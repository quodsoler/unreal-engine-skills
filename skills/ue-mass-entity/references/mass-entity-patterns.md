# Mass Entity Patterns

Copy-ready templates for UE 5.8. Every signature here is taken verbatim from the 5.8 headers in `Engine/Source/Runtime/MassEntity/Public`, `Engine/Source/Runtime/Mass/MassCore/Public/Mass` and the MassGameplay / MassAI plugin modules.

Build.cs for the samples below: `"MassEntity"`, `"MassCore"`, `"MassSignals"`, plus `"MassCommon"`, `"MassSpawner"`, `"MassMovement"` and `"MassRepresentation"` from the MassGameplay plugin (Experimental in 5.8).

---

## Element type declarations

```cpp
// MyMassTypes.h
#pragma once
#include "Mass/EntityElementTypes.h"
#include "MyMassTypes.generated.h"

// Per-entity mutable data.
USTRUCT()
struct FMyHealthFragment : public FMassFragment
{
    GENERATED_BODY()

    UPROPERTY(EditAnywhere, Category = "Mass")
    float Current = 100.f;

    UPROPERTY(EditAnywhere, Category = "Mass")
    float Max = 100.f;
};

// Zero-size marker. Tags must not declare data members.
USTRUCT()
struct FMyAliveTag : public FMassTag
{
    GENERATED_BODY()
};

USTRUCT()
struct FMyDeadTag : public FMassTag
{
    GENERATED_BODY()
};

// Sparse tag: added and removed without moving the entity between archetypes.
USTRUCT()
struct FMyStunnedTag : public FMassSparseTag
{
    GENERATED_BODY()
};

// Mutable value shared by every entity referencing the same instance.
USTRUCT()
struct FMySquadSharedFragment : public FMassSharedFragment
{
    GENERATED_BODY()

    UPROPERTY(EditAnywhere, Category = "Mass")
    int32 SquadID = 0;

    UPROPERTY(EditAnywhere, Category = "Mass")
    FLinearColor SquadColor = FLinearColor::White;
};

// Immutable configuration shared per archetype.
USTRUCT()
struct FMyMeshConfigFragment : public FMassConstSharedFragment
{
    GENERATED_BODY()

    UPROPERTY(EditAnywhere, Category = "Mass")
    TSoftObjectPtr<UStaticMesh> Mesh;

    UPROPERTY(EditAnywhere, Category = "Mass")
    float LODDistance = 5000.f;
};

// One instance per memory chunk, not per entity.
USTRUCT()
struct FMyChunkBudgetFragment : public FMassChunkFragment
{
    GENERATED_BODY()

    bool bShouldRun = true;
};
```

---

## Processor

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
#include "MassCommonTypes.h"
#include "MassExecutionContext.h"
#include "MassMovementFragments.h"
#include "Mass/EntityFragments.h"
#include "MyMassTypes.h"

UMyMovementProcessor::UMyMovementProcessor()
    : MovementQuery(*this)   // registers the query with this processor
{
    ProcessingPhase = EMassProcessingPhase::PrePhysics;
    ExecutionFlags = static_cast<int32>(EProcessorExecutionFlags::AllNetModes);
    ExecutionOrder.ExecuteInGroup = UE::Mass::ProcessorGroupNames::Movement;
    ExecutionOrder.ExecuteAfter.Add(TEXT("MassApplyMovementProcessor")); // class FName, no U prefix
    bRequiresGameThreadExecution = false;
}

void UMyMovementProcessor::ConfigureQueries(const TSharedRef<FMassEntityManager>& EntityManager)
{
    MovementQuery.AddRequirement<FTransformFragment>(
        EMassFragmentAccess::ReadWrite, EMassFragmentPresence::All);
    MovementQuery.AddRequirement<FMassVelocityFragment>(
        EMassFragmentAccess::ReadOnly, EMassFragmentPresence::All);
    MovementQuery.AddTagRequirement<FMyDeadTag>(EMassFragmentPresence::None);
    MovementQuery.AddConstSharedRequirement<FMassMovementParameters>(
        EMassFragmentPresence::All);
    MovementQuery.AddChunkRequirement<FMyChunkBudgetFragment>(
        EMassFragmentAccess::ReadOnly, EMassFragmentPresence::All); // filter uses the checked getter
    MovementQuery.SetChunkFilter([](const FMassExecutionContext& ChunkContext)
    {
        return ChunkContext.GetChunkFragment<FMyChunkBudgetFragment>().bShouldRun;
    });
}

void UMyMovementProcessor::Execute(FMassEntityManager& EntityManager, FMassExecutionContext& Context)
{
    MovementQuery.ForEachEntityChunk(Context, [](FMassExecutionContext& Context)
    {
        const int32 NumEntities = Context.GetNumEntities();
        const TArrayView<FTransformFragment> Transforms =
            Context.GetMutableFragmentView<FTransformFragment>();
        const TConstArrayView<FMassVelocityFragment> Velocities =
            Context.GetFragmentView<FMassVelocityFragment>();
        const FMassMovementParameters& MovementParams =
            Context.GetConstSharedFragment<FMassMovementParameters>();
        const float DeltaTime = Context.GetDeltaTimeSeconds();

        for (int32 Index = 0; Index < NumEntities; ++Index)
        {
            const FVector Velocity = Velocities[Index].Value.GetClampedToMaxSize(MovementParams.MaxSpeed);
            Transforms[Index].GetMutableTransform().AddToTranslation(Velocity * DeltaTime);
        }
    });
}
```

**Parallel variant** — safe only when the lambda touches no game-thread-only data:

```cpp
MovementQuery.ParallelForEachEntityChunk(Context,
    [](FMassExecutionContext& Context)
    {
        // same body
    },
    FMassEntityQuery::EParallelExecutionFlags::AutoBalance);
```

**Time-sliced variant** — `Limiter` must be a member so it survives between frames:

```cpp
// In the class body: UE::Mass::FExecutionLimiter Limiter{2000};
MovementQuery.ForEachEntityChunk(Context, Limiter, [](FMassExecutionContext& Context)
{
    // visits at most 2000 entities per Execute, finishing the current chunk
});
```

---

## Observer processor

```cpp
// MyHealthObserver.h
#pragma once
#include "MassObserverProcessor.h"
#include "MassEntityQuery.h"
#include "MyHealthObserver.generated.h"

UCLASS()
class MYGAME_API UMyHealthObserver : public UMassObserverProcessor
{
    GENERATED_BODY()
public:
    UMyHealthObserver();

protected:
    virtual void ConfigureQueries(const TSharedRef<FMassEntityManager>& EntityManager) override;
    virtual void Execute(FMassEntityManager& EntityManager, FMassExecutionContext& Context) override;

private:
    FMassEntityQuery ObserverQuery;
};
```

```cpp
// MyHealthObserver.cpp
#include "MyHealthObserver.h"
#include "MassExecutionContext.h"
#include "MyMassTypes.h"

UMyHealthObserver::UMyHealthObserver()
    : ObserverQuery(*this)
{
    // ObservedTypes is an array: one observer can watch several fragment or tag types.
    ObservedTypes.Add(FMyHealthFragment::StaticStruct());
    // Add == AddElement | CreateEntity, so entities born with the fragment fire too.
    ObservedOperations = EMassObservedOperationFlags::Add;
}

void UMyHealthObserver::ConfigureQueries(const TSharedRef<FMassEntityManager>& EntityManager)
{
    ObserverQuery.AddRequirement<FMyHealthFragment>(
        EMassFragmentAccess::ReadWrite, EMassFragmentPresence::All);
}

void UMyHealthObserver::Execute(FMassEntityManager& EntityManager, FMassExecutionContext& Context)
{
    ObserverQuery.ForEachEntityChunk(Context, [](FMassExecutionContext& Context)
    {
        const TArrayView<FMyHealthFragment> Healths =
            Context.GetMutableFragmentView<FMyHealthFragment>();
        for (int32 Index = 0; Index < Context.GetNumEntities(); ++Index)
        {
            Healths[Index].Current = Healths[Index].Max;
        }
    });
}
```

Override `virtual void Register()` to control registration when the default "register for every entry in `ObservedTypes`" behaviour is not what you want. `GetObservedTypes()` returns `TConstArrayView<const UScriptStruct*>`; `GetSingleObservedTypeChecked()` returns `TNotNull<const UScriptStruct*>` for single-type observers.

---

## Deferred commands

```cpp
MyQuery.ForEachEntityChunk(Context, [](FMassExecutionContext& Context)
{
    const TConstArrayView<FMassEntityHandle> Entities = Context.GetEntities();
    const TConstArrayView<FMyHealthFragment> Healths = Context.GetFragmentView<FMyHealthFragment>();

    for (int32 Index = 0; Index < Context.GetNumEntities(); ++Index)
    {
        if (Healths[Index].Current <= 0.f)
        {
            // Composition-only changes; one entity move each.
            Context.Defer().SwapTags<FMyAliveTag, FMyDeadTag>(Entities[Index]);
            Context.Defer().RemoveFragment<FMyHealthFragment>(Entities[Index]);

            // Sparse tag: no archetype change at all.
            Context.Defer().AddTag<FMyStunnedTag>(Entities[Index]);
        }
    }
});
```

Several element types in one entity move, for a single entity or a whole chunk:

```cpp
Context.Defer().AddElements<FMyHealthFragment, FMyAliveTag>(Entity);
Context.Defer().RemoveElements<FMyHealthFragment, FMyAliveTag>(Context.GetEntities());
```

Adding a fragment together with its initial value:

```cpp
Context.Defer().PushCommand<FMassCommandAddFragmentInstances<FMyHealthFragment>>(
    Entity, FMyHealthFragment{});
```

Destroying entities:

```cpp
Context.Defer().DestroyEntity(Entity);
Context.Defer().DestroyEntities(Context.GetEntities());
```

Arbitrary deferred work goes through a lambda command (`MassCommands.h:1673`); pick the alias whose bucket matches what the lambda does:

```cpp
Context.Defer().PushCommand<FMassDeferredSetCommand>([Entity](FMassEntityManager& Manager)
{
    FMassEntityView View(Manager, Entity);
    View.GetFragmentData<FMyHealthFragment>().Current = 0.f;
});
```

For a reusable typed command, derive from `FMassBatchedCommand`, override `Run(FMassEntityManager&)` (const `Execute` is `UE_DEPRECATED(5.7)`) and push it with `PushUniqueCommand(TUniquePtr<FMassBatchedCommand>&&)`.

Flush order by `EMassCommandOperationType` bucket (`MassCommandBuffer.cpp:110-119`): `Create`, `Add`, `ChangeComposition`, `Set`, then `Remove` and `Destroy` together (push order), and finally the unclassified `None` bucket.

---

## Single-entity access

```cpp
void ApplyDamage(FMassEntityManager& EntityManager, FMassEntityHandle Target, float Damage)
{
    if (!EntityManager.IsEntityValid(Target))
    {
        return;
    }

    FMassEntityView View(EntityManager, Target);

    FMyHealthFragment* Health = View.GetFragmentDataPtr<FMyHealthFragment>();
    if (Health == nullptr)
    {
        return;
    }

    Health->Current = FMath::Max(0.f, Health->Current - Damage);

    if (!View.HasTag<FMyDeadTag>() && Health->Current <= 0.f)
    {
        // Outside processor iteration, direct mutation is safe.
        EntityManager.SwapTagsForEntity(Target,
            FMyAliveTag::StaticStruct(), FMyDeadTag::StaticStruct());
    }
}
```

`FMassEntityView` also offers `GetFragmentData<T>()` (checked), `GetSharedFragmentData<T>()`, `GetConstSharedFragmentData<T>()` and their `*Ptr` variants. Never store a view across frames.

---

## FEntityBuilder

```cpp
#include "MassEntityBuilder.h"

FMassEntityHandle SpawnMyAgent(FMassEntityManager& EntityManager, const FTransform& SpawnTransform)
{
    UE::Mass::FEntityBuilder Builder(EntityManager);
    Builder.Add<FTransformFragment>(SpawnTransform);
    Builder.Add<FMyAliveTag>();

    FMyHealthFragment& Health = Builder.GetOrCreate<FMyHealthFragment>();
    Health.Current = 75.f;
    Health.Max = 100.f;

    return Builder.Commit();
}
```

- `Add<T>()` takes no arguments when `T` is a tag or a chunk fragment; for a fragment, `Add<T>(Args...)` forwards its constructor arguments.
- `Add(const FInstancedStruct&)` and `Add(TNotNull<const UScriptStruct*>)` handle runtime-typed elements.
- `SetForceDeferredCommit(true)` makes `Commit()` issue Mass commands even outside processing.
- `CommitAndReprepare()` commits and resets the builder so the same recipe can produce the next entity.

---

## Relations

```cpp
#include "MassRelationManager.h"
#include "Relations/MassChildOf.h"

void AttachChild(FMassEntityManager& EntityManager,
                 FMassEntityHandle ChildEntity, FMassEntityHandle ParentEntity)
{
    UE::Mass::FRelationManager& Relations = EntityManager.GetRelationManager();
    Relations.CreateRelationInstance<FMassChildOfRelation>(ChildEntity, ParentEntity);
}

TArray<FMassEntityHandle> GetChildren(FMassEntityManager& EntityManager, FMassEntityHandle ParentEntity)
{
    return EntityManager.GetRelationManager().GetRelationSubjects(
        FMassChildOfRelation::StaticStruct(), ParentEntity);
}
```

- A relation type derives from `FMassRelation` (which derives from `FMassTag`); per-instance payload goes in an `FMassRelationFragment`.
- `FMassChildOfRelation` plus `FMassChildOfFragment` (with its `Parent` handle) ship in `Relations/MassChildOf.h`.
- Deferred creation: `Context.Defer().PushCommand<FMassCommandMakeRelation<FMassChildOfRelation>>(ChildEntity, ParentEntity)`.
- `FRelationManager` also exposes `CreateRelationInstances`, `DestroyRelationInstance`, `GetRelationObjects` and `GetRelationTypeHandle`.
- From a builder: `Builder.AddRelation<FMassChildOfRelation>(ParentEntity)`.

---

## Archetype groups

```cpp
const UE::Mass::FArchetypeGroupType SquadGroupType =
    EntityManager.FindOrAddArchetypeGroupType(TEXT("Squad"));

// Assign entities to a specific group (ID) of that type.
const UE::Mass::FArchetypeGroupHandle SquadGroupHandle(SquadGroupType, UE::Mass::FArchetypeGroupID(SquadIndex));
EntityManager.BatchGroupEntities(SquadGroupHandle, SquadEntities);

// In ConfigureQueries: walk archetypes grouped (and optionally sorted) by that type.
MyQuery.GroupBy(SquadGroupType);
```

`GroupBy` has an overload taking a `TFunction<bool(const UE::Mass::FArchetypeGroupID, const UE::Mass::FArchetypeGroupID)>` sort predicate. `ResetGrouping()` clears it. `EntityManager.GetGroupForEntity(Entity, GroupType)` returns the entity's `UE::Mass::FArchetypeGroupHandle`.

---

## Custom trait

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

    UPROPERTY(EditAnywhere, Category = "Health")
    float DefaultMaxHealth = 100.f;

protected:
    virtual void BuildTemplate(FMassEntityTemplateBuildContext& BuildContext,
                               const UWorld& World) const override;
    virtual bool ValidateTemplate(const FMassEntityTemplateBuildContext& BuildContext,
                                  const UWorld& World,
                                  FAdditionalTraitRequirements& OutTraitRequirements) const override;
};
```

```cpp
// MyHealthTrait.cpp
#include "MyHealthTrait.h"
#include "MassEntityTemplateRegistry.h"
#include "Mass/EntityFragments.h"
#include "MyMassTypes.h"

void UMyHealthTrait::BuildTemplate(FMassEntityTemplateBuildContext& BuildContext,
                                   const UWorld& World) const
{
    FMyHealthFragment& HealthDefaults = BuildContext.AddFragment_GetRef<FMyHealthFragment>();
    HealthDefaults.Current = DefaultHealth;
    HealthDefaults.Max = DefaultMaxHealth;

    BuildContext.AddTag<FMyAliveTag>();
}

bool UMyHealthTrait::ValidateTemplate(const FMassEntityTemplateBuildContext& BuildContext,
                                      const UWorld& World,
                                      FAdditionalTraitRequirements& OutTraitRequirements) const
{
    // A transform is required by the movement processors this trait pairs with.
    OutTraitRequirements.Add(FTransformFragment::StaticStruct());
    return Super::ValidateTemplate(BuildContext, World, OutTraitRequirements);
}
```

`FMassEntityTemplateBuildContext` lives in `MassEntityTemplateRegistry.h` and offers `AddFragment<T>()`, `AddFragment_GetRef<T>()`, `AddTag<T>()`, `AddChunkFragment<T>()`, `AddSharedFragment(const FSharedStruct&)`, `AddConstSharedFragment(const FConstSharedStruct&)` and `AddTranslator<T>()`.

Assign the trait to a `UMassEntityConfigAsset` in the editor; `AMassSpawner` references config assets through its `EntityTypes` array and spawns `Count` entities via `DoSpawning()` / `DoDespawning()`. For editor-authored fragment lists with no C++, use `UMassAssortedFragmentsTrait` from `MassAssortedFragmentsTrait.h`.

---

## Subsystem thread-safety traits

Declare the truth about a subsystem you expose to queries, in the subsystem's own header, below the class:

```cpp
// MyMassSubsystem.h, after the UCLASS body. #include "Mass/ExternalSubsystemTraits.h" goes with the
// other includes, above "MyMassSubsystem.generated.h": UHT rejects any #include after the .generated.h line.

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

`TMassExternalSubsystemTraits` defaults `GameThreadOnly = true`, so any subsystem safe off the game thread must opt out explicitly. `TMassSharedFragmentTraits` defaults `GameThreadOnly = false`; specialise it only to force a shared fragment back onto the game thread. `TMassFragmentTraits` carries no threading flag, only `AuthorAcceptsItsNotTriviallyCopyable` (`Mass/ExternalSubsystemTraits.h:57-66`).

Queries bind subsystems with `AddSubsystemRequirement<T>(EMassFragmentAccess)`, and the processor reads them with `Context.GetSubsystem<T>()` / `Context.GetMutableSubsystem<T>()` (plus `*Checked` variants).
