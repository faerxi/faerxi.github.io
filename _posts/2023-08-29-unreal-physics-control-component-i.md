---
layout: post
title: UE 物理控制组件·原理篇
category: [coding, physics]
tag: [UE, C++, 物理引擎, 物理控制]
author: programming
math: true
publish: false
---

UE 5.1 新增了一个 **物理控制组件（Physics Control Component，以下简称 PCC）**，相比此前已有的物理动画组件，新组件在物理方面的控制更加精细，因此可以达到更好的效果。本篇基于 5.3 版本的源码对 PCC 进行了简单的分析。

## 版本说明

### 23/09/13 第一版

包含比较完整的弹簧模型分析，和常用公共接口的分析。物理控制组件 PImpl 类的实现目前分析不够深入，动画部分还没有搞清楚。

### 第二版

* 更新了方程的图解，只展示 $t \ge 0$ 的部分；优化了参数的设置，现在使用的系数是起始点和初速度。
* 修改了关于速度的控制目标错误的阐述。

## 基本概念

物理控制组件的核心是调节 Mesh 运动目标和特征的*控制*和调和物理动画的*身体修饰*。其中控制分为控制数据、控制目标和控制设置，分别调整了弹簧参数、物理控制的结束约束和其它设置。在详细介绍控制之前，让我们首先了解 PCC 中存储的一些参数：

* 子组件和骨骼：控制的主体。PCC 会控制这个组件（若是 Skeletal Mesh 则会控制指定的骨骼）。
* 父组件和骨骼：控制的目标。PCC 会将子组件“拉”到这里。若为空则使用世界空间。
* 控制数据、控制目标、控制乘数和控制设定：这些会在下面的小节中介绍。

### 控制数据

PCC 中将控制数据分成平动和转动两个类型，对两种弹簧分别设置了*强度（Strength）*、*阻尼比（Damping Ratio）*、额外阻尼和最大作用力。因此在控制数据中我们有 8 个参数可以设置。

* 强度：对应了弹簧的强度，越大则越快将 Mesh 拉向目标。
* 阻尼比：对应了弹簧的阻尼，这里归一化使得取 1 时就是临界阻尼，Mesh 会最高效地在目标处达到平衡。大于 1 则让 Mesh 受到过多阻力，小于 1 则会让 Mesh 在平衡点附近震荡。
* 额外阻尼：阻尼的修正（会加在计算好的阻尼，而非阻尼比上），会让 Mesh 在强度为零的时候也会受到阻尼影响。
* 最大作用力：限制弹簧对 Mesh 的作用力，超过这个力时会自动压回这个值。

对于转动来说，上面所有对距离和力的描述都要调整为角度和力矩。

下面我们将从数学角度讨论控制数据中参数的含义，可以酌情跳过。

### 弹簧模型

物理学中弹簧是一个非常常见的模型，其中包含了位移相关的弹力项 $kx$、和速度相关的阻力项 $\lambda \dot{x}$、振子的惯性项 $m\ddot{x}$ 和外部驱力 $F(t)$。综合起来可以参考下面的公式：

$$
m\ddot{x} + \lambda\dot{x} + kx = F(t)
$$

为了方便讨论，我将三维的情形简化为一维。上面出现的 $m$ 是振子的质量，$\lambda$ 称为弹簧阻尼，$k$ 称为弹簧强度。可以看到，Mesh 的受力大小 $m\ddot{x}$ 和偏移大小 $x$ 成正比，且方向相反，这反映了弹簧将物体拉向平衡点的特征；此外受力大小同样和速度大小成正比，且方向相反，这反映了阻尼阻碍物体运动的特征。

让我们首先考虑 $F(t) = 0$ 时上面方程的解。首先让我们将其变形为等价的形式：

$$
\ddot{x} + 2\zeta\omega_0\dot{x} + \omega_0^2x = 0
$$

其中

$$
\omega_0 = \sqrt{\frac{k}{m}} \qquad \zeta = \frac{\lambda}{2\sqrt{mk}}
$$

这两个常数都有特殊意义，我们在下面会提到。考虑这个变形的特征方程：

$$
r^2 + 2\zeta\omega_0r + \omega_0^2 = 0 \implies (r + \zeta\omega_0)^2 = (\zeta^2 - 1)\omega_0^2
$$

可以看到当 $\zeta^2$ 取大于，小于或等于 $1$ 时，方程的性质是不同的。 下面让我们分别讨论。

#### $\zeta > 1$ 时

此时很容易得到两个特征方程的解：

$$
r = (-\zeta\pm\sqrt{\zeta^2 - 1})\omega_0
$$

对应的原方程解就是

$$
x = Ae^{(-\zeta + \sqrt{\zeta^2 - 1})\omega_0t} + Be^{(-\zeta - \sqrt{\zeta^2 - 1})\omega_0t}
$$

可以看到这是一个随着 $t$ 增大而缓慢衰减的指数曲线。参考下面的图：

<iframe src="https://www.desmos.com/geometry/8tlhuyflbf" width="800" height="500"></iframe>

从这个图中其实可以看到 $\zeta$ 越靠近 $1$，$x$ 收敛到 $0$ 的速度就越快，反之则不容易。这种情况被称为 **过阻尼（Overdamped）**。

#### $\zeta = 1$ 时

此时的通解是

$$
x = Ae^{-\omega_0t} + Bte^{-\omega_0t}
$$

参考下面的图：

<iframe src="https://www.desmos.com/geometry/vjzknbfi7f" width="800" height="370"></iframe>

这种情况被称为 **临界阻尼（Critical Damped）**。

#### $\zeta < 1$ 时

此时我们可以得到一个正弦曲线：
$$
x = e^{-\zeta\omega_0t}\left[A\cos{(\sqrt{1-\zeta^2}\omega_0t)} + B\sin{(\sqrt{1-\zeta^2}\omega_0t)}\right]
$$
这是一个随 $t$ 增大逐渐减小震荡幅度的波。参考下面的的图（这里我将 $\zeta$ 的值调的很小，是因为 $\zeta \ll \omega_0$ 时才有明显的震荡效果；当 $\zeta$ 变大后函数会越来越近似于临界阻尼的情况）：

<iframe src="https://www.desmos.com/geometry/mlhudbvqss" width="800" height="500"></iframe>

$\zeta$ 越小，振荡的效果就越明显，因为方程中的 $\omega_0^2x$ 项成为主导；特别地，当 $\zeta = 0$ 时，振子会不停止地振动，频率为 $\omega_0$，因此它被称为弹簧的 **固有频率（Natural Frequency）**。而 $\zeta$ 显然和弹簧阻力效果有关，它被称为 **阻尼比（Damping Ratio）**。

上面出现的系数 $A$、$B$ 所取的初值是基于下面的预设：初始位置为 $x_0 = 1$，且初速度 $v_0 = 0$。因此三种情况分别对应的取值是：

* $\zeta > 1$：$A = B = 0.5$。
* $\zeta = 1$：$A = 1$，$B = \omega_0$。
* $\zeta < 1$：$A = 1$，$B = \frac{\zeta}{\sqrt{1 - \zeta^2}}$。

每个图左侧参数设置位置往下拉可以看到在任意的 $x_0$ 和 $v_0$ 时 $A$、$B$ 应取的值。

如果外力 $F(t)$ 存在，则取决于其形式：

* 常数 $F(t) = C$，则振子会在一个非零的地方 $x_0$ 平衡，且此处有 $kx_0 + C = 0$。
* 周期函数 $F(t) = f_d\cos{\omega_dt}$，则振子会逐步趋于这个频率振动，不过效果会受到系数 $\sqrt{\omega_0^2 - \omega_d^2}$ 影响。

物理控制组件中没有提供调控外驱力的参数，因此我们就不深入了。

### 转动

上一节我们讨论的都是 **平动（Translation）** 中的情形。实际上弹簧模型对于转动依然适用，只不过此时所有概念需要照搬到转动的语境。我们依然可以写出微分方程：

$$
I\ddot{\theta} + \mu\dot{\theta} + \kappa\theta = \tau(t)
$$

这里的 $I$ 是振子的 **转动惯量（Moment of Inertia）**，$\mu$ 是弹簧的转动阻尼，$\kappa$ 是弹簧的转动强度。从这些参数中我们依然可以提炼出固有频率 $\omega_0$ 和阻尼比 $\zeta$：

$$
\omega_0 = \sqrt{\frac{\kappa}{I}} \qquad \zeta = \frac{\mu}{2\sqrt{I\kappa}}
$$

它们的量纲与平动情况下是相同的。这就体现出我们前面将方程变为只含有这两个参数的形式的好处了，此前的结论在转动情况下依然成立。

### 驱动器和控制目标

UE 中的物理作用有时会通过 **驱动器（Driver）** 完成。这是一个使用弹簧模型的物理系统，它会持续向物体施加大小为 $F = kx + \lambda\dot{x}$ 的力（其中 $x$ 是和目标点的距离，$\dot{x}$ 是物体当前的速度；$k$ 和 $\lambda$ 则是力度和阻尼参数），或大小为 $\tau = \kappa\theta + \mu\dot{\theta}$ 的力矩（其中 $\theta$ 是和目标角度的偏移，$\dot{\theta}$ 是物体当前的角速度；$\kappa$ 和 $\mu$ 则是力度和阻尼参数），最终的效果便是将物体拉到平衡点，并和目标方向对齐。

> 需要添加：PD 控制器（Proportional-Derivative Controller）

在 PCC 的控制数据中，强度对应的便是一个和 $k$ 成正比的值（实际是 $\frac{\omega_0}{2\pi}$），而阻尼比对应的是一个和 $\lambda$ 与强度同时相关的值（实际是 $\frac{\lambda}{2m\omega_0}$）。如果设强度为 $f$，阻尼比沿用 $\zeta$，则方程中的参数实际应该是：

$$
k = 4\pi^2mf^2 \qquad \lambda = 4\pi mf\zeta + \lambda_0
$$

这里我按照 PCC 中的设置，对阻尼项加上了此前提到的额外阻尼 $\lambda_0$。此外 PCC 中完全忽略了质量的作用，因此可以将上面的 $m$ 省略，就得到（量纲不正确但可以视为等式右侧默认乘了 `1kg`）：

$$
\label{pcc-spring-formula}
k = 4\pi^2f^2 \qquad \lambda = 4\pi f\zeta + \lambda_0
$$

这里就可以看到额外阻尼的作用了：当我们将强度设为 0 时（比如想要撤销拉力），物理控制的阻尼自动也会变成 0，导致物体不受阻力地移动，这可能和预期不符。因此添加一个额外阻尼可以让物体更自然地停止运动。

最大力会确保 $F$ 不会超过某个值，额外阻尼则是一个不清楚原理的参数（个人觉得可以作为 $k$ 的修正项出现，但在 PCC 的注释中它似乎应该和 $\lambda$ 相关）。转动的情况同理。

目标位置和目标方向可以在 PCC 的控制目标中设置，其中包括：

* 位置：设置物体平动的平衡点。
* 速度：设置物体在经过平衡点时的速度。
* 方向：设置物体转动的平衡点。
* 角速度：设置物体在经过转动平衡点时的速度。PCC 中采用的是转/秒而非标准的弧度/秒。
* 是否将控制点和目标对齐：**控制点（Control Point）** 的概念源于物体并非大小为零的质点，因此可以将力施加在物体原点以外的地方，这个（组件空间的）偏移就是控制点。如果这里设置为是，则会将物体的控制点和目标对齐；否则会让目标与受力物体的原点对齐（这实际上也同时限制了物体达到目标点时的 transform）。



### 其它控制设置

**控制乘数（Control Multiplier）** 是用来为控制数据设置比例的工具，其包含：

* 平动强度在三方向上各自有一个乘数。
* 平动阻尼比在三方向上各自有一个乘数。
* 平动额外阻尼在三方向上各自有一个乘数。
* 最大力在三方向上各自有一个乘数。
* 转动强度只有一个乘数。
* 转动阻尼比只有一个乘数（UE 5.3 添加）。
* 转动额外阻尼只有一个乘数。
* 最大力矩只有一个乘数。

这里值得一提的是转动相关的都只有一个乘数，这是因为旋转不能分解，因此不能像平动那样三方向的分量同时进行。平动的弹簧可以直接将实际的强度 $k$ 用于三个方向，即设置三个弹簧同时作用于物体上，这样每个弹簧当然可以设置分别的乘数。对于旋转，它甚至不满足交换律（考虑 $(1, 0, 0)$ 沿 $y$ 轴旋转 90 度后沿着 $x$ 轴旋转 90 度到达 $(0, 1, 0)$，交换旋转顺序后则会到达 $(0, 0, -1)$）。因此它不存在类似于位置这样的分解（准确来说是不存在固定轴分解，因为可以利用欧拉角分解一个转动）。在源码中，设置方向目标也是首先寻找一个最短弧，然后根据这个弧的方向决定唯一的旋转弹簧（也即 `SLERP` 的控制方式）。

物理约束中的角驱动器可以选择使用 `TwistAndSwing` 来控制，此时 $x$ 方向的旋转被称为 Twist，$y$ 方向的旋转是 Swing2，$z$ 方向的旋转是 Swing1。

**控制设定（Control Setting）** 是其它的一些参数，包含：

* 控制点（`ControlPoint`）：即在此前介绍过的，力会被施加在这个点上。
* 是否使用动画（`bUseSkeletalAnimation`）：若是，则会将力施加在骨骼动画上。
* 骨骼动画速度乘数（`SkeletalAnimationVelocityMultiplier`）：骨骼动画的速度使用比例。
* 是否开启碰撞检测（`bEnableCollision`）：若不开启则忽略物体和目标组件的碰撞。
* 是否开启自动关闭（`bAutoDisable`）：若开启则每个 tick 都会首先关闭物理控制。





## 物理控制接口

### 创建物理控制

1. `CreateNamedControl` 创建一个自定义名称的物理控制。我们后续可以通过这个名字来更新这个物理控制的参数。如果名称有重复则不创建并返回 `false`。

   ```upp
   bool CreateNamedControl(
       FName Name,
       UMeshComponent* ParentMeshComponent,
       FName ParentBoneName,
       UMeshComponent* ChildMeshComponent,
       const FName ChildBoneName,
       const FPhysicsControlData ControlData,
       const FPhysicsControlTarget ControlTarget,
       const FPhysicsControlSettings ContorlSettings,
       FName Set,
       const bool bEnabled = true
   );
   ```

   这里子组件不能为空。对每个组件，如果其是一个 Skeletal Mesh，则将保存用于后续骨骼缓存（调用 `FPhysicsControlComponentImpl::AddSkeletalMeshReferenceForCaching`）。随后，将参数中物理控制相关的项拿来初始化一个 `FPhysicsControlRecord` 对象（控制记录），其用于存储物理控制的运行状态（包括父子组件和骨骼、控制数据、控制目标、控制设定）；后续通过控制的名称可以拿到这个控制记录。参数中的 `Set` 用来将这个控制纳入特定分组，之后我们可以为一个具名的控制组中所有物理控制一起设定参数（无论如何，每个物理控制都会在 `"All"` 组中）。`bEnabled` 会让物理控制立刻启用；如果创建控制时没有启用，也可以后续调用 `SetControlEnabled` 来手动开启。

   另有 `CreateControl` 和这个函数类似，不过它不需要输入指定的控制名，而是会自动生成一个，并返回这个名字。

2. `CreateControlsFromSkeletalMeshBelow` 为某个骨骼（可选择是否包含）的所有子骨骼创建物理控制，并返回所有创建的物理控制名称。注意这里一律使用父空间。要求物理资产非空。内部调用了 `CreateControl`。不会对 `ControlTarget` 进行初始化，因此需要后面设置目标。

   ```upp
   TArray<FName> CreateControlsFromSkeletalMeshBelow(
       USkeletalMeshComponent* SkeletalMeshComponent,
       const FName BoneName,
       const bool bIncludeSelf,
       const EPhysicsControlType ControlType,
       const FPhysicsControlData ControlData,
       const FPhysicsControlSettings ControlSettings,
       const FName Set,
       const bool bEnabled = true
   );
   ```

   注意到这里并没有指定控制目标，毕竟同时控制多个组件时不太可能设置同一个目标。`ControlType` 用于设置物理控制是否发生在父空间还是世界空间。参数 `SkeletalMeshComponent` 既是子组件又是父组件（当 `ControlType` 是父空间时），否则相当于仅指定了子组件；`BoneName` 是启用控制的最祖先骨骼；`bIncludeSelf` 决定 `BoneName` 是否也被控制。

3. `CreateControlsFromSkeletalMeshAndConstraintProfileBelow` 和 `CreateControlsFromSkeletalMeshBelow` 类似，但从物理资产的约束预设中提取控制数据来初始化物理控制。

   ```upp
   TArray<FName> CreateControlsFromSkeletalMeshAndConstraintProfileBelow(
       USkeletalMeshComponent* SkeletalMeshComponent,
       const FName BoneName,
       const bool bIncludeSelf,
       const FName ConstraintProfile,
       const FName Set,
       const bool bEnabled = true
   );
   ```

   和 `CreateControlsFromSkeletalMeshBelow` 对比可以看到，这个函数不需要给定控制数据和控制设定，这是因为：

   * 控制数据会直接用物理资产的约束预设，通过 `ConstraintProfile` 指定。会从子骨骼和父骨骼的关节约束中读取强度和阻尼等信息。实际效果可能会和设置不同，因为约束中的角驱动器可能采用了 `TwistAndSwing` 而非  `SLERP` 来控制骨骼（前者会将旋转分解为 twist 和 swing 两个“方向”，而后者使用 **球线性插值（Sphere Linear Interpolation, SLERP）** 将骨骼以最短距离控制到目标）。
   * 控制设定可以后续再设置；注意这个函数中会默认将骨骼动画的速度乘数设为 0，因为关节约束中并不会将动画速度作为目标。

4. `CreateControlsFromSkeletalMesh` 为指定的一个 Skeletal Mesh 中多个骨骼设置物理控制。要求物理资产非空。

   ```upp
   TArray<FName> CreateControlsFromSkeletalMesh(
       USkeletalMeshComponent* SkeletalMeshComponent,
       const TArray<FName>& BoneNames,
       const EPhysicsControlType ControlType,
       const FPhysicsControlData ControlData,
       const FPhysicsControlSettings ControlSettings,
       const FName Set,
       const bool bEnabled = true
   );
   ```

   本质上是比 `CreateControlsFromSkeletalMeshBelow` 用处更加宽泛的创建方式。

5. `CreateControlsFromSkeletalMeshAndConstraintProfile` 和 `CreateControlsFromSkeletalMesh` 类似，但从物理资产的约束预设中提取控制数据来初始化物理控制。

   ```upp
   TArray<FName> CreateControlsFromSkeletalMeshAndConstraintProfile(
       USkeletalMeshComponent* SkeletalMeshComponent,
       const TArray<FName>& BoneNames,
       const FName ConstraintProfile,
       const FName Set,
       const bool bEnabled = true
   );
   ```

   与 `CreateControlsFromSkeletalMesh` 的关系类似于 `CreateContorlsFromSkeletalMeshAndConstraintProfileBelow` 之于 `CreateControlsFromSkeletalMeshBelow`。

6. `CreateControlsFromLimbBones` 为一段肢体设置物理控制。肢体是以映射表的形式传入的，函数中会为其中每一个骨骼设置相同的物理控制。

   ```upp
   TMap<FName, FPhysicsControlNames> CreateControlsFromLimbBones(
       FPhysicsControlNames& AllControls,
       const TMap<FName, FPhysicsControlLimbBones>& LimbBones,
       const EPhysicsControlType ControlType,
       const FPhysicsControlData ControlData,
       const FPhysicsControlSettings ControlSettings,
       const bool bEnabled = true
   );
   ```

   这里出现的 `AllControls` 实际上是 `TArray<FName>` 的封装，其中包含了每个控制的名字。`LimbBones` 将这些名字映射到 `FPhysicsControlLimbBones` 上，其中存储了肢体所在的 Skeletal Mesh 和其中的所有骨骼名字。

7. `CreateControlsFromLimbBonesAndConstraintProfile` 从物理资产的约束预设中提取控制数据来初始化控制数据。和 `CreateControlsFromLimbBones` 的关系和此前遇到的两组函数类似。

   ```upp
   TMap<FName, FPhysicsControlName> CreateControlsFromLimbBonesAndConstraintProfile(
       FPhysicsControlNames& AllControls,
       const TMap<FName, FPhysicsControlLimbBones>& LimbBones,
       const FName ConstraintProfile,
       const bool bEnabled = true
   );
   ```

### 设置物理控制

1. `SetControlData` 为物理控制设置数据。若给定的控制名字不存在则返回 `false`（“看起来没必要返回 `bool`”的函数，都是为了再给定控制名字不存在时返回 `false`，之后不再赘述了）。

   ```upp
   bool SetControlData(
       const FName Name,
       const FPhysicsControlData ControlData,
       const bool bEnableControl = true
   );
   ```

   `bEnableControl` 用来顺便开启物理控制。这个函数有两个变体，`SetControlDatas` 和 `SetControlDatasInSet`，里面分别会接受一个名字数组和组名，用来设置给定的物理控制。

2. `SetControlMultiplier` 设置物理控制数据的乘数。

   ```upp
   bool SetControlMultiplier(
       const FName Name,
       const FPhysicsControlMultiplier ControlMultiplier,
       const bool bEnableControl = true
   );
   ```

   类似地，也有 `SetControlMultipliers` 和 `SetControlMultipliersInSet` 两个变种。

3. `SetControlLinearData` 仅设置物理控制中和平动相关的数据。

   ```upp
   bool SetControlLinearData(
       const FName Name,
       const float Strength = 1.0f,
       const float DampingRatio = 1.0f,
       const float ExtraDamping = 0.0f,
       const float MaxForce = 0.0f,
       const bool bEnableControl = true
   );
   ```

4. `SetControlAngularData` 仅设置物理控制中和转动相关的数据。

   ```upp
   bool SetControlAngularData(
       const FName Name,
       const float Strength = 1.0f,
       const float DampingRatio = 1.0f,
       const float ExtraDamping = 0.0f,
       const float MaxTorque = 0.0f,
       const bool bEnableControl = true
   );
   ```

5. `SetControlTarget` 设置物理控制的目标。

   ```upp
   bool SetControlTarget(
       const FName,
       const FPhysicsControlTarget ControlTarget,
       const bool bEnableControl = true
   );
   ```

   有 `SetControlTargets` 和 `SetControlTargetsInSet` 两个变种。

6. `SetControlTargetPositionAndOrientation` 仅设置物理控制中位置和方向的目标。

   ```upp
   bool SetControlTargetPositionAndOrientation(
       const FName Name,
       const FVector Position,
       const FRotator Orientation,
       const float VelocityDeltaTime,
       const bool bEnableControl = true,
       const bool bApplyControlPointToTarget = false
   );
   ```

   参数中的 `VelocityDeltaTime` 若不为零，则会用当前目标位置计算目标速度（目标位置相对当前位置的偏移与 `VelocityDeltaTime` 的比值），因此这个参数的作用类似于时间偏移 `DeltaTime` 的作用。`bApplyControlPointToTarget` 用于设置物理控制是否将控制点和目标对齐（回忆这也是物理控制目标的一个参数）。这实际上就是将控制目标中的速度项剥除后将其余项展开为函数参数。如果它是零，则让目标速度也为零。

   有 `SetControlTargetPositionsAndOrientations` 和 `SetControlTargetPositionsAndOrientationsInSet` 两个变种。此外也有 `SetControlTargetPosition` 和 `SetControlTargetOrientation` 以及带数组或组名参数的简化版及其变种。

7. `SetControlTargetPositionsAndOrientationsFromArray` 为多个物理控制分别设置目标位置和方向。若给定的控制数量和位置数量（或方向数量）不相同则返回 `false`。

   ```upp
   bool SetControlTargetPositionsFromArray(
       const TArray<FName>& Names,
       const TArray<FVector>& Positions,
       const TArray<FRotator>& Orientations,
       const float VelocityDeltaTime,
       const bool bEnableControl = true,
       const bool bApplyControlPointToTarget = false
   );
   ```

   有 `SetControlTargetPositionsFromArray` 和 `SetControlTargetOrientationsFromArray` 两个简化的变种。

8. `SetControlTargetPoses` 仅控制物理控制中位置和方向的目标；和 `SetControlTargetPositionAndOrientation` 的区别在于，这个函数会将子组件的 transform 向父组件对齐，且会自动将 `bApplyControlPointToTarget` 设置为 `true`。

   ```upp 
   bool SetControlTargetPoses(
       const FName Name,
       const FVector ParentPosition,
       const FRotator ParentOrientation,
       const FVector ChildPosition,
       const FRotator ChildOrientation,
       const float VelocityDeltaTime,
       const bool bEnableControl = true
   );
   ```

9. `SetControlUseSkeletalAnimation` 设置是否和动画混合。

   ```upp
   bool SetControlUseSkeletalAnimation(
       const FName Name,
       const bool bUseSkeletalAnimation = true,
       const float SkeletalAnimationVelocityMultiplier = 1.0f
   );
   ```

   有 `SetControlsUseSkeletalAnimation` 和 `SetControlsInSetUseSkeletalAnimation` 两个变种。

10. `SetControlAutoDisable` 设置是否每个 tick 自动关闭物理控制。

    ```upp
    bool SetControlAutoDisable(
        const FName Name,
        const bool bAutoDisable
    );
    ```

    有 `SetControlsAutoDisable` 和 `SetControlsInSetAutoDisable` 两个变种。


11. `SetControlDisableCollision` 设置是否关闭子组件和父组件的碰撞。

    ```upp
    bool SetControlDisableCollision(
        const FName Name,
        const bool bDisableCollision
    );
    ```

    有 `SetControlsDisableCollision` 和 `SetControlsInSetDisableCollision` 两个变种。

12. `SetControlEnabled` 开启或关闭物理控制。

    ```upp
    bool SetControlEnabled(
        const FName Name, 
        const bool bEnable = true
    );
    ```

    有 `SetControlsEnabled` 和 `SetControlsInSetEnabled` 两个变种。

13. `AddControlToSet` 将指定的控制（名字）加到一个组中。

    ```upp
    void AddControlToSet(
        FPhysicsControlNames& NewSet,
        const FName Control,
        const FName Set
    );
    ```

    如果组不存在则会创建一个组。

    有变种 `AddControlsToSet`，会将上面的 `Control` 参数变为一个数组。

有趣的是一些函数并没有传入数组和组名的变种。



### 获取控制信息

1. `GetControlData` 获取物理控制数据。

   ```upp
   bool GetControlData(
       const FName Name, 
       FPhysicsControlData& ControlData
   ) const;
   ```

2. `GetControlTarget` 获取物理控制目标。

   ```upp
   bool GetControlTarget(
       const FName Name, 
       FPhysicsControlTarget& ControlTarget
   ) const;
   ```

3. `GetControlMultiplier` 获取物理控制乘数。

   ```upp
   bool GetControlMultiplier(
       const FName Name, 
       FPhysicsControlMultiplier& ControlMultiplier
   ) const;
   ```

4. `GetControlAutoDisable` 获取物理控制是否会自动停止。

   ```upp
   bool GetControlAutoDisable(
       const FName Name
   ) const;
   ```

5. `GetControlEnabled` 获取物理控制是否已经开启。

   ```upp
   bool GetControlEnabled(
       const FName Name
   ) const;
   ```

6. `GetAllControlNames` 获取所有控制的名字。

   ```upp
   const TArray<FName>& GetAllControlNames(
   ) const;
   ```

7. `GetAllControlNamesInSet` 获取特定组中所有控制的名字。

   ```upp
   const TArray<FName>& GetAllControlNmaesInSet(
       const FName Set
   );
   ```

8. `GetSetsContainingControl` 获取所有拥有给定控制的组。

   ```upp
   TArray<FName> GetSetsContainingControl(
       const FName Control
   );
   ```

   注意这里返回的是一个 `TArray<FName>` 而非引用。



### 骨骼缓存

这里的接口都和目标信息的缓存相关。

1. `GetCachedBoneTransform` 获取给定骨骼世界空间下的 transform。

   ```upp
   TArray<FTransform> GetCachedBoneTransforms(
       const USkeletalMeshComponent* SkeletalMeshComponent,
       const TArray<FName>&          BoneNames
   );
   ```

   如果骨骼不存在，则会将返回单位变换。缓存会在 PCC 创建伊始就设置，因此如果在 tick 中调用这个函数或许已经过时了；不过如果手动更新组件的话，这个函数或许有用。这个结论对下面的所有 getter 有效。

   有两个变种 `GetCachedBonePosition` 和 `GetCachedBoneOrientation`，和用于获取多个骨骼 transform 等信息的 `GetCachedBoneTransforms`、`GetCachedBonePositions` 和 `GetCachedBoneOrientations`。

2. `GetCachedBoneVelocity` 获取给定骨骼世界空间下的速度。

   ```upp
   TVector GetCachedBoneVelocity(
       const USkeletalMeshComponent* SkeletalMeshComponent,
       const FName BoneName
   );
   ```

   如果骨骼不存在，则会返回零向量。有变种 `GetCachedBoneVelocities`。

3. `GetCachedBoneAngularVelocity` 获取给定骨骼世界空间下的角速度。

   ```upp
   TVector GetCachedBoneAngularVelocity(
       const USkeletalMeshComponent* SkeletalMeshComponent,
       const FName BoneName
   );
   ```

   如果骨骼不存在，则会返回零向量。有变种 `GetCachedBoneAngularVelocities`。

4. `SetCachedBoneData` 设置骨骼的缓存信息。

   ```upp
   bool SetCachedBoneData(
       const USkeletalMeshComponent* SkeletalMeshComponent,
       const FName BoneName, 
       const FTransform& TM,
       const FVector Velocity,
       const FVector AngularVelocity);
   ```

   可以看到这里一次性设置了骨骼的所有信息。

5. `ResetBodyModifierToCachedBoneTransform` 通知指定的身体修饰对应的物体，让其将 transform、速度等信息设为骨骼缓存中的信息。仅对 Skeletal Mesh 有效。

   ```upp
   bool ResetBodyModifierToCachedBoneTransform(
       const FName Name,
       const EResetToCachedTargetBehavior Behavior = EResetToCachedTargetBehavior::ResetImmediately);
   ```

   这里的 `EResetToCachedTargetBehavior` 一共有两种情况，一个是默认的 `ResetImmediately`，即立刻重置；或者是 `ResetDuringUpdateControls`，这会在 tick 或调用 `UpdateControls` 中重置，随后还会有物理更新，最后的结果才是真正的 transform。

   有两个变种 `ResetBodyModifiersToCachedBoneTransforms` 和 `ResetBodyModifiersInSetToCachedBoneTransforms`。



## 身体修饰接口

### 创建身体修饰

1. `CreateNamedBodyModifier` 用特定的名字创建一个身体修饰，如果名字已存在则返回 `false`。

   ```upp
   bool CreateNamedBodyModifier(
       const FName Name,
       UMeshComponent* MeshComponent,
       const FName BoneName,
       const FName Set,
       const EPhysicsMovementType MovementType = EPhysicsMovementType::Simulated,,
       const ECollisionEnabled::Type CollisionType = ECollisionEnabled::QueryAndPhysics,
       const float GravityMultiplier = 1.0f,
       const float PhysicsBlendWeight = 1.0f,
       const bool bUseSkeletalAnimation = true,
       const bool bUpdateKinematicFromSimulation = true
   );
   ```

   其中的 `MovementType` 决定 Mesh 是 Static（静止）、Kinematic（动画） 或 Simulated（物理）。CollisionType 和所有其它碰撞设置一样有多种选择。`GravityMultiplier` 是对项目中重力的乘数。`PhysicsBlendWeight` 决定了动画和物理的混比。`bUseSkeletalAnimation` 决定 kinematic 目标是否是在骨骼动画的参照系下，否则会在世界坐标下。`bUpdateKinematicFromSimulation` 决定是否应该通过模拟更新物体，即使它处于 kinematic 状态。用于异步物理中可以让 kinematic 部分和模拟部分的行为一致。

   有一个变体 `CreateBodyModifier`，其中不需要指定身体修饰名字，PCC 会自动生成一个并返回这个名字。

2. `CreateBodyModifierFromSkeletalMeshBelow` 为指定的 Skeletal Mesh 中某个骨骼（可以选择是否包含）的所有子骨骼设置身体修饰。

   ```upp
   TArray<FName> CreateBodyModifiersFromSkeletalMeshBelow(
       USkeletalMeshComponent* SkeletalMeshComponent,
       const FName BoneName,
       const bool bIncludeSelf,
       const FName Set,
       const EPhysicsMovementType MovementType = EPhysicsMovementType::Simluated,
       const ECollisionEnabled::Type CollisionType = ECollisionEnabled::QueryAndPhysics,
       const float GravityMultiplier = 1.0f,
       const float PhysicsBlendWeight = 1.0f,
       const bool bUseSkeletalAnimation = true,
       const bool bUpdateKinematicFromSimulation = true
   );
   ```

   和 `CreateControlsFromSkeletalMeshBelow` 类似，为一个骨骼和其所有后代骨骼创建身体修饰，并返回它们的名字。`bIncludeSelf` 决定输入的祖先骨骼是否也被纳入考虑范围。其它参数和 `CreateNamedBodyModifier` 基本一致。

3. `CreateBodyModifiersFromLimbBones` 为一段肢体设置身体修饰。

   ```upp
   TMap<FName, FPhysicsControlNames> CreateBodyModifiersFromLimbBones(
       FPhysicsControlNames& AllBodyModifiers,
       const TMap<FName, FPhysicsControlLimbBones>& LimbBones,
       const EPhysicsMovement::Type MovementType,
       const ECollisionEnabled::Type CollisionType,
       const float GravityMultiplier = 1.0f,
       const float PhysicsBlendWeight = 1.0f,
       const bool bUseSkeletalAnimation = true,
       const bool bUpdateKinematicFromSimulation = true
   );
   ```

   和 `CreateControlsFromLimbBones` 类似地，`AllBodyModifiers` 存储了每段肢体的控制名称，而 `LimbBones` 是这些名称和 `FPhysicsControlLimbBones` 的映射，其中会包含肢体所在的 Skeletal Mesh 以及所有骨骼的名称。

4. `CreateControlsAndBodyModifiersFromLimbBones` 同时创建物理控制和身体修饰，一步到位。

   ```upp
   void CreateControlsAndBodyModifiersFromLimbBones(
       FPhysicsControlNames&                       AllWorldSpaceControls,
       TMap<FName, FPhysicsControlNames>&          LimbWorldSpaceControls,
       FPhysicsControlNames&                       AllParentSpaceControls,
       TMap<FName, FPhysicsControlNames>&          LimbParentSpaceControls,
       FPhysicsControlNames&                       AllBodyModifiers,
       TMap<FName, FPhysicsControlNames>&          LimbBodyModifiers,
       USkeletalMeshComponent*                     SkeletalMeshComponent,
       const TArray<FPhysicsControlLimbSetupData>& LimbSetupData,
       const FPhysicsControlData                   WorldSpaceControlData,
       const FPhysicsControlSettings               WorldSpaceControlSettings,
       const bool                                  bEnableWorldSpaceControls,
       const FPhysicsControlData                   ParentSpaceControlData,
       const FPhysicsControlSettings               ParentSpaceControlSettings,
       const bool                                  bEnableParentSpaceControls,
       const EPhysicsMovementType                  PhysicsMovementType = EPhysicsMovementType::Static,
       const float                                 GravityMultiplier = 1.0f,
       const float                                 PhysicsBlendWeight = 1.0f
   );
   ```




### 设置身体修饰

1. `SetBodyModifierKinematicTarget` 设置身体修饰的 kinematic 目标（本地空间）。`bMakeKinematic`可以顺便将 Skeletal Mesh 设置为 kinematic 的。函数只在身体是 kinematic 时才会有效果。

   ```upp
   bool SetBodyModifierKinematicTarget(
       const FName Name,
       const FVector KinematicTargetPosition,
       const FRotator KinematicTargetOrientation,
       const bool bMakeKinematic
   );
   ```

   这里的 `bMakeKinematic` 会将控制的物体设置为 kinematic 的。

2. `SetBodyModifierMovementType` 设置身体修饰的移动类型。

   ```upp
   bool SetBodyModifierMovementType(
       const FName Name,
       const EPhysicsMovementType MovementType = EPhysicsMovementType::Simulated
   );
   ```

   有两个变种 `SetBodyModifiersMovementType` 和 `SetBodyModifiersInSetMovementType`。

3. `SetBodyModifierCollisionType` 设置身体修饰的碰撞类型。

   ```upp
   bool SetBodyModifierCollisionType(
       const FName Name,
       const ECollisionEnabled::Type CollisionType
   );
   ```

   有两个变种 `SetBodyModifiersCollisionType` 和 `SetBodyModifiersInSetCollisionType`。

4. `SetBodyModifierGravityMultiplier` 设置身体修饰的重力乘数。

   ```upp
   bool SetBodyModifierGravityMultiplier(
       const FName Name,
       const float GravityMultiplier
   );
   ```

   有两个变种 `SetBodyModifiersGravityMultiplier` 和 `SetBodyModifiersInSetGravityMultiplier`。

5. `SetBodyModifierPhysicsBlendWeight` 设置身体修饰的物理混比。

   ```upp
   bool SetBodyModifierPhysicsBlendWeight(
       const FName Name,
       const float PhysicsBlendWeight = 1.0f
   );
   ```

   有两个变种 `SetBodyModifiersPhysicsBlendWeight` 和 `SetBodyModifiersInSetPhysicsBlendWeight`。

6. `SetBodyModifierUseSkeletalAnimation` 设置身体修饰是否使用骨骼动画。此时的目标所在空间是骨骼动画的空间，而非世界空间。

   ```upp
   bool SetBodyModifierUseSkeletalAnimation(
       const FName Name,
       const bool bUseSkeletalAnimation
   );
   ```

   有两个变种 `SetBodyModifiersUseSkeletalAnimation` 和 `SetBodyModifiersInSetUseSkeletalAnimation`。

7. `SetBodyModifierUpdateKinematicFromSimulation` 设置身体修饰是否通过物理更新 kinematic。

   ```upp
   bool SetBodyModifierUpdateKinematicFromSimulation(
       const FName Name,
       bool bUpdateKinematicFromSimulation
   );
   ```

   有两个变种 `SetBodyModifiersUpdateKinematicFromSimulation` 和 `SetBodyModifiersInSetUpdateKinematicFromSimulation`。

8. `AddBodyModifierToSet`将指定的身体修饰（名字）添加到一个组中。

   ```upp
   void AddBodyModifierToSet(
       FPhysicsControlNames& NewSet, 
       const FName BodyModifier, 
       const FName Set
   );
   ```

   有一个变种 `AddBodyModifiersToSet`。



### 获取身体修饰信息

1. `GetAllBodyModifierNames` 获取所有身体修饰的名字。

   ```upp
   const TArray<FName>& GetAllBodyModifierNames(
   ) const;
   ```

2. `GetAllBodyModifierNamesInSet` 获取特定组中宏所有身体修饰的名字。

   ```upp
   const TArray<FName>& GetAllBodyModifierNamesInSet(
       const FName Set
   ) const;
   ```

3. `GetSetsContainingBodyModifier` 获取所有包含给定身体修饰的组。

   ```upp
   TArray<FName> GetSetContainingBodyModifier(
       const FName Control
   ) const;
   ```

   注意返回的是 `TArray<FName>` 而非引用。



## 组件通用接口

### 构造和初始化

PCC 采用 pimpl 惯用法，因此大多数信息都存在间接引用的对象上（后面我们会直接称其为 pimpl）。构造函数中首先为 pimpl 分配内存空间，随后设置一些基本信息。有一些细节需要注意：

* PCC 参与 tick，且在暂停时依然会 tick。
* PCC 有专门的 tick 分组，即 `TG_PrePhysics`。

`InitializeComponent` 会初始化所有控制，将其重置到默认状态。

### Tick

```upp
void UPhysicsControlComponent::TickComponent(
    float                        DeltaTime,
    enum ELevelTick              TickType,
    FActorComponentTickFunction* ThisTickFunction);
```

tick 仅在当前 `TickType` 是 `LEVELTICK_ALL`，即所有 tickable 都参与 tick 时才更新控制，毕竟计算目标速度时会涉及两次 tick 之间的位置差，如果目标没有参与 tick 的话会得到错误的更新信息。

tick 中仅调用了两个函数：

* `UpdateTargetCaches` 会更新骨骼的缓存信息。
* `UpdateControls` 是整个 tick 的核心，参见下面的讨论。

物理控制部分，首先设置每个物理控制的约束实例，调整碰撞设置（从控制设定中得到）并将平动和转动的驱动机参数归零。然后使用 pimpl 的 `ApplyControl` 来执行物理控制。让我们在下一个小节详细讨论其中的逻辑。

身体修饰部分，首先拿到每个身体修饰的身体实例，然后设置其中的物理模拟选项（`SetInstanceSimulatePhysics`）、物理混合权重（`PhysicsBlendWeight`）、以及是否通过模拟更新 kinematic（`SetUpdateKinematicFromSimulation`）。下面是在 `UBodyInstance` 中定义的函数：

```upp
void SetInstanceSimulatePhysics(
    bool bSimulate, 
    bool bMaintainPhysicsBlending = false, 
    bool bPreserveExistingAttachment = false
);
void SetUpdateKinematicFromSimulation(
    bool bUpdateKinematicFromSimulation
);
```

对于 Static 和 Kinematic 两种设定，都会调用 `SetInstanceSimulatePhysics(false, true)`，只不过后者还会再调用 pimpl 上的 `ApplyKinematicTarget`。Simulated 则会调用 `SetInstanceSimulatePhysics(true, true, true)`。

随后，会根据设置决定是否将身体重置到缓存的骨骼 transform、速度等信息上。

再之后，为身体实例设置碰撞设定。这似乎是将身体实例中的每个几何体都设置了碰撞，详细代码如下：

```upp
UBodySetup* BodySetup = BodyInstance->GetBodySetup();
if (BodySetup)
{
    int32 NumShapes = BodySetup->AggGeom.GetElementCount();
    for (int32 ShapeIndex = 0 ; ShapeIndex != NumShapes ; ++ShapeIndex)
    {
        BodyInstance->SetShapeCollisionEnabled(ShapeIndex, BodyModifier.CollisionType);
    }
}
```

最后，如果身体实例在物理模拟状态，则为其施加重力。

```upp
if (BodyInstance->IsInstanceSimulatingPhysics())
{
    float GravityZ = BodyInstance->GetPhysicsScene()->GetOwningWorld()->GetGravityZ();
    float AppliedGravityZ = BodyInstance->bEnableGravity ? GravityZ : 0.0f;
    float DesiredGravityZ = GravityZ * BodyModifier.GravityMultiplier;
    float GravityZToApply = DesiredGravityZ - AppliedGravityZ;
    BodyInstance->AddForce(FVector(0, 0, GravityZToApply), true, true);
}
```



## PImpl 的实现

PCC 使用 pimpl 类 `FPhysicsControlComponentImpl` 进行物理控制数据的存储和目标 Skeletal Mesh 的（位置、速度等）信息缓存，并将弹簧模型的逻辑封装其中。首先让我们看看它的公共成员变量。

* 缓存的 Skeletal Mesh 信息，对每个目标 Skeletal Mesh 都存储一份。

  ```upp
  TMap<TObjectPtr<USkeletalMeshComponent>, FCachedSkeletalMeshData> CachedSkeletalMeshDatas;
  ```

  这里的 `FCachedSkeletalMeshData` 包含 transform 信息（用于判断是否移动过快，此时会视为传送，不会更新速度） 和所有骨骼的位置、方向、速度和角速度。

* 修改的 Skeletal Mesh 信息，类似于一个脏数据标签。

  ```upp
  TMap<TObjectPtr<USkeletalMeshComponent>, FModifiedSkeletalMeshData> ModifiedSkeletalMeshDatas;
  ```

* 所有的控制信息。

  ```upp
  TMap<FName, FPhysicsControlRecord> PhysicsControlRecords;
  ```

* 所有的身体修饰信息。

  ```upp
  TMap<FName, FPhysicsBodyModifier> PhysicsBodyModifiers;
  ```

下面是一些包含复杂逻辑的成员函数：

* `UpdateCachedSkeletalBoneData` 通过 Skeletal Mesh 当前所有骨骼的 transform（调用 `GetEditableComponentSpaceTransforms`）和组件的 transform 得到它们的世界变换。同时根据 `Dt` 参数（可能来自于 `VelocityDeltaTime`）决定速度和角速度（大于零则根据 `v = x / t` 计算，否则默认为零）。
* `ApplyKinematicTarget` 将身体修饰中的位置和旋转信息作用在它指定的骨骼变换上（通过身体实例获得；Static Mesh 则直接可以拿到变换）。这个过程中可能会出现传送。
* `CalculateControlTargetData` 计算得到需要的 transform 和速度等信息（待补充）



### 核心逻辑

#### 施加控制

根据 `PhysicsControlRecords` 中的数据施加控制。

```;
void ApplyControl(
    FPhysicsControlRecord& Record
);
```

首先拿到物理控制的约束实例、父身体实例和子身体实例，确保三者不（全）为空。随后让控制数据更新到约束实例上（见下一节），计算变换和速度的目标（见下下节）并更新到约束实例的位置、方向、速度和角速度目标上。

```upp
// TargetTM 中同时包括了目标位置和方向，以及考虑了控制点的加入
ConstraintInstance->SetLinearPositionTarget(TargetTM.GetTranslation());
ConstraintInstance->SetAngularOrientationTarget(TargetTM.GetRotation());
// 目标速度的计算比较复杂（），参看下下节
ConstraintInstance->SetLinearVelocityTarget(TargetVelocity);
// UE 使用的是转/秒因此要除以 2*Pi
ConstraintInstance->SetAngularVelocityTarget(TargetAngularVelocity / UE_TWO_PI);
```

#### 令控制数据奏效

这部分逻辑在 `ApplyControlStrengths` 中。函数会将 `Record` 中的数据处理后转交给 `ConstraintInstance`，后者利用驱动器实现物理控制。

```upp
bool ApplyControlStrengths(
	FPhysicsControlRecord& Record, 
	FConstraintInstance* ConstraintInstance
);
```

数据处理部分依赖于一个帮助函数 `UE::PhysicsControlComponent::ConvertStrengthToSpringParams`：

```upp
template<typename TOut>
void ConvertStrengthToSpringParams(
	TOut& OutSpring, 
	TOut& OutDamping, 
	double InStrength, 
	double InDampingRatio, 
	double InExtraDamping
);
```

这个函数的逻辑就是公式 $(\ref{pcc-spring-formula})$ 给出的。通过此方法计算出实际的线和角强度与阻尼后再交由 `ConstraintInstance`：

```upp
ConstraintInstance->SetAngularDriveParams(AngularSpring, AngularDamping, MaxTorque);
ConstraintInstance->SetLinearDriveParams(LinearSpring, LinearDamping, MaxForce);
```

最后，函数返回控制数据是否真的奏效（而非一个平凡控制）：

```upp
double TestAngular = (AngularSpring + AngularDamping) * FMath::Max(UE_SMALL_NUMBER, MaxTorque);
FVector TestLinear = (LinearSpring + LinearDamping) * FVector(
    FMath::Max(UE_SMALL_NUMBER, MaxForce.X),
    FMath::Max(UE_SMALL_NUMBER, MaxForce.Y),
    FMath::Max(UE_SMALL_NUMBER, MaxForce.Z));
double Test = TestAngular + TestLinear.GetMax();
return Test > 0;
```

这个写法说实话非常奇怪，个人没有看懂其中的深意。仅当控制数据奏效时才进一步在 `ApplyControl` 中更新变换和速度目标。

#### 计算变换和速度目标

这部分逻辑在 `CalculateControlTargetData` 中。这个函数每一帧都会更新最新的目标速度和变换。

```upp
void CalculateControlTargetData(
	FTransform&                  OutTargetTM,
	FVector&                     OutTargetVelocity,
	FVector&                     OutTargetAngularVelocity,
	const FPhysicsControlRecord& Record,
	bool                         bCalculateVelocity
) const;
```

这是一个比较冗长的函数，让我们逐部分进行分析：

首先判断控制设定中有是否开启了骨骼动画（`Record.PhysicsControl.ControlSettings.bUseSkeletalAnimation`）。随后当父骨骼存在时：

```upp
FTransform ChildBoneTM = ChildBoneData.GetTM();
FTransform ParentBoneTM = ParentBoneData.GetTM();

// 得到子空间下某个物体（的变换）在父骨骼空间下变换的变换，因此取名为 SkeletalDeltaTM
// 注意 UE 中的变换组合是按照次序执行的（A * B 相当于先进行 A 变换，然后进行 B 变换），这和矩阵逻辑相反
FTransform SkeletalDeltaTM = ChildBoneTM * ParentBoneTM.Inverse();
OutTargetTM = SkeletalDeltaTM;

// 处理控制点；注意这里 GetRotation 得到的是一个四元数
// 它作用在控制点上用来得到父骨骼空间坐标下目标的 ControlPoint 所在方向
// 回忆当控制点非零且启用时，控制的目标将是“经过控制点变换”的目标
OutTargetTM.AddToTranslation(OutTargetTM.GetRotation() 
                           * Record.PhysicsControl.ControlSettings.ControlPoint);
```

接下来，如果需要计算速度（`bCalculateVelocity`），则：

```upp
// 计算当前 ControlPoint 在世界坐标下的位置和方向
FVector WorldControlPointOffset = ChildBoneTM.GetRotation() 
                                * Record.PhysicsControl.ControlSettings.ControlPoint;
FVector WorldChildControlPointPosition = ChildBoneTM.GetTranslation() 
                                       + WorldControlPointOffset;

// 计算父骨骼速度，其中考虑了转动因素
// 这本质上就是以父骨骼空间为参考系时子骨骼“相对”的速度
FVector ChildTargetVelocityDueToParent = ParentBoneData.Velocity 
                                       + ParentBoneData.AngularVelocity.Cross(
                                                        WorldChildControlPointPosition 
                                                      - ParentBoneTM.GetTranslation());
// 计算当前状态下子骨骼速度，其中考虑了转动因素
FVector ChildTargetVelocity = ChildBoneData.Velocity 
                            + ChildBoneData.AngularVelocity.Cross(
                                            WorldControlPointOffset);
```

根据这些速度可以进一步计算：

```upp
FVector SkeletalTargetVelocity = ParentBoneInvQ 
                               * (ChildTargetVelocity - ChildTargetVelocityDueToParent);
OutTargetVelocity += SkeletalTargetVelocity 
                   * Record.PhysicsControl.ControlSettings.SkeletalAnimationVelocityMultiplier;

// 角速度的变化比较简单粗暴，父骨骼和子骨骼角速度之差经过乘数放缩后更新到 OutTargetAngularVelocity 上
FVector SkeletalTargetAngularVelocity = ParentBoneInvQ 
                                      * (ChildBoneData.AngularVelocity - ParentBoneData.AngularVelocity);
OutTargetAngularVelocity += SkeletalTargetAngularVelocity 
                          * Record.PhysicsControl.ControlSettings.SkeletalAnimationVelocityMultiplier;
```

如果父骨骼不存在，则会直接使用世界空间（即令 `OutTargetTM = ChildBoneData.GetTM()`），随后如果需要计算速度，则：

```upp
FVector ChildTargetVelocity = ChildBoneData.Velocity 
                            + ChildBoneData.AngularVelocity.Cross(WorldControlPointOffset);
OutTargetVelocity += ChildTargetVelocity 
                   * Record.PhysicsControl.ControlSettings.SkeletalAnimationVelocityMultiplier;
OutTargetAngularVelocity += ChildBoneData.AngularVelocity 
                          * Record.PhysicsControl.ControlSettings.SkeletalAnimationVelocityMultiplier;
```

上面的 `OutTargetTM` 只是目标空间的变换，我们还需要进一步计算：

```upp
const FPhysicsControlTarget& Target = Record.PhysicsControl.ControlTarget;

FQuat TargetOrientationQ = Target.TargetOrientation.Quaternion();
FVector TargetPosition = Target.TargetPosition;

FVector ExtraTargetPosition(0);
if (!bUsedSkeletalAnimation && Record.PhysicsControl.ControlTarget.bApplyControlPointToTarget)
{
    ExtraTargetPosition = TargetOrientationQ * Record.PhysicsControl.ControlSettings.ControlPoint;
}

OutTargetTM = FTransform(TargetOrientationQ, TargetPosition + ExtraTargetPosition) * OutTargetTM;
```

速度的计算如下：

```upp
// UE 使用的角速度单位是转/秒，这里需要转换为标准单位 omega = 2 * PI * f
FVector TargetAngularVelocity = Target.TargetAngularVelocity * UE_TWO_PI;
OutTargetAngularVelocity += TargetAngularVelocity;
FVector ExtraVelocity = TargetAngularVelocity.Cross(ExtraTargetPosition);
OutTargetVelocity += Target.TargetVelocity + ExtraVelocity;
```



### 更新骨骼缓存





***

最后，让我们用一张表来展示 PCC 中所有能够设置的参数，以及它所在的分类。

| 参数             | 代码名称                              | 组件 | 控制目标 | 控制数据 | 控制设定             | 身体修饰           |
| ---------------- | ------------------------------------- | -------- | -------- | -------------------- | ------------------ | ---------------- |
| 子组件（受控制组件） | `ChildComponent` | ✅ |  |  |  |  |
| 子骨骼（子身体） | `ChildBone` | ✅ |  |  |  |  |
| 父组件 | `ParentComponent` | ✅ |  |  |  |  |
| 父骨骼 | `ParentBone` | ✅ |  |  |  |  |
| 控制乘数 | `ControlMultiplier` | ✅ |  |  |  |  |
| 位置             | `Position`                            |         | ✅        |          |                      |                    |
| 方向             | `Orientation`                         |         | ✅        |          |                      |                    |
| 速度             | `Velocity`                            |         | ✅        |          |                      |                    |
| 角速度           | `AngularVelocity`                     |         | ✅        |          |                      |                    |
| 控制点对齐       | `ApplyControlPointToTarget`           |         | ✅        |          | ℹ️ |                    |
| 平动强度         | `LinearStrength`                      |          |          | ✅        |                      |                    |
| 转动强度         | `AngularStrength`                     |          |          | ✅        |                      |                    |
| 平动阻尼         | `LinearDamping`                       |          |          | ✅        |                      |                    |
| 转动阻尼         | `AngularDamping`                      |          |          | ✅        |                      |                    |
| 平动额外阻尼     | `LinearExtraDamping`                  |          |          | ✅        |                      |                    |
| 转动额外阻尼     | `AngularExtraDamping`                 |          |          | ✅        |                      |                    |
| 最大力           | `MaxForce`                            |          |          | ✅        |                      |                    |
| 最大力矩         | `MaxTorque`                           |          |          | ✅        |                      |                    |
| 控制点           | `ControlPoint`                        |          |          |          | ✅                    |                    |
| 使用骨骼动画     | `UseSkeletalAnimation`                |          |          |          | ✅                    | ℹ️ |
| 骨骼动画速度乘数 | `SkeletalAnimationVelocityMultiplier` |          |          |          | ✅                    |                    |
| 开启碰撞         | `EnableCollision`                     |          |          |          | ✅                    |                    |
| 自动关闭         | `AutoDisable`                         |          |          |          | ✅                    |                    |
| 移动类型         | `MovementType`                        |          |          |          |                      | ✅                  |
| 碰撞类型         | `CollisionType`                       |          |          |          |                      | ✅                  |
| 重力乘数         | `GravityMultiplier`                   |          |          |          |                      | ✅                  |
| 物理混比         | `PhysicsBlendWeight`                  |          |          |          |  | ✅                  |
| 物理更新动画 | `UpdateKinematicFromSimulation`       |          |          |          |                      | ✅ |

