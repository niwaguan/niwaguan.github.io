---
layout: post
title: 理解Objective-C类对象的初始化
description: 程序在运行期间会创建无数的实例对象，这些实例对象依赖的类（对象） 也是需要提前处理好的。该篇讨论了相关的内容。
category:
  - iOS
tags:
  - Runtime
---



## 起源 _objc_init

> 严格来说`_objc_init`并不是最开始的起点，`_dyld_start`才是。这里暂不讨论`dyld`的加载。

一切要从`_objc_init`方法说起。`_objc_init`是Objc的初始化入口，由系统调用。这里除了必要的初始化外，向`dyld`注册了相关的回调，类（对象）的处理就是这些回调中进行的。

```c++
//
// Note: only for use by objc runtime
// Register handlers to be called when objc images are mapped, unmapped, and initialized.
// Dyld will call back the "mapped" function with an array of images that contain an objc-image-info section.
// Those images that are dylibs will have the ref-counts automatically bumped, so objc will no longer need to
// call dlopen() on them to keep them from being unloaded.  During the call to _dyld_objc_notify_register(),
// dyld will call the "mapped" function with already loaded objc images.  During any later dlopen() call,
// dyld will also call the "mapped" function.  Dyld will call the "init" function when dyld would be called
// initializers in that image.  This is when objc calls any +load methods in that image.
//
void _dyld_objc_notify_register(_dyld_objc_notify_mapped    mapped,
                                _dyld_objc_notify_init      init,
                                _dyld_objc_notify_unmapped  unmapped);
```

这个函数接收3个函数指针，分别用于接收加载镜像、初始化镜像、卸载镜像的事件。下面是这3个流程的大致说明。
![objc-init-overview](media/objc-init-overview.png)

接下来，我们就以这三条主线来研究系统是处理如何类（对象）的。

## map_images

在`map_images`阶段，又可以分为两个步骤：

1. 准备数据：统计镜像、`SEL`、`Message`，初始化全局数据结构
2. 加载类数据：根据统计的数量加载类、分类、协议以及实现`non-lazy class`

其中步骤2是核心。但在正式开始之前，我们需要先了解下几个相关内容：`future classes`、`remapped classes`、`gdb_objc_realized_classes`和`nonmeta classes`。这几个概念并没有详细的说明，但分析`map_images`又不可避免。所以下面采用顺藤摸瓜的方式探索，一起瞅瞅吧。

### future class

该类型的`class`存储在`NXMapTable`类型的`future_named_class_map`全局变量中。根据注释，`NXMapTable`是一种类似字典的结构，其中`key-value`必须是指针或整形。下面是`future_named_class_map`变量的调用路径图：

![objc-init-class-types-futureclass](media/objc-init-class-types-futureclass.png)

#### 初始化及基础支撑
`futureNamedClasses()`、`haveFutureNamedClasses()`、`popFutureNamedClass()`三个函数是直接访问`future_named_class_map`的，分别负责：初始化、判断是否存在`future class`、弹出一个`future class`。

#### 数据的添加
`objc_getFutureClass()`调用`_objc_allocateFutureClass()`是向`future_named_class_map`中插入数据的唯一入口，但是`objc_getFutureClass`却**没有被调用**。`_objc_allocateFutureClass()`的实现也比较简单：

```c++
// Allocate an unresolved future class for the given class name.
// Returns any existing allocation if one was already made.
// 通过name添加一个未决议的future class。如果已经存在了，就直接返回
Class _objc_allocateFutureClass(const char *name) {
    Class cls;
    NXMapTable *map = futureNamedClasses();
    // 尝试根据name读取class，如果有直接返回
    if ((cls = (Class)NXMapGet(map, name))) {
        // Already have a future class for this name.
        return cls;
    }
    // 没有的话，分配空间，添加到future_named_class_map中
    cls = _calloc_class(sizeof(objc_class));
    addFutureNamedClass(name, cls);
    return cls;
}
```

而`addFutureNamedClass()`负责分配`rw`、`ro`内存并关联到类数据中。需要注意的是，此时的数据中除了`rw`的`flag`加上了`future`的标识，其他均没有值。

```c++
// 插入一个future class
static void addFutureNamedClass(const char *name, Class cls) {
    void *old;
    // 为类的rw、ro数据段分配内存
    class_rw_t *rw = (class_rw_t *)calloc(sizeof(class_rw_t), 1);
    class_ro_t *ro = (class_ro_t *)calloc(sizeof(class_ro_t), 1);
    // 填充类的name、rw、ro数据
    ro->name = strdupIfMutable(name);
    rw->ro = ro;
    cls->setData(rw);
    // 将该类标记为future class
    cls->data()->flags = RO_FUTURE;
    // 插入容器中
    old = NXMapKeyCopyingInsert(futureNamedClasses(), name, cls);
}
```
#### 数据的消费
`future class`的唯一消费者是：`readClass()`。该方法负责读取编译器写入的类，在读取过程中会根据类名读取`future class`（如果有的话），并添加到`remapped_class_map`或`gdb_objc_realized_classes`或`allocatedClasses`记录中。

```c++
Class readClass(Class cls, bool headerIsBundle, bool headerIsPreoptimized) {
    // 获取类名
    const char *mangledName = cls->mangledName();
    // 丢失weak-linked superclass时，进行重映射
    if (missingWeakSuperclass(cls)) {
        addRemappedClass(cls, nil);
        cls->superclass = nil;
        return nil;
    }
    
    cls->fixupBackwardDeployingStableSwift();

    Class replacing = nil;
    // 查看是否有对应的future class
    if (Class newCls = popFutureNamedClass(mangledName)) {
        // This name was previously allocated as a future class.
        // Copy objc_class to future class's struct.
        // Preserve future's rw data block.
        // 记录future class的rw、ro数据
        class_rw_t *rw = newCls->data();
        const class_ro_t *old_ro = rw->ro;
        // 将传入的类数据复制到future class中
        memcpy(newCls, cls, sizeof(objc_class));
        // 将传入类的rw数据（实际此时存储的是ro数据）记录到future class的ro中
        rw->ro = (class_ro_t *)newCls->data();
        newCls->setData(rw);
        freeIfMutable((char *)old_ro->name);
        free((void *)old_ro);
        // 进行重映射
        addRemappedClass(cls, newCls);
        
        replacing = cls;
        cls = newCls;
    }
    
    if (headerIsPreoptimized  &&  !replacing) {
        // class list built in shared cache
        // fixme strict assert doesn't work because of duplicates
        // ASSERT(cls == getClass(name));
        ASSERT(getClassExceptSomeSwift(mangledName));
    } else {
        // 向不同的记录中添加class
        addNamedClass(cls, mangledName, replacing);
        addClassTableEntry(cls);
    }

    // for future reference: shared cache never contains MH_BUNDLEs
    if (headerIsBundle) {
        cls->data()->flags |= RO_FROM_BUNDLE;
        cls->ISA()->data()->flags |= RO_FROM_BUNDLE;
    }
    
    return cls;
}
```

需要注意的是`readClass()`并不是每次都会调用，它需要在`mustReadClasses()`返回`true`的时候才会调用。而这一流程正是在后续的`_read_images()`中。

至于`readClass()`的其他两个调用者：`_objc_realizeClassFromSwift()`和`objc_readClassPair()`也没有调用者，无法看出其是如何使用的。

至此，`future class`任然是个迷，只是知道系统会在某一个合适的时机生成、然后通过readClass读取其信息。

### remapped class

该类型的`class`存储在`objc::LazyInitDenseMap<Class, Class>`类型的`remapped_class_map`静态变量中。根据说明，这里主要存储两种情况下进行重映射的类：

* 现有类-已经实现的`future class`键值对
* 现有类-nil键值对，该类属于`weak-linked`类型

下面参考调用路径，继续分析：

![objc-init-class-types-remappedclass](media/objc-init-class-types-remappedclass.png)

#### 初始化

`remappedClasses()`函数是负责分配存储该类型`class`内存的，这是一个局部静态变量。

#### 数据的添加
`addRemappedClass()`负责向`remapped_class_map`中插入一个`{oldcls, newcls}`键值对。

#### 数据的消费
既然存在这样的重映射，那么对于给定一个`class`，可能不是最终使用的`class`，所以就需要一个方法去确定这个情况。`remapClass()`就是干这个的：

```c++
/// 判断一个类是否存在重映射。
/// 如果存在，返回映射的类；不存在返回入参
static Class remapClass(Class cls) {
    if (!cls) return nil;

    auto *map = remappedClasses(NO);
    if (!map)
        return cls;
    // 若没有在remapped_class_map中查找到，直接返回传入的类
    auto iterator = map->find(cls);
    if (iterator == map->end())
        return cls;
    return std::get<1>(*iterator);
}
```

按照上面的线索，可以知道`remapped_class_map`存储了那些原始类和使用类不统一的类。

### gdb_objc_realized_classes & nonmeta_class_map

`gdb_objc_realized_classes`表存储了不在`dyld`共享缓存中的类，该类可能已经被实现或者没有。

`nonmeta_class_map`表是`gdb_objc_realized_classes`表的补充，只存储了一种类型的类：在主表`gdb_objc_realized_classes`存在，但不是因为`同名`因素而存在的。
> 这里不是很好理解，原注释是： It only contains metaclasses whose classes would be in the runtime-allocated named-class table, but are not because some other class with the same name is in that table.

这两张表也都是`NXMapTable`类型。不同的是，前者存储`name-class`键值对，后者存储了`metaclass-class`键值对。

下面参考其对应的调用图继续分析：
![objc-init-class-types-gdb_objc_realized_classes](media/objc-init-class-types-gdb_objc_realized_classes.png)
![objc-init-class-types-nonmetaclass](media/objc-init-class-types-nonmetaclass-1.png)


#### 初始化
`gdb_objc_realized_classes`的初始化时在`_read_images()`函数中进行的。初始化时分配的内存为之前统计到类数量的`4/3`倍。而`nonmeta_class_map`的初始化是通过懒加载的方式进行，由其`getter`方法`nonMetaClasses()`负责。

#### 数据的添加
`addNamedClass()`负责向`gdb_objc_realized_classes`插入一个`name-class`键值对，或向`nonmeta_class_map`中插入`metaclass-class`键值对:

```c++
static void addNamedClass(Class cls, const char *name, Class replacing = nil) {
    Class old;
    if ((old = getClassExceptSomeSwift(name))  &&  old != replacing) {
        // getMaybeUnrealizedNonMetaClass uses name lookups.
        // Classes not found by name lookup must be in the
        // secondary meta->nonmeta table.
        addNonMetaClass(cls);
    } else {
        NXMapInsert(gdb_objc_realized_classes, name, cls);
    }
    ASSERT(!(cls->data()->flags & RO_META));
}
```

可以看到，在使用名称从`getClassExceptSomeSwift()`函数查询到`class`，并且和入参`replacing`不相等的时候，会插入到`nonmeta_class_map`中。再看看`getClassExceptSomeSwift()`函数的实现：

```c++
static Class getClassExceptSomeSwift(const char *name) {
    // 尝试从入参name直接查找类，若查找到，直接返回
    Class result = getClass_impl(name);
    if (result) return result;

    // 对名称进行转换，转换到swift类名，再次查找
    if (char *swName = copySwiftV1MangledName(name)) {
        result = getClass_impl(swName);
        free(swName);
        return result;
    }
    return nil;
}
```
这里的实现也比较简单，就是通过名称，调用`getClass_impl()`函数进行查找；若是`Swift`类名，还会多一次查找。所以`getClass_impl()`才是核心：

```c++
static Class getClass_impl(const char *name) {
    // 先从gdb_objc_realized_classes表中查找
    Class result = (Class)NXMapGet(gdb_objc_realized_classes, name);
    if (result) return result;
    
    // 没找到，尝试从dyld共享缓存中查找
    // Try table from dyld shared cache.
    // Note we do this last to handle the case where we dlopen'ed a shared cache
    // dylib with duplicates of classes already present in the main executable.
    // In that case, we put the class from the main executable in
    // gdb_objc_realized_classes and want to check that before considering any
    // newly loaded shared cache binaries.
    return getPreoptimizedClass(name);
}
```

通过上面的分析，我们知道在`addNamedClass()`中，一个类具体会插入到哪个表中有几种情况：

1. `getClassExceptSomeSwift()`未查找到 - 记录插入主表
2. `getClassExceptSomeSwift()`查找到, 结果和入参`replacing`不相等 - 记录插入的附表
3. `getClassExceptSomeSwift()`查找到, 结果和入参`replacing`相等 - 记录插入主表
在整个类初始化过程中，使用`getClassExceptSomeSwift()`函数查找类时，是查不到的，所以程序中多数的类会被插入到`gdb_objc_realized_classes`表中。

最后，对应`addNamedClass()`的入参`replacing`，我排查了对应的几个调用方，它只有2种可能：
1. `nil` 默认值，在`_objc_realizeClassFromSwift()`和`objc_duplicateClass()`以及`objc_registerClassPair`过程中也传入的是`nil`
2. `future class` 在`readClass()`过程中

所以，在`replacing`有值时，它一定是`future class`。

#### 数据的消费

类对象被添加到主表和附表中后，会在程序后续的运行中进行访问，具体细节这里先不分析。最后，在程序镜像卸载时，通过`removeNamedClass()`进行移除：

```c++
static void removeNamedClass(Class cls, const char *name) {
    ASSERT(!(cls->data()->flags & RO_META));
    // 先从gdb_objc_realized_classes中查找，若有直接移除
    if (cls == NXMapGet(gdb_objc_realized_classes, name)) {
        NXMapRemove(gdb_objc_realized_classes, name);
    } else {
        // 从nonmeta classes中移除
        removeNonMetaClass(cls);
    }
}
```

综上，我们知道了，在程序中使用的类对象大多添加到了`gdb_objc_realized_classes`，剩下一部分会添加到`nonmeta_class_map`中。

### _read_images

`_read_images`函数主要是读取镜像中的信息，包括`@selector`引用、类、分类、协议。依据前面讲到的内容，这里主要分析类相关的内容。

#### Discover classes

这里主要扫描镜像中的类列表，并通过`readClass()`函数将类添加到记录表中，并返回。返回值可能是`nil`、`类本身`或`future class`。若返回了`future class`，这里还会记录，后续将这种类型的类通过`realizeClassWithoutSwift()`函数实现。

#### Fix up remapped classes

在**Discover classes**阶段，调用`readClass()`可能会读取到需要进行重映射的类。这个阶段会将镜像中的类引用，修复到映射之后的值上。

```c++
static void remapClassRef(Class *clsref) {
    // 从remapped_class_map表中查找映射之后的类
    Class newcls = remapClass(*clsref);
    // 若读取到的类和入参不相同，就将其修复
    if (*clsref != newcls) *clsref = newcls;
}
```

#### Discover categories

这里主要扫描镜像中的分类列表，然后根据类是否`Realized`采取不同的动作。
* 对于`Realized`的类，将分类中的实例方法、实例属性添加到类对象上；将类方法、类属性添加到元类对象上（OC居然可以添加类属性，表示没用过）。这个过程主要的功臣是`attachCategories()`函数。
* 对于暂未`Realized`的类，只是简单的添加到`unattachedCategories`，后续通过`methodizeClass`处理。

`attachCategories()`：
```c++
/// @param cls 主类
/// @param cats_list 分类及其header信息
/// @param cats_count cats_list长度
/// @param flags 其他标志
static void attachCategories(Class cls, const locstamped_category_t *cats_list, uint32_t cats_count,
                 int flags)
{
    // 缓存大小64，原因是很少有类存在64个分类。减少插入的频率
    constexpr uint32_t ATTACH_BUFSIZ = 64;
    method_list_t   *mlists[ATTACH_BUFSIZ];
    property_list_t *proplists[ATTACH_BUFSIZ];
    protocol_list_t *protolists[ATTACH_BUFSIZ];

    uint32_t mcount = 0;
    uint32_t propcount = 0;
    uint32_t protocount = 0;
    bool fromBundle = NO;
    bool isMeta = (flags & ATTACH_METACLASS);
    auto rw = cls->data();

    for (uint32_t i = 0; i < cats_count; i++) {
        // 遍历传入的 分类 列表
        auto& entry = cats_list[i];
        // 取出实例方法列表或类方法列表
        method_list_t *mlist = entry.cat->methodsForMeta(isMeta);
        if (mlist) {
            // 若达到缓存上限，进行一次插入操作并清空计数器
            if (mcount == ATTACH_BUFSIZ) {
                prepareMethodLists(cls, mlists, mcount, NO, fromBundle);
                rw->methods.attachLists(mlists, mcount);
                mcount = 0;
            }
            // 向缓存中插入
            mlists[ATTACH_BUFSIZ - ++mcount] = mlist;
            fromBundle |= entry.hi->isBundle();
        }
        // 读取属性列表，记录在属性列表缓存中，并在达到缓存上限后进行插入
        property_list_t *proplist =
            entry.cat->propertiesForMeta(isMeta, entry.hi);
        if (proplist) {
            if (propcount == ATTACH_BUFSIZ) {
                rw->properties.attachLists(proplists, propcount);
                propcount = 0;
            }
            proplists[ATTACH_BUFSIZ - ++propcount] = proplist;
        }
        
        // 读取协议列表，记录在协议列表缓存中，并在达到缓存上限后进行插入
        protocol_list_t *protolist = entry.cat->protocolsForMeta(isMeta);
        if (protolist) {
            if (protocount == ATTACH_BUFSIZ) {
                rw->protocols.attachLists(protolists, protocount);
                protocount = 0;
            }
            protolists[ATTACH_BUFSIZ - ++protocount] = protolist;
        }
    }
    // 插入上面循环中记录的数据
    if (mcount > 0) {
        prepareMethodLists(cls, mlists + ATTACH_BUFSIZ - mcount, mcount, NO, fromBundle);
        rw->methods.attachLists(mlists + ATTACH_BUFSIZ - mcount, mcount);
        if (flags & ATTACH_EXISTING) flushCaches(cls);
    }
    rw->properties.attachLists(proplists + ATTACH_BUFSIZ - propcount, propcount);
    rw->protocols.attachLists(protolists + ATTACH_BUFSIZ - protocount, protocount);
}
```

需要注意的是，`attachCategories`方法是调用`attachLists`结构中的`attachLists`方法完成插入的。这个方法有个特点，在[之前的文章](https://waguan.cc/ios/2020/06/10/class-object/)中有提到，它会将之前的数据后移，将新的数据放置在前面。所以会导致最后加载的分类方法在列表最前面，存在同名方法按续查找会覆盖。

`methodizeClass`：

```c++
static void methodizeClass(Class cls, Class previously)
{
    bool isMeta = cls->isMetaClass();
    auto rw = cls->data();
    auto ro = rw->ro;

    // 将ro中的方法列表取出，排序，并添加到rw数据中
    // Install methods and properties that the class implements itself.
    method_list_t *list = ro->baseMethods();
    if (list) {
        prepareMethodLists(cls, &list, 1, YES, isBundleClass(cls));
        rw->methods.attachLists(&list, 1);
    }
    
    // 将ro中的属性列表添加到rw数据中
    property_list_t *proplist = ro->baseProperties;
    if (proplist) {
        rw->properties.attachLists(&proplist, 1);
    }
    
    // 将ro中的协议列表添加到rw数据中
    protocol_list_t *protolist = ro->baseProtocols;
    if (protolist) {
        rw->protocols.attachLists(&protolist, 1);
    }

    // Root classes get bonus method implementations if they don't have 
    // them already. These apply before category replacements.
    if (cls->isRootMetaclass()) {
        // root metaclass
        addMethod(cls, @selector(initialize), (IMP)&objc_noop_imp, "", NO);
    }

    // 处理分类
    if (previously) {
        if (isMeta) {
            objc::unattachedCategories.attachToClass(cls, previously,
                                                     ATTACH_METACLASS);
        } else {
            // When a class relocates, categories with class methods
            // may be registered on the class itself rather than on
            // the metaclass. Tell attachToClass to look for those.
            objc::unattachedCategories.attachToClass(cls, previously,
                                                     ATTACH_CLASS_AND_METACLASS);
        }
    }
    objc::unattachedCategories.attachToClass(cls, cls,
                                             isMeta ? ATTACH_METACLASS : ATTACH_CLASS);
}
```

#### Realize non-lazy class

`non-lazy`类型的类是指实现了`load`方法或者存在对应的静态变量。该阶段会扫描`non-lazy`类并将其实现，然后记录到`allocatedClasses`表中。这里的实现主要做了几件事情：

* 分配`RW`内存
* 分配`class`索引
* 实现当前类的父类及元类
* 确定是否使用原始`isa`指针
* 初始化`isa`
* 修复`ivar`偏移量
* 关联类与父类关系
* 关联分类

具体参考下面的代码：

```c++
static Class realizeClassWithoutSwift(Class cls, Class previously)
{
    const class_ro_t *ro;
    class_rw_t *rw;
    Class supercls;
    Class metacls;
    bool isMeta;

    if (!cls) return nil;
    if (cls->isRealized()) return cls;
    ASSERT(cls == remapClass(cls));

    // fixme verify class is not in an un-dlopened part of the shared cache?
    // 编译器确定的为ro数据
    ro = (const class_ro_t *)cls->data();
    if (ro->flags & RO_FUTURE) {
        // future class 在添加到其记录表中时就已经分配了rw内存
        rw = cls->data();
        ro = cls->data()->ro;
        cls->changeInfo(RW_REALIZED|RW_REALIZING, RW_FUTURE);
    } else {
        // 其他类需要分配rw内存
        rw = (class_rw_t *)calloc(sizeof(class_rw_t), 1);
        rw->ro = ro;
        rw->flags = RW_REALIZED|RW_REALIZING;
        cls->setData(rw);
    }

    isMeta = ro->flags & RO_META;
#if FAST_CACHE_META
    if (isMeta) cls->cache.setBit(FAST_CACHE_META);
#endif
    rw->version = isMeta ? 7 : 0;  // old runtime went up to 6


    // Choose an index for this class.
    // Sets cls->instancesRequireRawIsa if indexes no more indexes are available
    cls->chooseClassArrayIndex();
    
    // Realize superclass and metaclass, if they aren't already.
    // This needs to be done after RW_REALIZED is set above, for root classes.
    // This needs to be done after class index is chosen, for root metaclasses.
    // This assumes that none of those classes have Swift contents,
    //   or that Swift's initializers have already been called.
    //   fixme that assumption will be wrong if we add support
    //   for ObjC subclasses of Swift classes.
    supercls = realizeClassWithoutSwift(remapClass(cls->superclass), nil);
    metacls = realizeClassWithoutSwift(remapClass(cls->ISA()), nil);

#if SUPPORT_NONPOINTER_ISA
    // 根据各种情况处理是否需要原始isa指针
    if (isMeta) {
        // Metaclasses do not need any features from non pointer ISA
        // This allows for a faspath for classes in objc_retain/objc_release.
        cls->setInstancesRequireRawIsa();
    } else {
        // Disable non-pointer isa for some classes and/or platforms.
        // Set instancesRequireRawIsa.
        bool instancesRequireRawIsa = cls->instancesRequireRawIsa();
        bool rawIsaIsInherited = false;
        static bool hackedDispatch = false;

        if (DisableNonpointerIsa) {
            // Non-pointer isa disabled by environment or app SDK version
            instancesRequireRawIsa = true;
        }
        else if (!hackedDispatch  &&  0 == strcmp(ro->name, "OS_object"))
        {
            // hack for libdispatch et al - isa also acts as vtable pointer
            hackedDispatch = true;
            instancesRequireRawIsa = true;
        }
        else if (supercls  &&  supercls->superclass  &&
                 supercls->instancesRequireRawIsa())
        {
            // This is also propagated by addSubclass()
            // but nonpointer isa setup needs it earlier.
            // Special case: instancesRequireRawIsa does not propagate
            // from root class to root metaclass
            instancesRequireRawIsa = true;
            rawIsaIsInherited = true;
        }
        if (instancesRequireRawIsa) {
            cls->setInstancesRequireRawIsaRecursively(rawIsaIsInherited);
        }
    }
// SUPPORT_NONPOINTER_ISA
#endif

    // superclass和metaclass可能会被重映射
    cls->superclass = supercls;
    cls->initClassIsa(metacls);

    // 修复实例变量的偏移量
    if (supercls  &&  !isMeta) reconcileInstanceVariables(cls, supercls, ro);

    // Set fastInstanceSize if it wasn't set already.
    cls->setInstanceSize(ro->instanceSize);

    // Copy some flags from ro to rw
    if (ro->flags & RO_HAS_CXX_STRUCTORS) {
        cls->setHasCxxDtor();
        if (! (ro->flags & RO_HAS_CXX_DTOR_ONLY)) {
            cls->setHasCxxCtor();
        }
    }
    
    // Propagate the associated objects forbidden flag from ro or from
    // the superclass.
    if ((ro->flags & RO_FORBIDS_ASSOCIATED_OBJECTS) ||
        (supercls && supercls->forbidsAssociatedObjects())) {
        rw->flags |= RW_FORBIDS_ASSOCIATED_OBJECTS;
    }

    // Connect this class to its superclass's subclass lists
    if (supercls) {
        addSubclass(supercls, cls);
    } else {
        addRootClass(cls);
    }

    // 将该类的分类中的方法列表，属性列表，协议列表添加到rw中
    methodizeClass(cls, previously);
    return cls;
}
```


## load_images

该阶段主要处理类的load方法。思路比较清晰，先准备，然后调用。

### prepare_load_methods

准备load方法时，先处理类，再处理分类。在处理类的时候，会优先考虑类的父类。具体见代码：

```c++
void prepare_load_methods(const headerType *mhdr) {
    size_t count, i;
    // 加载所有的类
    classref_t const *classlist = 
        _getObjc2NonlazyClassList(mhdr, &count);
    for (i = 0; i < count; i++) {
        // 处理每一个类
        schedule_class_load(remapClass(classlist[i]));
    }

    // 加载分类
    category_t * const *categorylist = _getObjc2NonlazyCategoryList(mhdr, &count);
    for (i = 0; i < count; i++) {
        category_t *cat = categorylist[i];
        Class cls = remapClass(cat->cls);
        if (!cls) continue;  // category for ignored weak-linked class
        if (cls->isSwiftStable()) {
            _objc_fatal("Swift class extensions and categories on Swift "
                        "classes are not allowed to have +load methods");
        }
        // 防止类没有实现
        realizeClassWithoutSwift(cls, nil);
        ASSERT(cls->ISA()->isRealized());
        // 处理每一个分类
        add_category_to_loadable_list(cat);
    }
}

static void schedule_class_load(Class cls) {
    if (!cls) return;
    ASSERT(cls->isRealized());  // _read_images should realize
    // 已经load过的就直接返回。所以load方法只会调用一次
    if (cls->data()->flags & RW_LOADED) return;

    // Ensure superclass-first ordering
    // 优先处理父类，所以父类的load方法会先调用
    schedule_class_load(cls->superclass);

    add_class_to_loadable_list(cls);
    cls->setInfo(RW_LOADED); 
}

void add_class_to_loadable_list(Class cls) {
    IMP method;

    // 获取load方法的地址，若没有，直接返回
    method = cls->getLoadMethod();
    if (!method) return;  // Don't bother if cls has no +load method
    
    // 对容器扩容
    if (loadable_classes_used == loadable_classes_allocated) {
        loadable_classes_allocated = loadable_classes_allocated*2 + 16;
        loadable_classes = (struct loadable_class *)
            realloc(loadable_classes,
                              loadable_classes_allocated *
                              sizeof(struct loadable_class));
    }
    // 记录class和load方法地址
    loadable_classes[loadable_classes_used].cls = cls;
    loadable_classes[loadable_classes_used].method = method;
    loadable_classes_used++;
}


void add_category_to_loadable_list(Category cat)
{
    IMP method;

    loadMethodLock.assertLocked();
    // 获取分类中的load方法地址，若没有，直接返回
    method = _category_getLoadMethod(cat);
    if (!method) return;
    // 对容器扩容
    if (loadable_categories_used == loadable_categories_allocated) {
        loadable_categories_allocated = loadable_categories_allocated*2 + 16;
        loadable_categories = (struct loadable_category *)
            realloc(loadable_categories,
                              loadable_categories_allocated *
                              sizeof(struct loadable_category));
    }
    // 记录category和load方法地址
    loadable_categories[loadable_categories_used].cat = cat;
    loadable_categories[loadable_categories_used].method = method;
    loadable_categories_used++;
}
```

### call_load_methods

经过上一步的prepare，会得到`loadable_classes`和`loadable_categories`两个数组，分别存储类和分类的load方法信息。下面就是顺次遍历调用了：

```c++
void call_load_methods(void) {
    static bool loading = NO;
    bool more_categories;
    // Re-entrant calls do nothing; the outermost call will finish the job.
    if (loading) return;
    loading = YES;
    // 开启新的自动释放池
    void *pool = objc_autoreleasePoolPush();
    do {
        // 1. Repeatedly call class +loads until there aren't any more
        // 先调用类的load方法
        while (loadable_classes_used > 0) {
            call_class_loads();
        }
        // 2. Call category +loads ONCE
        // 再调用分类的load
        more_categories = call_category_loads();
        // 3. Run more +loads if there are classes OR more untried categories
    } while (loadable_classes_used > 0  ||  more_categories);
    // 关闭自动释放池
    objc_autoreleasePoolPop(pool);
    loading = NO;
}

static void call_class_loads(void) {
    int i;
    // Detach current loadable list.
    // 记录需要调用的loadable_classes，并将其置空
    struct loadable_class *classes = loadable_classes;
    int used = loadable_classes_used;
    loadable_classes = nil;
    loadable_classes_allocated = 0;
    loadable_classes_used = 0;
    
    // Call all +loads for the detached list.
    for (i = 0; i < used; i++) {
        Class cls = classes[i].cls;
        load_method_t load_method = (load_method_t)classes[i].method;
        if (!cls) continue; 

        // 根据函数地址，直接调用
        (*load_method)(cls, @selector(load));
    }
    
    // Destroy the detached list.
    if (classes) free(classes);
}

static bool call_category_loads(void) {
    int i, shift;
    bool new_categories_added = NO;
    
    // 记录需要调用的loadable_categories，并将其置空
    struct loadable_category *cats = loadable_categories;
    int used = loadable_categories_used;
    int allocated = loadable_categories_allocated;
    loadable_categories = nil;
    loadable_categories_allocated = 0;
    loadable_categories_used = 0;

    // Call all +loads for the detached list.
    for (i = 0; i < used; i++) {
        Category cat = cats[i].cat;
        load_method_t load_method = (load_method_t)cats[i].method;
        Class cls;
        // 可能该分类的原类没有被加载，直接返回。这样的会被保留，下次调用
        if (!cat) continue;

        cls = _category_getClass(cat);
        if (cls  &&  cls->isLoadable()) {
             // 根据函数地址，直接调用
            (*load_method)(cls, @selector(load));
            // 调用过的将category置空
            cats[i].cat = nil;
        }
    }

    // 压缩cats列表，去掉已经调用过load的条目
    shift = 0;
    for (i = 0; i < used; i++) {
        if (cats[i].cat) {
            cats[i-shift] = cats[i];
        } else {
            shift++;
        }
    }
    used -= shift;

    // Copy any new +load candidates from the new list to the detached list.
    new_categories_added = (loadable_categories_used > 0);
    for (i = 0; i < loadable_categories_used; i++) {
        // 容器扩容
        if (used == allocated) {
            allocated = allocated*2 + 16;
            cats = (struct loadable_category *)
                realloc(cats, allocated *
                                  sizeof(struct loadable_category));
        }
        // 将新列表中的添加到当前列表中
        cats[used++] = loadable_categories[i];
    }

    // 释放新的列表
    if (loadable_categories) free(loadable_categories);

    // 保存剩余的list，以便下次使用
    if (used) {
        loadable_categories = cats;
        loadable_categories_used = used;
        loadable_categories_allocated = allocated;
    } else {
        if (cats) free(cats);
        loadable_categories = nil;
        loadable_categories_used = 0;
        loadable_categories_allocated = 0;
    }
    return new_categories_added;
}
```

需要注意的是，在准备和调用时均是直接操作函数地址，并没有走OC的消息机制。

## unmap_image

这里负责卸载不在使用的镜像。是主要包含：

* 卸载镜像
* 移除Header信息

其中卸载镜像时，会先移除分类，再移除类及其元类并释放相关内存。具体细节参考`unmap_image_nolock`和`_unload_image`

`unmap_image_nolock`：
```c++
void unmap_image_nolock(const struct mach_header *mh) {

    header_info *hi;
    
    // Find the runtime's header_info struct for the image
    for (hi = FirstHeader; hi != NULL; hi = hi->getNext()) {
        if (hi->mhdr() == (const headerType *)mh) {
            break;
        }
    }

    if (!hi) return;
    
    // 卸载镜像
    _unload_image(hi);
    // 移除header
    removeHeader(hi);
    free(hi);
}
```

`_unload_image`：
```c++
void _unload_image(header_info *hi)
{
    size_t count, i;

    // Unload unattached categories and categories waiting for +load.

    // Ignore __objc_catlist2. We don't support unloading Swift
    // and we never will.
    // 移除分类
    category_t * const *catlist = _getObjc2CategoryList(hi, &count);
    for (i = 0; i < count; i++) {
        category_t *cat = catlist[i];
        Class cls = remapClass(cat->cls);
        if (!cls) continue;  // category for ignored weak-linked class

        // fixme for MH_DYLIB cat's class may have been unloaded already

        // unattached list
        objc::unattachedCategories.eraseCategoryForClass(cat, cls);

        // +load queue
        remove_category_from_loadable_list(cat);
    }

    // Unload classes.
    objc::DenseSet<Class> classes{};
    classref_t const *classlist;
    // 统计需要移除的类
    classlist = _getObjc2ClassList(hi, &count);
    for (i = 0; i < count; i++) {
        Class cls = remapClass(classlist[i]);
        if (cls) classes.insert(cls);
    }

    classlist = _getObjc2NonlazyClassList(hi, &count);
    for (i = 0; i < count; i++) {
        Class cls = remapClass(classlist[i]);
        if (cls) classes.insert(cls);
    }

    // First detach classes from each other. Then free each class.
    // This avoid bugs where this loop unloads a subclass before its superclass
    // 统一移除类
    for (Class cls: classes) {
        remove_class_from_loadable_list(cls);
        detach_class(cls->ISA(), YES);
        detach_class(cls, NO);
    }
    for (Class cls: classes) {
        free_class(cls->ISA());
        free_class(cls);
    }

}

```

## 总结

好了，到这里可以暂时舒缓下了，一起梳理下该篇讲的内容。我们以镜像的使用周期为主线，分别探索了镜像的读取（包含类、分类以及类的实现等），镜像的加载（load方法的扫描及调用），以及镜像的卸载。

这里还遗留了部分问题没有解决，比如：

* `methodizeClass`主要处理了`ro`中的方法以及分类中的信息，其中涉及到的方法排序具体是怎么样的？
* `non-lazy class`是在`_read_images`中进行了实现，那其他的类是在什么时候实现的？
* ...

希望大家带着自己的疑问，继续前行。共勉！