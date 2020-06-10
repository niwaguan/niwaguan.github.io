---
layout: post
title: OC的对象系统
description: 我们平时书写的Objective-C对象，其实是以C语言的结构体为支撑，该篇具体讲述了相关细节以及注意点。
category:
  - iOS
tags:
  - Runtime
  - Object
---

在上篇[理解Objective-C中的对象]({{ site.url }}/ios/2020/04/29/object/)中讲述了对象的相关内容，其实只涉及了OC`对象系统`中的`实例对象`，另外还有`类对象`和`元类对象`。今天一起来聊聊后面的两个。


## 对象的心脏 - `isa`

在正式开始之前，需要了解一个关键知识点--`isa`。

之前有提到过下面的代码：

```
/// Object.mm line 33
typedef struct objc_class *Class;
typedef struct objc_object *id;
/// objc.h line 40
struct objc_object {
    Class _Nonnull isa  OBJC_ISA_AVAILABILITY;
};
/// objc-runtime-new.h line 1145
struct objc_class : objc_object {
    // Class ISA;
    Class superclass;
    // ...
}
```

1. 实例对象其实都是`objc_object`结构体
2. Class都是`objc_class`结构体
3. `objc_class`继承至`objc_object`，所以类也是对象，称之为`类对象`
4. 实例对象和类对象都拥有`isa`成员。

在上节中我们知道`实例对象`存储的只有其成员变量信息。一个类除了成员变量，还会有其他的信息：实例方法，类方法等。这些信息都存在哪里呢？没错，就是`类对象`。`isa`就是访问类对象的关键所在。

关于`isa`，之前我们看到的对象是这个样子：

```
/// Represents an instance of a class.
struct objc_object {
    Class _Nonnull isa  OBJC_ISA_AVAILABILITY;
};
```
从这里，我们只能看到它有一个`isa`成员，占据`8`字节的空间。但是，我们可以在Runtime源码中发现这样的定义：

``` 
/// objc-private.h line 82
struct objc_object {
private:
    isa_t isa;
public:
    // ISA() assumes this is NOT a tagged pointer object
    Class ISA();
    // ... 其他方法
}
```

这里`objc_object`任然是个结构体，其成员变量也只有一个`isa`，不同的是`isa`的类型变成了`isa_t`。`isa_t`是个联合体：

```
/// objc-private.h line 68
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

可以看到这个联合体比较古怪，里面有`cls`，`bits`，`匿名结构体`三个成员。但是不变的是，联合体暂用的内存大小依然是最大成员需要的内存大小。我们来计算下：

* `Class cls;` Class 本质上是指针，占8字节。
* `uintptr_t bits;` uintptr_t本质是`unsigned long`的typedef，占8字节。
* `匿名结构体` 这里使用了位域，加起来一共64bit，也是8字节。

所以该联合体`isa_t`占用内存为8字节，从而使用`isa_t`类型的`struct objc_object`结构体占用内存也为8字节！和之前使用`Class isa;`作为成员变量的定义占用内存一样。

那既然占用内存相同，为什么这里的写法如此古怪？答案是优化。可以看到上面的匿名结构体的成员`shiftcls`后面有一句注释：`MACH_VM_MAX_ADDRESS 0x1000000000`。虚拟内存的最大地址为`0x1000000000`，使用33位足够了。64位的空间只用33位是不是太浪费了。所以可以用来存储一些其他信息。那么这个优化过程直接用代码写出来，标识每一位的具体用法，就是这个匿名结构体的作用。而其他两个成员是对这64位的另外两种解读，你可以认为这64位是存储`class`的，可以认为它就是二进制数据。当然也可以解读成这个匿名结构体所代表的的含义。

一图胜千言，下面我总结了这64位的用法：
![isa64bit](media/isa64bit.png)

至此，我们看清了`isa`的真正面目。它不仅存储了`实例对象`的`类对象`地址，而且还有更多信息。


## 对象系统

可以看出，`实例对象`的`isa`可以访问`类对象`。而`类对象`也有`isa`，通过这个可以访问什么呢？这里引用[gparker](https://twitter.com/gparker)大神的图：
![class diagram-w466](media/class%20diagram.png)

1. 类对象是实例对象的蓝图，实例对象的isa指向类对象
2. 类对象也是对象，它也需要蓝图，就是元类对象，类对象的isa指向元类对象
3. 元类对象同样也是对象，其isa指向根类（NSObject）的元类对象
4. 根元类对象的isa指向它自己
5. 类具有继承链，可通过superclass访问其父类（类对象和元类对象均有父类）
6. 根类对象的superclass为空，根元类对象的superclass指向根类对象

## 验证对象系统

