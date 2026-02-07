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
ISA_BITFIELD的构成
```cpp
#     define ISA_MASK        0x0000000ffffffff8ULL
#     define ISA_MAGIC_MASK  0x000003f000000001ULL
#     define ISA_MAGIC_VALUE 0x000001a000000001ULL
#     define ISA_HAS_CXX_DTOR_BIT 1
#     define ISA_BITFIELD      \
        uintptr_t nonpointer        : 1;    \
        uintptr_t has_assoc         : 1;    \
        uintptr_t has_cxx_dtor      : 1;   \
        uintptr_t shiftcls          : 33; /*MACH_VM_MAX_ADDRESS 0x1000000000*/ \
        uintptr_t magic             : 6;   \
        uintptr_t weakly_referenced : 1;    \
        uintptr_t unused            : 1;    \
        uintptr_t has_sidetable_rc  : 1;   \
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
