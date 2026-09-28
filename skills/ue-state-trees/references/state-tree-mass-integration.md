# State Tree Mass Entity Integration

Target engine: **UE 5.8**. Running State Trees over thousands of Mass entities. For Mass architecture itself — entity manager, archetypes, processors, traits — see `ue-mass-entity`.

The behaviour code lives in `MassAIBehavior` (plugin `MassAI`, Experimental in 5.8). `MassEntity` and `MassCore` are core engine modules under `Source/Runtime`, and Mass Signals moved to `Source/Runtime/Mass/MassSignals`.

```csharp
// MyGame.Build.cs
PublicDependencyModuleNames.AddRange(new string[]
{
    "Core",
    "CoreUObject",
    "Engine",
    "StateTreeModule",
    "GameplayStateTreeModule",
    "MassEntity",
    "MassCore",
    "MassSignals",
    "MassAIBehavior",
});
```

Add `MassCommon`, `MassMovement`, `MassNavigation` or `MassSmartObjects` only when you actually touch their fragments.

---

## UMassStateTreeSchema

`UMassStateTreeSchema : UStateTreeSchema` restricts an asset to Mass-compatible nodes through `IsStructAllowed` and collects every node's Mass requirements during `Link`, exposed as `GetDependencies()` returning `const TArray<FMassStateTreeDependency>&`.

Allowed node bases (`MassStateTreeTypes.h`):

| Base struct | Replaces |
|---|---|
| `FMassStateTreeTaskBase` | `FStateTreeTaskBase` |
| `FMassStateTreeConditionBase` | `FStateTreeConditionBase` |
| `FMassStateTreeEvaluatorBase` | `FStateTreeEvaluatorBase` |
| `FMassStateTreePropertyFunctionBase` | `FStateTreePropertyFunctionBase` |

Each adds one virtual:

```cpp
virtual void GetDependencies(UE::MassBehavior::FStateTreeDependencyBuilder& Builder) const;
```

`UE::MassStateTree::ExecutionFlags` is `EProcessorExecutionFlags::Standalone | EProcessorExecutionFlags::Server`, which is why Mass behaviours do not run on clients by default.

---

## Mass task template

`GetInstanceDataType()` is required here exactly as it is for ordinary State Tree nodes.

```cpp
// MyMassSeekTask.h
#pragma once

#include "Mass/EntityFragments.h"
#include "MassStateTreeDependency.h"
#include "MassStateTreeExecutionContext.h"
#include "MassStateTreeTypes.h"
#include "StateTreeExecutionContext.h"
#include "MyMassSeekTask.generated.h"

USTRUCT()
struct FMyMassSeekTaskInstanceData
{
    GENERATED_BODY()

    UPROPERTY(EditAnywhere, Category = "Input")
    FVector TargetLocation = FVector::ZeroVector;

    UPROPERTY(EditAnywhere, Category = "Parameter")
    float AcceptanceRadius = 150.f;
};

USTRUCT(meta = (DisplayName = "My Mass Seek"))
struct FMyMassSeekTask : public FMassStateTreeTaskBase
{
    GENERATED_BODY()

    using FInstanceDataType = FMyMassSeekTaskInstanceData;

    virtual const UStruct* GetInstanceDataType() const override { return FInstanceDataType::StaticStruct(); }

    /** Declared once when the asset is linked; feeds the processor's entity query. */
    virtual void GetDependencies(UE::MassBehavior::FStateTreeDependencyBuilder& Builder) const override
    {
        Builder.AddReadOnly<FTransformFragment>();
    }

    virtual EStateTreeRunStatus Tick(FStateTreeExecutionContext& Context,
        const float DeltaTime) const override
    {
        const FInstanceDataType& Data = Context.GetInstanceData(*this);
        const FMassStateTreeExecutionContext& MassContext =
            static_cast<const FMassStateTreeExecutionContext&>(Context);

        const FMassEntityHandle Entity = MassContext.GetEntity();
        const FTransformFragment& TransformFragment =
            MassContext.GetEntityManager().GetFragmentDataChecked<FTransformFragment>(Entity);

        const double Distance = FVector::Dist(
            TransformFragment.GetTransform().GetLocation(), Data.TargetLocation);

        return Distance <= Data.AcceptanceRadius
            ? EStateTreeRunStatus::Succeeded
            : EStateTreeRunStatus::Running;
    }
};
```

`FStateTreeDependencyBuilder` (`MassStateTreeDependency.h`) chains:

```cpp
Builder.AddReadOnly<FTransformFragment>()
       .AddReadWrite<FMyCustomFragment>()
       .AddReadOnly(MySubsystemHandle);   // overload taking a TStateTreeExternalDataHandle
```

`FTransformFragment` is declared in `MassCore` (`Mass/EntityFragments.h`). Movement, LOD and navigation fragments belong to their own plugin modules.

Conditions and evaluators follow the same shape — derive from `FMassStateTreeConditionBase` / `FMassStateTreeEvaluatorBase`, override `GetDependencies` plus `TestCondition` / `TreeStart`-`Tick`-`TreeStop`, and always add the `GetInstanceDataType()` override.

---

## FMassStateTreeExecutionContext

```cpp
// Owner, StateTree, InstanceData, MassExecutionContext
FMassStateTreeExecutionContext MassContext(Owner, *StateTree, InstanceData, MassExecContext);
MassContext.SetEntity(EntityHandle);   // set before Start()/Tick() for each entity
```

| Member | Returns |
|---|---|
| `GetEntity()` | `FMassEntityHandle` |
| `SetEntity(const FMassEntityHandle)` | `void` |
| `GetEntityManager()` | `FMassEntityManager&` |
| `GetMassEntityExecutionContext()` | `FMassExecutionContext&` |

`MassExecContext` is the `FMassExecutionContext&` handed to a processor's `Execute`. The context also installs `FMassExecutionExtension` (a `FStateTreeExecutionExtension`) so instance descriptions and linked-tree overrides work per entity.

The six-parameter form `(Owner, StateTree, InstanceData, EntityManager, SignalSubsystem, MassExecContext)` is deprecated; use the four-parameter constructor.

---

## Entity setup

`UMassStateTreeTrait : UMassEntityTraitBase` assigns the asset. Its `StateTree` property is filtered to assets whose schema is `UMassStateTreeSchema`, and `BuildTemplate` adds the fragments:

| Fragment | Kind | Contents |
|---|---|---|
| `FMassStateTreeInstanceFragment` | `FMassFragment` (per entity) | `FMassStateTreeInstanceHandle InstanceHandle`, `double LastUpdateTimeInSeconds` |
| `FMassStateTreeSharedFragment` | `FMassConstSharedFragment` (per archetype) | `TObjectPtr<UStateTree> StateTree` |

`FMassStateTreeSharedFragment` is a **const** shared fragment — never write to it at runtime.

---

## UMassStateTreeSubsystem

`UMassStateTreeSubsystem : UMassSubsystemBase` pools the per-entity instance data.

```cpp
FMassStateTreeInstanceHandle Handle = MassStateTreeSubsystem->AllocateInstanceData(StateTree);
FStateTreeInstanceData* Data = MassStateTreeSubsystem->GetInstanceData(Handle);  // nullptr if stale
const bool bValid = MassStateTreeSubsystem->IsValidHandle(Handle);
MassStateTreeSubsystem->FreeInstanceData(Handle);
```

The trait lifecycle does this for you; call it directly only from a custom processor. To substitute your own processor class, set `UMassBehaviorSettings::DynamicStateTreeProcessorClass` (Project Settings > Mass Behavior, `MassBehaviorSettings.h:29`); the subsystem copies it into its protected `DynamicProcessorClass` (`MassStateTreeSubsystem.h:131`).

---

## UMassStateTreeProcessor

`UMassStateTreeProcessor : UMassSignalProcessorBase` — signal-driven, not a plain `UMassProcessor`. The subsystem instantiates one per unique set of Mass requirements gathered from the asset's nodes; you are not expected to spawn them yourself.

| Override / method | Role |
|---|---|
| `ConfigureQueries(const TSharedRef<FMassEntityManager>&)` | Builds the entity query from the merged dependencies |
| `SignalEntities(FMassEntityManager&, FMassExecutionContext&, FMassSignalNameLookup&)` | Ticks the trees of the signalled entities |
| `SetExecutionRequirements(const FMassFragmentRequirements&, const FMassSubsystemRequirements&)` | Places the processor correctly in the graph; callable before initialisation only |
| `AddHandledStateTree(TNotNull<const UStateTree*>)` | Registers an asset with this processor instance |
| `ExportRequirements(FMassExecutionRequirements&) const` | Publishes the merged requirements |

Support processors in the same header: `UMassStateTreeActivationProcessor` (a `UMassProcessor` that signals newly spawned entities) and `UMassStateTreeFragmentDestructor` (a `UMassObserverProcessor` that frees pooled instance data).

---

## Signals

`namespace UE::Mass::Signals` in `MassStateTreeTypes.h`:

| Signal | Meaning |
|---|---|
| `StateTreeActivate` | Start or wake the entity's tree |
| `NewStateTreeTaskRequired` | The active task finished; select again |
| `DelayedTransitionWakeup` | A delayed transition's timer elapsed |
| `LookAtFinished`, `StandTaskFinished`, `AnimateTaskFinished`, `ContextualAnimTaskFinished` | Completion signals from the stock MassAI tasks |

`UMassSignalSubsystem` takes the **signal name first**:

```cpp
// MassSignalSubsystem.h
void SignalEntity(FName SignalName, const FMassEntityHandle Entity);
void SignalEntities(FName SignalName, TConstArrayView<FMassEntityHandle> Entities);
void DelaySignalEntity(FName SignalName, const FMassEntityHandle Entity, const float DelayInSeconds);
void SignalEntityDeferred(FMassExecutionContext& Context, FName SignalName, const FMassEntityHandle Entity);
void SignalEntitiesDeferred(FMassExecutionContext& Context, FName SignalName, TConstArrayView<FMassEntityHandle> Entities);
```

```cpp
SignalSubsystem.SignalEntity(UE::Mass::Signals::StateTreeActivate, Entity);
SignalSubsystem.SignalEntitiesDeferred(MassExecContext, UE::Mass::Signals::NewStateTreeTaskRequired, Entities);
```

Use the `*Deferred` variants from inside a processor's `Execute` so the signal is applied through the command buffer.

---

## Performance notes

- **Signals beat ticking.** The processor only runs trees for signalled entities. Finish tasks by signalling rather than polling in `Tick`.
- **Batch.** Prefer `SignalEntities` over a loop of `SignalEntity`, and the deferred variants inside processors.
- **Minimise write dependencies.** Every `AddReadWrite` narrows how much of the Mass graph can run in parallel with the State Tree processor. Declare `AddReadOnly` wherever possible, and declare exactly what the node touches — under-declaring is a data race, over-declaring is lost parallelism.
- **Pool instance data.** Always go through `UMassStateTreeSubsystem`; a per-entity `FStateTreeInstanceData` defeats the pooling.
- **Stay shallow.** `FStateTreeActiveStates::MaxStates = 8` is the hard cap; two or three levels keeps per-entity selection cost low.
- **Profile with Mass tooling.** A task costing 0.01 ms is 100 ms across 10,000 entities.
