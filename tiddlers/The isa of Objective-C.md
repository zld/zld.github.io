## 构成
```cpp
typedef unsigned long uintptr_t; // 8 bytes (64 bits) in arm64

union isa_t {
    isa_t() { }
    isa_t(uintptr_t value) : bits(value) { }

    uintptr_t bits;

private:
    Class cls;

public:
#if defined(ISA_BITFIELD)
    struct {
        ISA_BITFIELD;  // defined in isa.h
    };
	// ...
};

define ISA_BITFIELD
	uintptr_t nonpointer        : 1;
	uintptr_t has_assoc         : 1; //
	uintptr_t has_cxx_dtor      : 1;
	uintptr_t shiftcls          : 33; /*MACH_VM_MAX_ADDRESS 0x1000000000*/
	uintptr_t magic             : 6; //
	uintptr_t weakly_referenced : 1; //
	uintptr_t unused            : 1;
	uintptr_t has_sidetable_rc  : 1;
	uintptr_t extra_rc          : 19
```

| 字段名称                  | 位宽（bits） | 位位置（从低位起） | 代表含义                                                                 | 用途说明 |
|---------------------------|--------------|---------------------|--------------------------------------------------------------------------|----------|
| nonpointer                | 1           | 0                  | 是否为 non-pointer isa（1 表示是，0 表示传统纯指针）                    | 标识 isa 是否打包了额外元数据。如果为 0，则整个 isa 是普通 Class 指针。 |
| has_assoc                 | 1           | 1                  | 对象是否有 associated objects（关联对象）                               | 用于快速判断是否需要清理 associated objects（dealloc 时优化）。 |
| has_cxx_dtor              | 1           | 2                  | 对象是否有 C++ 析构函数（.cxx_destruct 方法）                           | 快速判断 dealloc 时是否需要调用 C++ 析构逻辑。 |
| shiftcls                  | 33          | 3–35               | 类指针（class pointer，经过移位和掩码处理）                             | 存储实际 Class 地址（由于指针对齐，压缩存储）。runtime 通过掩码提取真实类。 |
| magic                     | 6           | 36–41              | 魔法值（固定为 0x3f，用于调试和验证 isa 是否被正确初始化）               | runtime 用来自检 isa 是否有效（dealloc 或 retain 时检查，防止崩溃）。 |
| weakly_referenced         | 1           | 42                 | 对象是否有 weak 引用                                                    | 快速判断 dealloc 时是否需要清理 weak 表。 |
| deallocating              | 1           | 43                 | 对象是否正在 deallocating                                               | 标记对象正在释放，防止重复释放或循环引用问题。 |
| has_sidetable_rc          | 1           | 44                 | 引用计数是否溢出到 side table（额外存储）                               | 当引用计数超过 inline 存储范围时，使用 side table 存储大计数。 |
| extra_rc                  | 19          | 45–63              | 额外引用计数（实际 retain count - 1）                                    | 内联存储小引用计数（最多 19 位 +1），减少侧表访问，提升 retain/release 性能。 |

## init
nonpointer: 
* 压缩了状态
* 对 ISA_BITFIELD 进行初始化

!nonpointer（也就是pointer isa）: 
* 指向对象的 Class。
* 没有任何额外的状态位压缩。
* 所有额外信息（引用计数、关联对象等）都存储在 side table（外部表）里。

另外TaggedPointer也不是nonpointer，只不过不会走到`initIsa`

```cpp
inline void 
objc_object::initIsa(Class cls, bool nonpointer, UNUSED_WITHOUT_INDEXED_ISA_AND_DTOR_BIT bool hasCxxDtor)
{ 
    ASSERT(!isTaggedPointer()); 
    
    isa_t newisa(0);

    if (!nonpointer) {
        newisa.setClass(cls, this);
    } else {
        ASSERT(!DisableNonpointerIsa);
        ASSERT(!cls->instancesRequireRawIsa());


#if SUPPORT_INDEXED_ISA ----> Watch Only
        ASSERT(cls->classArrayIndex() > 0);
        newisa.bits = ISA_INDEX_MAGIC_VALUE;
        // isa.magic is part of ISA_MAGIC_VALUE
        // isa.nonpointer is part of ISA_MAGIC_VALUE
        newisa.has_cxx_dtor = hasCxxDtor;
        newisa.indexcls = (uintptr_t)cls->classArrayIndex();
#else
        newisa.bits = ISA_MAGIC_VALUE;
        // isa.magic is part of ISA_MAGIC_VALUE
        // isa.nonpointer is part of ISA_MAGIC_VALUE
#   if ISA_HAS_CXX_DTOR_BIT
        newisa.has_cxx_dtor = hasCxxDtor;
#   endif
        newisa.setClass(cls, this);
#endif
        newisa.extra_rc = 1;
    }

    // This write must be performed in a single store in some cases
    // (for example when realizing a class because other threads
    // may simultaneously try to use the class).
    // fixme use atomics here to guarantee single-store and to
    // guarantee memory order w.r.t. the class index table
    // ...but not too atomic because we don't want to hurt instantiation
    isa = newisa;
}
```