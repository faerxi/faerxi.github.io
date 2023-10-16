---
layout: post
title: UE 中的空间和变换
category: [coding]
tag: [UE, C++, 数学]
math: true
author: programming
typora-root-url: ./..
---

本篇让我们介绍 UE 中的空间和变换系统，其中会包含一些看起来比较棘手的空间变换例子。

## 基础知识

### 坐标系

UE 中，我们用 **坐标系（Coordinate System）** 来决定 Actor 的位置或朝向。UE 使用左手系，因此它的 $xyz$ 轴分别对应了 Actor 的 *前方*、*右方* 和 *上方*（区别于右手系里 $y$ 指向左方）。可以参考下面的图：

<img src="/assets/2023-09-14-unreal-space-and-transform/coordinate-example.png" alt="image-20230914122246102" style="zoom: 50%;" />

这个模型下面出现的红绿蓝色的坐标就是该 Actor 的坐标系。红色表示 $x$ 方向，绿色表示 $y$ 方向，蓝色表示 $z$ 方向。

> 这里可能会有一个疑问：之前明明说 $x$ 指向的是前方，为何这里的 $y$ 轴才是前方，而 $x$ 轴指向了左方？实际上，UE 中所有 Skeletal Mesh 的资产都会默认面向 $y$。其中原因我无从知晓，但这实际上类似于一个右手系的设置：将 $xy$ 互换，则 $x$ 指向前方，$y$ 指向左侧，$z$ 依然指向上方。
>
> 虽然 Skeletal Mesh 看起来 $y$ 才是前方，但如果调用上面的 `GetForwardVector` 方法，就会发现依然返回的是 $x$ 的方向。这是一个让人很不习惯的不一致。
{: .prompt-info }

在 UE 中，我们可以通过一些函数得到 Actor 的三个方向：

```upp
FVector AActor::GetActorForwardVector(   // x 方向
) const;
FVector AActor::GetActorRightVector(     // y 方向
) const;
FVector AActor::GetActorUpVector(        // z 方向
) const;
```

Actor 中的组件并不一定拥有和 Actor 相同的坐标系，因此它们也有相应的获取 $xyz$ 方向的函数：

```upp
FVector USceneComponent::GetForwardVector( // x 方向
) const;
FVector USceneComponent::GetRightVector(   // y 方向
) const;
FVector USceneComponent::GetUpVector(      // z 方向
) const;
```

从所定义的类型也可以看出，组件想要拥有坐标系，其至少需要是 `USceneComponent` 的子类。

确定一个坐标系之后，我们就可以判断空间中任意物体的位置和朝向了。同一物体取不同的坐标系时，会得到不同的结果。我们将一些特殊的坐标系成为 **空间（Space）**，列出如下：

* *世界空间（World Space）*：位于关卡的“中心”且有一个预设的 $xyz$ 方向。$z$ 方向和重力方向正好相反。这是现实世界中不存在的“绝对空间”。
* *组件空间（Component Space）*：组件的空间，比如 Skeletal Mesh 的组件空间就是 root 的坐标系。
* *父空间/本地空间（Parent Space/Local Space）*：对于一个层级结构中的结点，其父结点的空间，比如父骨骼的坐标系。

这些空间之间的转换是非常常见的操作，后面让我们从 UE 的变换系统出发，熟悉这个系统。

### 向量

我假定读者已经具有基本的向量知识，比如向量的加减法和标量乘法。向量的用处非常之多，从它能代表的量就可以看出来：

* 向量可以表示一个方向（即空间中的位移）。

* 向量也可以表示一个点，因为当把向量的起始点和坐标系原点对齐时，空间中每个点都可以唯一地对应一个向量。事实上，UE 中我们的坐标总是一个向量，它可以通过 `AActor::GetActorLocation` 或 `USceneComponent::GetComponentLocation` 得到。

* 向量可以表示一个旋转，只需将它的方向视为转动的轴，大小视为转动的量。可以通过下面的函数构建一个旋量（表示旋转的量，和方向相关）：

  ```upp
  static FRotator UKismetMathLibrary::RotatorFromAxisAndAngle(
      FVector Axis,
      float Angle
  );
  static FRotator UKismetMathLibrary::MakeRotFromX(
      const FVector& X
  );
  static FRotator UKismetMathLibrary::MakeRotFromXY(
      const FVector& X,
      const FVector& Y
  );
  ```

  这里只列出了从 $x$ 轴和 $xy$ 平面上的向量构建旋量的方式，其它四种情况（$y$、$z$、$yz$、$zx$）也是类似的。

向量之间有一些重要的运算：

* 点乘：获取两个向量的“投影积”，几何上的含义是一个向量对另一个向量的投影长度。用 $\cdot$ 表示：

  $$
  \vec{u} \cdot \vec{v} = uv\cos{\theta}
  $$
  
  这里的 $\theta$ 是两个向量的夹角。如果将 $\vec{u}$ 和 $\vec{v}$ 用坐标表示，点乘的结果可以很轻松地得到：
  
  $$
  \vec{u} \cdot \vec{v} = u_xv_x\hat{x} + u_yv_y\hat{y} + u_zv_z\hat{z}
  $$
  
  由于其计算便捷性，点乘通常被用来计算两个向量的夹角：
  
  $$
  \cos{\theta} = \frac{\vec{u} \cdot \vec{v}}{uv}
  $$
  
* 叉乘：获取两个向量的“作用积”，几何上的含义是两个向量形成的平行四边形的面积。用 $\times$ 表示：
  
  $$
  \vec{u} \times \vec{v} = uv\hat{n}\sin{\theta}
  $$

 这里的 $\hat{n}$ 是一个让 $\vec{u}$、$\vec{v}$、$\hat{n}$ 形成右手系的单位向量（即 $\vec{u}$ 作为前方、$\vec{v}$ 作为左方时 $\hat{n}$ 是上方）。

  叉乘的计算比较麻烦，在坐标系中可以用下面的式子：

$$
  \vec{u} \times \vec{v} = \begin{vmatrix}
  	\hat{x} & \hat{y} & \hat{z} \\
  	u_x & u_y & u_z \\
  	v_x & v_y & v_z
  \end{vmatrix} = (u_yv_z - u_zv_y)\hat{x} + (u_zv_x - u_xv_z)\hat{y} + (u_xv_y - u_yv_x)\hat{z}
$$

  注意到叉乘不难组交换律，且有 $\vec{v}\times\vec{u} = -\vec{u}\times\vec{v}$。

作为平移量而使用的向量，我们总是会提前声明它选取的坐标系（默认情况下就是世界空间）。

### 旋量（欧拉角）

旋转有很多种表示方式，包括上一节中我们提到的用向量表示旋转的方式。在 UE 中，我们参考航空器的标准——欧拉角来表示一个旋转：

* 翻滚角（Roll）：绕 $x$ 轴（向前）的旋转。
* 俯仰角（Pitch）：绕 $y$ 轴（向右）的旋转。
* 偏航角（Yaw）：绕 $z$ 轴（向下）的旋转。

注意到这个标准中的 $xyz$ 轴构成一个右手系，似乎和 UE 中不符；但实际上的旋转方向就是严格按照上面所述，只不过 $z$ 轴上的旋转看起来方向反了而已。可以参考下面的例子：

![image](/assets/2023-09-14-unreal-space-and-transform/rotation-axis-1.png){: .width=300 .height=400}

这里空间的三根轴如图所示（我选取的是世界坐标，因此也可以参考左下角的标注）。现在如果绕 $x$ 轴进行旋转：

![](/assets/2023-09-14-unreal-space-and-transform/rotation-axis-2.png)

可以看到旋转的方向和 $x$ 轴方向相同。$y$ 轴也是一样：

![](/assets/2023-09-14-unreal-space-and-transform/rotation-axis-3.png)

但对于 $z$ 轴，正向旋转的方向正好沿 $-z$ 方向。

![](/assets/2023-09-14-unreal-space-and-transform/rotation-axis-4.png)

因此，UE 实际上使用了两套手系：在坐标上采用左手系，但处理旋转时依然使用游戏以外领域更常见的右手系。两种手系在 $x$、$y$ 轴上没有区别，仅在 $z$ 轴上需要翻转方向。

> 推荐一个可视化交互的网页，其中可以直观地看到这个旋量系统的作用：[链接](https://compsci290-s2016.github.io/CoursePage/Materials/EulerAnglesViz/)。
>
{: .prompt-info }

UE 的旋量异常混乱，除了在不同轴使用不同手性，内部存储时也采用了奇怪的顺序（Pitch-Yaw-Roll），好在使用 Blueprint 时不会感受到这一点。

> UE 中同时存在三套旋量相关的顺序：
>
> * Pitch-X、Yaw-Y、Roll-Z：这是存储的顺序（即，`FRotator` 的构造时需要采用这个顺序），这实际上是其它引擎中采用的顺序。
> * Roll-X、Pitch-Y、Yaw-Z：这是航空业使用的顺序。虽然 UE 中 Z 的通用方向相反，但它依然采用这个转动方向，导致 $z$ 轴旋转实际上是左手系（或者反过来说，$xy$ 使用了和 UE 中不同的右手系）。
> * Yaw、Pitch、Roll：这个无关乎轴的选取，是欧拉角的通用计算顺序。分解转动时总是先考虑 Yaw，再考虑 Pitch，最后是 Roll。这源于欧拉角不能独立表示的缺陷。
>
{: .prompt-warning }

## 四元数

### 引入

在三维空间中，我们使用 *向量* 来表示位置和平移。向量的良好性质让位置和平移之间的许多运算得以轻松地实现（下面出现的所有量都是向量，但按照习惯将平移变换用大写字母表示）：

* 位置经平移后得到另一个位置：$T + \vec{p} = \vec{q}$。
* 平移可以叠加为新的平移：$(T + U) + \vec{p} = T + (U + \vec{p})$。
* 两个位置之间存在一个唯一的平移，使得一个位置可以被变换到另一个：$\vec{p} - \vec{q} = T$。

*旋量* 当然最好也有相同的特征：

* 方向经旋转后得到另一个方向：$\Gamma \vec{\mu} = \vec{\theta}$。
* 旋转可以叠加为新的旋转：$(\Gamma \Lambda)\vec{\mu} = \Gamma (\Lambda \vec{\mu})$。参考[欧拉旋转定理](https://en.wikipedia.org/wiki/Euler%27s_rotation_theorem)。

这两个特征我们可以通过欧拉角得到。然而第三点，也是最为关键的一点，用此前的欧拉角模型不能达成：比如万象锁中绕 $x$ 和 $z$ 轴的旋转可能会有完全相同的效果。更显然的例子是欧拉角的三个分量不设限时，增加或减少 $360^\circ$ 的倍数不会有任何不同。因此，我们需要一种新的代数，它在支持向量基本运算（加法、标量积）的同时可以定义出除法运算。即，对于旋量 $\vec{\mu}$ 和 $\vec{\theta}$，存在一个唯一的旋量 $\Gamma$ 满足 $\Gamma\vec{\mu} = \vec{\theta}$。

解决这个问题的代数就是四元数。



### 定义和运算规则

**四元数（Quaternion）** 是对复数的扩展，其中定义了三个虚单位 $i$、$j$ 和 $k$。所有四元数都可以表示为下面的形式：

$$
q = a + bi + cj + dk \quad a, b, c, d \in \mathbb{R} \label{def:quaternion}
$$

其中三个虚单位满足下面的运算性质，非常相似于三维空间中标准基的叉乘结果：

$$
ij = -ji = k \quad jk = -kj = i \quad ki = -ik = j
$$

每个虚单位的平方都等于 $-1$，这个类似于二维平面上的旋转（回忆复数乘法的几何意义）：

$$
i^2 = j^2 = k^2 = -1
$$

四元数常常写成一个标量和一个向量并列的形式：

$$
q = (r, \vec{v})
$$

这里 $r = a$ 是 $q$ 的标量部，$\vec{v} = bi + cj + dk$ 则是向量部。注意到 $(r, \vec{v})$ 依然是一个向量的形式，因此我们可以轻松地定义四元数的加法和标量乘法。

$$
q_1 + q_2 =(r_1 + r_2, \vec{v}_1 + \vec{v}_2) \qquad \lambda q = (\lambda r, \lambda\vec{v})
$$

四元数的乘积，也被称为 **哈密顿积（Hamilton Product）**，可以通过其定义 $(\ref{def:quaternion})$ 得到。这其实就是利用了我们熟悉的乘法分配律和虚单位的运算规律：

$$
\begin{align*}
	q_1q_2 &= (a_1a_2 - b_1b_2 - c_1c_2 + d_1d_2) \\
	&+ (a_1b_2 + b_1a_2 + c_1d_2 - d_1c_2)i \\
	&+ (a_1c_2 - b_1d_2 + c_1a_2 + d_1b_2)j \\
	&+ (a_1d_2 + b_1a_2 - c_1b_2 + d_1c_2)k \\
	&= (r_1r_2 - \vec{v}_1\vec{v}_2, r_1\vec{v}_2 + r_2\vec{v}_1 + \vec{v}_1 \times \vec{v}_2)
\end{align*}
$$

从上面的向量形式定义就可以看出，四元数的乘积不满足交换律（这该死的叉乘）。

四元数和复数类似，我们可以定义它的 **共轭（Conjugate）**：

$$
q^* = (r, -\vec{v})
$$

不难得到共轭的以下性质：

* $(q^*)^* = q$。
* $(pq)^* = q^*p^*$。
* $qq^* = q^*q$。
* $q^* = -\frac{1}{2}(q + iqi + jqj + kqk)$，这是计算四元数 $q$ 共轭的公式。

定义四元数的 **范数（Norm）** 为其和共轭乘积的平方根：

$$
\lVert q \rVert = \sqrt{qq^*} = \sqrt{a^2 + b^2 + c^2 + d^2}
$$

范数满足下面的性质：

* $\lVert \lambda q \rVert = \lvert\lambda\rvert \lVert q \rVert$。
* $\lVert pq \rVert = \lVert p \rVert \lVert q\rVert$。

四元数的 **逆（Reciprocal）** 可以如下计算：

$$
q^{-1} = \frac{q^*}{\lVert q \rVert^2}
$$

注意到 $qq^{-1} = q^{-1}q = 1$（即左右逆相等），这是一个良好的性质。现在我们就可以定义四元数除法了，当 $q \ne 0$ 时：

$$
\frac{p}{q} = \frac{pq^*}{\lVert q \rVert^2}
$$

**单位四元数（Unit Quaternion）** 是类似于单位向量的概念，它是指范数为 $1$ 的四元数。下一节中我们将介绍它作为旋量的用处。



### 三维空间的旋转

单位四元数用作三维空间的旋转时也被称为 Versor。它的通用形式是：

$$
q = e^{v\theta} = \cos\theta + v\sin\theta
$$

其中 $v$ 是一个标量部为零的四元数，且 $\lVert v \rVert = 1$；$\theta \in [0, \pi]$。注意到当 $v = i$ 时，这就是复平面上的旋转。不过，对于三维空间的旋转，$2\theta$ 才是其旋转的真正角度（注意到 $\theta$ 的范围只有半个圆周）。上面的 $q$ 的几何含义是绕轴 $v$ 旋转 $2\theta$ 度。旋转的结果是为一个向量（写成标量部为零的四元数）取其关于 Versor 的共轭：

$$
v' = qvq^{-1}
$$

这里 $v$ 是向量 $\vec{v}$ “投影”到四元数上得到的，最终的结果 $v'$ 也可以再次映射到向量 $\vec{v}'$ 上。注意到 $q^{-1}$（也就是绕 $v$ 旋转 $-2\theta$ 的四元数）是：

$$
q^{-1} = q^* = e^{-v\theta} = \cos(-\theta) + v\sin(-\theta) = \cos{\theta} - v\sin{\theta}
$$

相当于将 $q$ 的向量部乘以 $-1$，标量部则保持不变。这是一个相当方便的设定。

> 为了简便，我们也会直接将向量代入公式中，比如 $\vec{v}' = q\vec{v}'q^{-1}$，此时注意到：
> $$
> \vec{u}\vec{v} = uv = (0, \vec{u})(0, \vec{v}) = (-\vec{u}\cdot\vec{v}, \vec{u}\times\vec{v})
> $$
{: .prompt-info }

我们可以验证旋转公式的有效性：

$$
\begin{align*}
	\vec{u}' &= (\cos\theta + \vec{v}\sin\theta)\vec{u}(\cos\theta - \vec{v}\sin\theta) \\
	&= \vec{u}\cos^2\theta + (\vec{v}\vec{u} - \vec{u}\vec{v})\cos\theta\sin\theta - \vec{v}\vec{u}\vec{v}\sin^2\theta \\
    &= \vec{u}\cos^2\theta + 2(\vec{v}\times\vec{u})\cos\theta\sin\theta + (\vec{v}\times\vec{u} - \vec{v}\cdot\vec{u})\vec{v}\sin^2\theta \\
    &= \vec{u}\cos^2\theta + 2(\vec{v}\times\vec{u})\cos\theta\sin\theta + ((\vec{v}\times\vec{u})\cdot \vec{v} - (\vec{v}\times\vec{u})\times\vec{v} - (\vec{v}\cdot\vec{u})\vec{v})\sin^2\theta \\
    &= \vec{u}\cos^2\theta + 2(\vec{v}\times\vec{u})\cos\theta\sin\theta + (2(\vec{v}\cdot\vec{u})\vec{v} - \vec{u})\sin^2\theta \\
    &= (\vec{u}_\parallel + \vec{u}_\perp)\cos^2\theta + 2(\vec{v}\times\vec{u})\cos\theta\sin\theta + (\vec{u}_\parallel - \vec{u}_\perp)\sin^2\theta \\
    &= \vec{u}_\parallel(\cos^2\theta + \sin^2\theta) + 2(\vec{v}\times\vec{u})\cos\theta\sin\theta + \vec{u}_\perp(\cos^2\theta - \sin^2\theta) \\
    &= \vec{u}_\parallel + (\vec{v}\times\vec{u})\sin 2\theta + \vec{u}_\perp \cos 2\theta
\end{align*}
$$

最后得到的正是一个绕 $\vec{v}$ 旋转 $2\theta$ 的公式。参考下面的示意图：

![image-20230925085441377](/assets/2023-09-14-unreal-space-and-transform/rotation-formula.png)

这里，我们将 $\vec{u}$ 和旋转轴 $\vec{v}$ 起始位置对齐，然后以 $\vec{v}$ 为 $z$ 轴，$\vec{u}_\perp$ 为 $-x$ 轴构建三维坐标系。此时投影到 $xy$ 平面的有 $\vec{u}_\perp$（也就是图中和 $x$ 轴平行的 $\vec{u}-(\vec{u} \cdot \vec{v})\vec{v}$）和与 $y$ 轴平行的 $\vec{u}\times \vec{v}$（它们*确实*是单位向量，因为 $uv\sin\alpha$ （$\alpha$ 是两个向量的夹角）正对上 $-x$ 方向的单位向量）。此前旋转公式的三个部分分别对应了：

* $\vec{u}_\parallel$：$z$ 轴部分。
* $(\vec{v}\times\vec{u})\sin{2\theta}$：平行于 $y$ 轴的蓝色虚线。
* $\vec{u}_\perp \cos{2\theta}$：平行于 $x$ 轴的蓝色虚线。

这样相加之后我们就得到了旋转后的向量 $\vec{u}'$，符合预期。

虽然 Versor 的旋转公式不算简单，但它可以表示为一个线性变换 $\Gamma$：

$$
\Gamma = \begin{bmatrix}
	1 - 2(v_j^2 + v_k^2) & 2(v_iv_j - v_kv_r) & 2(v_iv_k - v_jv_r) \\
	2(v_iv_j + v_kv_r) & 1 - 2(v_k^2 + v_i^2) & 2(v_jv_k - v_iv_r) \\
	2(v_iv_k - v_jv_r) & 2(v_jv_k + v_iv_r) & 1 - 2(v_i^2 + v_j^2)
\end{bmatrix}
$$

除了在计算过程中，我们不会用到上面矩阵的展开形式，因此不需要记住它。

习惯上，我会把旋转变换记作大写的希腊字母，比如 $\Gamma$。除了线性，它还有一个此前我们提到的，欧拉角缺失的性质。对于方向 $\vec{p}$、$\vec{q}$，假设它们分别经由 $\vec{x}$ 旋转 $r_p$、$r_q$ 得到，则有：

$$
\vec{p} = r_p\vec{x}r_p^{-1} \qquad \vec{q} = r_q\vec{x}r_q^{-1}
$$

当 $r_p \ne 0$ 时，我们一定可以找到四元数 $r_{p\to q} = r_q/r_p$，此时有 $r_q = r_{p\to q}r_p$、$r_{q}^{-1} = r_p^{-1}r_{p\to q}^{-1}$。因此：

$$
\vec{q} = r_q\vec{x}r_q^{-1} = r_{p\to q}r_p\vec{x}r_p^{-1}r_{p\to q}^{-1} = r_{p\to q}\vec{p}r_{p\to q}^{-1}
$$

因此 $r_{p\to q}$ 是将 $\vec{p}$ 变换到 $\vec{q}$ 的唯一旋转。

从四元数的旋转定义出发，我们会发现方向和旋转似乎是“两回事”，因为方向是一个向量，而旋转是一个矩阵；即使都看作四元数，也并不是所有旋转都是方向：它可能带一个非零的标量部。这是因为我们没有将旋转“正规化”。如果将旋转 $\Gamma$ 定义为 $x$ 轴经过 $\Gamma$ 变换后得到的向量，那么方向和旋转的定位就统一了（这个过程类似于将平移定义为从坐标原点开始的向量）。



### 旋转的组合

利用四元数，我们可以



### UE 中的四元数

UE 提供了 `FQuat` 类型来表示一个四元数，并为它实现了许多常用的四元数操作以及和旋量之间的转换。下面对它们进行简单的介绍。

#### 构造

`FQuat` 是一个结构类型，因此可以直接提供其中所有成员变量的值来构建一个四元数：

```upp
FQuat UKismetMathLibrary::MakeQuat(
    float X, 
    float Y, 
    float Z, 
    float W
);
```

和数学中的习惯不同，UE 将四元数的标量和向量部分交换了。因此 `W` 是标量部，其它的组成向量部。 

此外，我们也可以从一个欧拉角得到四元数：

```upp
FQuat UKismetMathLibrary::MakeFromEuler(
    const FVector& Euler
);
```

虽然这里的参数是一个向量，但它本质上就是一个旋量 `Euler = (Roll, Pitch, Yaw)`。我们也可以让旋量转换为四元数（在 Blueprint 中这是一个类型转换节点；C++ 可以直接调用 `FRotator` 上的 `Quaternion` 成员函数）：

```upp
FQuat UKismetMathLibrary::Conv_RotatorToQuaternion(
    FRotator InRot
);
```

其它的构造方式有通过矩阵，或给定轴和旋转角构建（仅 C++）：

```upp
FQuat::FQuat(
    const FMatrix& M
)
FQuat::FQuat(
    FVector Axis,
    float AngleRad
)
```



#### 四元数性质

`FQuat` 上可以调用一些获取其性质的函数。比如获得一个旋转的轴和角度（在 Blueprint 中这些节点可以通过 `RotationAxis`、`Angle`、`Exp`、`Log` 直接得到；C++ 中也可以直接调用 `FQuat` 的成员函数得到）。

```upp
FVector UKismetMathLibrary::Quat_GetRotationAxis(
    const FQuat& Q
);
float UKismetMathLibrary::Quat_GetAngle(
    const FQuat& Q
);
FQuat UKismetMathLibrary::Quat_Exp(
    const FQuat& Q
);
FQuat UKismetMathLibrary::Quat_Log(
    const FQuat& Q
);
```



#### 四元数运算





## 空间变换



## 实例解析

### 两个空间的转换

这是非常常见的例子：把 A 空间的位置（或方向、变换）转换为 B 空间中的对应信息。首先让我们考察其中一种情况，即世界空间和 A 空间的转换。此时只需获取 A 空间的变换 $T$（即相对于世界空间的变换），随即将它（或它的逆）作用于位置/方向上即可：

* 从世界空间转换到 A 空间：使用 $T^{-1}$。
* 从 A 空间转换到世界空间：使用 $T$。

下面是一个例子：

<iframe src="https://blueprintue.com/render/xxxia16d/" width="800" height="550" scrolling="no" allowfullscreen></iframe>

这里出现的 `SM Actor` 是一个 Static Mesh Actor。我们获取其中的 Static Mesh Component 之后调用 `GetWorldTransform` 得到其世界空间的变换。`Transform Location` 和 `Inverse Transform Location` 分别是用这个变换和这个变换的逆作用于一个位置。对于旋转也是同理。

我们可以由此推出任意两个空间的变换方式（如 A 到 B）：只需要先将 A 空间转换为世界空间，再将其转换为 B 空间即可。请看下面的 Blueprint 代码（部分节点没有显示，请用按住右键拖拽背景）：

<iframe src="https://blueprintue.com/render/6iypbzl9/" width="800" height="600" scrolling="yes" allowfullscreen></iframe>



### 让特定方向朝向特定坐标

在同一空间中，我们已知一个方向和位置，现在想让这个方向指向这个位置，求它需要的旋转；比如让角色的头部始终看向某个点。由于两者处于同一空间，一个明显的方案是求出位置的旋量后让它和当前方向做差。然而，`FRotator` 并不支持减法运算（这实际上很合理，因为欧拉角确实不适用于减法）。好在我们可以利用四元数得到这个旋转。

<iframe src="https://blueprintue.com/render/rkunl_d1/" width="800" height="500" scrolling="no" allowfullscreen></iframe>

这里显示的 `Quat Rotator` 是从四元数到旋量的类型转换节点。最终我们得到的是世界空间中的旋量，因此还需要通过 `Inverse Transform` 将其转换为当前空间的旋量。

即使不用四元数，这个问题也可以通过朴素的向量计算得到。为了让 Arrow 瞄准 Target 方向，我们只需要计算两者的夹角和旋转轴（两者都可以通过叉乘得到）；随后再通过轴和角的信息构建旋量。

<iframe src="https://blueprintue.com/render/ma1sthpf/" width="800" height="450" scrolling="no" allowfullscreen></iframe>

同样地，我们依然需要用 `Inverse Transform` 将这个结果转换为当前空间的旋量。

实际上，UE 提供了一个帮助函数，`Look At Function`，我们可以直接轻松地得到目标的方向。

<iframe src="https://blueprintue.com/render/_qfcntok/" width="800" height="400" scrolling="no" allowfullscreen></iframe>

### 让特定方向平行于某个方向

在同一空间中，我们已知一个方向，和一个想要平行对齐的方向，求对齐需要的旋转；比如让角色的手部始终和地面平行，此时可以让和手部垂直的某个轴（可能是 $y$）与地面的法向量平行。
