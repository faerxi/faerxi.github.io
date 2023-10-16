---
layout: post
title: 实现一个层级结构
category: coding
tag: [C++]
author: programming
typora-root-url: ./..
---

常见的数据结构，比如*数组*、*哈希表*、*红黑树*等在日常编程中有相当多的出现率。本节将以实现一个数据库为目标，设计一个数据结构和相应的算法来支持一些应用场景和常见操作。

## 目标

想象我们有一集数据，它们拥有相同的形式（比如每项数据都是一个城市的词条），我们可以将其以哈希表形式存储：

```javascript
data = {
    "Shanghai": { /* 一些信息 */ },
    "Suzhou": { /* 一些信息 */ },
    // 更多城市
}
```

不过这样的结构缺少城市间联系的信息；一个解决方案是在城市信息中添加表现关联的信息：

```javascript
data = {
    "Shanghai": {
        "Province": "Shanghai",
        // 其它信息
    },
    "Suzhou": {
        "Province": "Jiangsu",
        // 其它信息
    },
    // 更多城市
}
```

此时可以通过 `data[city_1].Province == data[city_2].Province` 来确认两个城市是否属于一个省。不过如果已知一个省，寻找其中的所有城市依然是一个 $O(n)$ 的查询操作。我们可以使用 $O(n)$ 的哈希表来解决这个问题：

```javascript
data = { /* 省略信息 */ }
provinces = {
    "Shanghai": [
        "Shanghai"
    ],
    "Jiangsu": [
        "Nanjing",
        "Suzhou",
        // 更多城市
    ],
    // 更多省份
}
```

如果有大量这样的“中间信息”就需要更多的哈希表来确保轻松的访问。对于一些构成层级结构的中间信息，我们完全可以将其实现为一个树；同一个节点下的所有子节点总是拥有同一个特征，用它们的父节点表示。为了方便每个子节点的访问，可以将每个节点中存放一个哈希表：

```javascript
data = [
    "Shanghai": [
        "Shanghai": { /* 省略信息 */ }
    ],
    "Jiangsu": [
        "Nanjing": { /* 省略信息 */ },
        "Suzhou": { /* 省略信息 */ },
        // 更多城市
    ],
    // 更多省份
] // 根节点
```

这样的树可能也需要多个，因为节点的层级关系取决于角度；比如上面的数据可以重新按照方言和细分分类：

```javascript
data_dialect = [
    "Mandarin": [
        "Beijing": { /* 省略信息 */ },
        "Tianjin": { /* 省略信息 */ },
        // 更多城市
    ],
    "Wu": [
        "Shanghai": { /* 省略信息 */ },
        "Suzhou": { /* 省略信息 */ },
        // 更多城市
    ],
    // 更多方言
]
```

此时注意到没有必要在每个词条下都记录所有信息（会造成大量的冗余）。我们实际上只需要记录每个词条的*引用*即可，同时再用一个哈希表存储这些引用到实际词条信息的映射：

```javascript
view_province = [
    "Shanghai": [
        "Shanghai"
    ],
    "Jiangsu": [
        "Nanjing",
        "Suzhou",
        // ...
    ],
    // ...
]
view_dialect = [
    "Mandarin": [
        "Beijing",
        "Tianjin",
        // ...
    ],
    "Wu": [
        "Shanghai",
        "Suzhou",
        // ...
    ],
    // ...
]
data = [
    "Shanghai": { /* ... */ },
    "Beijing": { /* */ },
    // ...
]
```

这就形成了一个*数据-视图*模型。实际的数据由数组形式存储，然后根据实际*分类*构建不同的视图，数据结构可以采用哈希表树（也就是前面提到的，哈希表中每个元素都是指向新的哈希表的指针）。