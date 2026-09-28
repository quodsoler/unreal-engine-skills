---
name: ue-mass-entity
description: "Use when writing or debugging Mass Entity ECS code in Unreal Engine C++ — processors, queries, fragments, tags, observers, traits, spawning and large-scale agent simulation. Also use when the user mentions 'MassEntity', 'UMassProcessor', 'FMassEntityQuery', 'ForEachEntityChunk', 'FMassEntityManager', 'FMassFragment', 'FMassTag', 'FTransformFragment', 'UMassObserverProcessor', 'ObservedTypes', 'Defer', 'archetype', 'AMassSpawner', 'entity config asset', 'MassCrowd', 'ISM crowd', 'entity relations', or 'thousands of agents'. For State Tree behavior on entities, see ue-state-trees; for ZoneGraph and Smart Objects, see ue-ai-navigation; for thread-safety rules, see ue-async-threading."
metadata:
  version: "2.0.0"
  engine: "5.8"
---

# UE Mass Entity

Target engine: **UE 5.8**. APIs below are verified against the 5.8 headers; older forms are listed under "Deprecated — do not use".

Mass is an archetype Entity Component System for simulating thousands of agents with cache-friendly data layouts and multi-threaded processors. The core lives in the always-available runtime modules `MassEntity` and `MassCore` (plus `MassSignals`, `MassEngine`); gameplay-facing pieces (spawning, representation, movement, LOD, replication, actor bridging) come from the **MassGameplay (Experimental in 5.8)** plugin, and navigation/behavior from **MassAI (Experimental in 5.8)** and **MassCrowd (Experimental in 5.8)**.

## Context

Read `.agents/ue-project-context.md` if it exists (module names, conventions, enabled plugins, GAS/networking setup). Do not stop if it is missing.

Identify the area from the request and the codebase. Ask only when two plausible readings would produce different code.

| Request is about… | Go to |
|---|---|
| Build.cs, includes, "enable the plugin" | [Modules and Build.cs](#modules-and-buildcs) |
| Declaring fragments, tags, shared/chunk data | [Fragments, Tags and Archetypes](#fragments-tags-and-archetypes) |
| Creating/destroying entities, direct mutation | [FMassEntityManager](#fmassentitymanager) |
| Writing a processor, scheduling, phases | [UMassProcessor](#umassprocessor) |
| Selecting entities, requirements, filtering | [FMassEntityQuery](#fmassentityquery) |
| Reading/writing fragment data in a loop | [Iteration and FMassExecutionContext](#iteration-and-fmassexecutioncontext) |
| Adding/removing fragments during iteration | [Deferred Commands](#deferred-commands) |
| Reacting to fragment/tag add or remove | [UMassObserverProcessor](#umassobserverprocessor) |
| One entity outside a processor, fluent build | [Single-Entity Access and FEntityBuilder](#single-entity-access-and-fentitybuilder) |
| Parent/child links, archetype grouping | [Relations and Archetype Groups](#relations-and-archetype-groups) |
| Traits, config assets, `AMassSpawner` | [Traits, Config Assets and Spawning](#traits-config-assets-and-spawning) |
| ISM rendering, LOD, actor promotion | [Representation and LOD](#representation-and-lod) |
| Crowds, lanes, entity AI | [Crowd, Navigation and Behavior](#crowd-navigation-and-behavior) |

Longer templates live in [mass-entity-patterns.md](references/mass-entity-patterns.md); built-in types and fields in [mass-fragment-reference.md](references/mass-fragment-reference.md).

## Modules and Build.cs

There is no MassEntity plugin to enable. `MassEntity` and `MassCore` are runtime modules that ship with the engine — add them to `Build.cs` and they link. Only the gameplay layers are plugins, and those must be enabled in the `.uproject`.

```cs
PublicDependencyModuleNames.AddRange(new string[] {
    "Core", "CoreUObject", "Engine",
    "MassEntity",          // manager, processors, queries, commands, relations
    "MassCore",            // FMassFragment/FMassTag bases, FTransformFragment, traits
    "MassSignals",         // UMassSignalSubsystem, UMassSignalProcessorBase
    "MassCommon",          // UE::Mass::ProcessorGroupNames      (MassGameplay)
    "MassSpawner",         // UMassEntityTraitBase, AMassSpawner  (MassGameplay)
    "MassMovement",        // FMassVelocityFragment               (MassGameplay)
    "MassRepresentation",  // ISM visualization + LOD             (MassGameplay)
    "MassActors",          // UMassAgentComponent, actor bridging (MassGameplay)
    "MassNavigation",      // FMassMoveTargetFragment             (MassAI)
});
```

Include paths that moved into `MassCore` are namespaced under `Mass/`: `Mass/EntityElementTypes.h`, `Mass/EntityFragments.h`, `Mass/EntityHandle.h`, `Mass/ExternalSubsystemTraits.h`, `Mass/ArchetypeGroup.h`. `MassEntityTypes.h` still pulls the element bases in transitively; prefer the explicit `Mass/` path.

## Fragments, Tags and Archetypes

| Concept | Base / type | Purpose |
|---|---|---|
| Entity | `FMassEntityHandle` | Index + SerialNumber identity handle |
| Fragment | `FMassFragment` | Per-entity mutable data |
| Sparse fragment | `FMassSparseFragment` | Per-entity data stored outside the archetype (no entity move on add) |
| Tag | `FMassTag` | Zero-size marker used for filtering |
| Sparse tag | `FMassSparseTag` | Marker added/removed without an archetype change |
| Chunk fragment | `FMassChunkFragment` | One instance per memory chunk |
| Shared fragment | `FMassSharedFragment` | Mutable value shared by entities that reference it |
| Const shared fragment | `FMassConstSharedFragment` | Immutable shared configuration |
| Archetype | `FMassArchetypeHandle` | A unique fragment/tag composition |

All of these derive from `FMassElement` and are declared in `Mass/EntityElementTypes.h`.

```cpp
// MyMassTypes.h
#pragma once
#include "Mass/EntityElementTypes.h"
#include "MyMassTypes.generated.h"

USTRUCT()
struct FMyHealthFragment : public FMassFragment
{
    GENERATED_BODY()

    UPROPERTY(EditAnywhere, Category = "Mass")
    float Current = 100.f;

    UPROPERTY(EditAnywhere, Category = "Mass")
    float Max = 100.f;
};

USTRUCT()
struct FMyAliveTag : public FMassTag { GENERATED_BODY() };

USTRUCT()
struct FMyDeadTag : public FMassTag { GENERATED_BODY() };

USTRUCT()
struct FMySquadSharedFragment : public FMassSharedFragment
{
    GENERATED_BODY()

    UPROPERTY(EditAnywhere, Category = "Mass")
    int32 SquadID = 0;
};
```

Entities with the same composition share an archetype, and each fragment type is stored contiguously per chunk. Adding or removing a fragment or tag moves the entity to another archetype — that is why structural change is deferred during iteration. Sparse fragments and sparse tags exist precisely to avoid that move.

## FMassEntityManager

`FMassEntityManager` is not a `UObject`; it is a struct deriving from `TSharedFromThis<FMassEntityManager>` and `FGCObject`. Reach it through `UMassEntitySubsystem` (a `UMassSubsystemBase`, itself a `UWorldSubsystem`) or through `UE::Mass::Utils`.

```cpp
#include "MassEntitySubsystem.h"
#include "MassEntityManager.h"

UMassEntitySubsystem* MassSubsystem = GetWorld()->GetSubsystem<UMassEntitySubsystem>();
FMassEntityManager& EntityManager = MassSubsystem->GetMutableEntityManager();
// const access: MassSubsystem->GetEntityManager()
// free function: UE::Mass::Utils::GetEntityManagerChecked(*GetWorld())
```

```cpp
FMassEntityHandle Entity = EntityManager.CreateEntity(ArchetypeHandle);

FMassEntityHandle Reserved = EntityManager.ReserveEntity();
EntityManager.BuildEntity(Reserved, ArchetypeHandle);

// Batch creation: hold the returned context until observers should fire.
TArray<FMassEntityHandle> Entities;
TSharedRef<FMassEntityManager::FEntityCreationContext> CreationContext =
    EntityManager.BatchCreateEntities(ArchetypeHandle, 5000, Entities);

EntityManager.DestroyEntity(Entity);
EntityManager.BatchDestroyEntities(Entities);
```

`FMassEntityHandle::IsSet()` (and its alias `IsValid()`) only reports that Index and SerialNumber are non-zero. To ask whether an entity really exists, use the manager:

```cpp
EntityManager.IsEntityValid(Handle);   // handle refers to a live entity
EntityManager.IsEntityBuilt(Handle);   // reserved handle has been built
EntityManager.IsEntityActive(Handle);  // valid and built
```

Outside processor iteration, direct structural mutation is legal. The add/tag/swap calls take `TNotNull<const UScriptStruct*>` (`RemoveFragmentFromEntity` still takes a plain `const UScriptStruct*`, `MassEntityManager.h:430`), so pass `T::StaticStruct()` and never a possibly-null pointer.

```cpp
EntityManager.AddFragmentToEntity(Handle, FMyHealthFragment::StaticStruct());
EntityManager.RemoveFragmentFromEntity(Handle, FMyHealthFragment::StaticStruct());
EntityManager.AddTagToEntity(Handle, FMyDeadTag::StaticStruct());
EntityManager.RemoveTagFromEntity(Handle, FMyDeadTag::StaticStruct());
EntityManager.SwapTagsForEntity(Handle, FMyAliveTag::StaticStruct(), FMyDeadTag::StaticStruct());
```

Entity storage is concurrent: `FMassEntityManager::Initialize(const FMassEntityManagerStorageInitParams&)` takes `FMassEntityManager_InitParams_Concurrent` (`MaxEntityCount`, `MaxEntitiesPerPage`) and builds a `FConcurrentEntityStorage`. The subsystem does this for you; only standalone managers need the call.

## UMassProcessor

Subclass `UMassProcessor`, construct the query with `*this` so it registers itself, and override the two virtuals exactly as declared.

```cpp
// MyMovementProcessor.h
#pragma once
#include "MassProcessor.h"
#include "MassEntityQuery.h"
#include "MyMovementProcessor.generated.h"

UCLASS()
class MYGAME_API UMyMovementProcessor : public UMassProcessor
{
    GENERATED_BODY()
public:
    UMyMovementProcessor();
protected:
    virtual void ConfigureQueries(const TSharedRef<FMassEntityManager>& EntityManager) override;
    virtual void Execute(FMassEntityManager& EntityManager, FMassExecutionContext& Context) override;
private:
    FMassEntityQuery MovementQuery;
};
```

```cpp
// MyMovementProcessor.cpp
#include "MyMovementProcessor.h"
#include "MassCommonTypes.h"        // UE::Mass::ProcessorGroupNames
#include "MassExecutionContext.h"
#include "MassMovementFragments.h"   // FMassVelocityFragment, FMassMovementParameters
#include "MassSignalSubsystem.h"
#include "Mass/EntityFragments.h"    // FTransformFragment
#include "MyMassTypes.h"             // FMyHealthFragment, FMyDeadTag, FMySquadSharedFragment

UMyMovementProcessor::UMyMovementProcessor()
    : MovementQuery(*this)
{
    ProcessingPhase = EMassProcessingPhase::PrePhysics;
    ExecutionFlags = static_cast<int32>(EProcessorExecutionFlags::AllNetModes);
    ExecutionOrder.ExecuteInGroup = UE::Mass::ProcessorGroupNames::Movement;
    ExecutionOrder.ExecuteAfter.Add(TEXT("MassApplyMovementProcessor")); // class FName: no U prefix
    bRequiresGameThreadExecution = false;
}
```

`EMassProcessingPhase`: `PrePhysics`, `StartPhysics`, `DuringPhysics`, `EndPhysics`, `PostPhysics`, `FrameEnd`, `MAX`.

`EProcessorExecutionFlags`: `None`, `Standalone`, `Server`, `Client`, `Editor`, `EditorWorld`, `AllNetModes` (Standalone|Server|Client), `AllWorldModes` (AllNetModes|EditorWorld), `All`.

`FMassProcessorExecutionOrder` carries `ExecuteInGroup` (an `FName`) plus `ExecuteBefore` and `ExecuteAfter` arrays of processor or group names. A processor's name is its class `FName` without the `U` prefix (`MassProcessorDependencySolver.cpp:560`) — prefer `UOther::StaticClass()->GetFName()`; a mistyped name just becomes a dummy node with a Log-verbosity message (`:705`), so the ordering is silently lost. Named groups come from `UE::Mass::ProcessorGroupNames`: `UpdateWorldFromMass`, `SyncWorldToMass`, `Behavior`, `Tasks`, `Avoidance`, `ApplyForces`, `Movement`.

## FMassEntityQuery

Declare requirements in `ConfigureQueries` — never in the constructor, because the query is bound to the entity manager just before `ConfigureQueries` runs. Queries built with `FMassEntityQuery(UMassProcessor& Owner)` are already registered; a default-constructed member still needs `RegisterQuery(Query)`.

```cpp
void UMyMovementProcessor::ConfigureQueries(const TSharedRef<FMassEntityManager>& EntityManager)
{
    MovementQuery.AddRequirement<FTransformFragment>(
        EMassFragmentAccess::ReadWrite, EMassFragmentPresence::All);
    MovementQuery.AddRequirement<FMassVelocityFragment>(
        EMassFragmentAccess::ReadOnly, EMassFragmentPresence::All);
    MovementQuery.AddRequirement<FMyHealthFragment>(
        EMassFragmentAccess::ReadOnly, EMassFragmentPresence::Optional);
    MovementQuery.AddTagRequirement<FMyDeadTag>(EMassFragmentPresence::None);
    MovementQuery.AddSharedRequirement<FMySquadSharedFragment>(
        EMassFragmentAccess::ReadOnly, EMassFragmentPresence::All);
    MovementQuery.AddConstSharedRequirement<FMassMovementParameters>(
        EMassFragmentPresence::All);
    MovementQuery.AddSubsystemRequirement<UMassSignalSubsystem>(
        EMassFragmentAccess::ReadWrite);
}
```

| `EMassFragmentAccess` | Meaning |
|---|---|
| `None` | Filter only, no data binding |
| `ReadOnly` | Read through `GetFragmentView<T>()` (`TConstArrayView`) |
| `ReadWrite` | Read/write through `GetMutableFragmentView<T>()` (`TArrayView`) |

| `EMassFragmentPresence` | Meaning |
|---|---|
| `All` | Every listed element must be present |
| `Any` | At least one `Any`-marked element must be present |
| `None` | The element must be absent |
| `Optional` | Bound when present, skipped when absent |

`AddChunkRequirement<T>(Access, Presence)` binds a `FMassChunkFragment`. `SetChunkFilter(const FMassChunkConditionFunction&)` takes a `TFunction<bool(const FMassExecutionContext&)>` evaluated once per chunk, so the predicate may read chunk fragments, shared fragments and constant data — never per-entity fragments.

## Iteration and FMassExecutionContext

```cpp
void UMyMovementProcessor::Execute(FMassEntityManager& EntityManager, FMassExecutionContext& Context)
{
    MovementQuery.ForEachEntityChunk(Context, [](FMassExecutionContext& Context)
    {
        const int32 NumEntities = Context.GetNumEntities();
        const TArrayView<FTransformFragment> Transforms = Context.GetMutableFragmentView<FTransformFragment>();
        const TConstArrayView<FMassVelocityFragment> Velocities = Context.GetFragmentView<FMassVelocityFragment>();
        const float DeltaTime = Context.GetDeltaTimeSeconds();

        for (int32 Index = 0; Index < NumEntities; ++Index)
        {
            Transforms[Index].GetMutableTransform().AddToTranslation(Velocities[Index].Value * DeltaTime);
        }
    });
}
```

The lambda takes only `FMassExecutionContext&`; there is no entity-manager parameter. Other accessors: `GetEntities()` (`TConstArrayView<FMassEntityHandle>`), `GetFragmentView<T>()`, `GetMutableFragmentView<T>()`, `GetChunkFragment<T>()`, `GetConstSharedFragment<T>()`, `GetMutableSharedFragment<T>()`, `GetSubsystem<T>()`, `GetMutableSubsystem<T>()` and `Defer()`.

**Parallel:** `ParallelForEachEntityChunk(Context, Lambda, FMassEntityQuery::EParallelExecutionFlags::Default)`; the nested enum's flags are `Default`, `Force` and `AutoBalance` (`MassEntityQuery.h:67`).

**Time-slicing:** `ForEachEntityChunk(Context, Limiter, Lambda)` takes a `UE::Mass::FExecutionLimiter&` built from an entity count, stops once that many entities have been visited (always finishing the current chunk) and resumes from the stored position on the next call. Keep the limiter alive across frames as a processor member — a fresh one restarts from the beginning, and the limiter does not notice entities created or destroyed between calls.

## Deferred Commands

Inside `ForEachEntityChunk` never mutate composition through the entity manager: the entity moves archetype and the views you are holding dangle. Queue the change on `Context.Defer()`.

```cpp
MovementQuery.ForEachEntityChunk(Context, [](FMassExecutionContext& Context)
{
    const TConstArrayView<FMassEntityHandle> Entities = Context.GetEntities();
    const TConstArrayView<FMyHealthFragment> Healths = Context.GetFragmentView<FMyHealthFragment>();

    for (int32 Index = 0; Index < Context.GetNumEntities(); ++Index)
    {
        if (Healths[Index].Current <= 0.f)
        {
            Context.Defer().SwapTags<FMyAliveTag, FMyDeadTag>(Entities[Index]);
            Context.Defer().RemoveFragment<FMyHealthFragment>(Entities[Index]);
        }
    }
});
```

Convenience calls on `FMassCommandBuffer`: `AddFragment<T>`, `RemoveFragment<T>`, `AddTag<T>`, `RemoveTag<T>`, `SwapTags<TOld, TNew>`, `AddElements<T...>`, `RemoveElements<T...>`, `DestroyEntity`, `DestroyEntities`. The add/remove helpers forward to `FMassCommandAddElements<T...>` / `FMassCommandRemoveElements<T...>`, which handle any mix of fragments and tags in a single entity move (`SwapTags` uses `FMassCommandSwapTagsInternal`, `MassCommandBuffer.h:264`).

To add a fragment *and* its value, push the command type directly:

```cpp
Context.Defer().PushCommand<FMassCommandAddFragmentInstances<FMyHealthFragment>>(
    Entities[Index], FMyHealthFragment{});
```

For arbitrary deferred work push a lambda through `FMassDeferredCommand<OpType>`: `Context.Defer().PushCommand<FMassDeferredSetCommand>([Entity](FMassEntityManager& Manager) { ... });` (aliases `FMassDeferredCreate/Add/Remove/ChangeComposition/Set/DestroyCommand`, `MassCommands.h:1686-1744`). For a reusable typed command, subclass `FMassBatchedCommand`, override `Run(FMassEntityManager&)` (the const `Execute` is `UE_DEPRECATED(5.7)`, `MassCommands.h:202`) and call `PushUniqueCommand(TUniquePtr<FMassBatchedCommand>&&)`.

Commands flush by `EMassCommandOperationType` bucket, not enum order: `Create` → `Add` → `ChangeComposition` → `Set` → `Remove` and `Destroy` (one shared bucket, push order) → the unclassified `None` last (`MassCommandBuffer.cpp:110-119`). That ordering guarantees entities exist before they are modified and fragments exist before values are written.

## UMassObserverProcessor

Observers run when an observed fragment or tag is added or removed. Fill `ObservedTypes` (an array — one observer can watch several types) and `ObservedOperations`.

```cpp
// MyHealthObserver.cpp
UMyHealthObserver::UMyHealthObserver()
    : ObserverQuery(*this)
{
    ObservedTypes.Add(FMyHealthFragment::StaticStruct());
    ObservedOperations = EMassObservedOperationFlags::Add;
}
```

`EMassObservedOperationFlags`: `None`, `AddElement`, `RemoveElement`, `CreateEntity`, `DestroyEntity`, plus the composites `Add` (= `AddElement | CreateEntity`), `Remove` (= `RemoveElement | DestroyEntity`) and `All`. Use `Add`/`Remove` unless you deliberately want to ignore entity creation — `AddElement` alone does not fire for entities that are born with the fragment.

Override `Register()` to customise which types get registered, and read them back with `GetObservedTypes()` or `GetSingleObservedTypeChecked()`. Full observer template in [mass-entity-patterns.md](references/mass-entity-patterns.md).

## Single-Entity Access and FEntityBuilder

`FMassEntityView` gives typed access to one entity. It is transient — archetype memory relocates, so build a fresh view every time and never cache one.

```cpp
if (EntityManager.IsEntityValid(Handle))
{
    FMassEntityView View(EntityManager, Handle);
    if (FMyHealthFragment* Health = View.GetFragmentDataPtr<FMyHealthFragment>())
    {
        Health->Current -= Damage;
    }
    const bool bDead = View.HasTag<FMyDeadTag>();
}
```

`UE::Mass::FEntityBuilder` (`MassEntityBuilder.h`) composes an entity fluently and commits once, which beats repeated `AddFragmentToEntity` calls.

```cpp
#include "MassEntityBuilder.h"

UE::Mass::FEntityBuilder Builder(EntityManager);
Builder.Add<FTransformFragment>(SpawnTransform);
Builder.Add<FMyHealthFragment>();
Builder.Add<FMyAliveTag>();
FMyHealthFragment& Health = Builder.GetOrCreate<FMyHealthFragment>();
Health.Current = 50.f;
const FMassEntityHandle NewEntity = Builder.Commit();
```

`SetForceDeferredCommit(true)` routes the commit through the command buffer; `CommitAndReprepare()` reuses the builder for the next entity.

## Relations and Archetype Groups

Relations model typed links between entities. A relation type derives from `FMassRelation` (itself an `FMassTag`); the engine ships `FMassChildOfRelation` with `FMassChildOfFragment` in `Relations/MassChildOf.h`. Instances are created and queried through `UE::Mass::FRelationManager`, reached with `EntityManager.GetRelationManager()`.

```cpp
#include "MassRelationManager.h"
#include "Relations/MassChildOf.h"

UE::Mass::FRelationManager& Relations = EntityManager.GetRelationManager();
Relations.CreateRelationInstance<FMassChildOfRelation>(ChildEntity, ParentEntity);
TArray<FMassEntityHandle> Children = Relations.GetRelationSubjects(
    FMassChildOfRelation::StaticStruct(), ParentEntity);
```

Deferred equivalent: `Context.Defer().PushCommand<FMassCommandMakeRelation<FMassChildOfRelation>>(ChildEntity, ParentEntity)`. Builders take `AddRelation<T>(OtherEntity)`.

Archetype groups let a query process archetypes in a controlled order. `EntityManager.FindOrAddArchetypeGroupType(GroupName)` returns a `UE::Mass::FArchetypeGroupType`; `EntityManager.BatchGroupEntities(UE::Mass::FArchetypeGroupHandle(GroupType, UE::Mass::FArchetypeGroupID(N)), Entities)` assigns entities (`Mass/ArchetypeGroup.h:100`); `Query.GroupBy(GroupType)` (optionally with a sort predicate) makes iteration walk groups together.

## Traits, Config Assets and Spawning

`UMassEntityConfigAsset` (a `UDataAsset`) holds a list of traits that build an entity template. Traits subclass `UMassEntityTraitBase` and implement `BuildTemplate`; both virtuals are const and `ValidateTemplate` takes three parameters.

```cpp
// MyHealthTrait.h
#pragma once
#include "MassEntityTraitBase.h"
#include "MyHealthTrait.generated.h"

UCLASS(meta = (DisplayName = "My Health"))
class MYGAME_API UMyHealthTrait : public UMassEntityTraitBase
{
    GENERATED_BODY()
public:
    UPROPERTY(EditAnywhere, Category = "Health")
    float DefaultHealth = 100.f;

protected:
    virtual void BuildTemplate(FMassEntityTemplateBuildContext& BuildContext,
                               const UWorld& World) const override;
    virtual bool ValidateTemplate(const FMassEntityTemplateBuildContext& BuildContext,
                                  const UWorld& World,
                                  FAdditionalTraitRequirements& OutTraitRequirements) const override;
};
```

`FMassEntityTemplateBuildContext` (declared in `MassEntityTemplateRegistry.h`) offers `AddFragment<T>()`, `AddFragment_GetRef<T>()`, `AddTag<T>()`, `AddChunkFragment<T>()`, `AddSharedFragment(const FSharedStruct&)`, `AddConstSharedFragment(const FConstSharedStruct&)` and `AddTranslator<T>()`.

`UMassAssortedFragmentsTrait` (`MassAssortedFragmentsTrait.h`) adds an editor-authored list of fragments and tags without any C++. `AMassSpawner` references config assets through `EntityTypes` and drives `Count`, `DoSpawning()` and `DoDespawning()`.

## Representation and LOD

`UMassRepresentationSubsystem` pools Instanced Static Mesh components and spawned actors so thousands of entities render without one actor each.

| `EMassRepresentationType` | Use |
|---|---|
| `HighResSpawnedActor` | Full actor, closest entities |
| `LowResSpawnedActor` | Cheap actor at medium range |
| `SkinnedMeshInstance` | Instanced skinned mesh |
| `StaticMeshInstance` | ISM, the bulk of the crowd |
| `None` | Not rendered |

`EMassLOD::Type` is `High`, `Medium`, `Low`, `Off`, `Max`, stored as `TEnumAsByte<EMassLOD::Type>`; visibility is `EMassVisibility` (`CanBeSeen`, `CulledByFrustum`, `CulledByDistance`, `Max`). `UMassMovableVisualizationTrait` and `UMassStationaryVisualizationTrait` configure meshes, actor classes and LOD distances.

Thread-safety for anything a query touches is declared through traits in `Mass/ExternalSubsystemTraits.h`. `TMassExternalSubsystemTraits` defaults `GameThreadOnly = true`; `TMassSharedFragmentTraits` defaults `GameThreadOnly = false`. `TMassFragmentTraits` has no threading flag — only `AuthorAcceptsItsNotTriviallyCopyable = false` (`Mass/ExternalSubsystemTraits.h:57-66`). Specialise only when the default is wrong:

```cpp
template<>
struct TMassExternalSubsystemTraits<UMyMassSubsystem> final
{
    enum
    {
        GameThreadOnly = false,
        ThreadSafeWrite = false
    };
};
```

## Crowd, Navigation and Behavior

`UMassCrowdSubsystem` (MassCrowd, Experimental in 5.8) tracks ZoneGraph lane occupancy, density and waiting slots. Entities carry `FMassCrowdTag` and `FMassCrowdLaneTrackingFragment`; `UMassCrowdMemberTrait` wires them up. The subsystem specialises `TMassExternalSubsystemTraits<UMassCrowdSubsystem>` with `GameThreadOnly = false`, so parallel processors may declare it via `AddSubsystemRequirement`.

Movement targets live in `FMassMoveTargetFragment` (`MassNavigationFragments.h`, module `MassNavigation`). ZoneGraph itself and Smart Objects are covered by `ue-ai-navigation`.

Entity behavior runs on State Trees: `UMassStateTreeTrait` attaches the asset, `UMassStateTreeProcessor` derives from `UMassSignalProcessorBase` and ticks only signalled entities, and `FMassStateTreeSharedFragment` is a const shared fragment. Signal an entity with `UMassSignalSubsystem::SignalEntity(FName SignalName, FMassEntityHandle Entity)` — signal name first. Task and evaluator authoring belongs to `ue-state-trees`.

## Deprecated — do not use

| Do not emit | Use in 5.8 | Source |
|---|---|---|
| `ObservedType = T::StaticStruct();` | `ObservedTypes.Add(T::StaticStruct());` | `UE_DEPRECATED(5.8)` in `MassObserverProcessor.h` |
| `Operation = EMassObservedOperation::AddElement;` | `ObservedOperations = EMassObservedOperationFlags::Add;` | `UE_DEPRECATED(5.7)` in `MassObserverProcessor.h` |
| `GetObservedTypeChecked()` | `GetSingleObservedTypeChecked()` | `UE_DEPRECATED(5.8)` in `MassObserverProcessor.h` |
| `virtual void ConfigureQueries()` | `ConfigureQueries(const TSharedRef<FMassEntityManager>&)` | `UE_DEPRECATED(5.6)` in `MassProcessor.h` |
| `virtual void Initialize(UObject&)` | `InitializeInternal(UObject&, const TSharedRef<FMassEntityManager>&)` | `UE_DEPRECATED(5.6)` in `MassProcessor.h` |
| `ForEachEntityChunk(EntityManager, Context, Fn)` | `ForEachEntityChunk(Context, Fn)` | `UE_DEPRECATED(5.6)` in `MassEntityQuery.h` |
| `ParallelForEachEntityChunk(EntityManager, …)`, `EParallelForMode` | `ParallelForEachEntityChunk(Context, Fn, EParallelExecutionFlags)` | `UE_DEPRECATED(5.6)` in `MassEntityQuery.h` |
| `FMassEntityQuery Query{ FMyFragment::StaticStruct() };` | `FMassEntityQuery Query{*this};` + requirements in `ConfigureQueries` | `UE_DEPRECATED(5.6)` in `MassEntityQuery.h` |
| `Defer().AddTag_RuntimeCheck<T>()` and the other `*_RuntimeCheck` calls | `Defer().AddTag<T>()`, `AddFragment<T>()`, `RemoveFragment<T>()`, `RemoveTag<T>()`, `SwapTags<A,B>()` | `UE_DEPRECATED(5.8)` in `MassCommandBuffer.h` |
| `FMassCommandAddTagsInternal`, `FMassCommandAddFragmentsInternal` | `FMassCommandAddElements<T...>` | `UE_DEPRECATED(5.8)` in `MassCommands.h` |
| `FMassCommandRemoveTagsInternal`, `FMassCommandRemoveFragmentsInternal` | `FMassCommandRemoveElements<T...>` | `UE_DEPRECATED(5.8)` in `MassCommands.h` |
| `#include "MassExternalSubsystemTraits.h"`, `"MassEntityElementTypes.h"`, `"MassEntityFragments.h"`, `"MassEntityHandle.h"` | `Mass/ExternalSubsystemTraits.h`, `Mass/EntityElementTypes.h`, `Mass/EntityFragments.h`, `Mass/EntityHandle.h` | `UE_DEPRECATED_HEADER(5.8)` in each shim |
| `FMassEntityManager::Initialize()` | `Initialize(FMassEntityManagerStorageInitParams)` with `FMassEntityManager_InitParams_Concurrent` | `UE_DEPRECATED(5.8)` in `MassEntityManager.h` |
| `UMassRepresentationSubsystem::GetOrSpawnActorFromTemplate` | `GetOrRequestSpawnActorFromTemplate` | `UE_DEPRECATED(5.8)` in `MassRepresentationSubsystem.h:246` |
| `FMassLookAtTargetTag` | `FMassLookAtTargetFragment` | `UE_DEPRECATED(5.6)` in `MassLookAtFragments.h` |

## Common Mistakes

**Treating Mass as a plugin:** `MassEntity` and `MassCore` are runtime modules with no `.uplugin`. Adding `"MassEntity"` to `Build.cs` is enough; only MassGameplay, MassAI and MassCrowd need enabling in the `.uproject`.

**Wrong header for `FTransformFragment`:** it lives in `Mass/EntityFragments.h` (module `MassCore`). `MassCommonFragments.h` merely includes it; `MassCommon` and `MassEntity` list `MassCore` as a public dependency so it resolves transitively, but include `Mass/EntityFragments.h` and list `MassCore` explicitly rather than relying on that.

**Structural change during iteration:** calling `EntityManager.AddFragmentToEntity(...)` inside `ForEachEntityChunk` relocates the entity and invalidates every view in flight. Use `Context.Defer().AddFragment<T>(Entity)`.

**Requirements declared in the constructor:** the query is bound to its entity manager just before `ConfigureQueries` runs, so `AddRequirement` in the constructor trips the "Modifying requirements before initialization" check. Declare requirements only in `ConfigureQueries`.

**Forgetting to register a default-constructed query:** `FMassEntityQuery MyQuery;` silently matches nothing until `RegisterQuery(MyQuery)` runs. Prefer `MyQuery(*this)` in the constructor initializer list.

**Caching an `FMassEntityView`:** views point into archetype memory that moves. Rebuild the view from the handle at each use.

**`IsSet()` as an existence test:** a handle to a destroyed entity still reports `IsSet()`/`IsValid()` true. Ask `EntityManager.IsEntityValid(Handle)`.

**Access mismatch:** requesting `ReadOnly` and then calling `GetMutableFragmentView<T>()` asserts. Match the view to the declared `EMassFragmentAccess`.

**Observing only `AddElement`:** entities created with the fragment already present raise `CreateEntity`, not `AddElement`. Use `EMassObservedOperationFlags::Add`.

**Touching `UObject`s from a parallel processor:** set `bRequiresGameThreadExecution = true`, or declare the subsystem through `AddSubsystemRequirement` and give it a `TMassExternalSubsystemTraits` specialisation that states the truth.

## Related Skills

- `ue-state-trees` — State Tree assets, tasks, evaluators and the Mass State Tree schema
- `ue-ai-navigation` — ZoneGraph, Smart Objects, NavMesh and perception
- `ue-async-threading` — task graph, `ParallelFor` and thread-safety fundamentals
- `ue-actor-component-architecture` — actors and components on the other side of `UMassAgentComponent`
- `ue-cpp-foundations` — `USTRUCT`/`UCLASS` reflection, subsystems, module layout
- `ue-animation-system` — animation for entities promoted to actors or skinned instances
- `ue-procedural-generation` — PCG and ISM authoring that feeds Mass spawning
- `ue-mover` — the Mover plugin: movement modes, layered moves and rollback networking
