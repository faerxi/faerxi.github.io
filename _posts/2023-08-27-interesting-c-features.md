---
layout: post
title: 冷门且有趣的 C 特性
author: 码工法尔希
category: coding
tag: [C, 编程语言]
---

自从在 C89 分道扬镳后，C++ 和 C 的发展方向就出现了比较大的分歧。C++ 一直在向成为一个现代的拥有强大编译期运算能力的静态语言前进，而 C 的着重点似乎……非常不同：一方面 C 的主要应用场所，Linux 系统中更加看中的是 GNU 的 C 语法标准和 POSIX 中的 C 库标准，ISO C 的推进并不积极；另一方面也是 C 的主战场并不需要花里胡哨的新特性，这从它缓慢的迭代速度也可以看出来。

本篇文章会将 C 自进入 ISO 标准以来所有比较冷门且有趣的特性一一介绍。由于对 C 的社区不够了解，其中或许有一些特性并不冷门。我会以 C++ 程序员的角度来提出看法。


## 关键字

和其它语言的一大区别是，C 自 C99 之后就基本不再引入常见的，以小写字母组成的关键字，而是采用下划线加上大写字母开头的关键字，其原因可能是为了防止名字冲突（由于诸多历史项目的活跃，C 必须尽量避免名字冲突）。其中不乏一些在 C++ 中常见的：

```c
#include <stdio.h>

int main(void) {
    _Bool b;
    printf("%zu", _Alignof(char));
}
```

考虑到有不便之处，C 也为许多这样的关键字设立了一个标准库，并将全小写的宏定义为对应的关键字。以 `_Bool` 为例，我们只需导入 `<stdbool.h>` 就可以直接使用 `bool` 了。

```c
#include <stdalign.h>
#include <stdbool.h>
#include <stdio.h>

int main(void) {
    bool b;
    printf("%zu", alignof(char));
}
```

但从 C23 开始，ISO C 正式将一些宏设置为关键字（包括上面提到的 `bool` 和 `alignof`），同时这些头文件也被移除或废弃了。

### `restrict` 关键字

对于指针（不包括函数指针）和数组，我们可以对其使用 `restrict` 修饰符。此时相当于提示编译器这些参数在当前函数中不会出现地址冲突造成的引用问题。

```c
void copy_n(size_t n, char* restrict dest, char const* restrict src) {
    while (n--) {
        *dest++ = *src++;
    }
}

int main(void) {
    int arr[10], brr[10];
    copy_n(10, arr, brr);       // 没问题
    copy_n(8, arr, arr + 2);    // 未定义行为
}
```

`restrict` 只在其被声明的作用域中有效。对于结构体类型，这个作用域则取决于拥有该结构体类型的变量所在的作用域。

```c
struct foo {
    int* restrict p1, *restrict p2;
};

struct foo f;       // 全局有效

void bar(foo f) {}  // 只在函数 bar 中有效
```

作为 C99 引入的特性，它始终没有将其纳入 ISO C++。不过自从 C++23 起我们就可以使用 `[[assume]]` 属性来提示编译器做优化了，只不过表述起来要复杂一些。


### 泛型关键字 `_Generic`

C11 引入了泛型关键字 `_Generic`，其用法非常怪异，参考下面的例子：

```c
extern void magici(int);
extern void magicll(long long);
extern void magicf(float);
extern void magicd(double);

#define magic(x) _Generic((x),  // 这里的 (x) 用于判断类型，并分派到下面不同情况
    int: magici,
    long long: magicll,
    float: magicf,
    default: magicd             // default 是默认情况
)(x)                            // 调用函数

int main(void) {
    magic(42);                  // 调用 magici
    magic(4.2);                 // 调用 magicd
}
```

这从某种角度上可以实现编译期的参数多态，不过在数学以外的场景中不算常用。



## 结构体和联合体

### 匿名结构体和联合体

自从 C11 起，对于不具名的结构体和联合体，若其声明时没有同时声明一个该类型的变量，则其成员自动“提升”为所在作用域中的变量。

```c
struct {
    int x, y;
};                  // 匿名结构体，相当于定义了两个内存位置相邻的变量 x 和 y
union {
    struct {
        int r, g, b;
    };              // 匿名结构体，相当于在联合体中定义了变量 r, g, b，注意它们三个加起来和下面的 color 共享一段内存
    void* color;
};                  // 匿名联合体，相当于定义了 r, g, b, color 四个变量，其中 r, g, b 内存相邻，三者和 color 共享内存
```

这个功能在定义*和类型*时异常好用。可惜的是，C++17 中只引入了匿名联合体，且匿名结构体看起来遥遥无期。

### 指派器列表

C++ 从 C 这里继承了结构体和数组羸弱的初始化方式，即*聚合初始化*，这也是初始化混乱的主因之一。不曾想 C99 就引入了强大得多的指派器列表，C++ 很可惜地没有赶上这班车。

```c
struct foo {
    struct bar {
        int x, y, z;
    } r;
    struct baz {
        int a;
    } z;
    float f;
};

int main(void) {
    struct foo f = { .b = { .x = 0, .z = 42 }, .a.z = 0, .f = 1.f };
}
```

可以说初始化得轻松写意。虽然上面我是按照成员的顺序初始化的，但 C 对这个顺序并没有强制要求。

C++20 才姗姗来迟地跟进了这个特性，但却有许多要求。一方面是 C++ 执着于初始化顺序（明明在构造函数的初始化列表中不在意这个），另一方面也是 C++ 有很多特性不适合使用指派器列表初始化（比如不能平凡构造的对象、私有成员等）。因此这个特性在 C++ 中并不能展开拳脚，却在简洁的 C 中令人眼前一亮。

### 复合字面量

对于使用聚合初始化，或指派器列表初始化的对象，可以用显式类型转换的语法将其变为一个可以取地址的对象。

```c
struct foo {
    int x, y;
};

void swap_foo(struct foo* f1, struct foo* f2) {
    struct foo swap = *f1;
    *f1 = *f2;
    *f2 = swap;
}

int main(void) {
    struct foo f = { 1, 0 };
    swap_foo(f, &(struct foo){ .y = 2 });
}
```

C++ 中由于左值引用的存在，让人更加眼馋这个特性了，可惜目前遥遥无期。

## 数组

在 C++ 已经基本可以淘汰原生数组的同时（用 `std::array<T, N>` 代替），C 却将它玩出了花。更熟悉 C++ 的程序员或许会对下面介绍的一些特性感觉离奇。某种角度来说，比 C 类型严格的 C++ 甚至无法继承这些特性。

### 数组初始化

得益于 C99 引入的指派器列表，C 可以通过很神奇的语法初始化。

```c
int main(void) {
    int arr[3] = { [0] = 2, [2] = 1 };
}
```

这个语法甚至和用于初始化结构体的指派器兼容。

```c
struct foo {
    int x, y;
};

int main(void) {
    struct foo arr[] = { [0] = { .x = 0, .y = 42 }, [1].x = 42 };
}
```

很可惜的是，C++ 似乎并不青睐这样略显“魔幻”的语法，并没有随结构体指派器列表一同引入 C++20.


### 不确定长度数组

和 C++ 一样地，C 支持声明一个不确定长度的数组，此时它的性质和一个指针类似。通常我们会省略方括号中的数字，写成例如 `int[]` 的形式。

```c
int main(void) {
    extern int arr[];
    printf("arr[0] = %d", arr[0]);
}
int arr[] = { 1, 2, 3 };    // 这里虽然声明了不确定长数组，但其实际上等同于 int arr[3]
```

不确定长数组可以用于结构体的最后声明，此时它称为**柔性数组（Flexible Array）**。拥有柔性数组的结构体，其看起来可以随 `malloc` 的具体大小而变动。

```c
struct foo {
    size_t size;
    int arr[];
};

int main(void) {
    struct foo* vec = malloc(sizeof(struct foo) + sizeof(int) * 8);
    vec->arr[0] = 42;
}
```

当然，`foo` 本身的大小是固定的 `8`，即其不包含柔性数组的部分大小。柔性数组本质上就是一个语法糖，编译器会无视掉出现在结构体中最后出现的的不确定长度数组，并将其视作一个指向结构体结尾的地址（可能还要考虑对齐）的相应类型指针。得益于 C 语言并不会对内置数组类型进行下标检查，剩下的使用过程便畅通无阻了。


### 变量长度数组（VLA）

C++ 最眼馋 C 的特性之一就是**变量长度数组（Variable-Length Array, VLA）**。这是一种变长的栈数组，自 C99 起进入 ISO C。顾名思义，我们可以用一个非常量来声明一个数组。

```c
int main(void) {
    int n;
    scanf("%d", &n);
    if (n <= 0) {
        puts("Invalid array length");
    }
    int arr[n] = {};        // VLA，长度为 n 的整数数组
}
```

VLA 不允许以非自动存储期形式存在，因此它们一定没有链接。

```c
void foo(size_t n) {
    static int brr[n];      // 编译错误，静态存储期的 VLA
    extern int crr[n];      // 编译错误，外部链接的 VLA
}
```

和 VLA 相关的类型被称为**可变修饰类型（Variably-Modified Type）**，比如 VLA 的指针类型。它们同样只能出现在非文件作用域中。

```c
void foo(size_t n, int (*parr)[n]) {
    typedef int VLA[n];
    static int (*pbrr)[n] = parr;
}
```

当函数的参数是一个 VLA 时，其效果相当于声明了一个不确定长度的 VLA 类型，如 `int[*]`。这个记号的含义和 `int[]` 类似，只不过它只能出现在函数参数列表中。

```c
void foo(size_t n, int arr[n]);
// 效果相当于下面的声明
void foo(size_t n, int arr[*]);
```

得益于 C 中变换相当自由的指针类型系统，我们可以让一个 VLA 的指针指向动态分配的内存空间。

```c
int main(void) {
    int (*parr)[n] = malloc(n * sizeof(int));
}
```

从某种角度来看这很合理。

VLA 在标准中在每次进入其生命周期时都会重新分配一次内存；如果将它和跳转语句一同使用会有令人惊讶的效果：

```c
int main(void) {
    int x = 1;
label:;             // 让 label 对应一个空语句
    int arr[x];
    if (x++ < 10) {
        goto label;
    }
}
```

上面的代码中，会 10 次分配 `arr`，且每次的大小都不同。

### 数组修饰符

C 中的一些修饰符可以用来修饰数组类型，包括定长和不定长的数组；不过它出现的地方非常出乎意料：

```c
void foo(int arr[const 3]);     // 会将参数退化为 int const* 类型
void bar(int arr[static 42]);   // 接收数组至少应有 42 个元素
void baz(int arr[restrict], int brr[static restrict 10]); 
                                // 相当于两个 int restrict* 参数，且第二个有大小的要求
```

之所以有这样怪异的语法，是因为在 C23 之前，绝大多数在数组声明中出现的修饰符都是修饰其中元素，而非数组本身的。比如：

```c
const int arr[];                // 声明了元素类型为 const int 的数组
typedef int VLA[*];
void foo(VLA restrict arr, VLA restrict brr);   // arr 和 brr 的地址不同
```

因此为了修饰数组本身，C 将修饰符装入了方括号中。


## C++ 反哺的特性

C++ 基于 C89，因此可以认为所有 C++98 和 C89 重合的特性都是从 C 继承的。不过自此之后，C++ 向 C 反哺的特性频频出现，本节让我们罗列它们。

### `inline`

这或许令人惊奇，但 `inline` 是源自 C++，并在 C99 进入 C 的特性。它可以将函数内联到使用处。所有内联函数应该在使用时经过定义。

```c
// foo.h
inline int foo() {
    return 42;
}

// main.cpp
#include "foo.h"

int main(void) {
    printf("%d", foo());
}
```

### `_Static_assert`

完全等同于 C++ 的 `static_assert`，引自 C11。不过其不带错误信息的版本在 C23 时才引入。C23 时，ISO C 同时将此前在 `<assert.h>` 中定义的宏 `static_assert` 转正为关键字，同时将 `_Static_assert` 废弃。

```c
_Static_assert(false, "Wont' compile!");
```

### `nullptr`

一直以来，C 使用的是古老且有效的 `NULL` 宏作为空指针，其定义是 `(void*)0`。不过自 C23 起，ISO C 也加入了 C++ 中拥有 `nullptr_t` 类型的字面量 `nullptr`，其行为类似于一个右值。

### 枚举类型的基类

C23 起，我们可以指定一个枚举类型的 **基类（Underlying Type）**，其和 C++ 的语法和性质完全一致。

```c
enum Color : uint8_t { 
    Red, Green, Blue,
};
```

不过，即使加入了这个特性，我们依然没有办法提前声明一个枚举类型（像 C++ 那样）。


### 属性（Attribute）

C23 引入了 C++11 的 **属性（Attribute）** 语法，即用两层方括号包围的结构，包含但不限于下面这些：

* `[[deprecated]]`：用于标记名字是废弃的。
* `[[nodiscard]]`：用于标记某个返回值不应被忽略。
* `[[noreturn]]`：用于标记某个函数不会返回。

除了这些和 C++ 中含义基本一致的属性，C23 中也添加了一些独占的属性，比如 `[[unsequenced]]` 和 `[[reproducible]]`，其具体含义可以参考相关 paper。


## 奇幻的库函数

### `offsetof` 

用来得到结构体中某个成员的内存偏移量，这实际上也在 C++ 标准库中。不过 C++ 是有成员指针的，它代表了成员在类中的偏移量，C 可没有这东西。因此 `offsetof` 作为宏充满了魔法的意味：

```c
struct foo {
    int x, y;
};

int main(void) {
    printf("%zu", offsetof(struct foo, x));		// 输出 0
}
```

### `longjmp`

~~如果 `goto` 对你来说不够刺激，那么不妨试试 `longjmp`~~。这个函数可以加载之前保存的执行环境，从而达到从一个函数跳转到另一个函数的效果。

```c
#include <setjmp.h>
#include <stdnoreturn.h>

jmp_buf main_jmp_buf;

noreturn void foo(int ct) {
    long_jmp(main_jmp_buf, ct + 1);		// 这里的 ct + 1 会作为 setjmp 的返回值
}

int main(void) {
    volatile int ct = 0;
    if (setjmp(main_jmp_buf) != 3) {
        foo(ct);
    }
}
```

这个函数可以用来实现类似于异常处理的功能。



## 其它的特性

### `void` 作为函数参数类型

和 C++ 一样，C 允许将 `void` 作为函数的参数类型，此时相当于函数没有接受任何参数。不过，如果我们将 `void` 省略，由此得到的参数列表为空的函数，实际上却可能接受任意多个参数。这来自于 K&R 时代的 C 语法，即函数声明时不包含函数原型的语法。

```c
void foo();                 // 声明了函数 foo，接受的参数类型和个数尚未知

void foo(a, b) int a, b; {  // foo 的定义包含了函数原型
    return a + b; 
}
```

自 C23 起，C 采用和 C++ 一致的处理方式，而 K&R 的历史语法终于被移除了。


### 复数类型

C 将复数类型作为内置类型使用，并为其设置了 `_Complex` 和 `_Imaginary` 两个关键字。这一特性来源于 C99。为了声明一个复数变量，我们需要同时使用浮点类型修饰符和复数类型修饰符。

```c
#include <complex.h>

int main(void) {
    double complex z = I;
    printf("%.1f^2 = %.1f", z, z*z);
}
```

除了 `double`，我们也可以使用 `float` 或 `long double`。类似地，`imaginary` 也可以受这些修饰符修饰。

可能有读者要问，既然已经有了复数类型，那何必再引入一个虚数类型呢？毕竟从各种角度来说，虚数类型和实数类型基本完全一致，只是单位不同；我们只需要建立一个类似于二元组的类型即可。[cppreference][https://en.cppreference.com/w/c/language/arithmetic_types] 上的解释是说虚数类型的存在一方面减少了运算次数（比如虚数乘法 `z*w` 只需要一次运算，但若 `z` 和 `y` 是复数则需要四次运算。