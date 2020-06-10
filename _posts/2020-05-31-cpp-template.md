---
layout: post
title: C++中的模板
description: C++模板为生成通用的类型声明提供了更好的支持，该篇具体讲述了相关细节以及注意点。
category:
  - 基础
tags:
  - C++
---

C++模板为生成通用的类型声明提供了更好的支持，它可以将类型名（如`int`、`float`）作为参数名传递给接收方来建立类或函数。

## 问题场景

设想下，现在我们需要实现存储`int`类型的栈，如下：

![stack-structure](http://images-for-blog.oss-cn-beijing.aliyuncs.com/2020/06/10/stackstructure.png)

然后不到2分钟，三下五除二立马完成：

```c++
class Stack {
private:
    enum { MAX = 10 };
    int top;
    int items[MAX];
public:
    bool isEmpty();
    bool isFull();
    void push(int item);
    int pop();
};

bool Stack::isEmpty() {
    return top == 0;
}

bool Stack::isFull() {
    return top == MAX - 1;
}

void Stack:: push(int item) {
    if (isFull()) {
        return;
    }
    top += 1;
}

int Stack::pop() {
    if (isEmpty()) {
        return 0;
    }
    return items[top--];
}
```

结果没过几天，发现`int`类型不适合需求了，要改`float`.....这下可算张心眼了，万一下次再改咋办？索性把类型使用`typedef`定义下，所以我们有了第二版：

```c++
typedef int StackItem; // 哼哼，将需要存储的类型独立出来，让你改！
class WGStack {
private:
    enum { MAX = 10 };
    int top;
    StackItem items[MAX];
public:
    bool isEmpty();
    bool isFull();
    void push(StackItem item);
    StackItem pop();
};
```

这次修改之后，可以缓口气了。

嗯，咖啡还是很香的! 直到你即需要存储`int`型的栈，又需要`float`类型的栈....除了copy一份代码，似乎没有其他办法了...

不过，今天的大招可以解决该问题-C++模板，一起来瞅瞅！

## 基本语法

```c++
template<class Type> // 这里的class代表定义一个类型
// template<typename Type> typename可以代替class，可读性也比较高，推荐使用
class WGStack {
private:
    enum { MAX = 10 };
    int top;
    Type items[MAX];
public:
    bool isEmpty();
    bool isFull();
    void push(Type item);
    Type pop();
};

template<typename Type>
bool WGStack<Type>::isEmpty() {
    return top == 0;
}

template<typename Type>
bool WGStack<Type>::isFull() {
    return top == MAX - 1;
}

template<typename Type>
void WGStack<Type>::push(Type item) {
    if (isFull()) {
        return;
    }
    items[top++] = item;
}

template<typename Type>
Type WGStack<Type>::pop() {
    if (isEmpty()) {
        return 0;
    }
    return items[top--];
}
```

现在，我们既有了即可以存储`int`类型又可以存储`float`类型的栈：

```c++
WGStack<int> intStack;
WGStack<float> floatStack;
```

编译器会根据我们的声明，自动生成类，这样我们就不用自己复制粘贴了。

## 高级特性

看到上面的实现，是不是觉得这就是`OC`中的泛型啊！没错，C++的模板确实和泛型很像，但它比泛型更强大。

### 非类型参数

模板不仅支持类型参数，并且支持非类型参数：

```c++
// 这里的int size就是非类型参数
template<typename Type, int size>
class WGStack {
private:
    int top;
    Type items[size];
public:
    bool isEmpty();
    bool isFull();
    void push(Type item);
    Type pop();
};
```

之后，我们可以指定该栈的大小：

```c++
WGStack<char *, 10> s1 = WGStack<char *, 10>();
WGStack<char *, 100> s2 = WGStack<char *, 100>();
```

这里需要注意下面几个方面：
1. 非类型参数支持：整形、枚举、引用或指针，它是常量，不能修改
2. 这里编译器会生成2个类，分别是存储10、100个元素的栈；若想改进，可以将数量作为参数传入构造器，动态生成内部的数组。

### 递归

一个模板的类型参数是其本身，可以递归。

```c++
WGStack< WGStack<int, 10>, 20 > listStack;
```

这里定义了一个含有20个元素的`listStack`，而每一个元素又是一个包含10个int类型的栈。相当于二维数组。

### 多个类型参数

模板支持多个类型参数：

```c++
// 多个类型参数依次使用`,`隔开
template<typename T1, typename T2>
class WGPair {
private:
    T1 a;
    T2 b;
public:
    T1 first();
    T2 second();
};

template<typename T1, typename T2>
T1 WGPair<T1, T2>::first() {
    return a;
}

template<typename T1, typename T2>
T2 WGPair<T1, T2>::second() {
    return b;
}
```

### 默认值

类型参数和非类型参数都是可以有默认值的：

```c++
// 默认值直接使用`=`链接在类型后面；同理，非类型参数也可以
template<typename T1 = int, typename T2 = float>
class WGPair {
private:
    T1 a;
    T2 b;
public:
    T1 first();
    T2 second();
};
```

### 设置别名

别名可以简化代码书写，也可以提高可读性。例如上面的`WGStack`，可以有下面的别名设置：

```c++
// 使用typedef
typedef WGStack<int, 10> Int10Stack;
// 使用using关键字
using Int20Stack = WGStack<int, 20>;
// 设置别名时指定部分类型参数或非类型参数
template <typename T>
using Fixed100Stack = WGStack<T, 100>;
```

## 模板的具体化

由模板生成具体类，称为具体化。模板的具体化一共有3中形式：

### 隐式实例化

我们上面使用的例子，都属于该方式：直到需要生成具体类的时候，才会去生成，由编译器决定何时生成。

```c++
// 这里只是定义了变量，并不会导致编译器生成具体的类
WGStack< WGStack<int, 10>, 20 > listStack;
// 下面需要为变量申请内存了，才会生成具体类，并创建对象，分配空间
listStack = WGStack< WGStack<int, 10>, 20 >();
```

### 显式实例化

使用特定语法，书写语句，指示编译器生成具体类，即使没有使用该类生成对象。

```c++
template class WGStack<float, 10>;
template class WGPair<int, float>;
```

### 部分具体化

例如，对于上面的模板：

```c++
template<typename T1, typename T2>
class WGPair {
    //...
};
```

该模板指定了两个类型参数，加入我们想实现：`当T2为int类型时，生成的类有自己的行为`，那么就可以在现有基础上进行部分具体化：

```c++
// 之前的类型参数T2被部分具体化，直接省略；保留T1的声明
template <typename T1>
// 指定将T2部分具体化为int类型
class WGPair<T1, int> {
    // 具体实现
};
```

当我们将所有类型参数都指定时，就出现了特殊情况，称之为`显示具体化`。例如：

```c++
// 显示具体化，就是讲所有的泛型参数显示指定
template <>
class WGPair<int, int> {
    // 具体实现
};
```

需要注意，对于`WGPair<int, int> intPair;`声明，它符合`隐式实例化`、`部分具体化`以及`显示具体化`三种，编译器会选择具体化程度最高的模板`显示具体化`。

> tips:
> - 实例化，由模板到类的过程。
> - 具体化，由泛型到具体类型的过程。

还可以借助部分具体化，来使用现有类型参数填充其他类型参数：

```c++
/// 原有模板
template<typename T1, typename T2, typename T3>
class Trio { };
/// 使用T2指定T3的类型
template<typename T1, typename T2>
class Trio<T1, T2, T2> { };
/// 使用T1指定T2、T3的类型
template<typename T1>
class Trio<T1, T1*, T1*> { };
```

## 在成员变量中使用模板

模板可以作为结构、类或模板类的成员。

```c++
template<typename T>
class WGBeta {
    // 在内部定义的模板类
    template<typename V>
    class hold {
    private:
        V val;
    public:
        hold(V v = 0) : val(v) { }
        V value() { return val; }
    };
    // 在成员变量中使用模板，使用T类型具体化V
    hold<T> q;
    hold<int> n;
public:
    WGBeta(T t, int i) : q(t), n(i) {}
};
```

或者将内部的嵌套移动到外部实现，需要注意嵌套类的实现方式。

```c++
template<typename T>
class WGBeta {
    // 在模板类中声明模板类hold
    template<typename V> class hold;
    // 使用T类型具体化V
    hold<T> q;
    hold<int> n;
public:
    WGBeta(T t, int i) : q(t), n(i) {}
};

// 实现WGBeta类中声明的hold
// 1. 由于WGBeta和hold类是嵌套的关系，这里也需要指明模板的嵌套关系
template<typename T> template<typename V>
// 2. 指明WGBeta类中的嵌套类
class WGBeta<T>::hold {
private:
    V val;
public:
    hold(V v = 0) : val(v) {}
};
```

## 在模板参数中使用模板

模板可以声明类型参数，这个类型参数可以是另一个模板。
> 注意：和模板的递归使用有差别

```c++
// Crab是一个模板类
// 1. 其第一个模板类型参数为另一个模板，这里名称是Thing。可以看做是一个正常模板类声明，截止到类名的写法。
template<template<typename T> class Thing, typename U>
class Crab {
private:
    // 2. 使用Crab模板类型参数U具体化Thing中的T
    Thing<U> s1;
public:
    Crab() {}
};
```

## 模板类中的友元

模板类也可以声明友元函数，分为3类：非模板友元、约束模板友元、非约束模板友元。建议大家动手操作下，便于理解。

### 非模板友元

在模板类中声明正常的函数作为友元。例如：

```c++
void counts();

template <typename T>
class HasFriend {
    static int numbers;
    T value;
public:
    // 该模板所有实例化的友元函数
    friend void counts();
    // 使用模板作为参数的函数，对于每一个使用到的T需要有与之对应的实现
    friend void report(HasFriend<T> &);
    
    HasFriend(T t) : value(t) {
        numbers ++;
    }
    ~HasFriend() {
        numbers--;
    }
};

// 初始化模板类的静态成员
template <typename T> int HasFriend<T>::numbers = 0;

// 实现counts
void counts() {
    cout << "instance counts: " << HasFriend<int>::numbers << endl;
}

// 实现int类型的report
void report(HasFriend<int> & t) {
    cout << "report value: " << t.value << endl;
}
```

### 约束模板友元

友元函数是模板函数，有自己的类型参数。例如：

```c++

// 先声明模板函数
template <typename T> void counts();
template <typename T> void report(T &);

template <typename T>
class HasFriend {
    static int numbers;
    T value;
public:
    // 再将模板函数声明为模板类的友元
    friend void counts<T>();
    friend void report<>(HasFriend<T> &);
    
    HasFriend(T t) : value(t) {
        numbers ++;
    }
    ~HasFriend() {
        numbers--;
    }
};

// 初始化模板类的静态成员
template <typename T> int HasFriend<T>::numbers = 0;

// 实现模板函数
template <typename T>
void counts() {
    cout << "instance counts: " << HasFriend<T>::numbers << endl;
}

template <typename T>
void report(T &t) {
    cout << "report value: " << t.value << endl;
}
```

### 非约束模板友元

在类的内部可以创建`非约束模板友元`。例如：

```c++
template <typename T>
class ManyFriend {
    T value;
public:
    ManyFriend(T i) : value(i) { }
    // 嵌套的模板友元函数，这里的C,D似乎和ManyFriend<T>相等
    template <typename C, typename D> friend void show(C &c, D &d);
};

template <typename C, typename D> void show(C &c, D &d) {
    cout << "value1: " << c.value << ", value2: " << d.value << endl;
}

int main(int argc, const char * argv[]) {
    
    ManyFriend<int> mf1 = ManyFriend<int>(10);
    ManyFriend<float> mf2 = ManyFriend<float>(20);
    show(mf1, mf2); // value1: 10, value2: 20
    return 0;
}
```

总结下这三种友元的区别：

* 非模板友元，友元函数本身是常规函数，这样在该函数中访问模板实例化后的具体类时，无法做到通用性。
* 约束模板友元，友元函数是模板函数，这样模板类的类型参数和模板函数的类型参数可以相互具体化，具有通用性。
* 非约束模板友元，定义在模板类内部的模板函数，但是该模板函数的类型参数代表了，模板类的某一个实例化类。

## 参考

参考书籍均有`PDF`版本，如有需要添加微信`Niwaguan-_-`。

1. C++ Primer Plus 第六版