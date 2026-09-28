# Movement Pipeline

Internals of `UCharacterMovementComponent` in UE 5.8: tick dispatch, per-mode behaviour, floor detection and step-up, the movement-base update, root motion, the async physics path, and debugging. All names are from `GameFramework/CharacterMovementComponent.h`, `GameFramework/Character.h` and `CharacterMovementComponentAsync.h`.

---

## Tick dispatch

```
UCharacterMovementComponent::TickComponent(DeltaTime)
  |
  +-- Autonomous / authority -> ControlledCharacterMove(InputVector, DeltaSeconds)
  |                               +-- CharacterOwner->CheckJumpInput(DeltaSeconds)
  |                               +-- PerformMovement(DeltaTime)
  |
  +-- Simulated proxy       -> SimulatedTick(DeltaSeconds)
  |                               +-- SimulateMovement(DeltaTime)
  |                               +-- SmoothClientPosition(DeltaSeconds)
  |
  +-- Server for a remote AP -> ServerAutonomousProxyTick(DeltaSeconds)
```

`PerformMovement` applies pending launches (`HandlePendingLaunch`), accumulated forces (`ApplyAccumulatedForces`), root motion, then calls `StartNewPhysics(DeltaTime, Iterations)`:

```
StartNewPhysics(deltaTime, Iterations)
  +-- MOVE_Walking     -> PhysWalking(deltaTime, Iterations)
  +-- MOVE_NavWalking  -> PhysNavWalking(deltaTime, Iterations)
  +-- MOVE_Falling     -> PhysFalling(deltaTime, Iterations)
  +-- MOVE_Swimming    -> PhysSwimming(deltaTime, Iterations)
  +-- MOVE_Flying      -> PhysFlying(deltaTime, Iterations)
  +-- MOVE_Custom      -> PhysCustom(deltaTime, Iterations)
```

Each `Phys*` function sub-steps internally. A slice is capped at `MaxSimulationTimeStep`, the loop at `MaxSimulationIterations`, and any remainder below `MIN_TICK_TIME` is dropped. Every slide, step-up and landing consumes an iteration.

Inside a slice:

```
CalcVelocity(dt, Friction, bFluid, BrakingDeceleration)
  |
  v
SafeMoveUpdatedComponent(Delta, Rotation, bSweep, Hit)
  +-- MoveUpdatedComponentImpl()      // the actual primitive move
  +-- ResolvePenetrationImpl()        // on initial overlap; sets bJustTeleported
  |
  v
on blocking hit:
  HandleImpact(Hit, TimeSlice, MoveDelta)
  SlideAlongSurface(Delta, 1.f - Hit.Time, Hit.Normal, Hit, bHandleImpact)
    +-- ComputeSlideVector()
    +-- TwoWallAdjust(Delta, Hit, OldHitNormal)   // corner case
```

`ResolvePenetrationImpl` is overridden on CMC to set `bJustTeleported`, which suppresses velocity-from-position inference for that frame.

---

## PhysWalking

```
PhysWalking(deltaTime, Iterations)
  |
  +-- MaintainHorizontalGroundVelocity()
  +-- CalcVelocity(dt, GroundFriction, false, BrakingDecelerationWalking)
  |
  +-- MoveAlongFloor(Velocity, DeltaSeconds, &StepDownResult)
  |     +-- ComputeGroundMovementDelta(Delta, CurrentFloor.HitResult, bLineTrace)
  |     +-- SafeMoveUpdatedComponent(RampVector, Rotation, true, Hit)
  |     +-- on blocking hit: StepUp(GravDir, Delta, Hit, &StepDownResult)
  |           +-- on failure: SlideAlongSurface(...)
  |
  +-- FindFloor(NewLocation, CurrentFloor, bZeroDelta, DownwardSweepResult)
  |     +-- walkable floor  -> stay in MOVE_Walking
  |     +-- no floor        -> SetMovementMode(MOVE_Falling)
  |     +-- unwalkable      -> slide down or fall (ShouldCatchAir decides)
  |
  +-- AdjustFloorHeight()
        +-- keeps FloorDist between MIN_FLOOR_DIST and MAX_FLOOR_DIST
```

`ComputeGroundMovementDelta(const FVector& Delta, const FHitResult& RampHit, const bool bHitFromLineTrace) const` reorients the horizontal delta along the ramp so the character walks up slopes rather than into them. `MaintainHorizontalGroundVelocity()` zeroes vertical velocity afterwards; when `bMaintainHorizontalGroundVelocity` is false it also rescales the vector to preserve magnitude.

`virtual bool ShouldCatchAir(const FFindFloorResult& OldFloor, const FFindFloorResult& NewFloor)` returns false in the base class — override it to make a character launch off a crest instead of hugging it.

---

## Step-up

`virtual bool StepUp(const FVector& GravDir, const FVector& Delta, const FHitResult& Hit, FStepDownResult* OutStepDownResult = NULL)`:

```
1. Reject the hit if the impact point is above MaxStepHeight from the capsule bottom.
2. Sweep the capsule UP by the step height.
3. Sweep FORWARD by the remaining Delta.
4. Sweep DOWN to find the new floor, filling FStepDownResult.
5. Accept when the new floor is walkable and within MaxStepHeight;
   PerchRadiusThreshold / PerchAdditionalHeight allow hanging off an edge.
6. Otherwise revert to the pre-step position and let SlideAlongSurface handle it.
```

`FStepDownResult` carries `bComputedFloor` and `FloorResult`; `MoveAlongFloor` forwards it so `PhysWalking` can skip a redundant `FindFloor`. Step-up runs only while walking — a falling character slides instead.

---

## Floor detection chain

```
FindFloor(CapsuleLocation, OutFloorResult, bCanUseCachedLocation, DownwardSweepResult) const
  |
  v
ComputeFloorDist(CapsuleLocation, LineDistance, SweepDistance, OutFloorResult, SweepRadius, DownwardSweepResult) const
  |
  +-- 1. Downward capsule sweep with a shrunk radius (SweepRadius)
  |       - rejects hits within SWEEP_EDGE_REJECT_DISTANCE of the capsule's vertical edge
  |       - IsWalkable(Hit) tests the normal against WalkableFloorZ
  |
  +-- 2. Downward line trace (LineDistance)
  |       - validates the sweep at the exact contact point
  |       - sets bLineTrace and LineDist when it supplies the answer
  |
  v
FFindFloorResult: bBlockingHit, bWalkableFloor, bLineTrace, FloorDist, LineDist, HitResult
```

`IsWalkableFloor()` is `bBlockingHit && bWalkableFloor`. A steep blocking hit yields `bBlockingHit = true` with `bWalkableFloor = false`. `GetDistanceToFloor()` returns `LineDist` when `bLineTrace` is set, otherwise `FloorDist`. The cached result is the `CurrentFloor` member, refreshed once per `PhysWalking` iteration; pass `bCanUseCachedLocation = true` only when the capsule has not moved since.

`virtual bool IsWalkable(const FHitResult& Hit) const` is the per-surface override point (physical-material rules, one-way platforms). `WalkableFloorZ` is derived from `WalkableFloorAngle` through `SetWalkableFloorAngle(float)`; the pair desynchronises if either is written directly.

---

## PhysFalling

```
PhysFalling(deltaTime, Iterations)
  |
  +-- GetFallingLateralAcceleration(DeltaTime)
  |     +-- BoostAirControl() when speed is under AirControlBoostVelocityThreshold
  |     +-- GetAirControl() scales lateral input by AirControl (0..1)
  |     +-- ShouldLimitAirControl() / LimitAirControl() suppress control into a wall
  |
  +-- CalcVelocity(dt, FallingLateralFriction, false, BrakingDecelerationFalling)
  +-- NewFallVelocity(Velocity, Gravity, GravityTime)  // gravity integration, gravity-direction aware
  |
  +-- SafeMoveUpdatedComponent(Adjusted, Rotation, true, Hit)
  |     +-- valid landing spot -> ProcessLanded(Hit, remainingTime, Iterations)
  |     |     +-- SetPostLandedPhysics(Hit) -> usually SetMovementMode(MOVE_Walking)
  |     |     +-- ACharacter::Landed(Hit) -> OnLanded (Blueprint)
  |     |
  |     +-- wall hit -> HandleImpact(Hit) then SlideAlongSurface(...)
  |
  +-- no hit: carry the remaining time into the next iteration
```

`AirControl` at 0 gives a pure ballistic arc; at 1 the player steers freely in the air. `MaxJumpApexAttemptsPerSimulation` bounds how often a slice is split to land exactly on the apex. `bDontFallBelowJumpZVelocityDuringJump` holds vertical speed at `JumpZVelocity` while the jump key is held.

---

## NavMesh walking

`PhysNavWalking` projects the capsule onto the navmesh instead of sweeping against world geometry — cheap enough for large AI crowds, and the reason a nav-walking character can appear to stand on nothing.

```
PhysNavWalking(deltaTime, Iterations)
  +-- CalcVelocity(...)
  +-- move horizontally
  +-- FindNavFloor(TestLocation, NavFloorLocation)      // navmesh projection
  +-- bProjectNavMeshWalking: extra raycast to the real geometry for height
  +-- bSlideAlongNavMeshEdge: slide along the navmesh edge instead of
      projecting a point on the navmesh and moving to it
```

| Property | Effect |
|---|---|
| `bProjectNavMeshWalking` | Raycast to underlying geometry to conform height |
| `bProjectNavMeshOnBothWorldChannels` | Use `WorldStatic` and `WorldDynamic` for that raycast |
| `bSlideAlongNavMeshEdge` | Slide along the navmesh edge on contact |

`SetGroundMovementMode(EMovementMode)` picks which of `MOVE_Walking` / `MOVE_NavWalking` a character returns to after landing.

---

## Movement base update

A character standing on a moving object is carried by the base rather than by its own velocity.

```
PerformMovement(DeltaTime)
  |
  +-- MaybeUpdateBasedMovement(DeltaSeconds)
  |     +-- MovementBaseUtility::UseRelativeLocation(BaseData)?
  |     +-- base is simulating physics -> defer (bDeferUpdateBasedMovement)
  |     +-- otherwise UpdateBasedMovement(DeltaSeconds)
  |           +-- MovementBaseUtility::GetMovementBaseTransform(BaseData, BoneName, Loc, Quat)
  |           +-- move the capsule by the base delta
  |           +-- UpdateBasedRotation(FinalRotation, ReducedRotation)
  |           +-- on failure: OnUnableToFollowBaseMove(DeltaPosition, OldLocation, MoveOnBaseHit)
  |
  +-- ... physics ...
  |
  +-- SetBaseFromFloor(CurrentFloor)   // via SetBase(FMovementBaseInterfaceData*, BoneName)
  +-- SaveBaseLocation()
```

The base itself is `FMovementBaseInterfaceData` (`Interfaces/MovementBaseInterface.h`), holding `PhysicsObjectOwner` plus the `IPhysicsBodyInstanceOwner` that implements base behaviour. Read it with `GetMovementBaseInterfaceData()`; `GetMovementBaseObject()` returns the raw `UObject*` and needs an explicit cast to reach a `UPrimitiveComponent`.

Relative state replicates through `FBasedMovementInfo` (`BaseID`, `bServerHasBaseComponent`, `bRelativeRotation`, `bServerHasVelocity`, `BoneName`, `Location`, `Rotation`, `MovementBaseInterfaceData`). `HasRelativeLocation()`, `HasRelativeRotation()` and `IsBaseUnresolved()` answer whether the client has resolved the base yet — `IsBaseUnresolved()` is true when the server says there is a base but the object has not streamed in.

`bBaseOnAttachmentRoot`, `bStayBasedInAir` and `StayBasedInAirHeight` control whether a character keeps its base while airborne.

---

## Root motion

Two independent paths merge in `PerformMovement`.

```
[Gameplay sources]
  ApplyRootMotionSource(TSharedPtr<FRootMotionSource>)  -> uint16 LocalID
    +-- stored in CurrentRootMotion (FRootMotionSourceGroup)
    +-- each source's PrepareRootMotion() runs per tick
    +-- ApplyRootMotionToVelocity(deltaTime) folds it into Velocity
    +-- Priority orders sources; AccumulateMode Override vs Additive
    +-- Duration < 0 runs until RemoveRootMotionSource(InstanceName)

[Animation]
  UAnimInstance extracts montage root motion each tick
    +-- accumulated into RootMotionParams (FRootMotionMovementParams)
    +-- HasAnimRootMotion() gates the velocity override
    +-- replicated to simulated proxies through FRepRootMotionMontage
```

Network behaviour:

- Autonomous proxy predicts both paths. `FSavedMove_Character::SavedRootMotion` and `RootMotionMontage` / `RootMotionTrackPosition` are captured per move so replay reproduces them.
- Server IDs are mapped to client IDs (`FRootMotionServerToLocalIDMapping`) so a correction can match the right source.
- Corrections arrive through `ClientAdjustRootMotionPosition_Implementation` (animation) and `ClientAdjustRootMotionSourcePosition_Implementation` (sources) — both now take `FMovementBaseInterfaceData*`.
- Simulated proxies receive the replicated result and smooth it with `NetworkSmoothingMode`.

`ERootMotionFinishVelocityMode` on `FRootMotionSource::FinishVelocityParams` decides what happens to velocity when a source ends: maintain it, clamp it, or set an explicit value.

---

## Async physics path

CMC can run its simulation on the physics thread behind the CVar `p.AsyncCharacterMovement` (default `0`; the engine marks it not fully developed and discourages its use).

```
Game thread                          Physics thread
-----------                          --------------
BuildAsyncInput()                    FCharacterMovementComponentAsyncCallback
  +-- FillAsyncInput(InputVector, In)   +-- consumes FCharacterMovementComponentAsyncInput
  +-- AccumulateRootMotionForAsync()    +-- produces FCharacterMovementComponentAsyncOutput
PostBuildAsyncInput()
...
ProcessAsyncOutput()
  +-- ApplyAsyncOutput(Output)
```

Registration is `RegisterAsyncCallback()` / `IsAsyncCallbackRegistered()`. The input struct carries `FCharacterAsyncInput`, `FUpdatedComponentAsyncInput`, `FCharacterMovementGTInputs`, `FCachedMovementBaseAsyncData` and `FRootMotionAsyncData`; the output carries `FCharacterAsyncOutput` and `FUpdatedComponentAsyncOutput`. Custom `Phys*` state is not mirrored automatically — a subclass must copy it through these structs or it will not survive an async frame.

---

## Debugging

| Symptom | Where to look |
|---|---|
| Character falls through geometry | `FindFloor` / `ComputeFloorDist`, collision channel on the capsule, `MaxSimulationTimeStep` vs frame time |
| Cannot climb a slope | `WalkableFloorAngle` (set via `SetWalkableFloorAngle`), `IsWalkable` override |
| Snags on small ledges | `MaxStepHeight`, `PerchRadiusThreshold`, `PerchAdditionalHeight` |
| Slides off a moving platform | movement base resolution — `GetMovementBaseInterfaceData()`, `MovementBaseUtility::IsDynamicBase`, `bStayBasedInAir` |
| Rubber-banding under latency | missing state in `SetMoveFor` / `PrepMoveFor`; check `p.NetShowCorrections` |
| Jitter on other players' screens | `NetworkSmoothingMode`, `SmoothClientPosition` |
| Custom mode ignored on the server | `UpdateFromCompressedFlags` not reading the flag, or `GetCurrentNetworkMoveData()` not consulted in `MoveAutonomous` |
| Velocity spikes after teleport | `bJustTeleported` cleared too early, or a manual `SetActorLocation` instead of `SafeMoveUpdatedComponent` |

`DisplayDebug` on `ACharacter` and the `showdebug` console category print movement mode, velocity, floor state and base. Prediction correctness is best tested with artificial latency plus `p.NetShowCorrections 1`.
