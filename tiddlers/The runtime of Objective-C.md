## 是什么
Objective-C是一个面向对象的[动态语言](https://en.wikipedia.org/wiki/Dynamic_programming_language)，早期实现是基于面向过程的C和汇编语言，后面由C换到了C++。所以我理解为早期为了实现“面向过程->面向对象”而实现了runtime。

> The Objective-C language is a simple computer language designed to enable sophisticated object-oriented programming. Objective-C is defined as a small but powerful set of extensions to the standard ANSI C language. Its additions to C are mostly based on *Smalltalk*, one of the first object-oriented programming languages. Objective-C is designed to give C full object-oriented programming capabilities, and to do so in a simple and straightforward way. - [The Objective-C Programming Language](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ObjectiveC/Introduction/introObjectiveC.html)

可以看出ObjC是基于Smalltalk来设计的，而Smalltalk就是[late binding](https://en.wikipedia.org/wiki/Late_binding)的，其runtime比ObjC更纯粹。

很多语言，甚至说大部分主流语言都有runtime
| 语言          | 是否有 runtime | runtime 强度 | 备注             |
| ----------- | ----------- | ---------- | -------------- |
| C           | ❌           | 0          | 几乎纯编译          |
| C++         | ✅           | 1          | 静态 OOP         |
| Rust        | ✅           | 1–2        | 可选 runtime     |
| Objective-C | ✅           | 2          | Smalltalk 式消息  |
| Swift       | ✅           | 2          | 静态优先           |
| Go          | ✅           | 2–3        | GC + scheduler |
| Java        | ✅           | 3          | JVM            |
| C#          | ✅           | 3          | CLR            |
| Python      | ✅           | 4          | 解释器即 runtime   |
| JavaScript  | ✅           | 4          | 引擎             |
| Smalltalk   | ✅           | 4          | image 世界       |

只不过有的runtime是对外暴露的，可以用来做些奇技淫巧的事情，而有些只是作为语言实现封闭在内部。ObjC是几乎唯一一个“强类型、编译语言，却开放给用户使用完整runtime的主流语言”。

## 功能实现

### 核心运行时功能

| 功能点 | 实现文件 | 说明 |
|-------|---------|------|
| 运行时初始化 | `objc-runtime.mm` | 负责运行时的启动和初始化 |
| 类加载与卸载 | `objc-load.mm` | 处理类的加载和卸载过程 |
| 方法解析 | `objc-runtime-new.mm` | 实现方法的动态解析 |
| 运行时环境配置 | `objc-env.h` | 管理运行时环境变量和配置 |
| 线程安全管理 | `Threading/` 目录下的文件 | 提供线程安全相关的功能 |

### 类和对象管理

| 功能点 | 实现文件 | 说明 |
|-------|---------|------|
| 类的创建与管理 | `objc-class.mm` | 实现类的创建、修改和管理 |
| 对象实例化 | `NSObject.mm` | 处理对象的分配和初始化 |
| 类结构定义 | `objc-runtime-new.h` | 定义类的内部数据结构 |
| 协议管理 | `Protocol.mm` | 实现协议的创建和管理 |
| 类别（Category）支持 | `objc-runtime.mm` | 处理类别的加载和附加 |
| 方法列表管理 | `objc-class.mm` | 管理类的方法列表 |

### 消息传递机制

| 功能点 | 实现文件 | 说明 |
|-------|---------|------|
| 消息发送 | `Messengers.subproj/` 目录下的汇编文件 | 实现 `objc_msgSend` 等核心函数 |
| 消息转发 | `objc-runtime-new.mm` | 处理消息转发逻辑 |
| 方法查找 | `objc-runtime-new.mm` | 实现方法的快速查找 |
| 缓存管理 | `objc-cache.mm` | 管理方法缓存 |
| 选择器（Selector）管理 | `objc-sel.mm` | 处理选择器的注册和查找 |

### 内存管理

| 功能点 | 实现文件 | 说明 |
|-------|---------|------|
| 引用计数 | `NSObject.mm` | 实现 retain/release 等引用计数操作 |
| 弱引用支持 | `objc-weak.mm` | 实现弱引用表和自动置 nil 功能 |
| 自动释放池 | `NSObject.mm` | 实现自动释放池的管理 |
| 内存分配 | `objc-zalloc.mm` | 管理对象内存的分配 |
| 垃圾回收支持 | `NSObject.mm` | 提供垃圾回收相关功能 |

### 其他辅助功能

| 功能点 | 实现文件 | 说明 |
|-------|---------|------|
| 类型编码 | `objc-typeencoding.mm` | 处理类型编码和解析 |
| 关联对象 | `objc-references.mm` | 实现对象关联功能 |
| 同步机制 | `objc-sync.mm` | 提供 `@synchronized` 实现 |
| 异常处理 | `objc-exception.mm` | 处理异常的抛出和捕获 |
| 运行时调试 | `objc-test-env.c` | 提供运行时调试功能 |
| 指针认证 | `objc-ptrauth.h` | 实现指针认证相关功能 |


## 如何实现
* [](<#NSObject>)
* [](<#The Category of Objective-C>)
* [](<#NSProtocol>)
* [](<#objc_msgSend>)
* ARC
  * [](<#The weak of Objective-C>)
* [](<#autorelease>)
* autoreleasepool

## 怎么用