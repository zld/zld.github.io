## 结构组成
```cpp
struct category_t {
    const char *name;
    classref_t cls;
    WrappedPtr<method_list_t, PtrauthStrip> instanceMethods;
    WrappedPtr<method_list_t, PtrauthStrip> classMethods;
    struct protocol_list_t *protocols;
    struct property_list_t *instanceProperties;
    // Fields below this point are not always present on disk.
    struct property_list_t *_classProperties;

    method_list_t *methodsForMeta(bool isMeta) {
        if (isMeta) return classMethods;
        else return instanceMethods;
    }

    property_list_t *propertiesForMeta(bool isMeta, struct header_info *hi);
    
    protocol_list_t *protocolsForMeta(bool isMeta) {
        if (isMeta) return nullptr;
        else return protocols;
    }
};
```
## 加载流程

```cpp
_objc_init 
->     dyld_register_image_state_change_handler(dyld_image_state_bound,
                                             1/*batch*/, &map_2_images);
-> _read_images
-> remethodizeClass(Class cls)
-> attachCategories(Class cls, category_list *cats, bool flush_caches)
-> class_rw_t
```
1. 在运行时，读取Mach-O可执行文件，加载其中的image资源时(也就是map_2_images方法中的_read_images)，去读取编译时所存储到__objc_catlist的section段中的数据结构，并存储到category_t类型的一个临时变量中。
2. 遍历这个临时变量数组，依次读取。
3. 将catlist和原先类cls进行映射。
4. 调用remethodizeClass修改method_list，property_list，protocol_list结构，将分类的内容添加到原本类中。


```cpp
/***********************************************************************
* _read_images
* Perform initial processing of the headers in the linked 
* list beginning with headerList. 
*
* Called by: map_images_nolock
*
* Locking: runtimeLock acquired by map_images
**********************************************************************/
void _read_images(header_info **hList, uint32_t hCount)
{
    *************中间省略跟category无关的代码*************
    
    // Discover categories. 
    for (EACH_HEADER) {
        category_t **catlist = 
            _getObjc2CategoryList(hi, &count);
        for (i = 0; i < count; i++) {
            category_t *cat = catlist[i];
            Class cls = remapClass(cat->cls);

            if (!cls) {
                
                // Category's target class is missing (probably weak-linked).
                // Disavow any knowledge of this category.
                catlist[i] = nil;
                if (PrintConnecting) {
                    _objc_inform("CLASS: IGNORING category ???(%s) %p with "
                                 "missing weak-linked target class", 
                                 cat->name, cat);
                }
                continue;
            }

            // Process this category. 
            // First, register the category with its target class. 
            // Then, rebuild the class's method lists (etc) if 
            // the class is realized. 
            bool classExists = NO;
            if (cat->instanceMethods ||  cat->protocols  
                ||  cat->instanceProperties) 
            {
                addUnattachedCategoryForClass(cat, cls, hi);
                if (cls->isRealized()) {
                    remethodizeClass(cls);
                    classExists = YES;
                }
                if (PrintConnecting) {
                    _objc_inform("CLASS: found category -%s(%s) %s", 
                                 cls->nameForLogging(), cat->name, 
                                 classExists ? "on existing class" : "");
                }
            }

            if (cat->classMethods  ||  cat->protocols  
                /* ||  cat->classProperties */) 
            {
                addUnattachedCategoryForClass(cat, cls->ISA(), hi);
                if (cls->ISA()->isRealized()) {
                    remethodizeClass(cls->ISA());
                }
                if (PrintConnecting) {
                    _objc_inform("CLASS: found category +%s(%s)", 
                                 cls->nameForLogging(), cat->name);
                }
            }
        }
    }

    ts.log("IMAGE TIMES: discover categories");
        
   *************中间省略跟category无关的代码*************  
}

```

```cpp
// A fixed-size array that fills from the back, producing a contiguous chunk of
// memory with the elements ordered in reverse order from how they were added.
// This is a small wrapper around a C array and a count, with a bit of logic for
// computing insertion points. Built for attachCategories, which wants to build
// up arrays of category lists in reverse order.
template <typename T, uint32_t size>
struct ReversedFixedSizeArray {
    T array[size];
    uint32_t count = 0;

    bool isFull() const {
        return count >= size;
    }

    void add(T value) {
        ASSERT(!isFull());
        array[size - ++count] = value;
    }

    void clear() {
        count = 0;
    }

    T *begin() {
        return array + size - count;
    }
};
```

```cpp
// Attach method lists and properties and protocols from categories to a class.
// Assumes the categories in cats are all loaded and sorted by load order,
// oldest categories first.
static void
attachCategories(Class cls, const locstamped_category_t *cats_list,
                 uint32_t cats_count, Class catsListKey, int flags)
{
    if (slowpath(PrintReplacedMethods)) {
        printReplacements(cls, cats_list, cats_count);
    }
    if (slowpath(PrintConnecting)) {
        _objc_inform("CLASS: attaching %d categories to%s class '%s'%s",
                     cats_count, (flags & ATTACH_EXISTING) ? " existing" : "",
                     cls->nameForLogging(), (flags & ATTACH_METACLASS) ? " (meta)" : "");
        for (uint32_t i = 0; i < cats_count; i++)
            _objc_inform("    category: (%s) %p", cats_list[i].getCategory(catsListKey)->name, cats_list[i].getCategory(catsListKey));
    }

    /*
     * Only a few classes have more than 64 categories during launch.
     * This uses a little stack, and avoids malloc.
     *
     * Categories must be added in the proper order, which is back
     * to front. To do that with the chunking, we iterate cats_list
     * from front to back, build up the local buffers backwards,
     * and call attachLists on the chunks. attachLists prepends the
     * lists, so the final result is in the expected order.
     */
    // Category 必须**逆序**挂载（后加载的放最前面），因为方法查找是从前往后找第一个匹配的。
    // 但传入的 cats_list 是正序（老→新），所以用 ReversedFixedSizeArray 来“反着填”，
    // 再通过 attachLists 的“前插”行为，最终实现：**最后加载的 Category 方法排在方法表最前面**。
    constexpr uint32_t ATTACH_BUFSIZ = 64;
    struct Lists {
        ReversedFixedSizeArray<method_list_t *, ATTACH_BUFSIZ> methods;
        ReversedFixedSizeArray<property_list_t *, ATTACH_BUFSIZ> properties;
        ReversedFixedSizeArray<protocol_list_t *, ATTACH_BUFSIZ> protocols;
    };
    Lists preattachedLists;
    Lists normalLists;

    bool fromBundle = NO;
    bool isMeta = (flags & ATTACH_METACLASS);
    auto rwe = cls->data()->extAllocIfNeeded();

    for (uint32_t i = 0; i < cats_count; i++) {
        auto& entry = cats_list[i];

        method_list_t *mlist = entry.getCategory(catsListKey)->methodsForMeta(isMeta);
        bool isPreattached = entry.hi->info()->dyldCategoriesOptimized() && !DisablePreattachedCategories;
        Lists *lists = isPreattached ? &preattachedLists : &normalLists;
        if (mlist) {
            if (lists->methods.isFull()) {
                prepareMethodLists(cls, lists->methods.array, lists->methods.count, NO, fromBundle, __func__);
                rwe->methods.attachLists(lists->methods.array, lists->methods.count, isPreattached, PrintPreopt ? "methods" : nullptr);
                lists->methods.clear();
            }
            // 关键：用 ReversedFixedSizeArray.add() → 实际是往“逻辑尾部”插入
            // 因为是反向数组，所以越晚遍历到的 Category（越晚加载），越被放在数组的“前面”
            lists->methods.add(mlist);
            fromBundle |= entry.hi->isBundle();
        }

        property_list_t *proplist =
            entry.getCategory(catsListKey)->propertiesForMeta(isMeta, entry.hi);
        if (proplist) {
            if (lists->properties.isFull()) {
                rwe->properties.attachLists(lists->properties.array, lists->properties.count, isPreattached, PrintPreopt ? "properties" : nullptr);
                lists->properties.clear();
            }
            lists->properties.add(proplist);
        }

        protocol_list_t *protolist = entry.getCategory(catsListKey)->protocolsForMeta(isMeta);
        if (protolist) {
            if (lists->protocols.isFull()) {
                rwe->protocols.attachLists(lists->protocols.array, lists->protocols.count, isPreattached, PrintPreopt ? "protocols" : nullptr);
                lists->protocols.clear();
            }
            lists->protocols.add(protolist);
        }
    }

    auto attach = [&](Lists *lists, bool isPreattached) {
        if (lists->methods.count > 0) {
            prepareMethodLists(cls, lists->methods.begin(), lists->methods.count,
                               NO, fromBundle, __func__);
            rwe->methods.attachLists(lists->methods.begin(), lists->methods.count, isPreattached, PrintPreopt ? "methods" : nullptr);
            if (flags & ATTACH_EXISTING) {
                flushCaches(cls, __func__, [](Class c){
                    // constant caches have been dealt with in prepareMethodLists
                    // if the class still is constant here, it's fine to keep
                    return !c->cache.isConstantOptimizedCache();
                });
            }
        }

        rwe->properties.attachLists(lists->properties.begin(), lists->properties.count, isPreattached, PrintPreopt ? "properties" : nullptr);

        rwe->protocols.attachLists(lists->protocols.begin(), lists->protocols.count, isPreattached, PrintPreopt ? "protocols" : nullptr);
    };
    attach(&preattachedLists, true);
    attach(&normalLists, false);
}

```
```cpp
void attachLists(List* const * addedLists,
                 uint32_t addedCount,
                 bool preoptimized,
                 const char *logKind) {
    // ...

    if (storage.isNull() && addedCount == 1) {
        // 0 lists -> 1 list
        storage.set(*addedLists);
        validate();
    } else if (storage.isNull() || storage.template is<List *>()) {
        List *oldList = storage.template dyn_cast<List *>();
        // 0 or 1 list -> many lists
        uint32_t oldCount = oldList ? 1 : 0;
        uint32_t newCount = oldCount + addedCount;
        array_t *array = (array_t *)malloc(array_t::byteSize(newCount));
        storage.set(array);
        array->count = newCount;
        // 将原有的方法列表添加到数组的后面
        if (oldList) array->lists[addedCount] = oldList;
        // 将新的方法列表（Category方法列表）添加到数组的前面
        for (unsigned i = 0; i < addedCount; i++)
            array->lists[i] = addedLists[i];
        validate();
    } else if (array_t *array = storage.template dyn_cast<array_t *>()) {
        // many lists -> many lists
        uint32_t oldCount = array->count;
        uint32_t newCount = oldCount + addedCount;
        array_t *newArray = (array_t *)malloc(array_t::byteSize(newCount));
        newArray->count = newCount;

        for (int i = oldCount - 1; i >= 0; i--)
            newArray->lists[i + addedCount] = array->lists[i];
        for (unsigned i = 0; i < addedCount; i++)
            newArray->lists[i] = addedLists[i];
        free(array);
        storage.set(newArray);
        validate();
    } else if (auto *listList = storage.template dyn_cast<relative_list_list_t<List> *>()) {
        // ...
}
```

## 编译顺序如何影响方法顺序

1. **编译顺序**：编译器会按照文件在项目中的顺序编译Category，后编译的Category会在`cats_list`的后面。

2. **存储顺序**：在`attachCategories`函数中，使用`ReversedFixedSizeArray`来存储Category方法列表。`ReversedFixedSizeArray`的`add`方法会将新元素添加到数组的末尾，但是通过索引计算实现了反向存储。因此，后编译的Category会被存储在数组的前面。

3. **插入顺序**：调用`attachLists`方法时，会将`ReversedFixedSizeArray`中的方法列表按照存储顺序添加到类的方法列表的前面。因此，后编译的Category的方法会被放在类的方法列表的更前面。

4. **查找顺序**：方法查找时从方法列表的前面开始查找，找到第一个匹配的方法就返回。由于后编译的Category的方法被放在了方法列表的前面，因此会先被找到，从而"胜出"。

## 示例说明

假设有三个Category，编译顺序为A、B、C：

1. 编译后，`cats_list`的顺序为[A, B, C]。
2. 在`attachCategories`函数中，遍历`cats_list`，将每个Category的方法列表添加到`ReversedFixedSizeArray`中：
   - 添加A：数组变为[A]
   - 添加B：数组变为[B, A]
   - 添加C：数组变为[C, B, A]
3. 调用`attachLists`方法，将`ReversedFixedSizeArray`中的方法列表添加到类的方法列表的前面。
4. 类的方法列表变为[C的方法, B的方法, A的方法, 原有的方法]。
5. 当查找方法时，会先查找C的方法，然后是B的方法，然后是A的方法，最后是原有的方法。因此，C的同名方法会"胜出"。
