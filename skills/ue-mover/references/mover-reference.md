# Mover Reference (UE 5.8)

Catalogue of the shipped Mover types, the data model, backends, utility libraries and debugging surface. Mover is Experimental in 5.8; APIs and data formats are subject to change.

## Plugin and module layout

| Plugin | 5.8 maturity | Runtime modules | What it adds |
|---|---|---|---|
| `Mover` | Experimental | `Mover`, `MoverCVDData` (non-Shipping) | `UMoverComponent`, the default movement set, both non-physics backends |
| `MoverExamples` | Experimental | `MoverExamples` | Sample maps and pawns, including `AMoverExamplesCharacter` |
| `ChaosMover` | Experimental | `ChaosMover` | Physics-thread backend and Chaos character modes |
| `MoverAnimNext` | Experimental | `MoverAnimNext` | Bridge to the UAF animation graph |
| `MoverIntegrations` | Experimental | `MoverIntegrations`, `MoverMassIntegration` | Mass agent traits and a Mass input producer |
| `NetworkPrediction` | Beta | `NetworkPrediction` | Rollback networking under `UMoverNetworkPredictionLiaisonComponent` |

`Mover.uplugin` enables `NetworkPrediction`, `MotionWarping`, `PoseSearch`, `Water` and `ChaosVD` as dependencies. The `Mover` module's own public dependencies include `Core`, `NetCore`, `InputCore`, `NetworkPrediction`, `AnimGraphRuntime`, `MotionWarping`, `Water`, `GameplayTags` and `NavigationSystem`, so you normally only need `"Mover"` in your `Build.cs`.

Editor-only modules (`MoverEditor`, `MoverCVDEditor`) are `UncookedOnly` and should not appear in a runtime module's dependency list.

## Default movement set

All in `DefaultMovementSet/Modes/`. Every one derives from `UBaseMovementMode`.

| Mode | Header | Notes |
|---|---|---|
| `UWalkingMode` | `Modes/WalkingMode.h` | Ground movement with floor checks and step-up; overrides `Activate`, `GenerateMove_Implementation`, `SimulationTick_Implementation` |
| `UFallingMode` | `Modes/FallingMode.h` | Gravity, air control, landing; exposes `ProcessLanded` |
| `UFlyingMode` | `Modes/FlyingMode.h` | Free 3D movement; `bRespectDistanceOverWalkableSurfaces` keeps ground clearance |
| `USwimmingMode` | `Modes/SwimmingMode.h` | Water volumes; exposes `AttemptJump` |
| `UNavWalkingMode` | `Modes/NavWalkingMode.h` | Navmesh projection; `FindNavFloor`, `ProjectLocationFromNavMesh`, `SetCollisionForNavWalking` |
| `UAsyncWalkingMode` | `Modes/AsyncWalkingMode.h` | `Experimental` class specifier; worker-thread walking |
| `UAsyncFallingMode` | `Modes/AsyncFallingMode.h` | Worker-thread falling |
| `UAsyncFlyingMode` | `Modes/AsyncFlyingMode.h` | Worker-thread flying |
| `UAsyncNavWalkingMode` | `Modes/AsyncNavWalkingMode.h` | Worker-thread navmesh walking; CVars `Mover.AsyncNav.OverrideRaycastInterval`, `Mover.AsyncNav.UseNavMeshNormal` |
| `USimpleWalkingMode`, `USimpleFlyingMode`, `USimpleSpringWalkingMode`, `USmoothWalkingMode` | `Modes/Simple*.h`, `Modes/SmoothWalkingMode.h` | Reduced-complexity alternatives |

Mode names used by the shared settings: `DefaultModeNames::Walking`, `DefaultModeNames::Falling`, `DefaultModeNames::Flying`, `DefaultModeNames::Swimming` (declared in `MoverSimulationTypes.h`). `UNullMovementMode` (`UNullMovementMode::NullModeName`) is the do-nothing placeholder used when no mode is active.

Async modes are only useful when the backend runs asynchronously. Check with `UMoverComponent::IsBackendAsync()`, and mark your own mode `bSupportsAsync = true` only when `GenerateMove` and `SimulationTick` touch no scene components.

## Shared settings

Settings objects implement `IMovementSettingsInterface` (a `UINTERFACE(MinimalAPI, BlueprintType)` with `virtual FString GetDisplayName() const = 0;`). Each mode lists the classes it needs in `SharedSettingsClasses`, and `UMoverComponent` builds one shared instance per class into its private `SharedSettings` array. Look them up with `FindSharedSettings<T>()` / `FindSharedSettings_Mutable<T>()` (or `FindSharedSettings_BP` / `FindSharedSettings_Mutable_BP` from Blueprint).

`UCommonLegacyMovementSettings` (`DefaultMovementSet/Settings/CommonLegacyMovementSettings.h`) is what the default set uses:

| Field | Default | Meaning |
|---|---|---|
| `bShouldRemainVertical` | `true` | Keep the actor upright relative to gravity |
| `GroundMovementModeName` / `AirMovementModeName` / `SwimmingMovementModeName` | `Walking` / `Falling` / `Swimming` | Mode to switch to for each medium |
| `MaxWalkSlopeCosine` | `0.71` | Walkable slope, as `cos(angle)` |
| `FloorSweepDistance` | `40.0` cm | Floor scan distance |
| `bUseFlatBaseForFloorChecks` | `true` | Flat-bottom floor tests |
| `PerchRadiusThreshold` | `0.0` cm | Ledge perch rejection distance |
| `MaxStepHeight` | `40.0` cm | Step-up limit |
| `MaxSpeed` | `800` cm/s | Planar speed cap |
| `bUseAccelerationForVelocityMove` | `true` | Accelerate towards velocity input instead of setting it |
| `GroundFriction` | `8.0` | Direction-change responsiveness |
| `bUseSeparateBrakingFriction`, `BrakingFriction`, `BrakingFrictionFactor` | `8.0`, `2.0` | Braking model |
| `Acceleration` / `Deceleration` | `4000` cm/s² | Controlled / uncontrolled rate |
| `TurningRate` / `TurningBoost` | `500` deg/s, `8.0` | Rotation rate; negative `TurningRate` snaps |
| `bIgnoreBaseRotation` | `false` | Keep world rotation when the base rotates |

`UStanceSettings` (`Settings/StanceSettings.h`) backs the crouch modifier.

## Data model

`FMoverDataStructBase` (`MoverTypes.h`) is the base for everything carried in an input cmd, sync state or aux state. Derived types override:

```cpp
virtual FMoverDataStructBase* Clone() const;
virtual bool NetSerialize(FArchive& Ar, UPackageMap* Map, bool& bOutSuccess);
virtual UScriptStruct* GetScriptStruct() const;
virtual void ToString(FAnsiStringBuilderBase& Out) const;
virtual void AddReferencedObjects(FReferenceCollector& Collector);
virtual bool ShouldReconcile(const FMoverDataStructBase& AuthorityState) const;
virtual void Interpolate(const FMoverDataStructBase& From, const FMoverDataStructBase& To, float Pct);
virtual void Merge(const FMoverDataStructBase& From);
virtual void Decay(float DecayAmount);
```

`Merge` and `Decay` matter only for input data used by physics-based movement. `ShouldReconcile` and `Interpolate` matter for anything in the sync or aux state.

`FMoverDataCollection` holds one instance per struct type and provides `FindDataByType<T>()`, `FindMutableDataByType<T>()`, `FindOrAddDataByType<T>()`, `FindOrAddMutableDataByType<T>()`, `AddOrOverwriteData`, `AddDataByCopy`, `RemoveDataByType` and `GetDataArray()`. Blueprint access goes through `UMoverDataCollectionLibrary` (`K2_AddDataToCollection`, `K2_GetDataFromCollection`, `ClearDataFromCollection`).

Shipped data structs:

| Struct | Lives in | Contents |
|---|---|---|
| `FCharacterDefaultInputs` | input cmd | `SetMoveInput(EMoveInputType, FVector)`, `GetMoveInput()`, `GetMoveInputType()`, `GetMoveInput_WorldSpace()`, `OrientationIntent`, `GetOrientationIntentDir_WorldSpace()`, `ControlRotation`, `SuggestedMovementMode`, `bUsingMovementBase`, `MovementBase`, `MovementBaseBoneName`, `bIsJumpPressed`, `bIsJumpJustPressed` |
| `FMoverAIInputs` | input cmd | `RVOVelocityDelta` |
| `FMoverDefaultSyncState` | sync state | location / orientation / velocity / `AngularVelocityDegrees` (base-relative when a base is set), `MoveDirectionIntent`, movement base capture |

`EMoveInputType` values: `Invalid`, `DirectionalIntent` (per-axis magnitude in [-1,1]), `Velocity` (units per second), `None`.

`FMoverDefaultSyncState` accessors: `GetLocation_WorldSpace()` / `_BaseSpace()`, `GetIntent_*`, `GetVelocity_*`, `GetOrientation_*`, `GetTransform_*`, `GetAngularVelocityDegrees_*`, `GetMovementBase()`, `GetMovementBaseBoneName()`, `GetCapturedMovementBasePos()`, `GetCapturedMovementBaseQuat()`, `IsNearlyEqual()`. Writers: `SetTransforms_WorldSpace(...)`, `SetMovementBase(UPrimitiveComponent*, FName)`, `UpdateCurrentMovementBase()`. `UMoverDataModelBlueprintLibrary` exposes `SetDirectionalInput`, `SetVelocityInput`, `GetMoveDirectionIntentFromInputs`, `GetLocationFromSyncState`, `GetVelocityFromSyncState`, `GetAngularVelocityDegreesFromSyncState`, `GetOrientationFromSyncState`, `GetMoveDirectionIntentFromSyncState`.

`FProposedMove` mixing is driven by `EMoveMixMode`: `AdditiveVelocity` (default), `OverrideVelocity`, `OverrideAll`, `OverrideAllExceptVerticalVelocity`. `UMovementMixer::MixProposedMoves(const FProposedMove& MoveToMix, FVector UpDirection, FProposedMove& OutCumulativeMove)` performs the combination; assign a subclass to `UMoverComponent::MovementMixer` to change the rules.

`FMovementModeTickEndState` carries a mode's exit decision: `RemainingMs` (time refunded, producing another substep), `NextModeName`, `bEndedWithNoChanges` (lets the backend skip finalization work).

`FMoverTime` pairs a `FrameCount` (valid only under fixed ticking; `INDEX_NONE` otherwise) with `TimeMs`, and offers `ElapsedSecondsTo(Other, FixedFrameDtSeconds)`. `FMoverTimeStep::ToStartTime()` and `ToNextTime()` produce them.

## Layered moves, modifiers and effects shipped with the plugin

Header paths below are relative to `DefaultMovementSet/`; include them as `#include "DefaultMovementSet/LayeredMoves/BasicLayeredMoves.h"` (the same applies to the `Modes/...` and `Settings/...` paths above).

| Type | Header | Purpose |
|---|---|---|
| `FLayeredMove_LinearVelocity` | `LayeredMoves/BasicLayeredMoves.h` | Straight-line velocity, optional `MagnitudeOverTime` curve; supports async |
| `FLayeredMove_JumpImpulseOverDuration` | same | `UpwardsSpeed` applied over `DurationMs` |
| `FLayeredMove_JumpTo` | same | Ballistic arc to a destination |
| `FLayeredMove_MoveTo` / `FLayeredMove_MoveToDynamic` | same | Interpolate to a static / moving target |
| `FLayeredMove_RadialImpulse` | same | Blast forces |
| `FLayeredMove_MultiJump` | `LayeredMoves/MultiJumpLayeredMove.h` | Air jumps |
| `FLayeredMove_Launch` | `LayeredMoves/LaunchMove.h` | Launch velocity; also has instanced form `ULaunchMoveLogic` + `FLaunchMoveData` + `FLaunchMoveActivationParams` |
| `FLayeredMove_AnimRootMotion` | `LayeredMoves/AnimRootMotionLayeredMove.h` | Montage root motion, derives from `FLayeredMove_MontageStateProvider` |
| `FLayeredMove_AnimRootMotion_SimDriven` | same | Root motion advanced by the simulation |
| `FStanceModifier` | `MovementModifiers/StanceModifier.h` | Crouch / stance; `EStanceMode`, backed by `UStanceSettings` |
| `FTeleportEffect` / `FAsyncTeleportEffect` | `InstantMovementEffects/BasicInstantMovementEffects.h` | `TargetLocation`, `bUseActorRotation`, `TargetRotation` |
| `FJumpImpulseEffect` | same | Overrides vertical velocity and switches to `AirMovementModeName` |
| `FApplyVelocityEffect` | same | One-shot velocity change |

The instanced layered move path adds `ULayeredMoveLogic` (stateless `UObject`, `InstancedDataStructType`), `FLayeredMoveInstancedData` (replicated data) and `FLayeredMoveActivationParams`. `ULinearVelocityMoveLogic` + `FLinearVelocityMoveData` is the reference implementation. Register with `UMoverComponent::RegisterMove<T>()` / `K2_RegisterMove` / `K2_RegisterMoves`, unregister with `UnregisterMove<T>()` / `K2_UnregisterMove`, activate with `QueueLayeredMoveActivation` or `QueueLayeredMoveActivationWithContext`. Registration is flushed once per frame (`FlushPendingMoveRegistrations`), so it is thread-safe for async backends.

## Backend liaisons

`IMoverBackendLiaisonInterface` (`Backends/MoverBackendLiaison.h`):

```cpp
virtual double GetCurrentSimTimeMs() = 0;
virtual int32 GetCurrentSimFrame() = 0;
virtual bool IsAsync() const;
virtual bool IsFixedDt() const;
virtual bool ShouldResim() const;
virtual float GetEventSchedulingMinDelaySeconds() const;
virtual FMoverTime GetScheduledNetworkTime(const FMoverTime& Time) const;
virtual bool ReadPendingSyncState(OUT FMoverSyncState& OutSyncState);
virtual bool WritePendingSyncState(const FMoverSyncState& SyncStateToWrite);
virtual bool ReadPresentationSyncState(OUT FMoverSyncState& OutSyncState);
virtual bool WritePresentationSyncState(const FMoverSyncState& SyncStateToWrite);
virtual bool ReadPrevPresentationSyncState(OUT FMoverSyncState& OutSyncState);
virtual bool WritePrevPresentationSyncState(const FMoverSyncState& SyncStateToWrite);
virtual void OnSimulationRollback(const FMoverSyncState& NewSyncState, const FMoverTimeStep& NewBaseTimeStep, const FMoverTimeStep& PreRollbackTimeStep);
```

`UMoverComponent::BackendClass` is a `TSubclassOf<UActorComponent>` with `meta = (MustImplement = "/Script/Mover.MoverBackendLiaisonInterface")`.

**`UMoverNetworkPredictionLiaisonComponent`** (`Backends/MoverNetworkPredictionLiaison.h`) derives from `UNetworkPredictionComponent`. Its Network Prediction driver surface is `ProduceInput`, `ShouldReconcile` (sync and aux overloads), `RestoreFrame`, `FinalizeFrame`, `FinalizeSmoothingFrame`, `InitializeSimulationState` and `SimulationTick(const FNetSimTimeStep&, const TNetSimInput<KinematicMoverStateTypes>&, const TNetSimOutput<KinematicMoverStateTypes>&)`, where `KinematicMoverStateTypes = TNetworkPredictionStateTypes<FMoverInputCmdContext, FMoverSyncState, FMoverAuxStateContext>`.

**`UMoverStandaloneLiaisonComponent`** (`Backends/MoverStandaloneLiaison.h`) is a plain `UActorComponent` driving three tick functions — `FMoverStandaloneProduceInputTickFunction`, `FMoverStandaloneSimulateMovementTickFunction` and an apply-state tick — which map to `EMoverTickPhase::ProduceInput`, `SimulateMovement` and `ApplyState`. `AddTickDependency(UActorComponent* OtherComponent, EMoverTickDependencyOrder TickOrder, EMoverTickPhase TickPhase)` orders other components around a phase. `SetUseAsyncProduceInput` / `SetUseAsyncMovementSimulationTick` opt individual actors into worker-thread execution, gated globally by `mover.standalone.RunProduceInputOnAnyThread` and the matching simulation CVar.

**`UChaosMoverBackendComponent`** (`ChaosMover/Backends/ChaosMoverBackend.h`, `UCLASS(MinimalAPI, Within = MoverComponent)`) runs on the physics thread: `IsAsync()` and `IsFixedDt()` are true, and it exchanges `UE::ChaosMover::FSimulationInputData` / `FSimulationOutputData` via `ProduceInputData` and `ConsumeOutputData`, driving a Chaos character ground constraint through `UChaosMoverSimulation`. Its mode hierarchy is separate: `UChaosMovementMode` (derives from `UBaseMovementMode`), `UChaosCharacterMovementMode`, `UChaosWalkingMode`, `UChaosFallingMode`, `UChaosFlyingMode`, `UChaosSwimmingMode`, plus `UChaosCharacterMoverComponent` (derives from `UCharacterMoverComponent`). Known gaps: the teleport effect does not work, interaction with non-physics moving objects is unreliable, and there is extra latency between input and visible motion.

Scheduled variants (`ScheduleLayeredMove`, `ScheduleInstantMovementEffect`, `QueueScheduledLayeredMove`, `QueueScheduledInstantMovementEffect`) delay execution by `GetEventSchedulingMinDelaySeconds()` so every endpoint runs them on the same frame. They are only relevant to the ChaosMover networked-physics path; the plain `Queue*` calls are correct everywhere else.

## Movement utility libraries

All are `UBlueprintFunctionLibrary` subclasses under `MoveLibrary/`, callable from modes, layered moves and Blueprints.

| Library | Representative functions |
|---|---|
| `UMovementUtils` | `TrySafeMoveUpdatedComponent`, `TrySafeMoveUpdatedComponentNoMovementRecord`, `TryMoveToSlideAlongSurface`, `ComputeVelocityFromPositions`, `ComputeAngularVelocityDegrees`, `ComputeDirectionIntent`, `ApplyAngularVelocityToRotator`, `ApplyAngularVelocityToQuat`, `FindTeleportSpot` |
| `UGroundMovementUtils` | `ComputeControlledGroundMove`, `TryMoveToStepUp`, `TestMoveToStepOver`, `TryMoveToAdjustHeightAboveFloor`, `TryMoveToKeepMinHeightAboveFloor` |
| `UAirMovementUtils` | `ComputeControlledFreeMove`, `IsValidLandingSpot`, `TryMoveToFallAlongSurface`, `TestFallingMoveAlongHitSurface` |
| `UFloorQueryUtils` | `FindFloor`, `ComputeFloorDist`, `FloorSweepTest`, `IsHitSurfaceWalkable`, `IsWithinEdgeTolerance` |
| `UWaterMovementUtils` | water volume queries for `USwimmingMode` |
| `UBasedMovementUtils` | `GetMovementBaseVelocityAtPoint`, `AddTickDependency`, `RemoveTickDependency` |
| `UPlanarConstraintUtils` | `ConstrainDirectionToPlane` and friends, driven by `FPlanarConstraint` |
| `UAsyncMovementUtils` | threadsafe move tests for async modes |

The floor, air and ground libraries took a settings-struct refactor in 5.8: pass `FFloorCheckSettings` (`FloorSweepDistance`, `MaxWalkSlopeCosine`, `bUseFlatBaseForFloorChecks`, `PerchRadiusThreshold`) instead of loose floats. Results come back as `FFloorCheckResult` (`bBlockingHit`, `bWalkableFloor`, `bLineTrace`, `LineDist`, `FloorDist`, `HitResult`, `IsWalkableFloor()`).

`FMovementRecord` tracks the substeps of a move so the resulting velocity ignores non-contributing motion (for example the vertical part of a step-up). Call `SetDeltaSeconds(DeltaSeconds)` at the start of a tick, let the utility functions `Append` substeps, then read `GetRelevantVelocity()` when writing the ending sync state.

## Blackboards

`UMoverBlackboard` (`MoveLibrary/MoverBlackboard.h`) is a name-keyed, copy-in/copy-out store for values that must not enter the networked state: `TryGet<T>(FName, T& OutFoundValue)`, `Set<T>(FName, T)`, `Contains(FName)`, `Invalidate(FName)`, `Invalidate(EInvalidationReason)` and `InvalidateAll()`. `EInvalidationReason` is `FullReset` or `Rollback`.

Well-known keys in `namespace CommonBlackboard`: `LastFloorResult`, `LastWaterResult`, `LastFoundDynamicMovementBase`, `AccumulatedBasedTransformDelta`, `LastBasedMovementAppliedEventTime`, `TimeSinceSupported`, `LastModeChangeRecord`.

It is being replaced by the rollback-aware blackboard (`URollbackBlackboard`, reached through `UMoverComponent::GetRollbackBlackboardExternal()` and `FMoverSimContext::Blackboard`). Write code that tolerates a missing entry. Entry lifetime is `EBlackboardPersistencePolicy` (`Forever`, `ThroughNextFrame`, `CurrentFrameOnly`, `LimitedTime`) and rollback behaviour is `EBlackboardRollbackPolicy`. `TryGet<T>`/`Set<T>` are type-strict: a value stored as `double` must be read as `double`, or you get garbage.

## Gameplay tags and simulation events

Native tags from `MoverTypes.h`: `Mover_IsOnGround`, `Mover_IsInAir`, `Mover_IsFalling`, `Mover_IsFlying`, `Mover_IsSwimming`, `Mover_IsCrouching`, `Mover_IsNavWalking`, `Mover_AnimRootMotion`, `Mover_SkipAnimRootMotion`, `Mover_SkipVerticalAnimRootMotion`, `Mover_DisableLanding`. A mode contributes its `GameplayTags` container while active; layered moves and modifiers contribute through `HasGameplayTag` / `GetGameplayTags` overrides. External code adds loose tags with `AddGameplayTag`, `AddGameplayTags`, `RemoveGameplayTag`, `RemoveGameplayTags`, and queries with `HasGameplayTag`, `HasGameplayTagInState`, `GetGameplayTags`.

Simulation events derive from `FMoverSimulationEventData`: `FMovementModeChangedEventData`, `FTeleportSucceededEventData`, `FTeleportFailedEventData` (with `ETeleportFailureReason`), `FMoverGameplayTagChangeEventData`. They are dispatched to the matching dynamic delegates and to the C++-only `OnPostSimEventReceived`. `FMoverEventContext` tells you `EventTimeMs`, `ServerFrame`, `bIsDuringResimulation` and `bIsCausedByRollback` — gate cosmetic responses on those.

## Gravity, constraints and smoothing

```cpp
MoverComponent->SetGravityOverride(true, FVector(0.0, 0.0, -980.0));
const FVector Gravity = MoverComponent->GetGravityAcceleration();
const FVector Up      = MoverComponent->GetUpDirection();
const FQuat WorldToGravity = MoverComponent->GetWorldToGravityTransform();
MoverComponent->SetUpDirectionOverride(true, FVector::UpVector);
MoverComponent->SetPlanarConstraint(FPlanarConstraint());
```

`EMoverSmoothingMode` is `None` or `VisualComponentOffset` (the default). Visual smoothing moves the primary visual component relative to the movement root, so pair it with `SetPrimaryVisualComponent` / `GetPrimaryVisualComponent` and `SetBaseVisualComponentTransform`. `bSupportsKinematicBasedMovement` drives the based-movement tick that keeps an actor riding a moving platform; `SetUseDeferredGroupMovement` / `IsUsingDeferredGroupMovement` trade transform latency for throughput on crowds and require `s.GroupedComponentMovement.Enable`.

`bWarnOnExternalMovement` logs when something outside the simulation moves the actor; `bAcceptExternalMovement` makes the next state update absorb such a move instead of fighting it.

## Debugging

- **Log category** `LogMover` (`MoverLog.h`).
- **Gameplay Debugger**: activate the tool and toggle the Mover category from the numpad; `gdt.*` console commands change the selected actor.
- **Console commands** are prefixed `Mover.` and `mover.`. Useful ones: `Mover.LocalPlayer.ShowTrail`, `Mover.LocalPlayer.ShowTrajectory`, `Mover.LocalPlayer.ShowCorrections`, `mover.debug.ShowTeleportDiffs`, `mover.perf.SkipGenerateMoveIfOverridden`, `Mover.Input.CharacterDefaultInputsDecayAmountMultiplier`.
- **`UMoverDebugComponent`** (`Debug/MoverDebugComponent.h`) records motion history: `SetHistoryTracking(float SecondsToTrack, float SamplesPerSecond)` and `GetPastTrajectory()`.
- **Chaos Visual Debugger**: the `MoverCVDData` module traces sync state, active layered moves and modifiers when CVD support is compiled in.
- **`UMoverDeveloperSettings`** (`config = Engine`, shown as "Mover Settings") holds runtime knobs such as `MaxTimesToRefundSubstep`, which guards against two modes handing time back and forth forever.

## Known limitations in 5.8

From the plugin README and the headers:

- Network Prediction simulations tick before the world tick groups and in a fixed order, so tight ticking dependencies with other systems are hard to express.
- Fixed-rate simulation with a variable render rate needs `bEnableFixedTickSmoothing` plus `EMoverSmoothingMode::VisualComponentOffset`, or motion looks rough.
- Forward-predicting other players' pawns mispredicts on direction changes and jumps; `ENetworkLOD::Interpolated` is the safer default for simulated proxies.
- Blueprint coverage is incomplete; some work still requires C++.
- Replacing entries of `MovementModes` on level-placed instances can leave the actor unable to move — prefer runtime registration or untouched defaults.
- The default movement set assumes a capsule-like, horizontally symmetric root shape even though `UMoverComponent` itself does not.
- Only one movement modifier per type is supported unless you override `Matches`.
- Mover does not solve Gameplay Ability System synchronisation; GAS replicates independently.
