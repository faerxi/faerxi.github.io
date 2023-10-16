---
layout: post
title: 数学物理方法（一）向量分析
category: [math]
tag: [数学, 数学物理方法]
math: true
author: default
---

本篇让我们从零开始构建向量微积分

## 向量

本章中我们会先从向量的定义开始。前几节中的内容非常基础，可以作为复习。

### 定义和基本性质

**向量（Vector）** 是一系列数的排列，可以写成 $\mathbf{v} = (v_1, \dots, v_n)$ 的形式。包含 $n$ 个数的向量也被称为 $n$ 维向量。它满足下面的基本性质：

* 向量之间可以进行加法运算：$\mathbf{u} + \mathbf{v} = (u_1+v_1, \dots, u_n+v_n)$。
* 向量加法满足交换律和结合律：$\mathbf{u} + \mathbf{v} = \mathbf{v} + \mathbf{u}$ 且 $(\mathbf{u} + \mathbf{v}) + \mathbf{w} = \mathbf{u} + (\mathbf{v} + \mathbf{w})$。
* 向量可以进行标量乘法：$\lambda\mathbf{u} = (\lambda u_1, \dots, \lambda u_n)$；由于我们使用的数总是实数或复数，它们总满足交换律，因此 $\lambda\mathbf{u} = \mathbf{u}\lambda$。

向量减法可以定义为：
$$
\mathbf{u} - \mathbf{v} = \mathbf{u} + (-1)\mathbf{v}
$$
向量的 **范数（Norm）** 是一个满足下面要求的数 $n(\mathbf{v})$：

* $n(\mathbf{v}) \ge 0$ 且 $n(\mathbf{v}) = 0$ 当且仅当 $\mathbf{v} = \mathbf{0}$。
* 对任意标量 $\lambda$ 都有 $n(\lambda\mathbf{v}) = \vert\lambda\vert n(\mathbf{v})$。
* 三角不等式：$n(\mathbf{u} + \mathbf{v}) \le n(\mathbf{u}) + n(\mathbf{v})$。

其中有一系列常用的称为 $L^p$ 范数：
$$
L^p(\mathbf{v}) = \left(v_1^p + \dots + v_n^p\right)^{1/p}
$$


向量的大小是它的 $L^2$ 范数：
$$
|\mathbf{v}| = \sqrt{v_1^2 + \dots + v_n^2}
$$
这在几何上的含义就是空间中两个点的距离。

### 三维向量

在向量分析中，我们只对三维向量感兴趣；此后我们会将其简称为向量。首先，三维空间中有三个很重要的单位向量：
$$
\hat{x} = (1, 0, 0) \qquad \hat{y} = (0, 1, 0) \qquad \hat{z} = (0, 0, 1)
$$
利用这个记号，所有向量都可以写成：
$$
\mathbf{v} = v_x\hat{x} + v_y\hat{y} + v_z\hat{z}
$$


## 向量的微元

在物理中，我们将三元函数 $F(x, y, z)$ 称为 **域（Field）**，且分为 **向量场（Vector Field）** 和 **标量场（Scalar Field）**。

### Del 算符

定义 Del 算符：
$$
\nabla = \hat{x}\frac{\partial}{\partial x} + \hat{y}\frac{\partial}{\partial y} + \hat{z}\frac{\partial}{\partial z}
$$
将它和向量或标量结合会得到不同的物理量。首先是标量场的 **梯度（Gradient）**：
$$
\nabla \varphi = \frac{\partial \varphi_x}{\partial x}\hat{x} + \frac{\partial \varphi_y}{\partial y}\hat{y} + \frac{\partial \varphi_z}{\partial z}\hat{z}
$$
梯度描述了标量场上“下降速度最快”的方向分布。物理中力场可以定义为势场梯度的负值：
$$
\mathbf{F} = -\nabla V
$$
利用 Del 算符和点乘，我们可以得到矢量场的 **散度（Divergence）**：
$$
\nabla \cdot \mathbf{A} = \frac{\partial A_x}{\partial x} + \frac{\partial A_y}{\partial y} + \frac{\partial A_z}{\partial z}
$$
散度描述了矢量场上“发散程度”的分布。物理中磁场的散度是零，因为它是一个无源场：
$$
\nabla \cdot \mathbf{B} = 0
$$
利用 Del 算符和叉乘，我们可以得到是两场的 **旋度（Curl）**：
$$
\nabla \times \mathbf{A} = \begin{vmatrix}
	\hat{x} & \hat{y} & \hat{z} \\
	\frac{\partial}{\partial x} & \frac{\partial}{\partial y} & \frac{\partial}{\partial z} \\
	A_x & A_y & A_z
\end{vmatrix}
$$
旋度描述了矢量场上“旋转程度”的分布。物理中磁矢势的旋度是磁感应强度：
$$
\nabla \times \mathbf{A} = \mathbf{B}
$$
