
## 1. Weak 引用核心数据结构

```mermaid
classDiagram
    class SideTable {
        +spinlock_t slock
        +RefcountMap refcnts
        +weak_table_t weak_table
        +lock() void
        +unlock() void
        +forceResetLock() void
    }
    
    class weak_table_t {
        +weak_entry_t* weak_entries
        +size_t num_entries
        +size_t mask
        +size_t max_hash_displacement
        +weak_entry_t& weak_entry_for_referent(object)
        +void remove_weak_reference(object, referrer)
        +void append_weak_reference(object, referrer)
    }
    
    class weak_entry_t {
        +DisguisedPtr~objc_object~ referent
        +weak_referrer_t* referrers
        +weak_referrer_t inline_referrers[4]
        +size_t num_refs
        +size_t max_hash_displacement
        +bool out_of_line() bool
        +void add_referrer(referrer)
        +void remove_referrer(referrer)
    }
    
    class DisguisedPtr~T~ {
        +uintptr_t value
        +T get() T
        +void set(T) void
    }
    
    class weak_referrer_t {
        <<typedef>>
        DisguisedPtr~objc_object~*
    }
    
    SideTable "1" --> "1" weak_table_t : contains
    weak_table_t "1" --> "*" weak_entry_t : entries
    weak_entry_t "1" --> "1" DisguisedPtr~objc_object~ : referent
    weak_entry_t --> "*" weak_referrer_t : referrers
    weak_entry_t --> "4" weak_referrer_t : inline_referrers
```

## 2. Weak 引用完整生命周期流程

```mermaid
flowchart TD
    A[weak变量声明] --> B[编译器转换]
    B --> C{对象是否存在?}
    
    C -->|是| D[runtime调用objc_initWeak]
    C -->|否| E[指针设为nil]
    
    D --> F[在weak_table中查找或创建entry]
    F --> G[将weak指针地址添加到referrers]
    G --> H[返回指向对象的指针]
    
    subgraph "对象释放过程"
        I[dealloc调用] --> J[runtime调用clearDeallocating]
        J --> K[查找对象的SideTable]
        K --> L[获取weak_table中对应的entry]
        L --> M[遍历所有weak referrers]
        M --> N[将所有weak指针置为nil]
        N --> O[从weak_table中移除entry]
        O --> P[清理SideTable引用计数]
    end
    
    H --> Q[weak变量使用]
    Q --> R{对象是否被释放?}
    R -->|否| Q
    R -->|是| S[自动变为nil]
    S --> T[安全访问]
```

## 3. Weak 表查找和添加详细流程

```mermaid
flowchart TD
    A[objc_initWeak被调用] --> B[获取对象地址和weak指针地址]
    B --> C[调用storeWeak]
    
    subgraph storeWeak函数
        C --> D{操作类型判断}
        D -->|初始化| E[OLD = nil]
        D -->|重新赋值| F[OLD = 原对象]
        D -->|置nil| G[OLD = 对象]
        
        E --> H
        F --> H[调用weak_register_no_lock]
        G --> I[调用weak_unregister_no_lock]
    end
    
    subgraph weak_register_no_lock
        H --> J[查找对象的SideTable]
        J --> K[在weak_table中查找referent]
        K --> L{找到entry?}
        
        L -->|否| M[创建新的weak_entry_t]
        M --> N[插入weak_table]
        L -->|是| O[使用现有entry]
        
        N --> P[添加referrer到entry]
        O --> P
    end
    
    subgraph weak_unregister_no_lock
        I --> Q[查找SideTable和entry]
        Q --> R{找到entry?}
        R -->|是| S[从entry中移除referrer]
        S --> T{entry为空?}
        T -->|是| U[从weak_table移除entry]
        R -->|否| V[直接返回]
    end
    
    P --> W[更新OLD对象的引用]
    U --> W
    W --> X[返回新对象]
```

## 4. 对象释放时 weak 清理流程

```mermaid
sequenceDiagram
    participant App as 应用程序
    participant Runtime as Runtime
    participant ST as SideTable
    participant WT as Weak Table
    participant Entry as Weak Entry
    
    App->>Runtime: 对象引用计数为0
    Runtime->>Runtime: 调用dealloc
    Runtime->>ST: clearDeallocating()
    
    ST->>ST: lock()
    ST->>WT: 查找对象对应的weak_entry_t
    WT->>Entry: 获取entry
    
    loop 遍历所有referrers
        Entry->>Entry: 获取weak指针地址
        Entry->>Runtime: 将指针指向nil
        Runtime->>App: weak变量自动置nil
    end
    
    Entry->>WT: 从weak_table中移除entry
    WT->>ST: 释放entry内存
    ST->>ST: unlock()
    ST->>Runtime: 清理完成
    Runtime->>App: 对象完全释放
```

## 5. Weak 实现的关键函数和机制

```mermaid
mindmap
  root((Weak实现))
    
    核心数据结构
      SideTable
        : 自旋锁slock
        : 引用计数表refcnts
        : weak_table_t
      
      Weak Table
        : 哈希表结构
        : weak_entry_t数组
        : 自动扩容机制
      
      Weak Entry
        : 存储被引用对象
        : 存储weak指针地址数组
        : 内联优化(4个以内)
    
    关键操作
      初始化
        : objc_initWeak
        : storeWeak
        : weak_register_no_lock
      
      读取
        : objc_loadWeakRetained
        : 读取时retain/autorelease
      
      清理
        : clearDeallocating
        : weak_clear_no_lock
        : weak_unregister_no_lock
    
    优化策略
      内存优化
        : 内联referrers(4个以内)
        : 哈希表自动扩容
      
      性能优化
        : SideTable分离减少争用
        : 自旋锁保护
        : 惰性初始化
    
    线程安全
      : SideTable锁保护
      : 原子操作
      : 内存屏障
```

## 6. Weak 引用的内存管理细节

```mermaid
flowchart LR
    subgraph "应用程序内存空间"
        A[weak指针变量] --> B[指向对象]
        C[对象引用计数] --> D[strong引用计数]
        E[SideTable引用计数] --> F[weak引用计数]
    end
    
    subgraph "Runtime数据结构"
        G[SideTable池] --> H[SideTable1]
        G --> I[SideTable2]
        G --> J[SideTableN]
        
        H --> K[weak_table_t]
        K --> L[weak_entry_t数组]
        L --> M[entry1]
        L --> N[entry2]
        
        M --> O[referent: objc_object*]
        M --> P[referrers数组]
        P --> Q[weak指针地址1]
        P --> R[weak指针地址2]
    end
    
    B -.-> O
    A -.-> Q
```

## 7. Weak 引用操作的完整代码路径

```mermaid
flowchart TD
    A[__weak id obj = object] --> B[编译器生成objc_initWeak调用]
    
    subgraph "objc_initWeak"
        B --> C[storeWeak(&obj, object)]
    end
    
    subgraph "storeWeak核心逻辑"
        C --> D[获取对象新旧值]
        D --> E[获取SideTable锁]
        E --> F{新旧对象处理}
        
        F -->|新对象| G[weak_register_no_lock]
        F -->|旧对象| H[weak_unregister_no_lock]
        
        G --> I[在weak_table中添加引用]
        H --> J[从weak_table中移除引用]
        
        I --> K
        J --> K[更新弱引用计数]
    end
    
    K --> L[释放锁]
    L --> M[返回对象]
    
    subgraph "对象释放时"
        N[对象dealloc] --> O[runtime调用_objc_rootDealloc]
        O --> P[调用clearDeallocating]
        P --> Q[weak_clear_no_lock]
        Q --> R[遍历所有weak指针置nil]
        R --> S[清理weak_table]
    end
    
    subgraph "读取weak变量"
        T[id obj2 = obj] --> U[objc_loadWeakRetained]
        U --> V[增加引用计数]
        V --> W[autorelease对象]
        W --> X[返回对象]
    end
    
    M --> Y[weak变量可用]
    S --> Z[weak变量自动为nil]
    X --> AA[安全使用对象]
```

## 关键源码函数说明

根据 objc4 源码，weak 实现主要涉及以下关键函数：

1. **初始化 weak 引用**
   ```cpp
   id objc_initWeak(id *location, id newObj)
   void storeWeak(id *location, objc_object *newObj)
   ```

2. **weak 表操作**
   ```cpp
   void weak_register_no_lock(weak_table_t *weak_table, id referent_id, id *referrer_id)
   void weak_unregister_no_lock(weak_table_t *weak_table, id referent_id, id *referrer_id)
   ```

3. **对象释放清理**
   ```cpp
   void weak_clear_no_lock(weak_table_t *weak_table, id referent_id)
   void clearDeallocating()
   ```

4. **weak 变量读取**
   ```cpp
   id objc_loadWeakRetained(id *location)
   ```

## 总结

Objective-C 的 weak 引用实现是一个复杂的系统，主要特点包括：

1. **两级哈希表结构**：SideTable → weak_table_t → weak_entry_t
2. **线程安全**：通过自旋锁保护 SideTable
3. **内存优化**：使用内联数组存储少量 weak 指针
4. **自动清理**：对象释放时自动置 nil 所有 weak 引用
5. **延迟初始化**：SideTable 和 weak_table 按需创建

这种设计在保证线程安全的同时，提供了高效的 weak 引用管理，是 Objective-C 内存管理的重要组成部分。