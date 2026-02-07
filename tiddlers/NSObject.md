## NSObject的构成
`NSObject`就是个isa指针
```cpp
@interface NSObject <NSObject> {
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Wobjc-interface-ivars"
    Class isa  OBJC_ISA_AVAILABILITY;
#pragma clang diagnostic pop
}
```
`Class`也是`objc_object`
```cpp
typedef struct objc_class *Class;

struct objc_class : objc_object {/*...*/}

struct objc_object {
private:
    isa_t isa; // 以前的类型是Class，后来变更为isa_t
public:
//...
}
```
## isa的构成
[](<#The isa of Objective-C>)

---
Referenced by

<ul class="backlinks">
<$list filter="[backlinks[]!is[system]!title<currentTiddler>]">
  <li>
    <$link to={{!!title}}>
      <$text text={{!!title}} />
    </$link>
  </li>
</$list>
</ul>

