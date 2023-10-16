---
layout: post
title: UE 中的反射限定符
category: coding
tag: [UE, C++]
author: programming
typora-root-url: ./..
---

本篇整理 UE 中常见的反射限定符和它们的作用。

## `UCLASS`

`UCLASS` 用于修饰一个 UE 类，它必须是 `UObject` 的子类。

### Blueprint 相关

#### `BlueprintType`

该类型可以在 Blueprint 中作为类型出现。默认不可。

#### `Blueprintable`和 `NotBlueprintable`

该类型是否可以作为 Blueprint 类型的基类。默认继承直接基类的性质。因此可以使用相应的限定符来覆写直接基类的设定。此外也可以用 `IsBlueprintBase = (Bool)` 来等效替代这两个限定符。

#### `BlueprintSpawnableComponent`

dd该类型是否可以作为组件添加到 Actor 中（即从左侧的 Add 按钮中选择组件添加）。

#### `Abstract`

表明这个类型不能在关卡中生成。



### 显示和分类

#### `ClassGroup = (String)`

设置类的分组，会在 Actor 选单或 Component 选单中出现。

#### `Hidden`

隐藏这个类型，在编辑器中的任何类浏览器中都无法找到该类。

#### `ToolTip = (String)` 和 `ShortToolTip = (String)`

（Meta 属性）鼠标悬浮时显示这个类型的功能；覆写注释在此的作用。`ShortToolTip` 会用在需要比较简短注释的场景。

#### `DisplayName = (String)`

（Meta 属性）在 Blueprint 中实际显示的名字。

### 其它属性

#### `Deprecated`

该类被弃用，此类的对象将不参与序列化信息的生成。被子类继承。

#### `DeprecationMessage = (String)`

（Meta 属性）作为弃用类在 Blueprint 中使用的额外警告信息。