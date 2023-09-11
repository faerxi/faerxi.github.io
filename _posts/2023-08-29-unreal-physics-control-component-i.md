---
layout: post
title: UE 物理控制组件
category: [coding, physics-control]
tag: [UE, C++, 物理控制]
author: programmer
math: true
publish: false
---

UE 5.1 新增了一个 **物理控制组件（Physics Control Component，以下简称 PCC）**，相比此前已有的物理动画组件，新组件在物理方面的控制更加精细，因此可以达到更好的效果。

## 基本概念

物理控制组件的核心是调节 Mesh 运动目标和特征的*控制*和调和物理动画的*身体修饰*。其中控制分为控制数据、控制目标和控制设置，分别调整了弹簧参数、物理控制的结束约束和其它设置。

### 控制数据

PCC 中将控制数据分成平动和转动两个类型，对两种弹簧分别设置了*强度（Strength）*、*阻尼比（Damping Ratio）*、额外阻尼和最大作用力。因此在控制数据中我们有 8 个参数可以设置。

* 强度：对应了弹簧的强度，越大则越快将 Mesh 拉向目标。
* 阻尼比：对应了弹簧的阻尼，这里归一化使得取 1 时就是临界阻尼，Mesh 会最高效地在目标处达到平衡。大于 1 则让 Mesh 受到过多阻力，小于 1 则会让 Mesh 在平衡点附近震荡。
* 额外阻尼：阻尼的修正（会加在计算好的阻尼，而非阻尼比上），会让 Mesh 在强度为零的时候也会震动。这一点其实很奇怪，因为 $k=0$ 时依然振动的弹簧理应受到了外力 $F$。“阻尼”理应在运动时才会出现，这里的额外阻尼反而让弹簧自动振动起来，不是很自然。
* 最大作用力：限制弹簧对 Mesh 的作用力，超过这个力时会自动压回这个值。

对于转动来说，上面所有对距离和力的描述都要调整为角度和力矩。

下面我们将从数学角度讨论控制数据中参数的含义，可以酌情跳过。

### 弹簧模型

物理学中弹簧是一个非常常见的模型，其中包含了位移相关的弹力项 $kx$、和速度相关的阻力项 $\lambda \dot{x}$、振子的惯性力项 $m\ddot{x}$ 和外部驱力 $F(t)$。综合起来可以参考下面的公式：

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

<iframe src="https://www.desmos.com/geometry/ugix7unskf" width="800" height="500"></iframe>

从这个图中其实可以看到 $\zeta$ 越靠近 $1$，$x$ 收敛到 $0$ 的速度就越快，反之则不容易。这种情况被称为 **过阻尼（Overdamped）**。

#### $\zeta = 1$ 时

此时的通解是

$$
x = Ae^{-\omega_0t} + Bte^{-\omega_0t}
$$

参考下面的图：

<iframe src="https://www.desmos.com/geometry/fi2lzaua46" width="800" height="400"></iframe>

这种情况被称为 **临界阻尼（Critical Damped）**。

#### $\zeta < 1$ 时

此时我们可以得到一个正弦曲线：
$$
x = e^{-\zeta\omega_0t}\left[A\cos{(\sqrt{1-\zeta^2}\omega_0t)} + B\sin{(\sqrt{1-\zeta^2}\omega_0t)}\right]
$$
这是一个随 $t$ 增大逐渐减小震荡幅度的波。参考下面的的图（这里我将 $\zeta$ 的值调的很小，是因为 $\zeta \ll \omega_0$ 时才有明显的震荡效果；当 $\zeta$ 变大后函数会越来越近似于临界阻尼的情况）：

<iframe src="https://www.desmos.com/geometry/urz0k69ljs" width="800" height="500"></iframe>

$\zeta$ 越小，振荡的效果就越明显，因为方程中的 $\omega_0^2x$ 项成为主导；特别地，当 $\zeta = 0$ 时，振子会不停止地振动，频率为 $\omega_0$，因此它被称为弹簧的 **固有频率（Natural Frequency）**。而 $\zeta$ 显然和弹簧阻力效果有关，它被称为 **阻尼比（Damping Ratio）**。

如果外力 $F(t)$ 存在，则取决于其形式：

* 常数 $F(t) = C$，则振子会在一个非零的地方 $x_0$ 平衡，且此处有 $kx_0 + C = 0$。
* 周期函数 $F(t) = f_d\cos{\omega_dt}$，则振子会逐步趋于这个频率振动，不过效果会受到系数 $\sqrt{\omega_0^2 - \omega_d^2}$ 影响。

物理控制组件中没有提供调控驱力的参数，因此我们就不深入了。

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

### 动力机和控制目标

UE 中的物理作用有时会通过 **动力机（Motor）** 完成（下面我们会直接称为 motor）。这是一个使用弹簧模型的物理系统，它会持续向物体施加大小为 $F = kx + \lambda\dot{x}$ 的力（其中 $x$ 是和目标点的距离，$\dot{x}$ 是物体当前的速度；$k$ 和 $\lambda$ 则是力度和阻尼参数），或大小为 $\tau = \kappa\theta + \mu\dot{\theta}$ 的力矩（其中 $\theta$ 是和目标角度的偏移，$\dot{\theta}$ 是物体当前的角速度；$\kappa$ 和 $\mu$ 则是力度和阻尼参数），最终的效果便是将物体拉到平衡点，并和目标方向对齐。

在 PCC 的控制数据中，强度对应的便是一个和 $k$ 成正比的值，而阻尼比对应的是一个和 $\lambda$ 与强度同时相关的值。最大力会确保 $F$ 不会超过某个值，额外阻尼则是一个不清楚原理的参数（个人觉得可以作为 $k$ 的修正项出现，但在 PCC 的注释中它似乎应该和 $\lambda$ 相关）。转动的情况同理。

目标位置和目标方向可以在 PCC 的控制目标中设置，其中包括：

* 位置：设置物体平动的平衡点。
* 速度：设置物体在经过平衡点时的速度。
* 方向：设置物体转动的平衡点。
* 角速度：设置物体在经过转动平衡点时的速度。
* 





## 逻辑

### 创建物理控制

1. `CreateControl` 创建一个物理控制，其中包含了所有和控制相关的参数。每个物理控制都拥有一个名字，因此这里会默认给它一个独一无二的名字并作为返回值。内部调用了 `CreateNamedControl`。

   ```upp
   FName CreateControl(
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

2. `CreateNamedControl` 创建一个自定义名称的物理控制。如果名称有重复则不创建并返回 `false`。

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

   首先要求子组件不为空。对每个组件，如果其是一个 Skeletal Mesh，则将其缓存起来（调用 `FPhysicsControlComponentImpl::AddSkeletalMeshReferenceForCaching`）。随后，将参数中物理控制相关的项拿来初始化一个 `FPhysicsControlRecord` 对象，其用于存储物理控制的运行状态（包括父子组件和骨骼、控制数据、控制目标、控制设定）。

3. `CreateControlsFromSkeletalMeshBelow` 为某个骨骼（可选择是否包含）的所有子骨骼创建物理控制，并返回所有创建的物理控制名称。注意这里一律使用父空间。要求物理资产非空。内部调用了 `CreateControl`。不对 `ControlTarget` 进行初始化。

   ```upp
   TArray<FName> CreateControlsFromSkeletalMeshBelow(
       USkeletalMeshComponent* SkeletalMeshComponent,
       const FName BoneName,
       const bool bIncludeSelf,
       const EPhysicsControlType ControlType,
       const FPhysicsControlData ControlData,
       const FPhysicsControlSettings ControlSettings,
       const FName Set,
       const bool bEnabled
   );
   ```

4. `CreateControlsFromSkeletalMeshAndConstraintProfileBelow` 和 `CreateControlsFromSkeletalMeshBelow` 类似，但从物理资产的约束预设中提取控制数据来初始化物理控制。内部调用了 `CreateControl`。

   ```upp
   TArray<FName> CreateControlsFromSkeletalMeshAndConstraintProfileBelow(
       USkeletalMeshComponent* SkeletalMeshComponent,
       const FName BoneName,
       const bool bIncludeSelf,
       const FName ConstraintProfile,
       const FName Set,
       const bool bEnabled
   );
   ```

5. `CreateControlsFromSkeletalMesh` 为指定的一个 Skeletal Mesh 中多个骨骼设置物理控制。要求物理资产非空。内部调用了 `CreateControl`。不对 `ControlTarget` 进行初始化。

   ```upp
   TArray<FName> CreateControlsFromSkeletalMesh(
       USkeletalMeshComponent* SkeletalMeshComponent,
       const TArray<FName>& BoneNames,
       const EPhysicsControlType ControlType,
       const FPhysicsControlData ControlData,
       const FPhysicsControlSettings ControlSettings,
       const FName Set,
       const bool bEnabled
   );
   ```

6. `CreateControlsFromSkeletalMeshAndConstraintProfile` 和 `CreateControlsFromSkeletalMesh` 类似，但从物理资产的约束预设中提取控制数据来初始化物理控制。内部调用了 `CreateControl`。

   ```upp
   TArray<FName> CreateControlsFromSkeletalMeshAndConstraintProfile(
       USkeletalMeshComponent* SkeletalMeshComponent,
       const TArray<FName>& BoneNames,
       const FName ConstraintProfile,
       const FName Set,
       const bool bEnabled
   );
   ```

7. `CreateControlsFromLimbBones` 为一段肢体设置物理控制。肢体是以映射表的形式传入的，函数中会为其中每一个骨骼设置相同的物理控制。

   ```upp
   TMap<FName, FPhysicsControlNames> CreateControlsFromLimbBones(
       FPhysicsControlNames& AllControls,
       const TMap<FName, FPhysicsControlLimbBones>& LimbBones,
       const EPhysicsControlType ControlType,
       const FPhysicsControlData ControlData,
       const FPhysicsControlSettings ControlSettings,
       const bool bEnabled.
   );
   ```

8. `CreateControlsFromLimbBonesAndConstraintProfile` 和 `CreateControlsFromLimbBones` 类似，但从物理资产的约束预设中提取控制数据来初始化物理控制。内部调用了 `CreateConrol`。

   ```upp
   TMap<FName, FPhysicsControlName> CreateControlsFromLimbBonesAndConstraintProfile(
       FPhysicsControlNames& AllControls,
       const TMap<FName, FPhysicsControlLimbBones>& LimbBones,
       const FName ConstraintProfile,
       const bool bEnabled
   );
   ```

### 设置控制数据

1. `SetControlData` 为物理控制设置数据。若给定的控制名字不存在则返回 `false`。

   ```upp
   bool SetControlData(
       const FName Name,
       const FPhysicsControlData ControlData,
       const bool bEnableControl
   );
   ```

2. `SetControlDatas` 同时设置多个物理控制。内部调用了 `SetControlData`。

   ```upp
   void SetControlDatas(
       const TArray<FName>& Names,
       const FPhysicsControlData ControlData,
       const bool bEnableControl
   );
   ```

3. `SetControlDatasInSet` 设置特定组的物理控制，内部调用了 `SetControlDatas`。

   ```upp
   void SetControlDatasInSet(
       const FName SetName,
       const FPhysicsControlData ControlData,
       const bool bEnableControl
   );
   ```

4. `SetControlMultiplier` 设置物理控制数据的乘数。若给定的控制名字不存在则返回 `false`。

   ```upp
   bool SetControlMultiplier(
       const FName Name,
       const FPhysicsControlMultiplier ControlMultiplier,
       const bool bEnableControl
   );
   ```

5. `SetControlMultipliers` 设置多个物理控制数据的乘数。内部调用了 `SetControlMultiplier`。

   ```upp
   void SetControlMultipliers(
       const TArray<FName>& Names,
       const FPhysicsControlMultiplier ControlMultiplier,
       const bool bEnableControl
   );
   ```

6. `SetControlMultipliersInSet` 设置特定组的物理控制，内部调用了 `SetControlMultipliersInSet`。

   ```upp
   void SetControlMultipliersInSet(
       const FName SetName,
       const FPhysicsControlMultiplier ControlMultiplier,
       const bool bEnableControl
   );
   ```

7. `SetControlLinearData` 仅设置物理控制中和平动相关的数据。若给定的控制名字不存在则返回 `false`。

   ```upp
   bool SetControlLinearData(
       const FName Name,
       const float Strength,
       const float DampingRatio,
       const float ExtraDamping,
       const float MaxForce,
       const bool bEnableControl
   );
   ```

8. `SetControlAngularData` 仅设置物理控制中和转动相关的数据。若给定的控制名字不存在则返回 `false`。

   ```upp
   bool SetControlAngularData(
       const FName Name,
       const float Strength,
       const float DampingRatio,
       const float ExtraDamping,
       const float MaxTorque,
       const bool bEnableControl
   );
   ```

### 设置控制目标

1. `SetControlTarget` 设置物理控制的目标。

   ```upp
   bool SetControlTarget(
       const FName,
       const FPhysicsControlTarget ControlTarget,
       const bool bEnableControl
   );
   ```

2. `SetControlTarget` 设置多个物理控制的目标。内部调用了 `SetControlTarget`。

   ``` upp
   void SetControlTargets(
       const TArray<FName>& Names,
       const FPhysicsControlTarget ControlTarget,
       const bool bEnableControl
   );
   ```

3. `SetControlTargetsInSet` 设置特定组的物理控制。内部调用了 `SetControlTargets`。

   ```upp
   bool SetControlTargetsInSet(
       const FName SetName,
       const FPhysicsControlTarget ControlTarget,
       const bool bEnableControl
   );
   ```

4. `SetControlTargetPositionAndOrientation` 仅设置物理控制中位置和方向的目标。内部调用了 `SetControlTargetPosition` 和 `SetControlTargetOrientation`。参数中的 `VelocityDeltaTime` 若不为零，则会用当前目标位置计算目标速度（目标位置相对当前位置的偏移与 `VelocityDeltaTime` 的比值）。`bApplyControlPointToTarget` 则用于设置控制点（弹簧作用点）和目标位置的关系：若设置为 `true`，函数会试图将 Mesh 和目标位置（被假定为一个对象）的 Transform 对齐，否则仅仅将控制点和目标位置对齐。

   ```upp
   bool SetControlTargetPositionAndOrientation(
       const FName Name,
       const FVector Position,
       const FRotator Orientation,
       const float VelocityDeltaTime,
       const bool bEnableControl,
       const bool bApplyControlPointToTarget
   );
   ```

5. `SetControlTargetPosition` 仅设置物理控制中位置的目标。参数中的 `VelocityDeltaTime` 解释见上一条。

   ```
   bool SetControlTargetPosition(
       const FName Name,
       const FVector Position,
       const float VelocityDeltaTime,
       const bool bEnableControl,
       const bool bApplyControlPointToTarget
   );
   ```

6. `SetControlTargetOrientation` 仅控制物理控制中方向的目标。参数中的 `AngularVelocityDeltaTime` 和上一条中的 `VelocityDeltaTime` 原理类似。

   ```upp
   bool SetControlTargetOrientation(
       const FName Name,
       const FRotator Orientation,
       const float AngularVelocityDeltaTime,
       const bool bEnableControl,
       const bool bApplyControlPointToTarget
   );
   ```

7. `SetControlTargetPoses` 仅控制物理控制中位置和方向的目标；和 `SetControlTargetPositionAndOrientation` 的区别在于，这个函数会将子组件的 transform 向父组件对齐。

   ```upp 
   bool SetControlTargetPoses(
       const FName Name,
       const FVector ParentPosition,
       const FRotator ParentOrientation,
       const FVector ChildPosition,
       const FRotator ChildOrientation,
       const float VelocityDeltaTime,
       const bool bEnableControl
   );
   ```

### 设置控制设置

1. `SetControlUseSkeletalAnimation` 让物理控制以骨骼动画为目标。`SkeletalAnimationVelocityMultiplier` 决定了动画速度在目标速度中的占比。

   ```upp
   bool SetControlUseSkeletalAnimation(
       const FName Name,
       const bool bUseSkeletalAnimation = true,
       const float SkeletalAnimationVelocityMultiplier = 1
   );
   ```

2. `SetControlsUseSkeletalAnimation` 让多个物理控制以骨骼动画为目标。

3. `SetControlsInSetUseSkeletalAnimation` 让特定组中的物理控制以骨骼动画为目标。

4. `SetControlAutoDisable` 让物理控制在每个 tick 后自动关闭。

   ```upp
   bool SetCOntrolAutoDisable(
       const FName Name,
       const bool bAutoDisable
   );
   ```

5. `SetControlsAutoDisable` 让多个物理控制在每个 tick 后自动关闭。

6. `SetControlsInSetAutoDisable`：让特定组中的物理控制在每个 tick 后自动关闭。

7. `SetControlDisableCollision`：让物理控制中 Mesh 和它相连的组件间不发生碰撞。

   ```upp
   bool SetControlDisableCollision(
       const FName Name,
       const bool bDisableCollision
   );
   ```

8. `SetControlsDisableCollision`：让多个物理控制中 Mesh 和它相连的组件间不发生碰撞。

9. `SetControlsInSetDisableCollision`：让特定组中的物理组件中 Mesh 和它相连的组件间不发生碰撞。

### 创建身体修饰

1. `CreateBodyModifier` 创建一个身体修饰。其中的 `MovementType` 决定 Mesh 是 static、Kinematic 或 Simulated。CollisionType

   ```upp
   FName CreateBodyModifier(
       UMeshComponent* MeshComponent,
       const FName BoneName,
       const FName Set,
       consg EPhysicsMovementType MovementType,
       const ECollisionEnabled::Type CollisionType,
       const float GravityMultiplier = 1,
       const bool bUseSkeletalAnimation = true
   );
   ```

2. `CreateNamedBodyModifier` 用特定的名字创建一个身体修饰，如果名字已存在则返回 `false`。

   ```upp
   bool CreateNamedBodyModifier(
       const FName Name,
       UMeshComponent* MeshComponent,
       const FName BoneName,
       const FName Set,
       const EPhysicsMovementType MovementType,
       const ECollisionEnabled::Type CollisionType,
       const float GravityMultiplier = 1,
       const bool bUseSkeletalAnimation = true
   );
   ```

3. `CreateBodyModifierFromSkeletalMeshBelow` 为指定的 Skeletal Mesh 中某个骨骼（可以选择是否包含）的所有子骨骼设置身体修饰。

   ```upp
   TArray<FName> CreateBodyModifiersFromSkeletalMeshBelow(
       USkeletalMeshComponent* SkeletalMeshComponent,
       const FName BoneName,
       const bool bIncludeSelf,
       const FName Set,
       const EPhysicsMovementType MovementType,
       const ECollisionEnabled::Type, CollisionType,
       const float GravityMultiplier = 1,
       const bool bUseSkeletalAnimation = true
   );
   ```

4. `CreateBodyModifiersFromLimbBones` 为一段肢体设置身体修饰。

   ```upp
   TMap<FName, FPhysicsControlNames> CreateBodyModifiersFromLimbBones(
       FPhysicsControlNames& AllBodyModifiers,
       const TMap<FName, FPhysicsControlLimbBones>& LimbBones,
       const EPhysicsMovement::Type MovementType,
       const ECollisionEnabled::Type CollisionType,
       const float GravityMultiplier = 1,
       const bool bUseSkeletalAnimation = true
   );
   ```

### 设置身体修饰

1. `SetBodyModifierKinematicTarget` 设置身体修饰的 kinematic 目标。`bMakeKinematic`可以顺便将 Skeletal Mesh 设置为 kinematic 的。函数的效果只在身体是 kinematic 时才会有效果。

   ```upp
   bool SetBodyModifierKinematicTarget(
       const FName Name,
       const FVector KinematicTargetPosition,
       const FRotator KinematicTargetOrientation,
       const bool bMakeKinematic
   );
   ```

2. `SetBodyModifierMovementType` 设置身体修饰的移动类型。

   ```upp
   bool SetBodyModifierMovementType(
       const FName Name,
       const EPhysicsMovementType MovementType = EPhysicsMovementType::Simulated
   );
   ```

3. `SetBodyModifiersMovementType` 设置多个身体修饰的移动类型。

4. `SetBodyModifiersInSetMovementType` 设置特定组的身体修饰的移动类型。

5. `SetBodyModifierCollisionType` 设置身体修饰的碰撞类型。

   ```upp
   bool SetBodyModifierCollisionType(
       const FName Name,
       const ECollisionEnabled::Type CollisionType
   );
   ```

6. `SetBodyModifiersCollisionType` 设置多个身体修饰的碰撞类型。

7. `SetBodyModifiersInSetCollisionType` 设置特定组的身体修饰的移动类型。

8. `SetBodyModifierGravityMultiplier` 设置身体修饰的重力乘数。

   ```upp
   bool SetBodyModifierGravityMultiplier(
       const FName Name,
       const float GravityMultiplier
   );
   ```

9. `SetBodyModifiersGravityMultiplier` 设置多个身体修饰的重力乘数。

10. `SetBodyModifiersInSetGravityMultiplier` 设置特定组的身体修饰的重力乘数。

11. `SetBodyModifierUseSkeletalAnimation` 设置身体修饰是否使用骨骼动画。此时的目标所在空间是骨骼动画的空间，而非世界空间。

    ```upp
    bool SetBodyModifierUseSkeletalAnimation(
        const FName Name,
        const bool bUseSkeletalAnimation
    );
    ```

12. `SetBodyModifiersUseSkeletalAnimation` 设置多个身体修饰是否使用骨骼动画。

13. `SetBodyModifiersInSetUseSkeletalAnimation` 设置特定组的身体修饰是否使用骨骼动画。

14. 



### Tick 函数

PCC 重写了 `UActorComponent` 的 `TickComponent` 函数，它会随世界 tick 执行。在

