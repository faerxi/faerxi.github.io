---
layout: post
title: AutoScript 构思（二）类型和类别
category: [coding]
tag: [AutoScript, 编程语言]
author: programmer
publish: false
---

AutoScript 中，**类型（Type）** 和 **类别（Kind）** 是实体的两种重要性质，前者为实体完成了分类，而后者则是一种更加本质的“可调用性”的体现。

## 类别

所有实体都拥有类别；对象的类别总是 0（我们也因此称其为 *不可调用的（Uninvocable）*，而函数的类别取决于其经过多少次调用后才能得到一个对象。类型的类别则没有具体的特点。

```autoscript
x = 42;
f = const funct (x : Int) -> funct (y : Int) -> x + y;
Even = const funct (x : Int) -> x `mod` 2 == 0;

print kindof(x);    // 输出 0
print kindof(f);    // 输出 2
print kindof(Even); // 输出 1
```

一个实体有效当且仅当其拥有确定且唯一的类别。因此不存在无限类别的实体。

```autoscript
f = const funct(x : Int) -> f;    // 编译错误，无法推断 f 的类型
```

类别无法被声明，它往往已经包含在类型当中；对于信息严重不足的类型（如 `size 8`），能对其进行的操作也屈指可数。



## 类型

类型的本质是实体静态性质的谓词。所有类别至少为 1 且必定可以在编译期调用求值的实体都可以当作类型使用。下面的小节让我们首先介绍这一类类型。

### 一阶类型谓词

下面展示了一些类型：

```autoscript
Even = const funct (x : Int) -> x `mod` 2 == 0;
Top = const funct (x : Int) -> True;
Bottom = const funct (x : Int) -> False;
```

我们可以用它们作为声明来定义变量。这些类型的隐式转换需要初始化值满足这里给定的谓词。

```autoscript
x : Even = 42;
y : Top = "abc";
z : Botom = 1.0;     // 错误，谓词判断返回 false，因此 1.0 不能隐式转换为 Bottom 类型
```

一阶类型谓词只要求其类别为 1，因此我们可以利用部分调用来构建类型。

```autoscript
x : (> 0) = 42;
y : (<> 0) = 0;      // 错误，谓词判断返回 false，因此 0 不能隐式转换为 (<> 0) 类型
```

表达式在这里作为类型似乎有些奇特，但时刻记住在 AutoScript 中，类型仅仅是一类特殊的实体而已；我们直接写出来的和用表达式计算得到的并没有太多不同。只不过这里需要提到的是，AutoScript 会为每个类型创建一个常量存储期的变量。以上面的 `(> 0)` 为例：

```autoscript
x : (> 0) = 42;
// 编译器会自动生成下面的代码：
OpPack__gt_0 = const (> 0);
x : OpPack__gt_0 = 42;
```

### 内置类型

