

## 结构体

该部分主要梳理C语言中结构体相关知识，了解的同学可以跳过啦。

### 基本语法

C语言中定义一个结构体遵循下面的规则：

```c
struct 结构体名称 {
    // 成员
};
```

例如：

```c
struct book {
    char title[20];
    char author[20];
    float price;
};
```

之后你就可以像下面这样使用它：

```c
int main(int argc, const char * argv[]) {
    struct book a; // 只定义变量
    a.title = "三体";
    strcpy(a.author, "刘慈欣"); // 注意这里的赋值方式
    a.price = 158.0;
    
    struct book b = { "三体", "刘慈欣", 158.0 }; // 定义并给出全部成员的值
    struct book c = { // 定义并指定部分成员的值
        .title = "三体",
        .price = 158.0
    };
    struct book *pb = NULL;
    pb = &a;
    
    // 注意普通变量和指针变量访问结构体成员的差别
    printf("%s - %s - %.2f", b.title, b.author, b.price);
    printf("%s - %s - %.2f", pb->title, pb->author, pb->price);
    return 0;
}
```

如果你觉得`struct book a;`的形式定义变量太过麻烦，可以使用`typedef`：

```c
typedef struct book {
    char *title;
    char author[20];
    float price;
} Book;
// 或者将 book 省略
typedef struct {
    char *title;
    char author[20];
    float price;
} Book;
```

这里将`struct book`定义为`Book`类型，之后就可以直接像声明`int`类型的变量一样声明`Book`类型的变量：

```c
Book d = {};
Book *pd = NULL;
```

### 特性说明

1. 结构体的成员可以是另外一个结构体，也就是说结构可以嵌套：
    ```c
    typedef struct {
        char *name;
        int age;
    } Person;
    
    typedef struct book {
        char *title;
        Person author;
        float price;
    } Book;
    ```
    
    也可以有匿名嵌套：
    
    ```c
    typedef struct book {
        char *title;
        struct {
            char *name;
            int age;
        };
        float price;
    } Book;
    ```
    这样在访问Book中的匿名结构时，可以直接访问其中的name和age成员。
    
1. 结构体可以作为函数的参数及返回值，默认是以值传递的方式。需要的话也可以传递指针。
2. 结构体支持整体赋值：
      
    ```c
    Book a = { "三体", "刘慈欣", 158.0 };
    Book b = a; // 直接整体赋值
    ```
3. 伸缩型数组成员 - `flexible array member`

    带有伸缩型数组成员的结构体，该成员`必须是结构体中的最后一个成员`。下面是一个例子：
    
    ```c
    typedef struct book {
        char title[20];
        char author[20];
        float price;
        float scores[]; // 这里添加了伸缩型数组成员
    } Book;
    
    int scores = 5;
    Book *b = (Book *)malloc(sizeof(Book) + scores * sizeof(float));
    ```
    这里推荐使用`malloc`来为变量分配内存，因为`Book b;`这种方式不会为`scores`成员分配内存。但是我在实验的时候发现了特殊情况：
    
    ```c
    int main(int argc, const char * argv[]) {
        printf("size of Book: %lu\n", sizeof(Book)); // 44
        Book b;
        int index = 0;
        
        while (index < 100000) {
            b.scores[index] = 100; // 写入
            printf("score %d = %f\n", index, b.scores[index]); // 读取
            index ++;
        }
        
        return 0;
    }
    ```
    这里可以正常读写`scores`成员，直到`index=695`，程序因`EXC_BAD_ACCESS`崩溃。测试环境为`macOS Catalina 10.15.4 - Xcode 11.5(11E608c)`。
    
    原因未知，希望有大神赐教。
    
    但是这说明了这种方式是不稳定的，不建议使用。
1. 位字段 - `bit field`
    
    支持指定位数的成员，但成员类型需要为整形或枚举：
    
    ```c
    struct capability {
        unsigned int fly : 1;
        unsigned int run : 1;
    };
    ```
    
    
## 联合体

联合体支持在同一内存空间存储不同类型值，注意不能同时存储。例如：

```c
union hold { 
    int digit; 
    double bigfl; 
    char letter; 
};
```

这里虽然声明了3个成员，但只能存储一个int，或一个double或一个char的数据。也就是说使用不同成员名称进行赋值，后面的会覆盖之前的：

```c
hold h = {}; // 初始化, 或者  hold h = { .digit = 10 };
h.digit = 100;
h.bigfl = 101; // 覆盖之前的100
h.letter = 'c'; // 覆盖之前的101
```

当然，联合也是支持整体赋值，匿名嵌套等特性的。

## 参考

1. C Primer Plus 第六版
2. C++ Primer Plus 第六版