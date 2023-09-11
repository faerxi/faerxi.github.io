---
layout: post
title: UE 物理破碎效果
category: coding
tag: [Unreal Engine, C++]
publish: false
---

## 概述

借助 UE 5.0 引入的 *Chaos Destruction*，我们可以用 Unreal 编辑器中的 **破碎模式（Fracture Mode）** 构建 Static Mesh 的破碎模型（官方名称为 **几何体集（Geometry Collection）**）。在官方示例中，我们看到可破碎的墙体、地面，当我们对其施加物理场时，它们会产生破碎效果。这里的 **物理场（Physics Field）** 也是 UE 提供的物理模拟工具，也是主动触发破碎效果的唯一途径。为了制作更加复杂的物理场景，我们也可以用 Blueprint 中的 AnchorField、SleepField、MasterField 等预制的物理场来达到不同效果。

### 破碎效果的目标

官方破碎效果，其特点在于静态。无论是 1. 破碎模型的构建需要依赖 Unreal 编辑器的破碎模式；2. 破碎模型的行为绑定在一个 Static Mesh 上（实际上是一个几何体集 Mesh，但其可破碎以外的行为和 Static Mesh 非常相似）；3. 物理场的设置都是在关卡中进行的；4. 破碎效果的产生时机取决于物理场的设置，或 Mesh 受重力影响与其它物体碰撞产生的。

因此作为扩充，我希望可以实现：

* 手动触发的破碎效果，包括定向破碎和类似爆炸的效果。
* 让 Skeletal Mesh 拥有破碎的效果。
* 让 Skeletal Mesh 的破碎效果和布娃娃结合起来呈现。
* 为物理破碎添加合适的 Niagara 粒子特效，让整体效果更加炫酷。

### 难点初步分析

1. Skeletal Mesh 无法直接生成破碎模型，因此可能需要将它转化为 Static Mesh 后再使用破碎效果。
2. 布娃娃状态下身体的姿态无法预测，难以提前判断合适的姿势用于破碎模型生成。

### 相关类型

下面介绍一些实现破碎效果需要用到的 C++ 类型：

* `USkeletalMeshComponent`：
* `UGeometryCollection`：Static Mesh 通过破碎模式生成的可破碎模型，可以 `UGeometryCollectionComponent` 中的 `RestCollection` 属性为想要的可破碎模型。



## 实现日志

### 初版实现：具有层级关系的可破碎身体部分集，即可肢解网格体（Dismemberable Mesh）

一个 Skeletal Mesh 可以分成若干个 **身体部分（Body Part）**，这些部分组成了一个类似于骨骼的层级关系。当角色受击（或遭遇其它需要触发破碎效果的事件）时，根据受击的位置来肢解角色。直接的逻辑如下：

* 在受击的骨骼处生成一个破碎模型，并让它立刻破碎。
* 在受击骨骼的子骨骼处生成一个（或多个）破碎模型，让其受重力自动坠落。
* 隐藏受击骨骼和其子骨骼。

这样就可以做出受击骨骼被外力击碎，且附着骨骼自然脱落的假象。代码的实现概览如下：

```upp
USTRUCT()
struct UDismemberableBodyPart {
    GENERATE_BODY()

    // 身体部分对应骨骼名称
    UPROPERTY(EditAnywhere)
    FName Bone = "None";
    
    // 骨骼受击时破坏的模型
    UPROPERTY(EditAnywhere)
    TObjectPtr<UGeometryCollection> MainDebris = nullptr;
    
    // 骨骼受击时脱落的模型
    UPROPERTY(EditAnywhere)
    TObjectPtr<UGeometryCollection> AttachedDebris = nullptr;
};

UCLASS(BlueprintType, Blueprintable, EditInlineNew)
class UDismemberableMeshComponent : public USkeletalMeshComponent {
    GENERATE_BODY()
public:
    // 身体部分名称对可破碎身体部分的映射
    UPROPERTY(EditAnywhere)
    TObjectPtr<FName, UDismemberableBodyPart> BodyPartMap;
    
    // 身体部分对父部分的映射
    UPROPERTY(EditAnywhere)
    TMap<FName, FName> ParentPartMap;
};
```

上面展示的属性中，每个 BodyPart 都包含了其对应的（唯一）骨骼、该骨骼对应的可破碎模型，以及击碎该骨骼后应该脱离的可破碎模型。在可肢解组件中，则存放了身体部分的映射表和其层级结构的信息。上面，我特地将骨骼名称和身体部分名称区分开来，这出于下面的考量：

* 骨骼名称相对不友好。
* 希望同一个骨骼能根据条件造成不同破碎效果，此时若可破坏模型的设置都不同，就不能使用同一个身体部分。

因此设计上，一个骨骼可能对应多个身体部分。举例来说，同样是击打上臂，我们希望在冲击极大时可以直接轰掉整个上半身，此时就需要两个身体部分（比如起名为 `LeftUpperArm` 和 `UpperBody`），并在受击逻辑中选择破坏其中的一个。下面让我们顺势介绍可肢解组件中身体破坏的接口：

```upp
USTRUCT(BlueprintType)
struct FDismemberment {
    GENERATE_BODY()
    
    // 是否传递冲量到子身体部分
    UPROPERTY(BlueprintReadWrite, EditAnywhere)
    bool bEnablePropagation = false;
    
    // 是否对受击的身体部分使用冲量
    UPROPERTY(BlueprintReadWrite, EditAnywhere)
    bool bApplyImpulse = true;
    
    // 是否保持受击的身体部分完整性；若 true 则不会使其破碎
    UPROPERTY(BlueprintReadWrite, EditAnywhere)
    bool bKeepIntegrity = false;
    
    // 冲量大小
    UPROPERTY(BlueprintReadWrite, EditAnywhere)
    float ImpulseMagnitude = 0.f;
    
    // 冲量方向
    UPROPERTY(BlueprintReadWrite, EditAnywhere)
    FVector ImpulseDirection = FVector::ZeroVector;
    
    // 传递深度
    UPROPERTY(BlueprintReadWrite, EditAnywhere)
    int32_t PropagationDepth = 0.f;
    
    // 传递时冲量的衰减倍率
    UPROPERTY(BlueprintReadWrite, EditAnywhere)
    float PropagationFalloff = 1.f;
};

// 用给定的破碎信息击碎一个身体部分
UFUNCTION(BlueprintCallable)
void UDismemberableMeshComponent::Break(FName const& Part, FDismemberment const Info);
```

从 `FDismemeberment` 的设置中可以一窥肢解的逻辑：

* 受击部位的响应可以破碎或不破碎模型，也可以施加或不施加冲量。值得说明的是，通过物理场实现的效果中，冲量和破碎是分开的两个逻辑。
* 肢解可以一层层传递到子身体部分，此时就*不会*生成并让附着可破碎模型脱落。

不过，这些参数都在运行期设置，可能对策划不太友好。一个更理想的实现可能是将一部分信息放到 `UDismemberableMeshComponent` 中配置，这也就引出了第二版实现。



### 第二版实现：使用阈值组

考虑到策划可能需要调整身体部分的受击特性，我决定将一些可以编辑时确定的参数放到可肢解组件中（事实上，所有除了冲量大小以外的值都可以在运行之前设置）。考虑到不同的肢解效果本质上取决于冲量的大小，因此可以设计一个 **阈值组（Threshold Group）**，当冲量超过不同阈值后产生对应的效果。

```upp
USTRUCT()
struct FDismembermentThreshold {
    GENERATE_BODY()
    
    // 对应击碎的身体部分名称
    UPROPERTY(EditAnywhere)
    FName Part = "None";
    
    // 击碎需要的最小冲量
    UPROPERTY(EditAnywhere)
    float ImpulseThreshold = 0.f;
};

USTRUCT()
struct FDismembermentThresholdGroup {
    GENERATE_BODY();
    
    UPROPERTY(EditAnywhere)
    TArray<FDismembermentThresholdGroup> Thresholds;
};

UCLASS(BlueprintType, Blueprintable, EditInlineNew)
class UDismemberableMeshComponent : public USkeletalMeshComponent {
    GENERATE_BODY()
public:
    // 骨骼名称到阈值组的映射
    UPROPERTY(EditAnywhere)
    TMap<FName, FDismembermentThresholdGroup> GroupMap;
};
```

可以看到此时我们的 `UDismemberableMeshComponent` 中只需要存储所有相关骨骼的阈值组即可，抛弃了之前繁琐的层级关系；但与之相对地，每个阈值组可能会对应多个破碎逻辑，策划需要一一将其配对。但好处是所有的信息都会在蓝图中清楚地显示出来，这是不可避免的配表步骤。

经此改动，我们的破碎接口也应该有所调整：

```upp
void UDismemberableMeshComponent::Break(FName const& Bone, FVector const& ImpulseDirection, float const ImpulseMagnitude);
```





### 当前实现：预设的残肢 Skeletal Mesh 和逐骨骼破碎效果



```upp
UENUM(BlueprintType)
enum EDismemberableBodyPart : int8_t {
    Head, LeftClavicle, RightClavicle, LeftUpperArm, RightUpperArm, LeftLowerArm, RightLowerArm, LeftHand, Righthand,
    UpperTrunk, LowerTrunk,
    LeftThigh, RightThigh, LeftCalf, RightCalf, LeftFoot, RightFoot,
    
    // 组合的部位
    Trunk, LeftUpperLimb, RightUpperLimb, LeftLowerLimb, RightLowerLimb, UpperBody, LowerBody,
};

USTRUCT(BlueprintType)
struct FDismembermentGroup {
    GENERATE_BODY()
    
    UPROPERTY(EditAnywhere)
    TArray<EDismemberableBodyPart> BodyParts;
    
    UPROPERTY(EditAnywhere)
    bool bIsMinimal = false;
    
    UPROPERTY(EditAnywhere, meta = (EditCondition = "!bIsMinimal", EditConditionHides))
    TObject<USkeletalMesh> SkeletalMesh = nullptr;
    
    UPROPERTY(EditAnywhere, meta = (EditCondition = "bIsMinimal", EditConditionHides))
    TObject<UGeometryCollection> Debris = nullptr;
    
    UPROPERTY(BlueprintReadWrite, EditAnywhere)
    float Amount;
};

USTRUCT(BlueprintType)
struct FDismemberableBone {
    GENERATE_BODY()
        
    UPROPERTY(EditAnywhere)
    FName Bone;
        
    UPROPERTY(EditAnywhere)
    TArray<FDismembermentGroup> Thresholds;
};

UCLASS(BlueprintType, Blueprintable, EditInlineNew)
class UDismemberableMeshComponent : public USkeletalMeshComponent {
    GENERATE_BODY()
public:
    // 存储骨骼名称到其破碎预设的映射
    UPROPERTY(EditAnywhere, DisplayName = "Break Targets")
    TMap<EDismemberableBodyPart, FDismemberableBone> BoneThresholdMap;
    
protected:
    TMap<FName, EDismemberableBodyPart> BoneToBodyPartMap;
};
```

我们的破碎接口则需要至少两个，其一是直接根据受击骨骼、收到的冲量和预设来产生破坏；或者也可以手动指定要破坏的身体部分。

```upp
void UDismemberableMeshComponent::Break(FName const& Bone, FVector const& Direction, float const Magnitude);

void UDismemberableMeshComponent::BreakBodyParts(TArray<EDismemberableBodyPart> const& BodyParts, FVector const& Direction, float const Magnitude);
```

此外也有必要提供按照身体部分层级来一次性破坏多个部分的接口。

```upp
void UDismemberableMeshComponent::BreakAllBodyPartsBelow(EDismemberableBodyPart BodyPart, FVector const& Direction, float const Magnitude);

void UDismemberableMeshComponent::BreakAllBodyPartsAbove(EDismemberableBodyPart BodyPart, FVector const& Direction, float const Magnitude);

void UDismemberableMeshComponent::BreakAllBodyParts(FVector const& Direction, float const Magnitude);
```









## 文档

### 可破坏物

对于可破坏物，需要在其基类蓝图（或 C++）中将其网格体设置为 Destructible Mesh，即 `DestructibleMeshComponent`。具体设置如下：

* 寻找可破坏物的 Static Mesh，将其作为 Asset 导入到 UE 中，例 `SM_Sample`。
* 将 `SM_Sample` 拖入任意地图（为了不搞乱其它地图，可以新建一个专门用于生成破碎模型的地图），然后 Shift+6 进入破碎模式（Shift+1 则还原到默认的模式）。
* （此时理应已经）选中地图中的 `SM_Sample`，然后在左侧菜单中选择 *Generate\|New*，此时会弹出文件目录，需要选择破碎模型的储存路径，建议保存在 `SM_Sample` 所在路径或其子路径 `GeometryCollections` 下。建议修改破碎模型的名称，例 `GC_Sample`。
* 选择左侧菜单中的 *Fracture\|Uniform*，然后在 Fracture 面板中找到 *Uniform Voronoi* 栏，这里面的最小和最大值决定了破碎模型的破碎程度（UE 会选择两者之间的随机值将模型破碎）。然后在 Fracture 面板最下面点击 Fracture。至此破碎模型就已经生成完毕。
* （可选）在 `GC_Sample` 中找到 *Damage Threshold*，其中只有最上面一个元素生效，将其调小后破碎模型会碎得更加“清脆”，反之则更加“迟钝”。建议在几百到几千这个数量级。
* 在 Content Drawer 中选中一个目录（建议在 `SM_Sample` 相同路径或其子路径 `GeometryCollectionActors` 下）创建一个新的 Blueprint 类，点开 ALL CLASSES 查找基类 `GeometryCollectionActor`，然后将其重命名为 `BP_GC_Sample`。
* 打开 `BP_GC_Sample`，在 *Chaos Physics\|Rest Collection* 中选择 `GC_Sample`。
* 打开可破坏物的蓝图类，在左上方 Components 中选择其中的 Mesh（可能是其它名字，但鼠标悬浮在上面会显示其是一个 `DestructibleMeshComponent`）。在右侧 Details 菜单中寻找 *Destruction\|Debris*，选择 `BP_GC_Sample`，这样就完成了对可破坏物的基本设置。
* 让可破坏物破碎的部分需要让 Mesh 调用 `Break` 方法，这块程序来做。
* （可选）如果想要可破坏物在一定冲量下才会破碎，可以在 Mesh 的 *Destruction\|Impulse Threshold* 处调整最小冲量。如果调用 `Break` 方法时 `Magnitude` 参数小于这个值，就不会产生破碎效果。默认情况下任何非负的冲量大小都会让可破坏物破碎。



### 可肢解角色

对于可肢解角色，需要在其基类蓝图（或 C++）中将其网格体设置为 Dismemberable Mesh，即 `DismemberableMeshComponent`。具体设置如下：

#### 得到单个身体部分的步骤

* 在 UE 中寻找可肢解角色的 Skeletal Mesh（例 `SK_Sample`），在 Content Browser 中右键选择 *Asset Actions\|Export...*，将其作为 FBX 文件导出。
* 打开 Blender，确认我们使用 Blender Unit 而非 Metric Unit。可以在右侧工具栏中选择 Scene（一个锥体的图标），在 Units 栏中将 Unit System 设置为 None。
* 导入 FBX 文件，然后鼠标点击选中模型，按 Tab 进入 Edit Mode，然后按 A 全选所有点线面，再按 Shift+Space+8 使用二分工具。
* 先用 Shift+MMB（鼠标中键）移动到想要切割的身体部分附近，再用 Alt+MMB 拖拽场景到一个合适的视图。
* 从身体外用左键选择切割起点（按住不要松手），然后拖拽选择切割终点，Blender 会高亮所有和边切割的交点。松开鼠标后切割完成。
* （可选）选择左下角（可能折叠的）Bisect 菜单，这里可以再微调切割平面。
* 在 Bisect 菜单中选择 Clear Inner 或 Clear Outer 来选择模型中想要的部分。
* （可选）逐个选择身体边缘的三个顶点，按 F 生成面，目标是将截面补齐。由于我们用的是平面切割，最终一定会补齐为一个平面。
* 上方菜单 *File\|Export\|FBX* 将当前模型导出。
* 回到 UE，在合适的路径下（推荐在 `SK_Sample` 所在路径或其子路径 `BodyParts` 下）导入 FBX 文件，注意选择 Skeleton 为 `SK_Character_Skeleton`。命名为 `SK_Sample_BodyPart`（将 `BodyPart` 替换为特定的名称，详细见下面一段）。

#### 身体部分详解

项目中的所有 Skeletal Mesh 理应遵循一套 Skeleton 标准；若不是请给出和某个 Skeleton 标准的双向映射表。下面假设这个骨骼标准存在且拥有对应下面列出的身体部分的骨骼。括号中会标出 UE 官方骨骼的名称，和我们建议的后缀：

* 头部（`head`、`Head`）
* 大臂（`upperarm_l` 和 `upperarm_r`、`LeftUpperArm` 和 `RightUpperArm`）
* 小臂（`lowerarm_l` 和 `lowerarm_r`、`LeftLowerArm` 和 `RightLowerArm`）
* 手部（`hand_l` 和 `hand_r`、`LeftHand` 和 `RightHand`）
* 躯干上部（`spine_03`、`UpperTrunk`）
* 躯干下部（`spine_01`、`LowerTrunk`)
* 骨盆（`pelvis`）
* 大腿（`thigh_l` 和 `thigh_r`、`LeftThigh` 和 `RightThigh`）
* 小腿（`calf_l` 和 `calf_r`、`LeftCalf` 和 `RightThigh`）
* 脚部（`foot_l` 和 `foot_r`、`LeftFoot` 和 `RightFoot`）

总体来讲，击打到一个身体部分后的反馈是，特定部位对应的破碎模型立刻被破坏，剩余的部分则生成对应的 Skeletal Mesh。这些 Skeletal Mesh 会立刻模拟物理，并在特定情况下再让其破碎（比如受到二次撞击等）。注意到这个过程有一定规律性，不需要全部依次指定。下面是对每个部位受击的分析：

* 头部：立刻击碎头部，若伤害足够则击碎两肩。
* 大臂：让大臂从肩部脱落，伤害足够则击碎大臂并让小臂脱落，更大则同时让肩膀脱落。
* 小臂：让小臂从大臂脱落，伤害足够则击碎小臂并让手部脱落，更大则同时让大臂脱落。
* 手部：立刻击碎手部，若伤害足够则击碎小臂。
* 躯干上部：让躯干上部从身体脱落，伤害足够则击碎躯干上部并让头、两臂脱落。
* 躯干下部：让躯干下部从身体脱落，伤害足够则击碎躯干下部并让躯干上部和头、两臂一同脱落。，更大则让躯干上下部一同破碎并让头、两臂脱落。
* 大腿：让大腿从盆骨脱落，若伤害足够则击碎大腿。
* 小腿：让小腿从大腿脱落，若伤害足够则击碎小腿并让脚部脱落。
* 脚部：立刻击碎脚部，若伤害足够则击碎小腿。

因此我们需要下面这些 Skeletal Mesh 的资产（单独部位可以通过 Static Mesh 代替，这里不提及了），括号中是建议的后缀名称，和剩下的身体部分：

1. 小臂 + 手（`LowerArmExt`、`LowerArmRem`）
2. 大臂 + 小臂（`UpperArmExt1`、`UpperArmRem1`）
3. 大臂 + 小臂 + 手（`UpperArmExt2`、`UpperArmRem1`）
4. 肩膀 + 大臂（`ClavicleExt1`、`ClavicleRem1`）
5. 肩膀 + 大臂 + 小臂（`ClavicleExt2`、`ClavicleRem2`）
6. 肩膀 + 大臂 + 小臂 + 手（`ClavicleExt3`、`ClavicleRem3`）
7. 躯干上部 + 肩膀（两侧）（`UpperTrunkExt1`、`UpperArmExt2`、`UpperTrunkRem`）
8. 躯干上部 + 肩膀 + 大臂（两侧）（`UpperTrunkExt2`、`LowerArmExt`、`UpperTrunkRem`）
9. 躯干上部 + 肩膀 + 大臂 + 小臂（两侧）（`UpperTrunkExt3`、`UpperTrunkRem`、`Hand`）
10. 躯干上部 + 肩膀 + 大臂 + 小臂 + 手（两侧）（`UpperTrunkExt4`、`UpperTrunkRem`）
11. 头 + 躯干上部（`HeadExt1`、`ClavicleExt3`、`UpperTrunkRem`）
12. 头 + 躯干上部 + 肩膀（两侧）（`HeadExt2`、`UpperArmExt2`、`UpperTrunkRem`）
13. 头 + 躯干上部 + 肩膀 + 大臂（两侧）（`HeadExt3`、`LowerArmExt`、`UpperTrunkRem`）
14. 头 + 躯干上部 + 肩膀 + 大臂 + 小臂（两侧）（`HeadExt4`、`UpperTrunkRem`、`Hand`）
15. 头 + 躯干上部 + 肩膀 + 大臂 + 小臂 + 手（两侧）（`HeadExt5`、`UpperTrunkRem`）
16. 小腿 + 脚（`CalfExt`、`CalfRem`）
17. 大腿 + 小腿（`ThighExt1`、`ThighRem1`）
18. 大腿 + 小腿 + 脚（`ThighExt2`、`ThighRem2`）
19. 躯干下部 + 大腿（`LowerTrunkExt1`、`LowerTrunkRem1`）
20. 躯干下部 + 大腿 + 小腿（`LowerTrunkExt2`、`LowerTrunkRem2`）
21. 躯干下部 + 大腿 + 小腿 + 脚（`LowerTrunkExt3`、`LowerTrunkRem3`）





