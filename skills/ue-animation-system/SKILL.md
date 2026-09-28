---
name: ue-animation-system
description: "Use when writing or debugging Unreal Engine animation C++: AnimInstance subclasses, thread-safe update, FAnimInstanceProxy, montage playback and delegates, anim notifies, animation curves, blend spaces, anim state machines, linked anim layers, root motion. Also use when the user mentions 'UAnimInstance', 'NativeUpdateAnimation', 'NativeThreadSafeUpdateAnimation', 'Montage_Play', 'OnMontageEnded', 'UAnimNotify', 'UAnimNotifyState', 'branching point', 'GetCurveValue', 'blend space', 'AnimGraph', 'LinkAnimClassLayers', 'motion matching', 'motion warping', 'root motion'. For movement and RootMotionSource, see ue-character-movement; for montage-driven abilities, see ue-gameplay-abilities."
metadata:
  version: "2.0.0"
  engine: "5.8"
---

# UE Animation System

Target engine: **UE 5.8**. APIs below are verified against the 5.8 headers; older forms are listed under "Deprecated — do not use".

Covers the runtime animation path in the `Engine` module — `UAnimInstance`, `FAnimInstanceProxy`, `UAnimMontage`, `UAnimNotify`/`UAnimNotifyState`, curves, blend spaces, anim state machines, linked anim layers and root motion — plus the animation plugins shipped in 5.8 (`PoseSearch`, `MotionWarping`, `Chooser`, `IKRig`, `ControlRig`, `BlendStack`, `OptimusCore`, `UAF`). Build.cs: `Engine` for everything under `Runtime/Engine/Classes/Animation`, `AnimGraphRuntime` for the helper anim nodes such as `FAnimNode_LayeredBoneBlend`; plugin module names are listed per row below.

## Context

Read `.agents/ue-project-context.md` if it exists (module names, conventions, enabled plugins, GAS/networking setup). Do not stop if it is missing.

Identify the area from the request and the codebase. Ask only when two plausible readings would produce different code.

| Request is about… | Go to |
|---|---|
| AnimInstance subclass, thread-safe update, proxy | [AnimInstance and Proxy](#animinstance-and-proxy) |
| Playing/stopping montages, section jumps, end callbacks | [Montages](#montages) |
| Notifies, notify states, branching points, notify timing | [Anim Notifies](#anim-notifies) |
| Curve values, morph targets, curve metadata | [Animation Curves](#animation-curves) |
| Blend spaces, state machines, aim offsets, layered blends | [Blend Spaces and State Machines](#blend-spaces-and-state-machines) |
| Linked anim layers, per-mode locomotion swapping | [Linked Anim Graphs and Layers](#linked-anim-graphs-and-layers) |
| Root motion extraction and modes | [Root Motion](#root-motion) |
| Motion matching, warping, IK Rig, Control Rig, deformers | [5.8 Animation Plugins](#58-animation-plugins) |

## AnimInstance and Proxy

`UAnimInstance` (`Animation/AnimInstance.h`) runs in two phases. Game thread: `NativeUpdateAnimation` — safe to read gameplay state. Worker thread: `NativeThreadSafeUpdateAnimation` and blend-tree evaluation. Cache gameplay reads on the game thread; read only those cached values on the worker thread.

Verbatim virtuals (`AnimInstance.h:1436-1457` public, `:1716-1719` protected):

```cpp
virtual void NativeInitializeAnimation();
virtual void NativeUpdateAnimation(float DeltaSeconds);
virtual void NativeThreadSafeUpdateAnimation(float DeltaSeconds);
virtual void NativePostEvaluateAnimation();
virtual void NativeUninitializeAnimation();
virtual void NativeBeginPlay();
virtual FAnimInstanceProxy* CreateAnimInstanceProxy();               // protected
virtual void DestroyAnimInstanceProxy(FAnimInstanceProxy* InProxy);  // protected
```

`FAnimInstanceProxy` (`Public/Animation/AnimInstanceProxy.h`) is the worker-thread data container. Its override points are **protected** (`:603-669`):

```cpp
virtual void Initialize(UAnimInstance* InAnimInstance);
virtual void PreUpdate(UAnimInstance* InAnimInstance, float DeltaSeconds);
virtual void Update(float DeltaSeconds) {}
virtual bool Evaluate(FPoseContext& Output) { return false; }
virtual void PostUpdate(UAnimInstance* InAnimInstance) const;
virtual void Uninitialize(UAnimInstance* InAnimInstance);
```

Reach the proxy with the protected templates `GetProxyOnGameThread<T>()` (`:1746`) and `GetProxyOnAnyThread<T>()` (`:1769`).

```cpp
// MyAnimInstance.h
#pragma once

#include "CoreMinimal.h"
#include "Animation/AnimInstance.h"
#include "Animation/AnimInstanceProxy.h"
#include "MyAnimInstance.generated.h"

class ACharacter;
class UCharacterMovementComponent;

USTRUCT()
struct FMyAnimInstanceProxy : public FAnimInstanceProxy
{
    GENERATED_BODY()

    FMyAnimInstanceProxy() = default;
    explicit FMyAnimInstanceProxy(UAnimInstance* InAnimInstance)
        : FAnimInstanceProxy(InAnimInstance) {}

    // Written on the game thread in PreUpdate, read on the worker thread in Update.
    FVector Velocity = FVector::ZeroVector;
    float Speed = 0.f;
    bool bIsFalling = false;

protected:
    virtual void PreUpdate(UAnimInstance* InAnimInstance, float DeltaSeconds) override;
    virtual void Update(float DeltaSeconds) override;
};

UCLASS()
class MYGAME_API UMyAnimInstance : public UAnimInstance
{
    GENERATED_BODY()

public:
    virtual void NativeInitializeAnimation() override;
    virtual void NativeUpdateAnimation(float DeltaSeconds) override;
    virtual void NativeThreadSafeUpdateAnimation(float DeltaSeconds) override;
    virtual void NativeUninitializeAnimation() override;

protected:
    virtual FAnimInstanceProxy* CreateAnimInstanceProxy() override;
    virtual void DestroyAnimInstanceProxy(FAnimInstanceProxy* InProxy) override;

    UPROPERTY(Transient)
    TObjectPtr<ACharacter> OwningCharacter;

    UPROPERTY(Transient)
    TObjectPtr<UCharacterMovementComponent> MovementComp;

    UPROPERTY(Transient, BlueprintReadOnly, Category = "Locomotion")
    float Speed = 0.f;

    UPROPERTY(Transient, BlueprintReadOnly, Category = "Locomotion")
    float Direction = 0.f;

    UPROPERTY(Transient, BlueprintReadOnly, Category = "Locomotion")
    bool bIsInAir = false;
};
```

```cpp
// MyAnimInstance.cpp
#include "MyAnimInstance.h"
#include "GameFramework/Character.h"
#include "GameFramework/CharacterMovementComponent.h"
#include "Kismet/KismetMathLibrary.h"

void FMyAnimInstanceProxy::PreUpdate(UAnimInstance* InAnimInstance, float DeltaSeconds)
{
    FAnimInstanceProxy::PreUpdate(InAnimInstance, DeltaSeconds);

    // Game thread: safe to touch the owning actor here.
    if (const ACharacter* Character = Cast<ACharacter>(InAnimInstance->TryGetPawnOwner()))
    {
        const UCharacterMovementComponent* Movement = Character->GetCharacterMovement();
        Velocity = Movement->Velocity;
        bIsFalling = Movement->IsFalling();
    }
}

void FMyAnimInstanceProxy::Update(float DeltaSeconds)
{
    FAnimInstanceProxy::Update(DeltaSeconds);
    Speed = Velocity.Size2D(); // Worker thread: proxy members only.
}

FAnimInstanceProxy* UMyAnimInstance::CreateAnimInstanceProxy()
{
    return new FMyAnimInstanceProxy(this);
}

void UMyAnimInstance::DestroyAnimInstanceProxy(FAnimInstanceProxy* InProxy)
{
    delete InProxy;
}

void UMyAnimInstance::NativeInitializeAnimation()
{
    Super::NativeInitializeAnimation();

    OwningCharacter = Cast<ACharacter>(TryGetPawnOwner());
    if (OwningCharacter)
    {
        MovementComp = OwningCharacter->GetCharacterMovement();
    }
}

void UMyAnimInstance::NativeUpdateAnimation(float DeltaSeconds)
{
    Super::NativeUpdateAnimation(DeltaSeconds);

    if (!OwningCharacter || !MovementComp)
    {
        return;
    }

    const FVector CurrentVelocity = MovementComp->Velocity;
    Speed = CurrentVelocity.Size2D();
    bIsInAir = MovementComp->IsFalling();

    if (Speed > 3.f)
    {
        Direction = UKismetMathLibrary::NormalizedDeltaRotator(
            CurrentVelocity.ToOrientationRotator(),
            OwningCharacter->GetActorRotation()).Yaw;
    }
}

void UMyAnimInstance::NativeThreadSafeUpdateAnimation(float DeltaSeconds)
{
    Super::NativeThreadSafeUpdateAnimation(DeltaSeconds);

    // Worker thread: read the proxy, never the owning actor. This runs before the
    // proxy's Update(), so read what PreUpdate wrote, not what Update derives.
    const FMyAnimInstanceProxy& Proxy = GetProxyOnAnyThread<FMyAnimInstanceProxy>();
    Speed = Proxy.Velocity.Size2D();
    bIsInAir = Proxy.bIsFalling;
}

void UMyAnimInstance::NativeUninitializeAnimation()
{
    Super::NativeUninitializeAnimation();
    OwningCharacter = nullptr;
    MovementComp = nullptr;
}
```

Query pending-update state with `NeedsUpdate()` (`AnimInstance.h:512`); the `bNeedsUpdate` member is deprecated.

## Montages

Source: `Animation/AnimInstance.h`, `Animation/AnimMontage.h`. Verbatim signatures (`AnimInstance.h:626-796`):

```cpp
float Montage_Play(UAnimMontage* MontageToPlay, float InPlayRate = 1.f,
    EMontagePlayReturnType ReturnValueType = EMontagePlayReturnType::MontageLength,
    float InTimeToStartMontageAt = 0.f, bool bStopAllMontages = true);
// Same trailing params; override the asset's blend-in (FMontageBlendSettings adds BlendProfile and
// BlendMode, e.g. EMontageBlendMode::Inertialization)
float Montage_PlayWithBlendIn(UAnimMontage* MontageToPlay, const FAlphaBlendArgs& BlendIn, ...);
float Montage_PlayWithBlendSettings(UAnimMontage* MontageToPlay, const FMontageBlendSettings& BlendInSettings, ...);
void  Montage_Stop(float InBlendOutTime, const UAnimMontage* Montage = NULL);
void  Montage_Pause(const UAnimMontage* Montage = NULL);
void  Montage_Resume(const UAnimMontage* Montage);
void  Montage_JumpToSection(FName SectionName, const UAnimMontage* Montage = NULL);
void  Montage_SetNextSection(FName SectionNameToChange, FName NextSection,
          const UAnimMontage* Montage = NULL);
void  Montage_SetPlayRate(const UAnimMontage* Montage, float NewPlayRate = 1.f);
bool  Montage_IsPlaying(const UAnimMontage* Montage) const;
bool  Montage_IsActive(const UAnimMontage* Montage) const;
bool  Montage_GetIsStopped(const UAnimMontage* Montage) const;
float Montage_GetPosition(const UAnimMontage* Montage) const;
void  Montage_SetPosition(const UAnimMontage* Montage, float NewPosition);
float Montage_GetBlendTime(const UAnimMontage* Montage) const;
float Montage_GetPlayRate(const UAnimMontage* Montage) const;
FName Montage_GetCurrentSection(const UAnimMontage* Montage = NULL) const;
void  Montage_SetEndDelegate(FOnMontageEnded& InOnMontageEnded, UAnimMontage* Montage = NULL);
void  Montage_SetBlendingOutDelegate(FOnMontageBlendingOutStarted& InOnMontageBlendingOut,
          UAnimMontage* Montage = NULL);
```

`EMontagePlayReturnType` (`AnimInstance.h:70`) is `MontageLength` or `Duration`. Once blend-out starts, `Montage_IsPlaying` and `Montage_IsActive` return false and `Montage_GetIsStopped` returns true, even though the pose is still blending (`Stop()` removes the instance from `ActiveMontagesMap`, `AnimInstance.cpp:3709`). Use `OnMontageBlendingOut` / `Montage_SetBlendingOutDelegate` for blend-out start and `OnMontageEnded` for the end. Delegate types (`AnimInstance.h:79-80`) are single-cast:

```cpp
DECLARE_DELEGATE_TwoParams(FOnMontageEnded, UAnimMontage*, bool /*bInterrupted*/)
DECLARE_DELEGATE_TwoParams(FOnMontageBlendingOutStarted, UAnimMontage*, bool /*bInterrupted*/)
```

For broadcast, use the dynamic multicast members `OnMontageStarted`, `OnMontageEnded` and `OnMontageBlendingOut` (`:758-770`).

```cpp
void UMyWeaponComponent::PlayAttackMontage(USkeletalMeshComponent* Mesh, UAnimMontage* Montage)
{
    UAnimInstance* AnimInst = Mesh ? Mesh->GetAnimInstance() : nullptr;
    if (!AnimInst || !Montage)
    {
        return;
    }

    // Play FIRST: the delegate setters resolve the active montage instance,
    // which does not exist until Montage_Play has created it.
    if (AnimInst->Montage_Play(Montage) <= 0.f)
    {
        return;
    }

    FOnMontageEnded EndDelegate;
    EndDelegate.BindUObject(this, &UMyWeaponComponent::OnAttackEnded);
    AnimInst->Montage_SetEndDelegate(EndDelegate, Montage);

    FOnMontageBlendingOutStarted BlendOutDelegate;
    BlendOutDelegate.BindUObject(this, &UMyWeaponComponent::OnAttackBlendingOut);
    AnimInst->Montage_SetBlendingOutDelegate(BlendOutDelegate, Montage);
}

void UMyWeaponComponent::OnAttackEnded(UAnimMontage* Montage, bool bInterrupted) {}
void UMyWeaponComponent::OnAttackBlendingOut(UAnimMontage* Montage, bool bInterrupted) {}
```

`USkeletalMeshComponent` (`Components/SkeletalMeshComponent.h`):

```cpp
UAnimInstance* GetAnimInstance() const;                                  // :1111
void PlayAnimation(class UAnimationAsset* NewAnimToPlay, bool bLooping); // :1258
virtual void SetAnimInstanceClass(class UClass* NewClass);               // :1096
void LinkAnimClassLayers(TSubclassOf<UAnimInstance> InClass);            // :1194
void UnlinkAnimClassLayers(TSubclassOf<UAnimInstance> InClass);          // :1205
UAnimInstance* GetLinkedAnimGraphInstanceByTag(FName InTag) const;       // :1160
void SetAnimationMode(EAnimationMode::Type InAnimationMode,
    bool bForceInitAnimScriptInstance = true);                           // :1246
```

`PlayAnimation` puts the component in single-node mode; `SetAnimInstanceClass` puts it in Animation Blueprint mode. Mixing both on one component silently drops one of them.

Multiplayer: with GAS, drive montages through `UAbilityTask_PlayMontageAndWait::CreatePlayMontageAndWaitProxy` (`GameplayAbilities/Public/Abilities/Tasks/AbilityTask_PlayMontageAndWait.h:66`) — see `ue-gameplay-abilities`. Without GAS, replicate the montage choice yourself and call `Montage_Play` from the replication callback; never call it independently per net role.

## Anim Notifies

Source: `Animation/AnimNotifies/AnimNotify.h`, `AnimNotifyState.h`. Override these exact signatures:

```cpp
// UAnimNotify (AnimNotify.h:85-86, :59)
virtual void Notify(USkeletalMeshComponent* MeshComp, UAnimSequenceBase* Animation,
    const FAnimNotifyEventReference& EventReference);
virtual void BranchingPointNotify(FBranchingPointNotifyPayload& BranchingPointPayload);
FString GetNotifyName() const;  // BlueprintNativeEvent: override GetNotifyName_Implementation

// UAnimNotifyState (AnimNotifyState.h:74-80)
virtual void NotifyBegin(USkeletalMeshComponent* MeshComp, UAnimSequenceBase* Animation,
    float TotalDuration, const FAnimNotifyEventReference& EventReference);
virtual void NotifyTick(USkeletalMeshComponent* MeshComp, UAnimSequenceBase* Animation,
    float FrameDeltaTime, const FAnimNotifyEventReference& EventReference);
virtual void NotifyEnd(USkeletalMeshComponent* MeshComp, UAnimSequenceBase* Animation,
    const FAnimNotifyEventReference& EventReference);
```

`Received_Notify` (`AnimNotify.h:62`) and `Received_NotifyBegin`/`Received_NotifyTick`/`Received_NotifyEnd` (`AnimNotifyState.h:45-51`) are `BlueprintImplementableEvent` entry points — implement them in Blueprint, never override them in C++.

Montage notify timing is per event: `FAnimNotifyEvent::MontageTickType` (`Public/Animation/AnimTypes.h:318`) is `EMontageNotifyTickType::Queued` (fired at the end of the evaluation phase — the header states it is not suitable for changing sections or montage position) or `BranchingPoint` (fired as encountered, suitable for section changes). A native notify forces the branching-point path by setting `bIsNativeBranchingPoint = true` in its constructor (`AnimNotify.h:123`, `AnimNotifyState.h:108`).

Bind named montage notifies from outside the AnimInstance with `OnPlayMontageNotifyBegin` / `OnPlayMontageNotifyEnd` (`AnimInstance.h:1820-1823`, type `FPlayMontageAnimNotifyDelegate`).

See [references/anim-notify-reference.md](references/anim-notify-reference.md) for the built-in notify catalog, complete custom notify and notify-state examples, `FAnimNotifyEvent` fields and linked-instance propagation.

## Animation Curves

Curves are addressed by `FName` in 5.8 — no smart names, no curve UIDs, no mapping lookups.

```cpp
// UAnimInstance (AnimInstance.h:1241-1273)
float GetCurveValue(FName CurveName) const;
bool  GetCurveValue(FName CurveName, float& OutValue) const;
bool  GetCurveValueWithDefault(FName CurveName, float DefaultValue, float& OutValue) const;
void  SetMorphTarget(FName MorphTargetName, float Value);

// USkeletalMeshComponent (SkeletalMeshComponent.h:664-667)
bool GetCurveValue(FName CurveName, float DefaultValue, float& Value) const;
const FBlendedHeapCurve& GetAnimCurves() const;
```

Per-curve flags (does the curve drive a material or a morph target) live in `CurveMetaData` on the skeleton (`Animation/Skeleton.h:387-467`): `GetCurveMetaData(FName)`, `AddCurveMetaData(FName CurveName, bool bTransact = true)`, `GetCurveMetaDataNames(TArray<FName>& OutNames)`, `SetCurveMetaDataMorphTarget(FName CurveName, bool bOverrideMorphTarget)`.

## Blend Spaces and State Machines

Blend spaces are assets sampled by AnimGraph nodes; drive them from `UPROPERTY` members the AnimGraph reads.

| Asset class | Header | Use |
|---|---|---|
| `UBlendSpace` | `Animation/BlendSpace.h:470` | Two-axis blend, typically Direction x Speed |
| `UBlendSpace1D` | `Animation/BlendSpace1D.h:19` | Single axis, typically Speed |
| `UAimOffsetBlendSpace` | `Animation/AimOffsetBlendSpace.h:20` | Additive aim poses over a base pose |

Axis smoothing is `FInterpolationParameter` (`BlendSpace.h:80-110`): `InterpolationTime`, `DampingRatio`, `MaxSpeed`, and `InterpolationType` of type `TEnumAsByte<EFilterInterpolationType>` — values `BSIT_Average`, `BSIT_Linear`, `BSIT_Cubic`, `BSIT_EaseInOut`, `BSIT_ExponentialDecay`, `BSIT_SpringDamper` (`Engine/EngineTypes.h:1356`).

Bind native C++ to a compiled state machine from `NativeInitializeAnimation` (`AnimInstance.h:1461-1473`):

```cpp
// CanStartMoving() and OnLandEntered() are members of UMyAnimInstance.
// DECLARE_DELEGATE_RetVal(bool, FCanTakeTransition)                     AnimInstance.h:118
// DECLARE_DELEGATE_ThreeParams(FOnGraphStateChanged,
//     const struct FAnimNode_StateMachine&, int32, int32)              AnimInstance.h:121
AddNativeTransitionBinding(FName("LocomotionSM"), FName("Idle"), FName("Walk"),
    FCanTakeTransition::CreateUObject(this, &UMyAnimInstance::CanStartMoving),
    FName("IdleToMoving"));

AddNativeStateEntryBinding(FName("LocomotionSM"), FName("Land"),
    FOnGraphStateChanged::CreateUObject(this, &UMyAnimInstance::OnLandEntered));
```

Query at runtime with `GetStateMachineIndex(FName MachineName)` (`:1202`), `GetStateMachineInstanceFromName(FName MachineName)` (`:1191`) and `GetInstanceStateWeight(int32 MachineIndex, int32 StateIndex)` (`:1116`).

See [references/locomotion-setup.md](references/locomotion-setup.md) for a full blend space, state machine, layered-blend-per-bone and aim-offset setup.

## Linked Anim Graphs and Layers

Source: `Animation/AnimNode_LinkedAnimGraph.h`, `Animation/AnimNode_LinkedAnimLayer.h`.

```cpp
// UAnimInstance (AnimInstance.h:850-910)
virtual void LinkAnimClassLayers(TSubclassOf<UAnimInstance> InClass);
virtual void UnlinkAnimClassLayers(TSubclassOf<UAnimInstance> InClass);
UAnimInstance* GetLinkedAnimLayerInstanceByClass(TSubclassOf<UAnimInstance> InClass,
    bool bCheckForChildClass = false) const;
void LinkAnimGraphByTag(FName InTag, TSubclassOf<UAnimInstance> InClass);
UAnimInstance* GetLinkedAnimGraphInstanceByTag(FName InTag) const;
```

`LinkAnimClassLayers(nullptr)` restores the default layer implementations. Notify flow across linked instances is set by `SetReceiveNotifiesFromLinkedInstances(bool bSet)` and `SetPropagateNotifiesToLinkedInstances(bool bSet)` (`:523-531`), or per node by `bReceiveNotifiesFromLinkedInstances` / `bPropagateNotifiesToLinkedInstances` (`AnimNode_LinkedAnimGraph.h:76-80`). `SetUseMainInstanceMontageEvaluationData(bool bSet)` (`:537`) makes linked layers share the main instance's montage evaluation.

Blend the swap with `PendingBlendInDuration` / `PendingBlendOutDuration` on the linked node (`AnimNode_LinkedAnimGraph.h:60-67`), or request one from code:

```cpp
// AnimInstance.h:1006
AnimInst->RequestSlotGroupInertialization(FName("DefaultGroup"), 0.2f);
```

## Root Motion

`ERootMotionMode::Type` (`Animation/AnimEnums.h:27-42`): `NoRootMotionExtraction`, `IgnoreRootMotion`, `RootMotionFromEverything`, `RootMotionFromMontagesOnly`. The header marks `RootMotionFromEverything` as unsuitable for networked multiplayer and `RootMotionFromMontagesOnly` as the networked choice.

```cpp
// AnimInstance.h:372, :1066, :1662
TEnumAsByte<ERootMotionMode::Type> RootMotionMode;   // UPROPERTY(EditDefaultsOnly)
void SetRootMotionMode(TEnumAsByte<ERootMotionMode::Type> Value);
FRootMotionMovementParams ConsumeExtractedRootMotion(float Alpha);

// Suppress root motion for part of a montage (AnimMontage.h:562-564)
if (FAnimMontageInstance* Inst = AnimInst->GetActiveInstanceForMontage(Montage))
{
    Inst->PushDisableRootMotion();
    // ... later ...
    Inst->PopDisableRootMotion();
}
```

Montage root motion and `RootMotionSource` are separate systems: the montage path feeds the pose-derived delta into the movement component, while `RootMotionSource` applies scripted movement with no animation attached. Networked prediction, correction and `bAllowPhysicsRotationDuringAnimRootMotion` (`GameFramework/CharacterMovementComponent.h:1083`) belong to `ue-character-movement`.

## 5.8 Animation Plugins

Each is disabled by default in the .uproject unless noted; add the module to Build.cs when you use its types.

| Plugin / module | 5.8 maturity | Key types and headers |
|---|---|---|
| `PoseSearch` | production | `FAnimNode_MotionMatching` (`PoseSearch/AnimNode_MotionMatching.h:18`), `UPoseSearchDatabase` (`PoseSearch/PoseSearchDatabase.h:502`); trajectories are `FTransformTrajectorySample` (`Runtime/Engine/Public/Animation/TrajectoryTypes.h:18`) |
| `Chooser` | production | `UChooserTable` (`Chooser.h:22`), `UChooserFunctionLibrary` (`ChooserFunctionLibrary.h:18`) — data-driven asset selection, commonly feeding motion matching |
| `MotionWarping` | Beta in 5.8 | `UMotionWarpingComponent` (`MotionWarpingComponent.h:100`), `URootMotionModifier` (`RootMotionModifier.h:71`); rotation offsets use `AdditionalRotationOffset` (`RootMotionModifier.h:407`) |
| `IKRig` | production, enabled by default | `UIKRigDefinition` (`Rig/IKRigDefinition.h:206`), `UIKRetargeter` (`Retargeter/IKRetargeter.h:70`) — runtime IK solving and animation retargeting |
| `ControlRig` | production, enabled by default | `UControlRig` (`ControlRig.h:63`), `FAnimNode_ControlRig` (`AnimNode_ControlRig.h:21`) — procedural rigs evaluated inside the AnimGraph |
| `BlendStack` | production | `FAnimNode_BlendStack` and `FAnimNode_BlendStack_Standalone` (`BlendStack/AnimNode_BlendStack.h:272,150`) — the blend stack motion matching is built on |
| `OptimusCore` (Deformer Graph) | Beta in 5.8 | `UOptimusDeformer` (`OptimusDeformer.h:132`) — GPU mesh deformation graphs |
| `UAF` | Experimental in 5.8 | `Plugins/Experimental/UAF/UAF`, module `UAF`: `UUAFRigVMAsset` (`AnimNextRigVMAsset.h:61`), `UUAFComponent` (`Component/AnimNextComponent.h:77`) — the successor animation framework |

## Deprecated — do not use

| Do not emit | Use in 5.8 | Source |
|---|---|---|
| `FSmartName`, `FSmartNameMapping`, `FSmartNameContainer` | plain `FName` curve names | `UE_DEPRECATED(5.8)` in `Animation/SmartName.h:20,80,121` |
| Skeleton smart-name curve tables | `CurveMetaData` accessors on `USkeleton` | `UE_DEPRECATED(5.7)` in `Animation/Skeleton.h:505` |
| `AnimCurveUID` | `FName` curve names | `UE_DEPRECATED(5.7)` in `Animation/Skeleton.h:384` |
| `UAnimInstance::bNeedsUpdate` | `NeedsUpdate()` | `UE_DEPRECATED(5.7)` in `Animation/AnimInstance.h:389` |
| `USkeletalMeshComponent::SetAnimClass` | `SetAnimInstanceClass` | `UE_DEPRECATED(5.5)` in `Components/SkeletalMeshComponent.h:1084` |
| `USkeletalMeshComponent::AnimCurves` member | `GetAnimCurves()` / `GetCurveValue()` | `UE_DEPRECATED(5.6)` in `Components/SkeletalMeshComponent.h:470` |
| `UAnimSequence::RetargetSourceAsset` member | `GetRetargetSourceAsset` / `SetRetargetSourceAsset` | `UE_DEPRECATED(5.5)` in `Animation/AnimSequence.h:305` |
| `FTrajectorySample` | `FTransformTrajectorySample` | `UE_DEPRECATED(5.6)` in `Public/Animation/MotionTrajectoryTypes.h:10` |
| `FPoseSearchQueryTrajectorySample` | `FTransformTrajectorySample` | `UE_DEPRECATED(5.6)` in `PoseSearch/PoseSearchTrajectoryTypes.h:16` |
| `FPoseSearchDatabaseSequence`, `FPoseSearchDatabaseBlendSpace` | `FPoseSearchDatabaseAnimationAsset` | `UE_DEPRECATED(5.7)` in `PoseSearch/PoseSearchDatabase.h:228` |
| `EMotionWarpRotationType::OppositeDefault` / `OppositeFacing` | `AdditionalRotationOffset` | `UE_DEPRECATED(5.8)` in `MotionWarping/RootMotionModifier.h:309,312` |
| `FAnimInstanceProxy::GetSlotInertializationRequest` | `GetSlotInertializationRequestData` | `UE_DEPRECATED(5.5)` in `Public/Animation/AnimInstanceProxy.h:444` |
| `SetSubInstanceClassByTag`, `GetSubInstanceByTag` | `LinkAnimGraphByTag`, `GetLinkedAnimGraphInstanceByTag` | `UE_DEPRECATED(4.24)` in `Animation/AnimInstance.h:860,845` |
| `SetLayerOverlay`, `ClearLayerOverlay` | `LinkAnimClassLayers`, `UnlinkAnimClassLayers` | `UE_DEPRECATED(4.24)` in `Animation/AnimInstance.h:868,880` |
| `GetLayerSubInstanceByClass` | `GetLinkedAnimLayerInstanceByClass` | `UE_DEPRECATED(4.24)` in `Animation/AnimInstance.h:906` |

## Common Mistakes

**Touching the owning actor in `NativeThreadSafeUpdateAnimation`:** that function runs on a worker thread, so `TryGetPawnOwner()`, component queries and traces are unsafe there. Cache what you need in `NativeUpdateAnimation` or in `FAnimInstanceProxy::PreUpdate`, and read only those members on the worker thread.

**Binding montage delegates before `Montage_Play`:** `Montage_SetEndDelegate` and `Montage_SetBlendingOutDelegate` resolve the active montage instance, which does not exist yet. Call `Montage_Play` first, check the return value is greater than zero, then bind.

**Overriding `Received_Notify` in C++:** it is a `BlueprintImplementableEvent`. Override `Notify(USkeletalMeshComponent*, UAnimSequenceBase*, const FAnimNotifyEventReference&)` instead.

**Jumping sections from a queued notify:** queued notifies fire after the evaluation phase, so `Montage_JumpToSection` from one lands a frame late. Set `bIsNativeBranchingPoint = true` and override `BranchingPointNotify` for section control.

**Looking curves up by smart name or UID:** `FSmartName`, `FSmartNameMapping` and `AnimCurveUID` are deprecated. Pass `FName` to `GetCurveValue` and keep per-curve flags in `CurveMetaData`.

**Declaring a proxy override public:** `Initialize`, `PreUpdate`, `Update`, `Evaluate` and `PostUpdate` are protected on `FAnimInstanceProxy`, and so are `CreateAnimInstanceProxy` / `DestroyAnimInstanceProxy` on `UAnimInstance`. Keep the access level or the override will not compile.

**Skipping the base call:** every `Native*` override calls its `Super::` form, and every proxy override calls `FAnimInstanceProxy::<Name>`; the base implementations drive montage ticking, notify queues and proxy bookkeeping.

## Related Skills

- `ue-character-movement` — `UCharacterMovementComponent`, `RootMotionSource`, networked root motion prediction and correction.
- `ue-gameplay-abilities` — `UAbilityTask_PlayMontageAndWait`, GAS montage replication, ability-driven animation.
- `ue-actor-component-architecture` — `USkeletalMeshComponent` setup, attachment, component tick ordering.
- `ue-sequencer-cinematics` — Level Sequence animation tracks and cinematic playback on skeletal meshes.
- `ue-mass-entity` — crowd-scale animation through Mass representation instead of per-actor AnimInstances.
- `ue-cpp-foundations` — delegate binding, `UPROPERTY`/`UFUNCTION` specifiers, `TObjectPtr`.
- `ue-niagara-effects` — Niagara systems spawned from notifies and skeletal mesh data interfaces.
- `ue-audio-system` — UAudioComponent, MetaSounds, submixes, attenuation and concurrency
- `ue-mover` — the Mover plugin: movement modes, layered moves and rollback networking
