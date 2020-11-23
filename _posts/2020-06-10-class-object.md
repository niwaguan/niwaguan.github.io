---
layout: post
title: 理解Objective-C中的类对象
description: 类对象记录了类的元信息，主要包含了方法列表，协议列表，属性列表等。是OC对象系统的重要组成部分。该篇介绍了相关的数据结构。
category:
  - iOS
tags:
  - Runtime
---

在之前的文章中，我们提到了类对象，但是没有细说。今天一起来扒扒这里面有啥东西。

类对象，也就是我们平时所的`Class`，其在`Runtime`中的定义是结构体`struct objc_class`的指针：

```oc
/// Object.mm line 33
typedef struct objc_class *Class;
```

而`struct objc_class`继承至`struct objc_object`（c++中的结构体可以继承，并且可以定义方法），这就说明`类对象也是对象`。除了继承来的`isa`成员，`struct objc_class`还存储了父类，缓存信息以及类的具体信息：

```c++
struct objc_class : objc_object {
    // Class ISA;
    Class superclass;
    cache_t cache;             // formerly cache pointer and vtable
    class_data_bits_t bits;    // class_rw_t * plus custom rr/alloc flags

    class_rw_t *data() const {
        return bits.data();
    }
    ...
}
```

下面就一起来看看该结构体的具体成员。参考下图：
![class-obj-fields](https://images-for-blog.oss-cn-beijing.aliyuncs.com/2020/06/16/classobjfields.png)

## isa - 我是谁

`isa`是对象的心脏，它解决了对象是谁的问题。`isa`的真正面目是`union isa_t`：

```
union isa_t {
    isa_t() { }
    isa_t(uintptr_t value) : bits(value) { }

    Class cls;
    uintptr_t bits;
    struct {
    # if __arm64__
        #   define ISA_MASK        0x0000000ffffffff8ULL
        #   define ISA_MAGIC_MASK  0x000003f000000001ULL
        #   define ISA_MAGIC_VALUE 0x000001a000000001ULL
        uintptr_t nonpointer        : 1;
        uintptr_t has_assoc         : 1;
        uintptr_t has_cxx_dtor      : 1;
        uintptr_t shiftcls          : 33; /*MACH_VM_MAX_ADDRESS 0x1000000000*/
        uintptr_t magic             : 6;
        uintptr_t weakly_referenced : 1;
        uintptr_t deallocating      : 1;
        uintptr_t has_sidetable_rc  : 1;
        uintptr_t extra_rc          : 19
        #   define RC_ONE   (1ULL<<45)
        #   define RC_HALF  (1ULL<<18)

    # elif __x86_64__
        // ... x86_64架构下的定义，结构一致
    #endif
    };
};
```

也许你会疑惑，之前看到的`struct objc_object`中的成员`isa`是`Class`类型，为啥这里又出现一个联合体。下面说说我的理解：

首先，看看这两种类型所占用的内存大小是否一样。

`union isa_t`结构体有 3 个成员：

- `Class cls;` Class 本质上是指针，占 8 字节。
- `uintptr_t bits;` `uintptr_t` 本质是`unsigned long`的 typedef，占 8 字节。
- `匿名结构体` 这里使用了位域，加起来一共 64bit，也是 8 字节。

根据联合体 size 的大小计算规则，整个联合体的大小是 8 字节，和之前看到的 Class 类型的 isa 并无差别，它们占用的内存大小相同。

可以理解为对这 8 字节的内存解读方式的区别。如果按 Class 解读这 8 字节，那么这块内存就只能存储一个地址了；如果按这种怪异的方式解读，相比之下存储的信息就多了，具体见下图：
![](https://images-for-blog.oss-cn-beijing.aliyuncs.com/2020/06/16/isa64bit.png)

其次，为什么会设计出这个怪异的结构。

在上面的代码里，可以看到在`arm64`架构下，虚拟地址的最大值为`0x1000000000`，使用 33 位足够了。那么 8 字节（64 位）的空间只用 33 位是不是有点可惜，那么这种怪异的结构自然也就出来了。要知道在程序运行期间，可是有无限多个对象要创建的，在这里进行`变态级的优化`，效果是非常可观的。

## superclass - 我从哪里来

这个成员比较简单，记录了该类的继承结构，指向其父类对象。

## cache - 我干的最频繁的事

这里以散列表的方式存储方法缓存，提高消息发送时的查找效率。

```c++
struct cache_t {
    struct bucket_t *_buckets; // 列表
    mask_t _mask; // 掩码，列表的数量 - 1
    mask_t _occupied; // 已经占用的数量
    // ...
}
```

在这个散列表中的每一个元素`bucket_t`，存储了一个方法的 SEL，和 IMP。

```c++
typedef uintptr_t cache_key_t;

struct bucket_t {
    MethodCacheIMP _imp;
    cache_key_t _key;
}
```

既然是散列表，就会牵扯到索引的计算和冲突解决。先来看看索引的计算：

```c++
// 哈希函数，直接使用SEL和mask做&运算
static inline mask_t cache_hash(SEL sel, mask_t mask)
{
    return (mask_t)(uintptr_t)sel & mask;
}
```

这里有个注意点，`一个数和mask&运算的结果<=mask`。mask 也间接反映了`cache_t`中缓存方法的数量：

```c++
// 获取容量 mask + 1
mask_t cache_t::capacity()
{
    return mask() ? mask()+1 : 0;
}
```

冲突的解决，这里直接采用当前索引加一或减一的方式：

```c++
#if __arm__  ||  __x86_64__  ||  __i386__
// objc_msgSend has few registers available.
// Cache scan increments and wraps at special end-marking bucket.
#define CACHE_END_MARKER 1
static inline mask_t cache_next(mask_t i, mask_t mask) {
    return (i+1) & mask; // 加一再进行&运算
}

#elif __arm64__
// objc_msgSend has lots of registers available.
// Cache scan decrements. No end marker needed.
#define CACHE_END_MARKER 0
static inline mask_t cache_next(mask_t i, mask_t mask) {
    return i ? i-1 : mask; // 直接减一，减到零后从mask向后遍历
}
#else
#error unknown architecture
#endif
```

## bits - 我的方方面面

该字段存储了一个类相关的具体信息，比如方法列表、协议列表，属性等。

`bits`的类型为`class_data_bits_t`，它只是封装了`class_rw_t`（类的可读写数据）和自定义的标识以及操作内部数据的方法。这个结构体只有一个成员`bits`，占用 8 字节的内存。其中这个`class_rw_t* data()`方法可以获取类的可读写数据：

```
class_rw_t* data() const {
    return (class_rw_t *)(bits & FAST_DATA_MASK);
}
```

看来这个`bits`成员也存储了不止一种数据。有兴趣的可以可以查看`objc-runtime-new.h`文件中`FAST_`开头的宏定义。每一个宏和`bits`进行`&`运算都可以得到相应的信息。

### class_rw_t

该结构存储了一个类的可读写信息。其中只读信息又独立出来，使用`class_ro_t`结构表示，这个放到下节讲述。而我们平时书写的方法、属性以及协议也都存储在这里。

```oc
struct class_rw_t {
    // ...
    method_array_t methods;
    property_array_t properties;
    protocol_array_t protocols;
    // ...
}
```

这三者的结构类似，都继承至`list_array_tt`，而`list_array_tt`是一个 C++模板。一个为存储元数据的通用实现。其中，`Element`是元数据的类型，比如`method_t`；`List`是包含元数据的列表，比如`method_list_t`。`list_array_tt`类型的值有 3 种：

- 空
- 一个`List`指针
- 一个数组，元素是`List`指针

下面是详细的说明，建议认真查看。

```c++
template <typename Element, typename List>
class list_array_tt {
    // array_t begin
    // 实际的存储结构
    struct array_t {
        uint32_t count; // List的个数
        List* lists[0]; // 结构体中的伸缩型数组成员

        // 由于array_t中包含伸缩型数组成员，而系统默认不会为这样的成员分配内存
        // 需要通过malloc自己手动申请具体的内存大小，这个方法是用于快捷计算n个元素
        // 的array_t需要的内存大小
        static size_t byteSize(uint32_t count) {
            return sizeof(array_t) + count*sizeof(lists[0]);
        }
        // 使用成员count计算需要的内存大小
        size_t byteSize() {
            return byteSize(count);
        }
    };
    // array_t end

 protected:
    // iterator begin
    // list_array_tt的迭代器
    class iterator {
        List **lists;
        List **listsEnd;
        // 在模板中需要指定参数类型定义成员
        typename List::iterator m, mEnd;

     public:
        iterator(List **begin, List **end)
            : lists(begin), listsEnd(end)
        {
            if (begin != end) {
                // m, mEnd 存储了单个List的起始位置
                m = (*begin)->begin();
                mEnd = (*begin)->end();
            }
        }
        // 重写了 *、!=、 ++ 操作符
        const Element& operator * () const {
            return *m;
        }
        Element& operator * () {
            return *m;
        }

        bool operator != (const iterator& rhs) const {
            if (lists != rhs.lists) return true;
            if (lists == listsEnd) return false;  // m is undefined
            if (m != rhs.m) return true;
            return false;
        }

        const iterator& operator ++ () {
            ASSERT(m != mEnd);
            m++;
            // 一个List遍历完毕，且不是最后一个List时，将m，和mEnd指向下一个List
            if (m == mEnd) {
                ASSERT(lists != listsEnd);
                lists++;
                if (lists != listsEnd) {
                    m = (*lists)->begin();
                    mEnd = (*lists)->end();
                }
            }
            return *this;
        }
    };
    // iterator end

 private:
    // 嵌套的匿名联合体，可以在list_array_tt中直接访问其成员
    union {
        List* list;
        uintptr_t arrayAndFlag; // 实际数据的地址和标志
    };

    // 是否为多个List。多个时会将地址存储在arrayAndFlag中；单个会存储在list中
    bool hasArray() const {
        return arrayAndFlag & 1;
    }

    // arrayAndFlag的存取方法
    array_t *array() {
        return (array_t *)(arrayAndFlag & ~1);
    }
    void setArray(array_t *array) {
        arrayAndFlag = (uintptr_t)array | 1;
    }

 public:
    // 所有List中Element的总和
    uint32_t count() {
        uint32_t result = 0;
        for (auto lists = beginLists(), end = endLists();
             lists != end;
             ++lists)
        {
            result += (*lists)->count;
        }
        return result;
    }
    // 从开头到结尾的所有List的迭代器
    iterator begin() {
        return iterator(beginLists(), endLists());
    }
    // 最后一个List的迭代器
    iterator end() {
        List **e = endLists();
        return iterator(e, e);
    }
    // 统计List的个数
    uint32_t countLists() {
        if (hasArray()) {
            return array()->count;
        } else if (list) {
            return 1;
        } else {
            return 0;
        }
    }
    // 第一个List
    List** beginLists() {
        if (hasArray()) {
            return array()->lists;
        } else {
            return &list;
        }
    }
    // 最后一个List
    List** endLists() {
        if (hasArray()) {
            return array()->lists + array()->count;
        } else if (list) {
            return &list + 1;
        } else {
            return &list;
        }
    }
    // 新增addedCount个List
    void attachLists(List* const * addedLists, uint32_t addedCount) {
        if (addedCount == 0) return;
        if (hasArray()) { // 现有List多个，新增多个
            uint32_t oldCount = array()->count;
            uint32_t newCount = oldCount + addedCount;
            // 扩展内存空间
            setArray((array_t *)realloc(array(), array_t::byteSize(newCount)));
            array()->count = newCount;
            // 将之前的List向后移动
            memmove(array()->lists + addedCount, array()->lists,
                    oldCount * sizeof(array()->lists[0]));
            // 保存新的Lists
            memcpy(array()->lists, addedLists,
                   addedCount * sizeof(array()->lists[0]));
        }
        else if (!list  &&  addedCount == 1) {
            // 现在为空，并且新增一个List，直接将地址保存在list成员中
            list = addedLists[0];
        }
        else {
            // 当前有一个List或空，新增多个
            List* oldList = list;
            uint32_t oldCount = oldList ? 1 : 0;
            uint32_t newCount = oldCount + addedCount;
            // 分配内存并保存地址
            setArray((array_t *)malloc(array_t::byteSize(newCount)));
            array()->count = newCount;
            // 如果之前有一个List，将其放到最后的位置
            if (oldList) array()->lists[addedCount] = oldList;
            // 将添加的Lists拷贝到新内存中
            memcpy(array()->lists, addedLists,
                   addedCount * sizeof(array()->lists[0]));
        }
    }
    // 释放内存。先释放每一个List，再释放容器
    void tryFree() {
        if (hasArray()) {
            for (uint32_t i = 0; i < array()->count; i++) {
                try_free(array()->lists[i]);
            }
            try_free(array());
        }
        else if (list) {
            try_free(list);
        }
    }
    // 复制一份
    template<typename Result>
    Result duplicate() {
        Result result;

        if (hasArray()) {
            array_t *a = array();
            result.setArray((array_t *)memdup(a, a->byteSize()));
            for (uint32_t i = 0; i < a->count; i++) {
                result.array()->lists[i] = a->lists[i]->duplicate();
            }
        } else if (list) {
            result.list = list->duplicate();
        } else {
            result.list = nil;
        }

        return result;
    }
};
```

总结来说，`list_array_tt`，是一个容器，直接存储的类型需要为数组（typename List），然后数组里面的元素是元数据（typename Element），并提供相应的操作方法，如迭代器，添加新的`Lists`等。

一个`method_array_t`就是：
![method_array_t](https://images-for-blog.oss-cn-beijing.aliyuncs.com/2020/06/16/methodarrayt.png)

`properties`和`protocols`的`List`和`Element`分别是`property_list_t - property_t`、`protocol_list_t - protocol_ref_t`

再来看看`method_list_t`和`property_list_t`以及`protocol_list_t`。这三者是`list_array_tt`的直接元素。其中`method_list_t`和`property_list_t`都继承至`entsize_list_tt`。这是一个 C++模板，包含了两个类型参数一个非类型参数。这是一个具有`non-fragile`特性[^1]的数组通用实现。下面是具体的解读：

```c++
template <typename Element, typename List, uint32_t FlagMask>
struct entsize_list_tt {
    uint32_t entsizeAndFlags; // Element大小+标志位
    uint32_t count; // List的元素个数
    Element first;  // 第一个元素
    // 从entsizeAndFlags成员中取出Element大小
    uint32_t entsize() const {
        return entsizeAndFlags & ~FlagMask;
    }
    // 从entsizeAndFlags成员中取出标志
    uint32_t flags() const {
        return entsizeAndFlags & FlagMask;
    }
    // 随机读取支持
    Element& getOrEnd(uint32_t i) const {
        ASSERT(i <= count);
        return *(Element *)((uint8_t *)&first + i*entsize());
    }
    Element& get(uint32_t i) const {
        ASSERT(i < count);
        return getOrEnd(i);
    }
    // 快捷获取List的大小
    size_t byteSize() const {
        return byteSize(entsize(), count);
    }
    static size_t byteSize(uint32_t entsize, uint32_t count) {
        // count - 1 因为entsize_list_tt结构中会分配一个first成员
        return sizeof(entsize_list_tt) + (count-1)*entsize;
    }
    // 拷贝
    List *duplicate() const {
        auto *dup = (List *)calloc(this->byteSize(), 1);
        dup->entsizeAndFlags = this->entsizeAndFlags;
        dup->count = this->count;
        std::copy(begin(), end(), dup->begin());
        return dup;
    }

    // 迭代器,参考源码
    struct iterator;
}
```

`protocol_list_t`是一个单独实现的结构体。只包含了记录数量，内存地址成员，以及迭代方法等。

好了，容器解决了。再瞅瞅具体的元数据。

- method_t

  ```c++
  struct method_t {
      SEL name;
      const char *types;
      MethodListIMP imp;
  }
  ```

  `method_t`描述了一个方法的基本信息，如：名称、编码类型[^2]、具体实现。

- property_t

  ```c++
  struct property_t {
      const char *name;
      const char *attributes;
  };
  ```

  主要包含名称，以及对应的属性（类型，内存管理方式，对应实例变量名称等）。

- protocol_ref_t

  `protocol_ref_t`是一个指针，指向`struct protocol_t`。它也继承至`objc_object`。主要有协议名称、遵循的协议列表、（可选）实例方法、（可选）类方法、实例属性等内容。

### class_ro_t

`class_ro_t`主要记录了一个类在编译期确定的信息，不能改变。

```c++
struct class_ro_t {
    uint32_t flags; // 类标志
    uint32_t instanceStart; //
    uint32_t instanceSize;  // 实例大小
#ifdef __LP64__
    uint32_t reserved;
#endif

    const uint8_t * ivarLayout;

    const char * name; // 类名称
    method_list_t * baseMethodList; // 原始类中的方法
    protocol_list_t * baseProtocols; // 原始类中的协议
    const ivar_list_t * ivars; // 实例变量

    const uint8_t * weakIvarLayout; //
    property_list_t *baseProperties; // 原始类中的属性
    // Swift 初始化器
    // This field exists only when RO_HAS_SWIFT_INITIALIZER is set.
    _objc_swiftMetadataInitializer __ptrauth_objc_method_list_imp _swiftMetadataInitializer_NEVER_USE[0];
    _objc_swiftMetadataInitializer swiftMetadataInitializer() const {
        if (flags & RO_HAS_SWIFT_INITIALIZER) {
            return _swiftMetadataInitializer_NEVER_USE[0];
        } else {
            return nil;
        }
    }

    method_list_t *baseMethods() const {
        return baseMethodList;
    }
    // 拷贝
    class_ro_t *duplicate() const {
        if (flags & RO_HAS_SWIFT_INITIALIZER) {
            size_t size = sizeof(*this) + sizeof(_swiftMetadataInitializer_NEVER_USE[0]);
            class_ro_t *ro = (class_ro_t *)memdup(this, size);
            ro->_swiftMetadataInitializer_NEVER_USE[0] = this->_swiftMetadataInitializer_NEVER_USE[0];
            return ro;
        } else {
            size_t size = sizeof(*this);
            class_ro_t *ro = (class_ro_t *)memdup(this, size);
            return ro;
        }
    }
};
```

可以看到，这里使用的复杂数据类型，在`class_rw_t`都出现过。理解起来应该比较轻松。其中`ivar_list_t`是成员变量列表，它继承至`entsize_list_tt`，其中的元素类型为`ivar_t`。一个`ivar_t`代表一个成员。

```c++
struct ivar_t {
#if __x86_64__
    // *offset was originally 64-bit on some x86_64 platforms.
    // We read and write only 32 bits of it.
    // Some metadata provides all 64 bits. This is harmless for unsigned
    // little-endian values.
    // Some code uses all 64 bits. class_addIvar() over-allocates the
    // offset for their benefit.
#endif
    int32_t *offset; // 地址相对于实例对象的偏移量
    const char *name; // 名称
    const char *type; // 类型编码
    // alignment is sometimes -1; use alignment() instead
    uint32_t alignment_raw; // 字节对齐数量
    uint32_t size; // 所占字节大小

    uint32_t alignment() const {
        if (alignment_raw == ~(uint32_t)0) return 1U << WORD_SHIFT;
        return 1 << alignment_raw;
    }
};
```

## 总结

`Runtime`的学习就是理解其中的数据结构，以及如何使用这些数据结构。该篇和之前的[理解 Objective-C 中的对象](https://waguan.cc/ios/2020/04/29/object/)介绍了实例对象和类对象对应的结构，是后续学习的基础。共勉！

## 参考

[^1]: http://www.sealiesoftware.com/blog/archive/2009/01/27/objc_explain_Non-fragile_ivars.html
[^2]: https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtTypeEncodings.html#//apple_ref/doc/uid/TP40008048-CH100-SW1
