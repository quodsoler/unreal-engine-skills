# Locomotion Setup Reference (UE 5.8)

End-to-end character locomotion: the AnimInstance that feeds the AnimGraph, blend
spaces, the anim state machine, layered blending, linked layers and root motion.

Source headers:
- `Engine/Source/Runtime/Engine/Classes/Animation/AnimInstance.h`
- `Engine/Source/Runtime/Engine/Classes/Animation/BlendSpace.h`, `BlendSpace1D.h`, `AimOffsetBlendSpace.h`
- `Engine/Source/Runtime/Engine/Classes/Animation/AnimEnums.h` (`ERootMotionMode`)
- `Engine/Source/Runtime/Engine/Classes/Animation/AnimNode_LinkedAnimGraph.h`, `AnimNode_LinkedAnimLayer.h`
- `Engine/Source/Runtime/AnimGraphRuntime/Public/AnimNodes/AnimNode_LayeredBoneBlend.h`
- `Engine/Source/Runtime/Engine/Classes/Animation/AnimData/BoneMaskFilter.h` (`FInputBlendPose`, `FBranchFilter`)

Build.cs: `Engine` plus `AnimGraphRuntime` when you reference anim node structs such as `FAnimNode_LayeredBoneBlend` directly.

---

## 1. The locomotion AnimInstance

Everything the AnimGraph samples is a `UPROPERTY` written on the game thread in
`NativeUpdateAnimation`.

```cpp
// MyLocomotionAnimInstance.h
#pragma once

#include "CoreMinimal.h"
#include "Animation/AnimInstance.h"
#include "MyLocomotionAnimInstance.generated.h"

class ACharacter;
class UCharacterMovementComponent;
struct FAnimNode_StateMachine;

UCLASS()
class MYGAME_API UMyLocomotionAnimInstance : public UAnimInstance
{
    GENERATED_BODY()

public:
    virtual void NativeInitializeAnimation() override;
    virtual void NativeUpdateAnimation(float DeltaSeconds) override;

protected:
    bool CanStartMoving() const;
    void OnLandStateEntered(const FAnimNode_StateMachine& Machine, int32 PrevStateIndex, int32 NextStateIndex);

    UPROPERTY(Transient)
    TObjectPtr<ACharacter> OwningCharacter;

    UPROPERTY(Transient)
    TObjectPtr<UCharacterMovementComponent> MovementComp;

    /** Ground speed on the XY plane; drives the blend space speed axis. */
    UPROPERTY(Transient, BlueprintReadOnly, Category = "Locomotion")
    float GroundSpeed = 0.f;

    /** Movement direction relative to actor facing, -180 to 180. */
    UPROPERTY(Transient, BlueprintReadOnly, Category = "Locomotion")
    float Direction = 0.f;

    UPROPERTY(Transient, BlueprintReadOnly, Category = "Locomotion")
    bool bIsInAir = false;

    UPROPERTY(Transient, BlueprintReadOnly, Category = "Locomotion")
    bool bIsIdle = true;

    UPROPERTY(Transient, BlueprintReadOnly, Category = "Locomotion")
    bool bIsCrouching = false;

    UPROPERTY(Transient, BlueprintReadOnly, Category = "Locomotion")
    float Acceleration = 0.f;

    UPROPERTY(Transient, BlueprintReadOnly, Category = "Locomotion")
    float LeanAmount = 0.f;

    UPROPERTY(Transient, BlueprintReadOnly, Category = "AimOffset")
    float AimYaw = 0.f;

    UPROPERTY(Transient, BlueprintReadOnly, Category = "AimOffset")
    float AimPitch = 0.f;

private:
    float PreviousActorYaw = 0.f;
};
```

```cpp
// MyLocomotionAnimInstance.cpp
#include "MyLocomotionAnimInstance.h"
#include "GameFramework/Character.h"
#include "GameFramework/CharacterMovementComponent.h"
#include "Kismet/KismetMathLibrary.h"

namespace
{
    constexpr float IdleSpeedThreshold = 10.f;
    constexpr float LeanInterpSpeed = 4.f;
    constexpr float LeanMaxAngle = 30.f;
}

void UMyLocomotionAnimInstance::NativeInitializeAnimation()
{
    Super::NativeInitializeAnimation();

    OwningCharacter = Cast<ACharacter>(TryGetPawnOwner());
    if (OwningCharacter)
    {
        MovementComp = OwningCharacter->GetCharacterMovement();
        PreviousActorYaw = OwningCharacter->GetActorRotation().Yaw;
    }

    // Native bindings must be registered here, before the graph evaluates.
    AddNativeTransitionBinding(FName("LocomotionSM"), FName("Idle"), FName("Move"),
        FCanTakeTransition::CreateUObject(this, &UMyLocomotionAnimInstance::CanStartMoving),
        FName("IdleToMoving"));

    AddNativeStateEntryBinding(FName("LocomotionSM"), FName("Land"),
        FOnGraphStateChanged::CreateUObject(this, &UMyLocomotionAnimInstance::OnLandStateEntered));
}

void UMyLocomotionAnimInstance::NativeUpdateAnimation(float DeltaSeconds)
{
    Super::NativeUpdateAnimation(DeltaSeconds);

    if (!OwningCharacter || !MovementComp || DeltaSeconds <= 0.f)
    {
        return;
    }

    const FVector Velocity = MovementComp->Velocity;
    const FRotator ActorRot = OwningCharacter->GetActorRotation();

    GroundSpeed = Velocity.Size2D();
    bIsInAir = MovementComp->IsFalling();
    bIsCrouching = MovementComp->IsCrouching();
    bIsIdle = GroundSpeed < IdleSpeedThreshold && !bIsInAir;
    Acceleration = MovementComp->GetCurrentAcceleration().Size2D();

    // Keep the last valid direction while decelerating so the blend space does not snap.
    if (GroundSpeed > IdleSpeedThreshold)
    {
        Direction = UKismetMathLibrary::NormalizedDeltaRotator(
            Velocity.ToOrientationRotator(), ActorRot).Yaw;
    }

    const FRotator AimDelta = UKismetMathLibrary::NormalizedDeltaRotator(
        OwningCharacter->GetBaseAimRotation(), ActorRot);
    AimYaw = FMath::Clamp(AimDelta.Yaw, -90.f, 90.f);
    AimPitch = FMath::Clamp(AimDelta.Pitch, -90.f, 90.f);

    const float CurrentYaw = ActorRot.Yaw;
    const float YawRate = UKismetMathLibrary::NormalizedDeltaRotator(
        FRotator(0.f, CurrentYaw, 0.f), FRotator(0.f, PreviousActorYaw, 0.f)).Yaw / DeltaSeconds;
    PreviousActorYaw = CurrentYaw;

    const float TargetLean = FMath::Clamp(YawRate / LeanMaxAngle, -1.f, 1.f);
    LeanAmount = FMath::FInterpTo(LeanAmount, TargetLean, DeltaSeconds, LeanInterpSpeed);
}

bool UMyLocomotionAnimInstance::CanStartMoving() const
{
    return GroundSpeed > IdleSpeedThreshold && !bIsInAir;
}

void UMyLocomotionAnimInstance::OnLandStateEntered(const FAnimNode_StateMachine& Machine,
    int32 PrevStateIndex, int32 NextStateIndex)
{
    // Landing feedback: camera shake, rumble, a footstep burst.
}
```

Delegate types used above (`AnimInstance.h:118,121`):

```cpp
DECLARE_DELEGATE_RetVal(bool, FCanTakeTransition);
DECLARE_DELEGATE_ThreeParams(FOnGraphStateChanged,
    const struct FAnimNode_StateMachine& /*Machine*/, int32 /*PrevStateIndex*/, int32 /*NextStateIndex*/);
```

Registration functions (`AnimInstance.h:1461-1473`): `AddNativeTransitionBinding`,
`AddNativeStateEntryBinding`, `AddNativeStateExitBinding`. Call them from
`NativeInitializeAnimation`, not `BeginPlay` — the bindings are matched against the
compiled state machine before the AnimGraph runs.

---

## 2. Blend spaces

| Asset class | Header | Shape |
|---|---|---|
| `UBlendSpace1D` | `Animation/BlendSpace1D.h:19` | one axis |
| `UBlendSpace` | `Animation/BlendSpace.h:470` | two axes |
| `UAimOffsetBlendSpace` | `Animation/AimOffsetBlendSpace.h:20` | two axes, additive |

Axis smoothing is `FInterpolationParameter` (`BlendSpace.h:80-110`):

| Field | Type | Note |
|---|---|---|
| `InterpolationTime` | `float` | editor label "Smoothing Time"; 0 disables smoothing |
| `DampingRatio` | `float` | only for `BSIT_SpringDamper`; 1 reaches target without overshoot |
| `MaxSpeed` | `float` | units per second; only used when > 0 |
| `InterpolationType` | `TEnumAsByte<EFilterInterpolationType>` | editor label "Smoothing Type" |

`EFilterInterpolationType` (`Engine/EngineTypes.h:1356`): `BSIT_Average`, `BSIT_Linear`,
`BSIT_Cubic`, `BSIT_EaseInOut`, `BSIT_ExponentialDecay`, `BSIT_SpringDamper`.

### 1D ground locomotion

| Axis | Name | Min | Max | Grid |
|---|---|---|---|---|
| X | Speed | 0 | 600 | 3 |

Samples: Idle at 0, Walk_Fwd at 200, Run_Fwd at 600. Smoothing: `InterpolationTime = 0.15`,
`InterpolationType = BSIT_SpringDamper`, `DampingRatio = 1.0`. The Blend Space 1D Player
node's X pin binds to `GroundSpeed`.

### 2D directional locomotion

| Axis | Name | Min | Max | Grid |
|---|---|---|---|---|
| X | Direction | -180 | 180 | 5 |
| Y | Speed | 0 | 600 | 3 |

```
Direction \ Speed |  0 (Idle) | 200 (Walk) | 600 (Run)
------------------+-----------+------------+------------
       0 (Fwd)    | Idle      | Walk_Fwd   | Run_Fwd
      90 (Right)  | Idle      | Walk_Right | Run_Right
     -90 (Left)   | Idle      | Walk_Left  | Run_Left
     180 (Bwd)    | Idle      | Walk_Bwd   | Run_Bwd
```

Place the same idle sample at every direction value at Speed 0 so the blend collapses to
it cleanly. Direction axis smoothing is tighter than speed: `InterpolationTime = 0.10`
on X, `0.15` on Y, both `BSIT_SpringDamper`. Pins: X to `Direction`, Y to `GroundSpeed`.

### Aim offset

| Axis | Name | Min | Max |
|---|---|---|---|
| X | Yaw | -90 | 90 |
| Y | Pitch | -90 | 90 |

Five additive poses — centre, left, right, up, down — authored relative to the reference
pose. The Aim Offset node goes after the base locomotion pose; X pin `AimYaw`, Y pin
`AimPitch`.

---

## 3. State machine

State machine name `LocomotionSM` (the `FName` passed to `AddNativeTransitionBinding`
must match the graph exactly).

```
[Entry] -> [Idle] <-> [Move]
              |          |
              v          v
        [Crouch Idle] <-> [Crouch Move]
              |
              v
        [Jump Start] -> [In Air] -> [Land] -> [Idle]
```

| State | Content | Leaves when |
|---|---|---|
| Idle | idle loop, or the blend space at speed 0 | `GroundSpeed > 10 && !bIsInAir`, or `bIsInAir`, or `bIsCrouching` |
| Move | `BS_GroundLocomotion2D` | `GroundSpeed <= 10 && !bIsInAir`, or `bIsInAir` |
| Jump Start | play-once jump start | automatic rule on the sequence player |
| In Air | falling loop | `!bIsInAir` |
| Land | play-once landing | automatic rule on the sequence player |
| Crouch Idle | crouch idle loop | `GroundSpeed > 3`, or `!bIsCrouching` |
| Crouch Move | crouch blend space | `GroundSpeed <= 3`, or `!bIsCrouching` |

Suggested cross-fades:

| From | To | Duration | Blend curve |
|---|---|---|---|
| Idle | Move | 0.15s | Cubic In-Out |
| Move | Idle | 0.20s | Cubic In-Out |
| Move | Jump Start | 0.10s | Linear |
| In Air | Land | 0.05s | Linear |
| Land | Idle | 0.20s | Cubic In-Out |
| Idle | Crouch Idle | 0.15s | Linear |

Use "Automatic Rule Based on Sequence Player" for the play-once states (Jump Start, Land)
so they hand over as the animation finishes.

Runtime queries (`AnimInstance.h:1116,1191,1202`):

```cpp
const int32 MachineIndex = GetStateMachineIndex(FName("LocomotionSM"));
if (const FAnimNode_StateMachine* Machine = GetStateMachineInstanceFromName(FName("LocomotionSM")))
{
    const float MoveWeight = GetInstanceStateWeight(MachineIndex, Machine->GetCurrentState());
}
```

---

## 4. AnimGraph layout and upper/lower body split

```
[Blend Space Player]  ->  [Aim Offset]  ->  inside states of  [LocomotionSM]
                                                                    |
                                                    base pose  -->  [Layered Blend Per Bone]  --> [Output Pose]
                                                                    ^
                                        slot 'UpperBody'  ---------/
```

`FAnimNode_LayeredBoneBlend` (`AnimGraphRuntime/Public/AnimNodes/AnimNode_LayeredBoneBlend.h:21`)
takes a base pose plus `TArray<FPoseLink> BlendPoses` (`:32`) with matching
`TArray<float> BlendWeights` (`:55`). Which bones each blend pose owns depends on
`ELayeredBoneBlendMode BlendMode` (`:13-17,36`):

| `BlendMode` | Configured by |
|---|---|
| `BranchFilter` | `TArray<FInputBlendPose> LayerSetup` (`:51`) — each entry holds `TArray<FBranchFilter> BranchFilters` of `{ FName BoneName; int32 BlendDepth; }` (`AnimData/BoneMaskFilter.h:16-33`) |
| `BlendMask` | a `UBlendProfile` blend mask asset on the node |

For an upper-body montage layer with `BranchFilter`: one blend pose fed by the
`UpperBody` slot, one branch filter with `BoneName = spine_01`. Use
`bMeshSpaceRotationBlend` (`:94`) when the layered pose must keep its world-space
orientation rather than blending in local space.

The montage must target the same slot: its `SlotAnimTracks[0].SlotName`
(`Animation/AnimMontage.h:701`) has to equal the Slot node's name, and the slot must exist
in the skeleton's slot group settings. On a mismatch the `UpperBody` Slot node never picks
the montage up (`UAnimMontage::IsValidSlot`); it plays only on a Slot node matching its track,
such as `DefaultSlot` when the track was left at its default.

---

## 5. Modular locomotion with linked anim layers

For several locomotion modes (ground, swim, climb), give each mode its own AnimInstance
implementing an anim layer interface, and swap the implementation instead of growing one
state machine.

1. Create an Anim Layer Interface asset — the engine interface behind it is
   `UAnimLayerInterface` (`Animation/AnimLayerInterface.h:12`) — with one layer function.
2. The main Animation Blueprint places a **Linked Anim Layer** node
   (`FAnimNode_LinkedAnimLayer`, `Animation/AnimNode_LinkedAnimLayer.h:21`) for it.
3. Each mode is an AnimInstance subclass implementing the interface.

```cpp
// MyClimbingLayer.h
#pragma once

#include "CoreMinimal.h"
#include "Animation/AnimInstance.h"
#include "MyClimbingLayer.generated.h"

UCLASS()
class MYGAME_API UMyClimbingLayer : public UAnimInstance
{
    GENERATED_BODY()

public:
    virtual void NativeUpdateAnimation(float DeltaSeconds) override;

protected:
    UPROPERTY(Transient, BlueprintReadOnly, Category = "Climbing")
    float ClimbSpeed = 0.f;
};
```

```cpp
// MyCharacter.cpp — swapping the implementation
void AMyCharacter::EnterClimbing()
{
    if (UAnimInstance* AnimInst = GetMesh()->GetAnimInstance())
    {
        AnimInst->LinkAnimClassLayers(UMyClimbingLayer::StaticClass());
    }
}

void AMyCharacter::ExitClimbing()
{
    if (UAnimInstance* AnimInst = GetMesh()->GetAnimInstance())
    {
        AnimInst->UnlinkAnimClassLayers(UMyClimbingLayer::StaticClass());
    }
}
```

`LinkAnimClassLayers(nullptr)` resets every layer to its default implementation
(`AnimInstance.h:877`). `UnlinkAnimClassLayers(TSubclassOf<UAnimInstance>)` (`:888`) unlinks
only the layers that class provides. Retrieve a live layer with
`GetLinkedAnimLayerInstanceByClass(InClass, bCheckForChildClass)` (`:910`).

### Blending the swap

Place an Inertialization node above the linked layer output. The node's
`PendingBlendInDuration` and `PendingBlendOutDuration`
(`Animation/AnimNode_LinkedAnimGraph.h:67,60`), with optional `PendingBlendInProfile` /
`PendingBlendOutProfile` (`:71,64`), control the blend. From code:

```cpp
// AnimInstance.h:1006
// void RequestSlotGroupInertialization(FName InSlotGroupName, float Duration,
//                                      const UBlendProfile* BlendProfile = nullptr);
AnimInst->RequestSlotGroupInertialization(FName("DefaultGroup"), 0.2f);
```

---

## 6. Root motion

`ERootMotionMode::Type` (`Animation/AnimEnums.h:27-42`), with the header's own descriptions:

| Mode | Behaviour |
|---|---|
| `NoRootMotionExtraction` | leave root motion in the animation |
| `IgnoreRootMotion` | extract root motion but do not apply it |
| `RootMotionFromEverything` | take root motion from all animations contributing to the final pose; not suitable for network multiplayer |
| `RootMotionFromMontagesOnly` | take root motion only from montages; suitable for network multiplayer |

```cpp
// Set the mode from code (AnimInstance.h:1066)
if (UAnimInstance* AnimInst = GetMesh()->GetAnimInstance())
{
    AnimInst->SetRootMotionMode(ERootMotionMode::RootMotionFromMontagesOnly);
}
```

`RootMotionMode` is also a `UPROPERTY(Category = RootMotion, EditDefaultsOnly)`
(`AnimInstance.h:372`), so it can be set on the Animation Blueprint's class defaults.

Per-montage suppression uses the counters on `FAnimMontageInstance`
(`Animation/AnimMontage.h:562-564`): `PushDisableRootMotion()`, `PopDisableRootMotion()`,
`IsRootMotionDisabled()`. The built-in `UAnimNotifyState_DisableRootMotion`
(`AnimNotifies/AnimNotifyState_DisableRootMotion.h:8`) wraps that pair for a timed window.

Whether a montage contributes translation and rotation at all is set on the asset:
`bEnableRootMotionTranslation` and `bEnableRootMotionRotation` (`AnimMontage.h:711,715`).

Everything downstream — how the movement component consumes the delta, prediction,
correction and `RootMotionSource` — belongs to `ue-character-movement`.

---

## 7. Pitfalls

**Direction flips near zero speed.** Only write `Direction` above a speed threshold, as in
the example; otherwise the orientation rotator of a near-zero velocity vector is noise.

**Blend space snapping on turns.** Use `BSIT_SpringDamper` on the direction axis with
`InterpolationTime` between 0.08 and 0.12. Lower reads as snappy, higher as sluggish.

**Aim offset fighting the state machine.** Apply the aim offset inside the locomotion
states, or as an additive layer on the state machine output — not both.

**Crouch/stand pops.** Crouch and stand blend spaces need matching samples at speed 0, or
the transition has to be long enough (0.15s and up) to hide the difference.

**Upper body slot silently ignored.** The slot name on the montage track, the Slot node
and the skeleton's slot group must all match (`FName` comparison, so case does not matter).

**Native transition binding never fires.** `AddNativeTransitionBinding` has to run in
`NativeInitializeAnimation`, and the machine, state and transition names must match the
compiled graph exactly.
