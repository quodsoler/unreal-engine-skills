---
name: ue-character-movement
description: "Use when writing or debugging UCharacterMovementComponent code: custom movement modes, floor detection, jumping, crouching, root motion, moving platforms, or client-side movement prediction. Also use when the user mentions 'CharacterMovementComponent', 'CMC', 'PhysCustom', 'PhysWalking', 'MOVE_Custom', 'FSavedMove_Character', 'GetCompressedFlags', 'FCharacterNetworkMoveData', 'MovementBase', 'LaunchCharacter', 'WalkableFloorAngle', 'SetGravityDirection', 'NavWalking', or 'FRootMotionSource'. For the Mover plugin, see ue-mover; for replication mechanics, see ue-networking-replication."
metadata:
  version: "2.0.0"
  engine: "5.8"
---

# UE Character Movement

Target engine: **UE 5.8**. APIs below are verified against the 5.8 headers; older forms are listed under "Deprecated — do not use".

This skill covers `UCharacterMovementComponent` (CMC) and `ACharacter`: the movement-mode pipeline, floor detection and step-up, custom movement modes, client-side prediction and correction, root motion, custom gravity, and the 5.8 movement-base rework. Everything here ships in the `Engine` module — no plugin and no extra `Build.cs` entry beyond `"Engine"`. Public headers: `GameFramework/CharacterMovementComponent.h` (which already pulls in `GameFramework/RootMotionSource.h`, `GameFramework/CharacterMovementReplication.h`, `CharacterMovementComponentAsync.h` and `Interfaces/MovementBaseInterface.h`), `GameFramework/Character.h`.

## Context

Read `.agents/ue-project-context.md` if it exists (module names, conventions, enabled plugins, GAS/networking setup). Do not stop if it is missing.

Identify the area from the request and the codebase. Ask only when two plausible readings would produce different code.

| Request is about… | Go to |
|---|---|
| Choosing between CMC and the Mover plugin | [CMC vs Mover](#cmc-vs-mover) |
| Class hierarchy, movement modes, mode transitions | [CMC Architecture](#cmc-architecture) |
| Standing on a lift, rotating platform, vehicle, physics body | [Movement Base](#movement-base) |
| Where a tick goes, `PhysWalking`/`PhysFalling`, sweeps, sliding | [Movement Pipeline](#movement-pipeline) |
| Falling through floors, step-up, walkable angle, perching | [Floor Detection](#floor-detection) |
| Wall-run, climb, dash, grapple, any new movement type | [Custom Movement Modes](#custom-movement-modes) |
| Multiplayer desync, rubber-banding, saved moves, compressed flags | [Network Prediction](#network-prediction) |
| Knockback, dashes, animation-driven motion | [Root Motion](#root-motion) |
| Walking on walls/ceilings, planet gravity | [Custom Gravity](#custom-gravity) |
| AI agents, navmesh-driven walking, RVO avoidance | [NavMesh Walking](#navmesh-walking) |
| Jump/crouch/launch from gameplay code | [ACharacter API](#acharacter-api) |
| Speeds, friction, braking, rotation flags | [Key Properties](#key-properties) |

Deep templates live in [cmc-extension-patterns.md](references/cmc-extension-patterns.md); per-mode internals and flow diagrams live in [movement-pipeline.md](references/movement-pipeline.md).

## CMC vs Mover

Mover (Experimental in 5.8) is a separate plugin (`Mover`, not enabled by default) built around `UMoverComponent`, `UBaseMovementMode`, `FLayeredMoveBase` and `UCharacterMoverComponent`. It is not a drop-in CMC replacement.

| Situation | Use |
|---|---|
| Shipping title, any engine-supported platform | CMC |
| Existing `ACharacter` hierarchy, animation and AI already wired to CMC | CMC |
| Networked gameplay that must be stable today | CMC |
| Non-capsule, non-humanoid movers (vehicles, drones) that still want rollback networking | Mover |
| Movement composed from data-driven modes/modifiers instead of a `Phys*` subclass | Mover |
| Movement simulated on the physics thread with Chaos | Mover (`ChaosMover`) |
| Prototype that can absorb Experimental API churn | Mover |

Mover details belong to `ue-mover`. Do not mix the two components on one pawn.

## CMC Architecture

```
UActorComponent -> UMovementComponent -> UNavMovementComponent -> UPawnMovementComponent -> UCharacterMovementComponent
```

`UNavMovementComponent` also implements `INavMovementInterface`; `UCharacterMovementComponent` additionally implements `IRVOAvoidanceInterface` and `INetworkPredictionInterface`, and is declared `UCLASS(MinimalAPI)`.

CMC is a default subobject on `ACharacter` (`ACharacter::CharacterMovementComponentName`). `ACharacter` owns the capsule, mesh and high-level verbs (`Jump`, `Crouch`, `LaunchCharacter`); CMC owns simulation, floor detection and prediction.

### Movement Modes

`EMovementMode` (`Engine/EngineTypes.h`):

| Mode | Meaning |
|---|---|
| `MOVE_None` | No movement processing |
| `MOVE_Walking` | Ground movement with floor detection and step-up |
| `MOVE_NavWalking` | Walking driven by navmesh projection |
| `MOVE_Falling` | Airborne: gravity, air control, landing |
| `MOVE_Swimming` | Fluid movement with buoyancy |
| `MOVE_Flying` | Free 3D movement, no gravity |
| `MOVE_Custom` | User-defined; dispatches to `PhysCustom` with a `uint8` sub-mode |

```cpp
// UCharacterMovementComponent. SetMovementMode is BlueprintCallable;
// OnMovementModeChanged is protected and not a UFUNCTION (override it in C++)
virtual void SetMovementMode(EMovementMode NewMovementMode, uint8 NewCustomMode = 0);
virtual void OnMovementModeChanged(EMovementMode PreviousMovementMode, uint8 PreviousCustomMode);
```

`ACharacter::OnMovementModeChanged(EMovementMode PrevMovementMode, uint8 PreviousCustomMode = 0)` fires after the CMC hook and broadcasts `MovementModeChangedDelegate` (`FMovementModeChangedSignature`). Put enter/exit logic for custom modes in the CMC override; the current sub-mode is the `uint8 CustomMovementMode` member.

## Movement Base

The movement base is the object a character stands on and moves with. In 5.8 it is no longer a `UPrimitiveComponent*`: it is `FMovementBaseInterfaceData` (`Interfaces/MovementBaseInterface.h`), which holds a `TWeakObjectPtr<UObject> PhysicsObjectOwner` plus the `IPhysicsBodyInstanceOwner` that supplies the base behaviour. This lets non-component physics objects act as bases.

### Reading the base

```cpp
// UCharacterMovementComponent
UObject* GetMovementBaseObject() const;                                    // BlueprintCallable
const FMovementBaseInterfaceData* GetMovementBaseInterfaceData() const;
FMovementBaseInterfaceData* GetMovementBaseInterfaceData_Mutable();
const FMovementBaseInterfaceData* GetLastServerMovementBaseInterfaceData() const;
```

`APawn` declares `GetMovementBaseObject()`, `GetMovementBaseInterfaceData()` and `GetMovementBaseInterfaceData_Mutable()`; `ACharacter` overrides all three as `final`. `FMovementBaseInterfaceData` itself exposes `IsValid()`, `Clear()`, `Set()`, `GetBodyInstanceOwner()`, `GetMovementBaseObject()` and `GetMovementBaseObjectOwner()`.

```cpp
static void MyLogCurrentBase(const UCharacterMovementComponent& Movement)
{
    const FMovementBaseInterfaceData* BaseData = Movement.GetMovementBaseInterfaceData();
    if (!MovementBaseUtility::IsMovementBaseDataValid(BaseData))
    {
        return;
    }

    const FName BoneName = Movement.GetCharacterOwner()->GetBasedMovement().BoneName;
    const FVector BaseVelocity = MovementBaseUtility::GetMovementBaseVelocity(BaseData, BoneName);

    // Component-specific work now needs an explicit cast.
    if (const UPrimitiveComponent* BaseComponent = Cast<UPrimitiveComponent>(BaseData->GetMovementBaseObject()))
    {
        UE_LOG(LogMyGame, Verbose, TEXT("Based on %s at %s"), *BaseComponent->GetName(), *BaseVelocity.ToString());
    }
}
```

### Setting the base

```cpp
virtual void SetBase(FMovementBaseInterfaceData* MovementBaseInterfaceData, const FName BoneName = NAME_None, bool bNotifyActor = true); // CMC and ACharacter
void SetBaseFromFloor(const FFindFloorResult& FloorResult);                                                                             // CMC
```

Build the struct from what you already have:

```cpp
FMovementBaseInterfaceData NewBase = MovementBaseUtility::GetMovementBaseDataFromHitResult(&Hit);
SetBase(&NewBase, Hit.BoneName);
```

`MovementBaseUtility::GetMovementBaseDataFromPhysicsOwner(UObject*)` does the same from a replicated `PhysicsObjectOwner`.

### Utility namespace

Every `MovementBaseUtility` helper now takes `FMovementBaseInterfaceData*` instead of `UPrimitiveComponent*`: `IsDynamicBase`, `IsSimulatedBase`, `UseRelativeLocation`, `AddTickDependency`, `RemoveTickDependency`, `GetMovementBaseVelocity`, `GetMovementBaseTangentialVelocity`, `GetMovementBaseTransform`, `TransformLocationToWorld`, `TransformLocationToLocal`, `TransformDirectionToWorld`, `TransformDirectionToLocal`. New in 5.8: `DoesMovementBaseDataMatch`, `IsMovementBaseDataValid`, `GetMovementBaseDataFromPhysicsOwner`, `GetMovementBaseDataFromHitResult`.

### Stored and replicated base state

| Old member | 5.8 member | Holder |
|---|---|---|
| `MovementBase` | `MovementBaseInterfaceData` | `FBasedMovementInfo` |
| `MovementBase` | `PhysicsObjectOwner` | `FRepRootMotionMontage` |
| `MovementBase` | `MovementBasePhysicsObjectOwner` | `FCharacterNetworkMoveData` |
| `StartBase` / `EndBase` | `StartMovementBaseInterfaceData` / `EndMovementBaseInterfaceData` | `FSavedMove_Character` |
| `LastServerMovementBase` | `LastMovementBaseInterfaceData` | `UCharacterMovementComponent` |

Every server-authority and correction virtual that used to take a `UPrimitiveComponent* ClientMovementBase` now has an `FMovementBaseInterfaceData*` overload: `ServerMoveHandleClientError`, `ServerCheckClientError`, `ServerExceedsAllowablePositionError`, `ServerShouldUseAuthoritativePosition`, `ClientAdjustPosition_Implementation`, `ClientVeryShortAdjustPosition_Implementation`, `ClientAdjustRootMotionPosition_Implementation`, `ClientAdjustRootMotionSourcePosition_Implementation`, `OnClientCorrectionReceived` (plus the non-virtual `RevertMove`). Override the new overload — overriding the deprecated one silently stops being called.

`ACharacter::BaseChange()` and `ACharacter::OnRep_ReplicatedBasedMovement()` are the notification hooks; `ACharacter::GetBasedMovement()` returns the `FBasedMovementInfo`.

## Movement Pipeline

`PerformMovement(float DeltaTime)` (protected) is the autonomous-path entry point; simulated proxies go through `SimulatedTick`, which calls `PerformMovement` only while root motion is active and otherwise `SimulateMovement` (a `MoveSmooth` extrapolation that skips the `Phys*` functions). `PerformMovement` reaches `StartNewPhysics(float deltaTime, int32 Iterations)`, which dispatches on `MovementMode`. All `Phys*` functions are `virtual`; `PhysFalling` is public (`CharacterMovementComponent.h:1663`), the other five are protected (`:1995-2007`):

```cpp
virtual void PhysWalking(float deltaTime, int32 Iterations);
virtual void PhysNavWalking(float deltaTime, int32 Iterations);
virtual void PhysFalling(float deltaTime, int32 Iterations);
virtual void PhysSwimming(float deltaTime, int32 Iterations);
virtual void PhysFlying(float deltaTime, int32 Iterations);
virtual void PhysCustom(float deltaTime, int32 Iterations);
```

Inside a `Phys*` function:

```cpp
// BlueprintCallable
virtual void CalcVelocity(float DeltaTime, float Friction, bool bFluid, float BrakingDeceleration);

// UMovementComponent; also has an FRotator overload
virtual bool SafeMoveUpdatedComponent(const FVector& Delta, const FQuat& NewRotation, bool bSweep,
                                      FHitResult& OutHit, ETeleportType Teleport = ETeleportType::None);
```

`SafeMoveUpdatedComponent` wraps `MoveUpdatedComponent` and calls `ResolvePenetration` on initial overlap — always prefer it inside custom `Phys*` code. When overriding it, add `using UMovementComponent::SafeMoveUpdatedComponent;` next to the declaration or the `FRotator` overload is hidden.

On a blocking hit, CMC's override of `SlideAlongSurface(const FVector& Delta, float Time, const FVector& Normal, FHitResult& Hit, bool bHandleImpact)` projects the remaining delta along the surface (and refuses to slide up unwalkable slopes while walking). Corner hits go through `TwoWallAdjust(FVector& WorldSpaceDelta, const FHitResult& Hit, const FVector& OldHitNormal) const`. During walking, `ComputeGroundMovementDelta(const FVector& Delta, const FHitResult& RampHit, const bool bHitFromLineTrace) const` reorients the delta along the ramp, and `MaintainHorizontalGroundVelocity()` flattens velocity afterwards (rescaling to preserve magnitude when `bMaintainHorizontalGroundVelocity` is false).

`SetUpdatedComponent(USceneComponent* NewUpdatedComponent)` is overridden on CMC; it is what binds the component being moved (the capsule) and re-registers ticking. Sub-stepping is bounded by `MaxSimulationTimeStep` and `MaxSimulationIterations`; `MIN_TICK_TIME` is the floor below which a slice is discarded. CMC can also run the whole simulation on the physics thread behind the CVar `p.AsyncCharacterMovement` (default `0`; the engine marks it not fully developed) through `FCharacterMovementComponentAsyncInput` / `FCharacterMovementComponentAsyncOutput` / `FCharacterMovementComponentAsyncCallback` — custom `Phys*` state is not mirrored into that path automatically. Full per-mode breakdowns and the async plumbing: [movement-pipeline.md](references/movement-pipeline.md).

## Floor Detection

`FFindFloorResult` lives in `CharacterMovementComponentAsync.h` (`USTRUCT(BlueprintType)`), not in the CMC header:

| Member | Meaning |
|---|---|
| `bBlockingHit` | Sweep hit something, not in initial penetration |
| `bWalkableFloor` | Hit surface passes the walkability test |
| `bLineTrace` | Result came from the line-trace fallback |
| `FloorDist` | Distance from the swept capsule |
| `LineDist` | Distance from the line trace (valid only if `bLineTrace`) |
| `HitResult` | Full `FHitResult` |

Helpers: `IsWalkableFloor()` (`bBlockingHit && bWalkableFloor`), `GetDistanceToFloor()`, `Clear()`, `SetFromSweep()`, `SetFromLineTrace()`.

```cpp
// Both are const on UCharacterMovementComponent
virtual void FindFloor(const FVector& CapsuleLocation, FFindFloorResult& OutFloorResult,
                       bool bCanUseCachedLocation, const FHitResult* DownwardSweepResult = NULL) const;

virtual void ComputeFloorDist(const FVector& CapsuleLocation, float LineDistance, float SweepDistance,
                              FFindFloorResult& OutFloorResult, float SweepRadius,
                              const FHitResult* DownwardSweepResult = NULL) const;
```

`FindFloor` delegates to `ComputeFloorDist` (downward capsule sweep, then a line trace to validate the contact point). The cached result is `CurrentFloor`. `AdjustFloorHeight()` snaps the capsule between `MIN_FLOOR_DIST` and `MAX_FLOOR_DIST`.

Walkability: `WalkableFloorAngle` (degrees, clamped 0–90) and the derived `WalkableFloorZ` (default `0.71`, about 44.8 degrees). Set the angle through `SetWalkableFloorAngle(float)` — writing `WalkableFloorZ` directly desynchronises the pair. `virtual bool IsWalkable(const FHitResult& Hit) const` is the override point for per-surface rules.

Step-up runs through `virtual bool StepUp(const FVector& GravDir, const FVector& Delta, const FHitResult& Hit, FStepDownResult* OutStepDownResult = NULL)`, bounded by `MaxStepHeight`, with `PerchRadiusThreshold` and `PerchAdditionalHeight` controlling ledge perching.

## Custom Movement Modes

`MOVE_Custom` plus the `uint8 CustomMovementMode` gives 256 sub-modes that reuse CMC's prediction pipeline.

```cpp
// MyCharacterMovement.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/CharacterMovementComponent.h"
#include "MyCharacterMovement.generated.h"

UENUM(BlueprintType)
enum class EMyMovementMode : uint8
{
    None      = 0,
    Climbing  = 1,
    Dash      = 2
};

UCLASS()
class MYGAME_API UMyCharacterMovement : public UCharacterMovementComponent
{
    GENERATED_BODY()

public:
    UMyCharacterMovement();

    UPROPERTY(Transient)
    uint8 bWantsToClimb : 1;

    UPROPERTY(Transient)
    FVector ClimbSurfaceNormal;

    virtual void PhysCustom(float deltaTime, int32 Iterations) override;
    virtual void OnMovementModeChanged(EMovementMode PreviousMovementMode, uint8 PreviousCustomMode) override;
    virtual float GetMaxSpeed() const override;

protected:
    void PhysClimb(float deltaTime, int32 Iterations);
};
```

```cpp
void UMyCharacterMovement::PhysCustom(float deltaTime, int32 Iterations)
{
    Super::PhysCustom(deltaTime, Iterations);   // dispatches ACharacter::K2_UpdateCustomMovement

    if (CustomMovementMode == static_cast<uint8>(EMyMovementMode::Climbing))
    {
        PhysClimb(deltaTime, Iterations);
    }
}
```

Call `Super::PhysCustom(deltaTime, Iterations)` **first**: the base implementation forwards to `ACharacter::K2_UpdateCustomMovement(float DeltaTime)`, so skipping it or calling it last breaks the Blueprint hook and any base bookkeeping added later. This skill is the authority on that ordering.

Enter and leave a mode with `SetMovementMode(MOVE_Custom, static_cast<uint8>(EMyMovementMode::Climbing))` and `SetMovementMode(MOVE_Falling)`. Cap speed by overriding `virtual float GetMaxSpeed() const override` (the base returns `MaxCustomMovementSpeed` for `MOVE_Custom`). A full climbing mode with prediction is in [cmc-extension-patterns.md](references/cmc-extension-patterns.md).

## Network Prediction

The autonomous client simulates locally, records each frame as an `FSavedMove_Character`, sends it to the server, and replays unacknowledged moves when the server sends a correction. Custom movement state that is not saved and restored will be lost on replay. The transport itself (RPC rules, roles, relevancy, push model) belongs to `ue-networking-replication`.

### FSavedMove_Character

Useful fields: `bPressedJump`, `bWantsToCrouch`, `bForceMaxAccel`, `bForceNoCombine`, `TimeStamp`, `DeltaTime`, `StartLocation`, `StartVelocity`, `StartFloor`, `StartRotation`, `StartControlRotation`, `StartMovementBaseInterfaceData`, `SavedLocation`, `SavedVelocity`, `EndMovementBaseInterfaceData`, `Acceleration`, `MaxSpeed`, `SavedRootMotion`.

Virtuals to override, verbatim:

```cpp
virtual void Clear();
virtual void SetMoveFor(ACharacter* C, float InDeltaTime, FVector const& NewAccel, class FNetworkPredictionData_Client_Character& ClientData);
virtual void PrepMoveFor(ACharacter* C);
virtual uint8 GetCompressedFlags() const;
virtual bool CanCombineWith(const FSavedMovePtr& NewMove, ACharacter* InCharacter, float MaxDelta) const;
virtual void PostUpdate(ACharacter* C, EPostUpdateMode PostUpdateMode);
virtual bool IsImportantMove(const FSavedMovePtr& LastAckedMove) const;
virtual void CombineWith(const FSavedMove_Character* OldMove, ACharacter* InCharacter, APlayerController* PC, const FVector& OldStartLocation);
```

### Compressed flags

`FSavedMove_Character::CompressedFlags` is a nested `enum`:

| Flag | Value | Purpose |
|---|---|---|
| `FLAG_JumpPressed` | `0x01` | Jump input |
| `FLAG_WantsToCrouch` | `0x02` | Crouch input |
| `FLAG_Reserved_1` | `0x04` | Engine reserved |
| `FLAG_Reserved_2` | `0x08` | Engine reserved |
| `FLAG_Custom_0` | `0x10` | Yours |
| `FLAG_Custom_1` | `0x20` | Yours |
| `FLAG_Custom_2` | `0x40` | Yours |
| `FLAG_Custom_3` | `0x80` | Yours |

Pack them in `GetCompressedFlags()`, unpack them in `virtual void UpdateFromCompressedFlags(uint8 Flags)` on the CMC. Four bits total; anything larger goes through custom move data.

### Prediction data

`FNetworkPredictionData_Client_Character` owns `SavedMoves`, `FreeMoves`, `PendingMove`, `LastAckedMove`, `MaxSavedMoveCount` and the mesh-smoothing offsets. Override `virtual FSavedMovePtr AllocateNewMove()` to hand back your subclass, and lazily create the data in `virtual class FNetworkPredictionData_Client* GetPredictionData_Client() const override` (the CMC member is `ClientPredictionData`; `GetPredictionData_Client_Character()` is the typed accessor).

### Packed move RPCs

Movement RPCs live on `ACharacter`:

```cpp
void ServerMovePacked(const FCharacterServerMovePackedBits& PackedBits);          // Server, unreliable, WithValidation
void ClientMoveResponsePacked(const FCharacterMoveResponsePackedBits& PackedBits); // Client, unreliable, WithValidation
```

For state that does not fit in the four custom flags, derive `FCharacterNetworkMoveData` (override `ClientFillNetworkMoveData(const FSavedMove_Character& ClientMove, ENetworkMoveType MoveType)` and `Serialize(UCharacterMovementComponent&, FArchive&, UPackageMap*, ENetworkMoveType)`), derive `FCharacterNetworkMoveDataContainer` and repoint `NewMoveData` / `PendingMoveData` / `OldMoveData`, then call `SetNetworkMoveDataContainer(...)` in the CMC constructor with a container that outlives the component. `GetCurrentNetworkMoveData()` reaches the active data from inside `MoveAutonomous` or `UpdateFromCompressedFlags`. The response side mirrors this with `FCharacterMoveResponseDataContainer` and `SetMoveResponseDataContainer(...)`. Templates: [cmc-extension-patterns.md](references/cmc-extension-patterns.md).

## Root Motion

`FRootMotionSource` (`GameFramework/RootMotionSource.h`) drives movement from gameplay code. Base fields: `Priority` (`uint16`, higher wins), `LocalID` (`uint16`), `InstanceName`, `Duration` (negative means it runs until removed), `AccumulateMode` (`ERootMotionAccumulateMode::Override` or `Additive`), `FinishVelocityParams` (`ERootMotionFinishVelocityMode`).

| Struct | Key fields | Use |
|---|---|---|
| `FRootMotionSource_ConstantForce` | `Force`, `StrengthOverTime` | Knockback, wind |
| `FRootMotionSource_RadialForce` | `Location`, `LocationActor`, `Radius`, `Strength`, `bIsPush`, `bNoZForce` | Explosions, vortex |
| `FRootMotionSource_MoveToForce` | `StartLocation`, `TargetLocation`, `PathOffsetCurve` | Dash to a fixed point |
| `FRootMotionSource_MoveToDynamicForce` | `StartLocation`, `SetTargetLocation()` | Homing dash |
| `FRootMotionSource_JumpForce` | `Rotation`, `Distance`, `Height`, `TimeMappingCurve` | Targeted jump arc |

```cpp
// UCharacterMovementComponent
uint16 ApplyRootMotionSource(TSharedPtr<FRootMotionSource> SourcePtr);
TSharedPtr<FRootMotionSource> GetRootMotionSource(FName InstanceName);
TSharedPtr<FRootMotionSource> GetRootMotionSourceByID(uint16 RootMotionSourceID);
void RemoveRootMotionSource(FName InstanceName);
void RemoveRootMotionSourceByID(uint16 RootMotionSourceID);
bool HasRootMotionSources() const;
```

```cpp
TSharedPtr<FRootMotionSource_ConstantForce> Knockback = MakeShared<FRootMotionSource_ConstantForce>();
Knockback->InstanceName = TEXT("Knockback");
Knockback->Duration = 0.3f;
Knockback->Force = KnockbackDirection * KnockbackStrength;
Knockback->AccumulateMode = ERootMotionAccumulateMode::Override;
Knockback->Priority = 5;
GetCharacterMovement()->ApplyRootMotionSource(Knockback);
```

Active sources live in `CurrentRootMotion` (`FRootMotionSourceGroup`) and are folded into velocity by `virtual void ApplyRootMotionToVelocity(float deltaTime)`. Animation root motion is a separate path: `RootMotionParams` (`FRootMotionMovementParams`) filled from the anim graph, queried with `HasAnimRootMotion()`, and replicated through `FRepRootMotionMontage`. Montage authoring and `UAnimInstance::SetRootMotionMode` belong to `ue-animation-system`.

## Custom Gravity

```cpp
virtual void SetGravityDirection(const FVector& GravityDir);
bool HasCustomGravity() const;
FVector GetGravityDirection() const;
FQuat GetWorldToGravityTransform() const;
FQuat GetGravityToWorldTransform() const;
FVector RotateGravityToWorld(const FVector& World) const;
FVector RotateWorldToGravity(const FVector& Gravity) const;
FVector ProjectToGravityFloor(const FVector& Vector) const;
FVector::FReal GetGravitySpaceZ(const FVector& Vector) const;
void SetGravitySpaceZ(FVector& Vector, const FVector::FReal Z) const;
```

`GravityDirection`, `WorldToGravityTransform`, `GravityToWorldTransform` and `bHasCustomGravity` are protected — always go through the accessors, and always set the direction with `SetGravityDirection` so the cached quaternions are rebuilt. The default is `UCharacterMovementComponent::DefaultGravityDirection`. Walking, falling and floor detection all operate in gravity space when `HasCustomGravity()` is true; in custom `Phys*` code use `ProjectToGravityFloor` / `GetGravitySpaceZ` instead of hard-coded `Z` arithmetic.

## NavMesh Walking

`MOVE_NavWalking` projects the capsule onto the navmesh instead of sweeping against world geometry — far cheaper for crowds, and the reason AI characters can stand on nothing. `FindNavFloor(const FVector& TestLocation, FNavLocation& NavFloorLocation) const` performs the projection.

| Property | Effect |
|---|---|
| `bProjectNavMeshWalking` | Raycast to underlying geometry to conform height |
| `bProjectNavMeshOnBothWorldChannels` | Use `WorldStatic` and `WorldDynamic` for that raycast |
| `bSlideAlongNavMeshEdge` | Slide along the navmesh edge instead of projecting a point and moving to it |

Avoidance: `bUseRVOAvoidance`, `AvoidanceWeight`, and `AvoidanceGroup` (`FNavAvoidanceMask`) set through `SetAvoidanceGroupMask(const FNavAvoidanceMask&)`. Nav data, queries and pathfinding belong to `ue-ai-navigation`.

## Key Properties

| Property | Default | Notes |
|---|---|---|
| `MaxWalkSpeed` | 600 | Ground speed cap |
| `MaxWalkSpeedCrouched` | 300 | Half of `MaxWalkSpeed` by default |
| `MaxSwimSpeed` | 300 | |
| `MaxFlySpeed` | 600 | |
| `MaxCustomMovementSpeed` | 600 | Cap returned by `GetMaxSpeed()` in `MOVE_Custom` |
| `MaxAcceleration` | 2048 | |
| `BrakingDecelerationWalking` | 2048 | Matches `MaxAcceleration` by default |
| `GroundFriction` | 8 | |
| `GravityScale` | 1 | Multiplier on world gravity |
| `JumpZVelocity` | 420 | |
| `AirControl` | 0.05 | Lateral control while falling |
| `MaxStepHeight` | 45 | |
| `WalkableFloorZ` | 0.71 | About 44.8 degrees; set via `SetWalkableFloorAngle` |
| `MaxSimulationTimeStep` | 0.05 | Sub-step cap |
| `MaxSimulationIterations` | 8 | Sub-steps per frame |
| `NetworkSmoothingMode` | `Exponential` (`CharacterMovementComponent.cpp:718`) | `ENetworkSmoothingMode::Disabled`, `Linear`, `Exponential` |
| `bOrientRotationToMovement` | false | Rotate toward velocity |
| `bUseControllerDesiredRotation` | false | Rotate toward controller rotation |

`bOrientRotationToMovement` and `bUseControllerDesiredRotation` fight each other each frame — pick one.

## ACharacter API

```cpp
virtual void Jump();                                     // BlueprintCallable, sets bPressedJump
virtual void StopJumping();                              // BlueprintCallable
bool CanJumpInternal() const;                            // BlueprintNativeEvent, DisplayName "CanJump"
virtual bool CanJumpInternal_Implementation() const;     // override this
virtual void LaunchCharacter(FVector LaunchVelocity, bool bXYOverride, bool bZOverride);
virtual void Landed(const FHitResult& Hit);
virtual void Crouch(bool bClientSimulation = false);
virtual void UnCrouch(bool bClientSimulation = false);
```

Jump state: `bPressedJump`, `JumpMaxHoldTime`, `JumpMaxCount`, `JumpCurrentCount`, `JumpCurrentCountPreJump`. The CMC side is `virtual bool DoJump(bool bReplayingMoves, float DeltaTime)` and `virtual bool CanAttemptJump() const`. Blueprint events: `OnLanded`, `OnLaunched`, `OnWalkingOffLedge`, `OnReachedJumpApex`.

`bXYOverride`/`bZOverride` replace the corresponding velocity components instead of adding to them; `LaunchCharacter` routes through `UCharacterMovementComponent::Launch(FVector const&)` and `HandlePendingLaunch()`.

Crouching: `bIsCrouched` is the replicated flag; the capsule half-height is `SetCrouchedHalfHeight(const float)` / `GetCrouchedHalfHeight()` on the CMC.

Accessors: `GetCharacterMovement()`, `GetCapsuleComponent()` (`UCapsuleComponent`), `GetMesh()` (`USkeletalMeshComponent`).

## Deprecated — do not use

| Do not emit | Use in 5.8 | Source |
|---|---|---|
| `ACharacter::GetMovementBase()` | `GetMovementBaseObject()` / `GetMovementBaseInterfaceData()` | `UE_DEPRECATED(5.8)` in `GameFramework/Character.h` |
| `UCharacterMovementComponent::GetMovementBase()` | `GetMovementBaseObject()` | `UE_DEPRECATED(5.8)` in `GameFramework/CharacterMovementComponent.h` |
| `SetBase(UPrimitiveComponent*, FName, bool)` | `SetBase(FMovementBaseInterfaceData*, FName, bool)` | `UE_DEPRECATED(5.8)` in `Character.h`, `CharacterMovementComponent.h` |
| Any `MovementBaseUtility` helper taking `const UPrimitiveComponent*` | the same name taking `const FMovementBaseInterfaceData*` (`AddTickDependency`/`RemoveTickDependency` take a non-const `UPrimitiveComponent*` in the old form) | `UE_DEPRECATED(5.8)` in `Character.h` |
| `FBasedMovementInfo::MovementBase` | `FBasedMovementInfo::MovementBaseInterfaceData` | `UE_DEPRECATED(5.8)` in `Character.h` |
| `FRepRootMotionMontage::MovementBase` | `FRepRootMotionMontage::PhysicsObjectOwner` | `UE_DEPRECATED(5.8)` in `Character.h` |
| `FCharacterNetworkMoveData::MovementBase` | `MovementBasePhysicsObjectOwner` | `UE_DEPRECATED(5.8)` in `GameFramework/CharacterMovementReplication.h` |
| `FSavedMove_Character::StartBase` / `EndBase` | `StartMovementBaseInterfaceData` / `EndMovementBaseInterfaceData` | `UE_DEPRECATED(5.8)` in `CharacterMovementComponent.h` |
| `GetLastServerMovementBase()` / `LastServerMovementBase` | `GetLastServerMovementBaseInterfaceData()` / `LastMovementBaseInterfaceData` | `UE_DEPRECATED(5.8)` in `CharacterMovementComponent.h` |
| `ServerMoveHandleClientError`, `ServerCheckClientError`, `ClientAdjustPosition_Implementation`, `RevertMove` and the other `UPrimitiveComponent*` correction virtuals | the `FMovementBaseInterfaceData*` overloads | `UE_DEPRECATED(5.8)` in `CharacterMovementComponent.h` |
| `DoJump(bool bReplayingMoves)` | `DoJump(bool bReplayingMoves, float DeltaTime)` | `UE_DEPRECATED_FORGAME(5.5)` in `CharacterMovementComponent.h` |
| Public `CrouchedHalfHeight` | `SetCrouchedHalfHeight()` / `GetCrouchedHalfHeight()` | `UE_DEPRECATED_FORGAME(5.0)` in `CharacterMovementComponent.h` |
| `SetAvoidanceGroup(int32)` | `SetAvoidanceGroupMask(const FNavAvoidanceMask&)` | `meta=(DeprecatedFunction)` in `CharacterMovementComponent.h` |
| `UNavMovementComponent::bUseAccelerationForPaths`, `FixedPathBrakingDistance`, `bUpdateNavAgentWithOwnersCollision`, `bUseFixedBrakingDistanceForPaths`, `bStopMovementAbortPaths` | the matching `NavMovementProperties.*` field | `UE_DEPRECATED(5.5)` in `GameFramework/NavMovementComponent.h` |
| `ServerMove`, `ServerMoveDual`, `ServerMoveOld`, `ClientAdjustPosition`, `ClientAckGoodMove` RPCs | `ServerMovePacked` / `ClientMoveResponsePacked` | `DEPRECATED_CHARACTER_MOVEMENT_RPC` in `CharacterMovementReplication.h` (`SUPPORT_DEPRECATED_CHARACTER_MOVEMENT_RPCS` defaults to `0`) |

## Common Mistakes

**Treating the movement base as a component:** `GetMovementBase()` is deprecated and `FBasedMovementInfo::MovementBase` no longer carries the base. Read `GetMovementBaseInterfaceData()` and cast `GetMovementBaseObject()` only when you genuinely need a `UPrimitiveComponent`.

**Overriding the `UPrimitiveComponent*` correction virtuals:** a subclass that overrides the deprecated `ServerCheckClientError` / `ClientAdjustPosition_Implementation` overload compiles but is never called, so corrections silently fall back to base behaviour. Override the `FMovementBaseInterfaceData*` overloads.

**Calling `Super::PhysCustom` last, or not at all:** the base implementation forwards to `ACharacter::K2_UpdateCustomMovement`. Call it first, then run your mode.

**Writing `Velocity` directly instead of `CalcVelocity`:** bypasses friction, braking and acceleration clamping.
```cpp
// WRONG
Velocity = GetLastInputVector() * MaxWalkSpeed;
// RIGHT
CalcVelocity(DeltaTime, GroundFriction, false, BrakingDecelerationWalking);
```

**Using `MoveUpdatedComponent` in custom `Phys*` code:** it does not resolve initial penetration, so the capsule gets stuck in geometry. Use `SafeMoveUpdatedComponent`.

**Not saving custom state in `FSavedMove_Character`:** anything not captured in `SetMoveFor` and restored in `PrepMoveFor` is lost during correction replay, which shows up as rubber-banding only under latency.

**Reusing a stack-local network move data container:** `SetNetworkMoveDataContainer` stores a pointer. A container that goes out of scope leaves a dangling pointer on the CMC.

**Setting `WalkableFloorZ` directly:** it is computed from `WalkableFloorAngle`. Call `SetWalkableFloorAngle()` so both stay consistent.

**Hard-coding `Z` while custom gravity is active:** use `ProjectToGravityFloor`, `GetGravitySpaceZ` and `RotateWorldToGravity` instead of `Velocity.Z`.

## Related Skills

- `ue-mover` — the Mover plugin (Experimental in 5.8): `UMoverComponent`, movement modes as data, layered moves, ChaosMover
- `ue-networking-replication` — RPC rules, net roles, `DOREPLIFETIME`, push model, relevancy; the transport CMC prediction rides on
- `ue-animation-system` — montages, anim graph, `SetRootMotionMode`, animation-driven locomotion
- `ue-gameplay-framework` — `ACharacter` in the GameMode/Controller/Pawn hierarchy, possession, input routing
- `ue-physics-collision` — sweeps, collision channels and responses used by floor detection and step-up
- `ue-gameplay-abilities` — abilities and ability tasks that trigger movement modes or root motion sources
- `ue-ai-navigation` — navmesh generation and queries behind `MOVE_NavWalking`, avoidance configuration
- `ue-actor-component-architecture` — actor and component lifecycle, attachment, spawning and tick configuration
- `ue-gameplay-cameras` — spring arms, view targets, camera modifiers, shakes and the Gameplay Camera System
- `ue-input-system` — Enhanced Input actions, mapping contexts, triggers, modifiers and user settings
