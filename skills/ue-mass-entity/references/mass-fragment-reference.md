# Mass Fragment Reference

Built-in element types, shared fragments and traits, with the header and Build.cs module each one needs in UE 5.8. Headers under `Mass/` belong to the `MassCore` runtime module; the rest come from the MassGameplay, MassAI and MassCrowd plugins (all Experimental in 5.8).

---

## Element base types

All declared in `Mass/EntityElementTypes.h` (module `MassCore`), all deriving from `FMassElement`. `MassEntityTypes.h` pulls them in transitively.

| Base | Kind | Notes |
|---|---|---|
| `FMassFragment` | Per-entity data | Stored contiguously per chunk |
| `FMassSparseFragment` | Per-entity data | Lives outside the archetype; add/remove causes no entity move |
| `FMassTag` | Marker | Must declare no data members |
| `FMassSparseTag` | Marker | Add/remove without an archetype change |
| `FMassChunkFragment` | Per-chunk data | One instance per memory chunk |
| `FMassSharedFragment` | Shared mutable | One instance shared by referencing entities |
| `FMassConstSharedFragment` | Shared immutable | Configuration set at template build time |
| `FMassRelation` | Relation type | Derives from `FMassTag`; declared in `MassEntityRelations.h` (module `MassEntity`) |

---

## Per-entity fragments

| Fragment | Header | Module | Key fields |
|---|---|---|---|
| `FTransformFragment` | `Mass/EntityFragments.h` | `MassCore` | `FTransform Transform` (protected) |
| `FAgentRadiusFragment` | `MassCommonFragments.h` | `MassCommon` | `float Radius = 40.f` |
| `FAgentHeightFragment` | `MassCommonFragments.h` | `MassCommon` | `float Height = 180.f` |
| `FMassVelocityFragment` | `MassMovementFragments.h` | `MassMovement` | `FVector Value` |
| `FMassForceFragment` | `MassMovementFragments.h` | `MassMovement` | `FVector Value` |
| `FMassMoveTargetFragment` | `MassNavigationFragments.h` | `MassNavigation` (MassAI) | `FVector Center`, `FVector Forward`, `float DistanceToGoal`, `float SlackRadius`, `FMassInt16Real DesiredSpeed`, `EMassMovementAction IntentAtGoal` |
| `FMassRepresentationFragment` | `MassRepresentationFragments.h` | `MassRepresentation` | `EMassRepresentationType CurrentRepresentation`, `PrevRepresentation` |
| `FMassRepresentationLODFragment` | `MassRepresentationFragments.h` | `MassRepresentation` | `TEnumAsByte<EMassLOD::Type> LOD`, `PrevLOD`, `EMassVisibility Visibility`, `float LODSignificance` |
| `FMassCrowdLaneTrackingFragment` | `MassCrowdFragments.h` | `MassCrowd` | `FZoneGraphLaneHandle TrackedLaneHandle` |
| `FMassChildOfFragment` | `Relations/MassChildOf.h` | `MassEntity` | `FMassEntityHandle Parent` |

### FTransformFragment

`Transform` is protected. Use `GetTransform()` (const ref), `GetMutableTransform()` (mutable ref) and `SetTransform(const FTransform&)`. A converting constructor from `FTransform` exists, which is what makes `Builder.Add<FTransformFragment>(SpawnTransform)` work.

This type moved into `MassCore`. `MassCommonFragments.h` only `#include`s it; `MassCommon` lists `MassCore` as a public dependency (`MassCommon.Build.cs:19`) so it resolves transitively, but include `Mass/EntityFragments.h` and add `MassCore` explicitly.

### FMassVelocityFragment / FMassForceFragment

Both hold a single `FVector Value`. Forces are integrated into velocity, velocity into `FTransformFragment`, by the processors in `Movement/MassMovementProcessors.h` (`UMassApplyMovementProcessor`).

### FMassMoveTargetFragment

`DesiredSpeed` is an `FMassInt16Real`, not a float. The action is private: read it with `GetCurrentAction()` / `GetPreviousAction()` and change it with `CreateNewAction(EMassMovementAction, const UWorld&)`. `EMassMovementAction` (`MassNavigationTypes.h`) is `Stand`, `Move`, `Animate`.

---

## Chunk fragments

A chunk fragment derives from `FMassChunkFragment` and holds one instance per memory chunk. Add it to a template with `BuildContext.AddChunkFragment<T>()`, bind it with `Query.AddChunkRequirement<T>(Access, Presence)`, and read it with `Context.GetChunkFragment<T>()`. Chunk fragments are the only per-entity-adjacent data a `SetChunkFilter` predicate may read.

`FMassRepresentationLODFragment` is **not** a chunk fragment — it is a per-entity `FMassFragment`.

---

## Shared and const shared fragments

| Fragment | Kind | Header | Module | Key fields |
|---|---|---|---|---|
| `FMassMovementParameters` | Const shared | `MassMovementFragments.h` | `MassMovement` | `float MaxSpeed = 200.f`, `float MaxAcceleration = 250.f`, `float DefaultDesiredSpeed = 140.f`, `float DefaultDesiredSpeedVariance = 0.1f` |
| `FMassRepresentationParameters` | Const shared | `MassRepresentationFragments.h` | `MassRepresentation` | `EMassRepresentationType LODRepresentation[EMassLOD::Max]`, `float NotVisibleUpdateRate = 0.5f`, `uint8 bKeepLowResActors : 1` |
| `FMassStateTreeSharedFragment` | Const shared | `MassStateTreeFragments.h` | `MassAIBehavior` (MassAI) | `TObjectPtr<UStateTree> StateTree` |

`FMassMovementParameters` has no deceleration field; deceleration falls out of the force/velocity integration.

`FMassRepresentationParameters::LODRepresentation` maps each `EMassLOD::Type` slot to an `EMassRepresentationType`, which is how an entity is promoted to an actor up close and demoted back to an ISM instance at distance.

Bind shared data with `Query.AddSharedRequirement<T>(Access, Presence)` or `Query.AddConstSharedRequirement<T>(Presence)`; read it with `Context.GetMutableSharedFragment<T>()` / `Context.GetConstSharedFragment<T>()`. `Presence::Any` is rejected for both.

---

## Traits

Traits derive from `UMassEntityTraitBase` (`MassEntityTraitBase.h`, module `MassSpawner`) and implement `BuildTemplate(FMassEntityTemplateBuildContext&, const UWorld&) const`. Optional validation is `ValidateTemplate(const FMassEntityTemplateBuildContext&, const UWorld&, FAdditionalTraitRequirements&) const` — three parameters.

| Trait | Header | Module | Purpose |
|---|---|---|---|
| `UMassAssortedFragmentsTrait` | `MassAssortedFragmentsTrait.h` | `MassSpawner` | Editor-authored `Fragments` and `Tags` arrays of `FInstancedStruct` |
| `UMassMovableVisualizationTrait` | `MassMovableVisualizationTrait.h` | `MassRepresentation` | ISM/actor visualization for moving agents |
| `UMassStationaryVisualizationTrait` | `MassStationaryVisualizationTrait.h` | `MassRepresentation` | Visualization for agents that never move |
| `UMassDistanceVisualizationTrait` | `MassDistanceVisualizationTrait.h` | `MassRepresentation` | Distance-driven representation switching |
| `UMassReplicationTrait` | `MassReplicationTrait.h` | `MassReplication` | Registers the entity with Mass replication |
| `UMassStateTreeTrait` | `MassStateTreeTrait.h` | `MassAIBehavior` (MassAI) | Attaches a State Tree asset to the entity |
| `UMassCrowdMemberTrait` | `MassCrowdMemberTrait.h` | `MassCrowd` | Adds `FMassCrowdTag` and lane tracking |
| `UMassSmartObjectUserTrait` | `MassSmartObjectUserTrait.h` | `MassSmartObjects` | Lets the entity claim Smart Objects |

`UMassVisualizationTrait` is soft-deprecated (its display name is "DEPRECATED Visualization"). It is still the base class of the movable and stationary traits and still owns the shared properties `StaticMeshInstanceDesc` (`FStaticMeshInstanceVisualizationDesc`), `SkinnedMeshInstanceDesc`, `HighResTemplateActor`, `LowResTemplateActor`, `Params` (`FMassRepresentationParameters`) and `LODParams` (`FMassVisualizationLODParameters`). Author new content against `UMassMovableVisualizationTrait` or `UMassStationaryVisualizationTrait`.

Assign traits to a `UMassEntityConfigAsset` (a `UDataAsset` in `MassEntityConfigAsset.h`); `AMassSpawner` lists config assets in `EntityTypes` and spawns `Count` entities.

---

## Enums

### EMassRepresentationType (`MassRepresentationTypes.h`, `MassRepresentation`)

| Value | Use |
|---|---|
| `HighResSpawnedActor` | Full actor, closest entities |
| `LowResSpawnedActor` | Cheap actor at medium range |
| `SkinnedMeshInstance` | Instanced skinned mesh |
| `StaticMeshInstance` | Instanced static mesh, the bulk of a crowd |
| `None` | Not rendered |

`UE::Mass::Representation::IsValidMeshRepresentation()` returns true for the two instanced-mesh values.

### EMassLOD (`MassLODTypes.h`, `MassLOD`)

`EMassLOD` is a namespaced enum — `namespace EMassLOD { enum Type : int { High, Medium, Low, Off, Max }; }` — so it is stored as `TEnumAsByte<EMassLOD::Type>` and `EMassLOD::Max` doubles as the array size for `LODRepresentation`.

### EMassVisibility (`MassLODTypes.h`, `MassLOD`)

`CanBeSeen`, `CulledByFrustum`, `CulledByDistance`, `Max`.

### EMassObservedOperationFlags (`MassEntityTypes.h`, `MassEntity`)

`None`, `AddElement`, `RemoveElement`, `CreateEntity`, `DestroyEntity`, `Add` (= `AddElement | CreateEntity`), `Remove` (= `RemoveElement | DestroyEntity`), `All`.

### EMassCommandOperationType (`MassCommands.h`, `MassEntity`)

Flush order is by bucket, not enum order (`MassCommandBuffer.cpp:110-119`): `Create`, `Add`, `ChangeComposition`, `Set`, then `Remove` and `Destroy` together; commands left as `None` run last.

---

## Crowd types

| Type | Kind | Header |
|---|---|---|
| `FMassCrowdTag` | Tag | `MassCrowdFragments.h` |
| `FMassCrowdLaneTrackingFragment` | Fragment | `MassCrowdFragments.h` |
| `FMassCrowdObstacleFragment` | Fragment | `MassCrowdFragments.h` |
| `UMassCrowdSubsystem` | World subsystem | `MassCrowdSubsystem.h` |

`UMassCrowdSubsystem` derives from `UMassSubsystemBase` and tracks lane occupancy, density and waiting slots over a ZoneGraph. It specialises `TMassExternalSubsystemTraits<UMassCrowdSubsystem>` with `GameThreadOnly = false` and `ThreadSafeWrite = false`, so parallel processors may declare it through `AddSubsystemRequirement`.

---

## Signals

`UMassSignalSubsystem` (`MassSignalSubsystem.h`, module `MassSignals`) wakes processors for specific entities instead of ticking every entity every frame. The signal name comes first:

```cpp
SignalSubsystem->SignalEntity(FName("MyGame.Damaged"), Entity);
SignalSubsystem->DelaySignalEntity(FName("MyGame.Respawn"), Entity, 3.f);
SignalSubsystem->SignalEntityDeferred(Context, FName("MyGame.Damaged"), Entity);
```

Processors that react to signals derive from `UMassSignalProcessorBase` (`MassSignalProcessorBase.h`), which is also the base of `UMassStateTreeProcessor`.
