# State Tree Patterns

Target engine: **UE 5.8**. Complete, compilable node templates. Every signature is copied from the 5.8 headers in `Engine/Plugins/Runtime/StateTree/Source/StateTreeModule/Public`.

Build.cs: `StateTreeModule` for the node types, `GameplayStateTreeModule` for `UStateTreeComponent`, `UStateTreeAIComponent` and their schemas, `GameplayTags` for `FGameplayTag`.

Two rules every template below follows:

1. `using FInstanceDataType = …;` **and** `virtual const UStruct* GetInstanceDataType() const override { return FInstanceDataType::StaticStruct(); }`. Without the override the framework allocates nothing and `Context.GetInstanceData(*this)` is undefined behaviour.
2. Derive from the `*CommonBase` struct (`FStateTreeTaskCommonBase`, `FStateTreeConditionCommonBase`, `FStateTreeEvaluatorCommonBase`, `FStateTreeConsiderationCommonBase`) so `UStateTreeComponentSchema::IsStructAllowed` accepts the node and it shows up in the editor picker.

---

## Task with instance data and an event

```cpp
// MyTimedEventTask.h
#pragma once

#include "GameplayTagContainer.h"
#include "StateTreeExecutionContext.h"
#include "StateTreeTaskBase.h"
#include "MyTimedEventTask.generated.h"

USTRUCT()
struct FMyTimedEventTaskInstanceData
{
    GENERATED_BODY()

    /** Editable and bindable, because it lives on the instance data. */
    UPROPERTY(EditAnywhere, Category = "Parameter")
    float TriggerDelay = 3.0f;

    UPROPERTY(EditAnywhere, Category = "Parameter")
    FGameplayTag EventToSend;

    UPROPERTY()
    float ElapsedTime = 0.f;

    UPROPERTY()
    bool bHasTriggered = false;
};

USTRUCT(meta = (DisplayName = "My Timed Event Task"))
struct FMyTimedEventTask : public FStateTreeTaskCommonBase
{
    GENERATED_BODY()

    using FInstanceDataType = FMyTimedEventTaskInstanceData;

    virtual const UStruct* GetInstanceDataType() const override { return FInstanceDataType::StaticStruct(); }

    virtual EStateTreeRunStatus EnterState(FStateTreeExecutionContext& Context,
        const FStateTreeTransitionResult& Transition) const override
    {
        FInstanceDataType& Data = Context.GetInstanceData(*this);
        Data.ElapsedTime = 0.f;
        Data.bHasTriggered = false;
        return EStateTreeRunStatus::Running;
    }

    virtual EStateTreeRunStatus Tick(FStateTreeExecutionContext& Context,
        const float DeltaTime) const override
    {
        FInstanceDataType& Data = Context.GetInstanceData(*this);
        Data.ElapsedTime += DeltaTime;

        if (!Data.bHasTriggered && Data.ElapsedTime >= Data.TriggerDelay)
        {
            Data.bHasTriggered = true;
            Context.SendEvent(Data.EventToSend, FConstStructView(), TEXT("MyTimedEventTask"));
            return EStateTreeRunStatus::Succeeded;
        }
        return EStateTreeRunStatus::Running;
    }

    virtual void ExitState(FStateTreeExecutionContext& Context,
        const FStateTreeTransitionResult& Transition) const override
    {
        FInstanceDataType& Data = Context.GetInstanceData(*this);
        Data.bHasTriggered = false;
    }

    virtual FString GetDebugInfo(const FStateTreeReadOnlyExecutionContext& Context) const override
    {
        return TEXT("MyTimedEventTask");
    }
};
```

---

## Task reading the context actor and a world subsystem

The component schemas publish the owning actor as *context data* named `Actor`. Bind it with a `Category = "Context"` property on the instance data — the editor fills it automatically. Subsystems come through external data handles, which the schema validates via `IsExternalItemAllowed` (it accepts `AActor`, `UActorComponent` and `UWorldSubsystem` subclasses).

```cpp
// MyReachTargetTask.h
#pragma once

#include "GameFramework/Actor.h"
#include "StateTreeExecutionContext.h"
#include "StateTreeLinker.h"
#include "StateTreeTaskBase.h"
#include "MyNavigationSubsystem.h" // complete type: the inline EnterState calls into it
#include "MyReachTargetTask.generated.h"

USTRUCT()
struct FMyReachTargetTaskInstanceData
{
    GENERATED_BODY()

    /** Filled by the schema's "Actor" context data. */
    UPROPERTY(EditAnywhere, Category = "Context")
    TObjectPtr<AActor> Actor = nullptr;

    /** Bindable input, typically driven by an evaluator. */
    UPROPERTY(EditAnywhere, Category = "Input")
    FVector TargetLocation = FVector::ZeroVector;

    UPROPERTY(EditAnywhere, Category = "Parameter")
    float AcceptanceRadius = 100.f;
};

USTRUCT(meta = (DisplayName = "My Reach Target"))
struct FMyReachTargetTask : public FStateTreeTaskCommonBase
{
    GENERATED_BODY()

    using FInstanceDataType = FMyReachTargetTaskInstanceData;

    virtual const UStruct* GetInstanceDataType() const override { return FInstanceDataType::StaticStruct(); }

    virtual bool Link(FStateTreeLinker& Linker) override
    {
        Linker.LinkExternalData(NavigationSubsystemHandle);
        return true;
    }

    virtual EStateTreeRunStatus EnterState(FStateTreeExecutionContext& Context,
        const FStateTreeTransitionResult& Transition) const override
    {
        const FInstanceDataType& Data = Context.GetInstanceData(*this);
        if (Data.Actor == nullptr)
        {
            return EStateTreeRunStatus::Failed;
        }

        UMyNavigationSubsystem& Navigation = Context.GetExternalData(NavigationSubsystemHandle);
        return Navigation.RequestPath(Data.Actor, Data.TargetLocation)
            ? EStateTreeRunStatus::Running
            : EStateTreeRunStatus::Failed;
    }

    virtual EStateTreeRunStatus Tick(FStateTreeExecutionContext& Context,
        const float DeltaTime) const override
    {
        const FInstanceDataType& Data = Context.GetInstanceData(*this);
        if (Data.Actor == nullptr)
        {
            return EStateTreeRunStatus::Failed;
        }

        const double Distance = FVector::Dist(Data.Actor->GetActorLocation(), Data.TargetLocation);
        return Distance <= Data.AcceptanceRadius
            ? EStateTreeRunStatus::Succeeded
            : EStateTreeRunStatus::Running;
    }

    TStateTreeExternalDataHandle<UMyNavigationSubsystem> NavigationSubsystemHandle;
};
```

`Context.GetExternalData(Handle)` returns a reference and asserts for `Required` handles. Declare the handle as
`TStateTreeExternalDataHandle<UMyNavigationSubsystem, EStateTreeExternalDataRequirement::Optional>` and call
`Context.GetExternalDataPtr(Handle)` when the data may be absent.

---

## Latent task finished from a callback

`FStateTreeWeakExecutionContext` is copyable and safe to capture. It re-resolves the owner, the asset and the task index when used, so the callback can run on a later frame.

```cpp
// MyAsyncLoadTask.h
#pragma once

#include "Engine/AssetManager.h"
#include "Engine/StreamableManager.h"
#include "StateTreeAsyncExecutionContext.h"
#include "StateTreeExecutionContext.h"
#include "StateTreeTaskBase.h"
#include "MyAsyncLoadTask.generated.h"

USTRUCT()
struct FMyAsyncLoadTaskInstanceData
{
    GENERATED_BODY()

    UPROPERTY(EditAnywhere, Category = "Parameter")
    FSoftObjectPath AssetToLoad;

    TSharedPtr<FStreamableHandle> StreamableHandle;
};

USTRUCT(meta = (DisplayName = "My Async Load"))
struct FMyAsyncLoadTask : public FStateTreeTaskCommonBase
{
    GENERATED_BODY()

    FMyAsyncLoadTask()
    {
        bShouldCallTick = false;
    }

    using FInstanceDataType = FMyAsyncLoadTaskInstanceData;

    virtual const UStruct* GetInstanceDataType() const override { return FInstanceDataType::StaticStruct(); }

    virtual EStateTreeRunStatus EnterState(FStateTreeExecutionContext& Context,
        const FStateTreeTransitionResult& Transition) const override
    {
        FInstanceDataType& Data = Context.GetInstanceData(*this);
        const FStateTreeWeakExecutionContext WeakContext = Context.MakeWeakExecutionContext();

        Data.StreamableHandle = UAssetManager::GetStreamableManager().RequestAsyncLoad(
            Data.AssetToLoad,
            FStreamableDelegate::CreateLambda([WeakContext]()
            {
                WeakContext.FinishTask(EStateTreeFinishTaskType::Succeeded);
            }));

        return Data.StreamableHandle.IsValid()
            ? EStateTreeRunStatus::Running
            : EStateTreeRunStatus::Failed;
    }

    virtual void ExitState(FStateTreeExecutionContext& Context,
        const FStateTreeTransitionResult& Transition) const override
    {
        FInstanceDataType& Data = Context.GetInstanceData(*this);
        if (Data.StreamableHandle.IsValid())
        {
            Data.StreamableHandle->CancelHandle();
            Data.StreamableHandle.Reset();
        }
    }
};
```

When the tree is currently ticking, a task can also finish itself synchronously with
`Context.FinishTask(*this, EStateTreeFinishTaskType::Failed);`.

To read or write data outside the tick, promote the weak context:

```cpp
const FStateTreeStrongExecutionContext StrongContext = WeakContext.MakeStrongExecutionContext();
if (StrongContext.SendEvent(MyTag_Finished))
{
    // the state that created the context is still active
}
```

Every mutating method on the weak and strong contexts returns `false` when the owner, the asset or the
originating state is gone, so there is no separate validity check. `MakeStrongReadOnlyExecutionContext()`
returns the read-only flavour.

---

## Condition template

```cpp
// MyHealthCondition.h
#pragma once

#include "StateTreeConditionBase.h"
#include "StateTreeExecutionContext.h"
#include "MyHealthCondition.generated.h"

USTRUCT()
struct FMyHealthConditionInstanceData
{
    GENERATED_BODY()

    /** Bind this to an evaluator output or a context property. */
    UPROPERTY(EditAnywhere, Category = "Input")
    float CurrentHealth = 100.f;

    UPROPERTY(EditAnywhere, Category = "Parameter")
    float Threshold = 25.f;
};

USTRUCT(meta = (DisplayName = "My Health Below Threshold"))
struct FMyHealthCondition : public FStateTreeConditionCommonBase
{
    GENERATED_BODY()

    using FInstanceDataType = FMyHealthConditionInstanceData;

    virtual const UStruct* GetInstanceDataType() const override { return FInstanceDataType::StaticStruct(); }

    /** Pure function: selection may call this several times in one frame. */
    virtual bool TestCondition(FStateTreeExecutionContext& Context) const override
    {
        const FInstanceDataType& Data = Context.GetInstanceData(*this);
        return Data.CurrentHealth < Data.Threshold;
    }
};
```

`Operand` (`EStateTreeExpressionOperand::And` / `Or`) and `DeltaIndent` on the base struct build boolean expressions without nesting. `EvaluationMode` (`Evaluated`, `ForcedTrue`, `ForcedFalse`) lets designers pin a condition while debugging.

---

## Evaluator template

Evaluators tick before transitions and before task ticks, once per tree rather than per state. `DeltaTime` is `0` when they are ticked during state pre-selection, so never integrate time without checking.

```cpp
// MyNearestThreatEvaluator.h
#pragma once

#include "GameFramework/Actor.h"
#include "StateTreeEvaluatorBase.h"
#include "StateTreeExecutionContext.h"
#include "StateTreeLinker.h"
#include "MyPerceptionSubsystem.h" // complete type: the inline Tick calls into it
#include "MyNearestThreatEvaluator.generated.h"

USTRUCT()
struct FMyNearestThreatEvaluatorInstanceData
{
    GENERATED_BODY()

    /** Filled by the schema's "Actor" context data. */
    UPROPERTY(EditAnywhere, Category = "Context")
    TObjectPtr<AActor> Actor = nullptr;

    UPROPERTY(EditAnywhere, Category = "Parameter")
    float QueryInterval = 0.5f;

    /** Outputs — bind task and condition inputs to these. */
    UPROPERTY(EditAnywhere, Category = "Output")
    TObjectPtr<AActor> NearestThreat = nullptr;

    UPROPERTY(EditAnywhere, Category = "Output")
    float DistanceToNearest = 0.f;

    UPROPERTY()
    float TimeSinceLastQuery = 0.f;
};

USTRUCT(meta = (DisplayName = "My Nearest Threat"))
struct FMyNearestThreatEvaluator : public FStateTreeEvaluatorCommonBase
{
    GENERATED_BODY()

    using FInstanceDataType = FMyNearestThreatEvaluatorInstanceData;

    virtual const UStruct* GetInstanceDataType() const override { return FInstanceDataType::StaticStruct(); }

    virtual bool Link(FStateTreeLinker& Linker) override
    {
        Linker.LinkExternalData(PerceptionSubsystemHandle);
        return true;
    }

    virtual void TreeStart(FStateTreeExecutionContext& Context) const override
    {
        FInstanceDataType& Data = Context.GetInstanceData(*this);
        Data.NearestThreat = nullptr;
        Data.DistanceToNearest = 0.f;
        Data.TimeSinceLastQuery = Data.QueryInterval; // force a query on the first tick
    }

    virtual void Tick(FStateTreeExecutionContext& Context, const float DeltaTime) const override
    {
        FInstanceDataType& Data = Context.GetInstanceData(*this);
        Data.TimeSinceLastQuery += DeltaTime;
        if (Data.TimeSinceLastQuery < Data.QueryInterval || Data.Actor == nullptr)
        {
            return;
        }
        Data.TimeSinceLastQuery = 0.f;

        UMyPerceptionSubsystem& Perception = Context.GetExternalData(PerceptionSubsystemHandle);
        Data.NearestThreat = Perception.FindNearestThreat(Data.Actor, Data.DistanceToNearest);
    }

    virtual void TreeStop(FStateTreeExecutionContext& Context) const override
    {
        FInstanceDataType& Data = Context.GetInstanceData(*this);
        Data.NearestThreat = nullptr;
    }

    TStateTreeExternalDataHandle<UMyPerceptionSubsystem> PerceptionSubsystemHandle;
};
```

`UMyPerceptionSubsystem` here is a `UWorldSubsystem` exposing `AActor* FindNearestThreat(AActor* Querier, float& OutDistance) const;`. `UStateTreeComponentSchema::IsExternalItemAllowed` only accepts `AActor`, `UActorComponent` and `UWorldSubsystem` subclasses, so external data must be one of those.

---

## Consideration template

Considerations feed `TrySelectChildrenWithHighestUtility` and `TrySelectChildrenAtRandomWeightedByUtility`. Override the protected `GetScore`; the framework calls the public `GetNormalizedScore`.

```cpp
// MyDistanceConsideration.h
#pragma once

#include "StateTreeConsiderationBase.h"
#include "StateTreeExecutionContext.h"
#include "MyDistanceConsideration.generated.h"

USTRUCT()
struct FMyDistanceConsiderationInstanceData
{
    GENERATED_BODY()

    UPROPERTY(EditAnywhere, Category = "Input")
    float Distance = 0.f;

    UPROPERTY(EditAnywhere, Category = "Parameter")
    float MaxDistance = 2000.f;
};

USTRUCT(meta = (DisplayName = "My Distance Score"))
struct FMyDistanceConsideration : public FStateTreeConsiderationCommonBase
{
    GENERATED_BODY()

    using FInstanceDataType = FMyDistanceConsiderationInstanceData;

    virtual const UStruct* GetInstanceDataType() const override { return FInstanceDataType::StaticStruct(); }

protected:
    virtual float GetScore(FStateTreeExecutionContext& Context) const override
    {
        const FInstanceDataType& Data = Context.GetInstanceData(*this);
        if (Data.MaxDistance <= 0.f)
        {
            return 0.f;
        }
        return 1.f - FMath::Clamp(Data.Distance / Data.MaxDistance, 0.f, 1.f);
    }
};
```

Stock considerations to bind instead of writing your own: `FStateTreeConstantConsideration`, `FStateTreeFloatInputConsideration` (with an `FStateTreeConsiderationResponseCurve`) and `FStateTreeEnumInputConsideration` in `Considerations/StateTreeCommonConsiderations.h`.

---

## Delegate-driven task

A task exposes an `FStateTreeDelegateListener` on its instance data, binds it while the state is active and unbinds on exit. A transition with the `OnDelegate` trigger reacts to the matching dispatcher.

```cpp
// MyWaitForSignalTask.h
#pragma once

#include "StateTreeDelegate.h"
#include "StateTreeExecutionContext.h"
#include "StateTreeTaskBase.h"
#include "MyWaitForSignalTask.generated.h"

USTRUCT()
struct FMyWaitForSignalTaskInstanceData
{
    GENERATED_BODY()

    /** Bound in the editor to a dispatcher declared by another node. */
    UPROPERTY(EditAnywhere, Category = "Input")
    FStateTreeDelegateListener SignalListener;
};

USTRUCT(meta = (DisplayName = "My Wait For Signal"))
struct FMyWaitForSignalTask : public FStateTreeTaskCommonBase
{
    GENERATED_BODY()

    FMyWaitForSignalTask()
    {
        bShouldCallTick = false;
    }

    using FInstanceDataType = FMyWaitForSignalTaskInstanceData;

    virtual const UStruct* GetInstanceDataType() const override { return FInstanceDataType::StaticStruct(); }

    virtual EStateTreeRunStatus EnterState(FStateTreeExecutionContext& Context,
        const FStateTreeTransitionResult& Transition) const override
    {
        const FInstanceDataType& Data = Context.GetInstanceData(*this);
        const FStateTreeWeakExecutionContext WeakContext = Context.MakeWeakExecutionContext();

        Context.BindDelegate(Data.SignalListener, FSimpleDelegate::CreateLambda([WeakContext]()
        {
            WeakContext.FinishTask(EStateTreeFinishTaskType::Succeeded);
        }));

        return EStateTreeRunStatus::Running;
    }

    virtual void ExitState(FStateTreeExecutionContext& Context,
        const FStateTreeTransitionResult& Transition) const override
    {
        const FInstanceDataType& Data = Context.GetInstanceData(*this);
        Context.UnbindDelegate(Data.SignalListener);
    }
};
```

The dispatching side declares `UPROPERTY(EditAnywhere) FStateTreeDelegateDispatcher Dispatcher;` on its own instance data and calls `Context.BroadcastDelegate(Data.Dispatcher);`.

---

## Writing back to a bound property with FStateTreePropertyRef

`TStateTreePropertyRef<T>` lets a node write into a property that lives elsewhere in the tree (another node's instance data, a global parameter) instead of returning it as an output binding. `GetMutablePtr` is an inline template that calls into `FPropertyBindingBindingCollection`, so Build.cs also needs `PropertyBindingUtils` (`Plugins/Runtime/PropertyBindingUtils`) or the link fails.

```cpp
USTRUCT()
struct FMyStoreResultTaskInstanceData
{
    GENERATED_BODY()

    UPROPERTY(EditAnywhere, Category = "Output")
    TStateTreePropertyRef<float> ResultRef;

    UPROPERTY(EditAnywhere, Category = "Parameter")
    float ValueToStore = 1.f;
};

// inside EnterState
FInstanceDataType& Data = Context.GetInstanceData(*this);
if (float* Result = Data.ResultRef.GetMutablePtr(Context))
{
    *Result = Data.ValueToStore;
}
```

---

## Running a tree yourself

A custom runner owns the instance data and builds a context per tick. This is what `UStateTreeComponent` does internally.

```cpp
// MyTreeRunnerComponent.h
#pragma once

#include "Components/ActorComponent.h"
#include "StateTreeInstanceData.h"
#include "StateTreeReference.h"
#include "MyTreeRunnerComponent.generated.h"

struct FStateTreeExecutionContext;

UCLASS(ClassGroup = AI, meta = (BlueprintSpawnableComponent))
class MYGAME_API UMyTreeRunnerComponent : public UActorComponent
{
    GENERATED_BODY()

public:
    UMyTreeRunnerComponent();

    virtual void BeginPlay() override;
    virtual void EndPlay(const EEndPlayReason::Type EndPlayReason) override;
    virtual void TickComponent(float DeltaTime, ELevelTick TickType,
        FActorComponentTickFunction* ThisTickFunction) override;

protected:
    bool CollectExternalData(const FStateTreeExecutionContext& Context, const UStateTree* StateTree,
        TArrayView<const FStateTreeExternalDataDesc> Descs, TArrayView<FStateTreeDataView> OutDataViews) const;

    UPROPERTY(EditAnywhere, Category = "StateTree")
    FStateTreeReference StateTreeRef;

    UPROPERTY(Transient)
    FStateTreeInstanceData InstanceData;
};
```

```cpp
// MyTreeRunnerComponent.cpp
#include "MyTreeRunnerComponent.h"
#include "Components/StateTreeComponentSchema.h"
#include "StateTree.h"
#include "StateTreeExecutionContext.h"

UMyTreeRunnerComponent::UMyTreeRunnerComponent()
{
    PrimaryComponentTick.bCanEverTick = true;
}

void UMyTreeRunnerComponent::BeginPlay()
{
    Super::BeginPlay();

    const UStateTree* StateTree = StateTreeRef.GetStateTree();
    if (StateTree == nullptr || !StateTree->IsReadyToRun())
    {
        return;
    }

    FStateTreeExecutionContext Context(*this, *StateTree, InstanceData);
    Context.SetCollectExternalDataCallback(FOnCollectStateTreeExternalData::CreateUObject(
        this, &UMyTreeRunnerComponent::CollectExternalData));

    FStateTreeExecutionContext::FStartParameters StartParams;
    StartParams.InitialGlobalParameters = StateTreeRef.GetGlobalParameters();
    Context.Start(StartParams);
}

void UMyTreeRunnerComponent::TickComponent(float DeltaTime, ELevelTick TickType,
    FActorComponentTickFunction* ThisTickFunction)
{
    Super::TickComponent(DeltaTime, TickType, ThisTickFunction);

    const UStateTree* StateTree = StateTreeRef.GetStateTree();
    if (StateTree == nullptr)
    {
        return;
    }

    FStateTreeExecutionContext Context(*this, *StateTree, InstanceData);
    Context.SetCollectExternalDataCallback(FOnCollectStateTreeExternalData::CreateUObject(
        this, &UMyTreeRunnerComponent::CollectExternalData));
    Context.Tick(DeltaTime);
}

void UMyTreeRunnerComponent::EndPlay(const EEndPlayReason::Type EndPlayReason)
{
    const UStateTree* StateTree = StateTreeRef.GetStateTree();
    if (StateTree != nullptr)
    {
        FStateTreeExecutionContext Context(*this, *StateTree, InstanceData);
        Context.Stop();
    }

    Super::EndPlay(EndPlayReason);
}

bool UMyTreeRunnerComponent::CollectExternalData(const FStateTreeExecutionContext& Context,
    const UStateTree* StateTree, TArrayView<const FStateTreeExternalDataDesc> Descs,
    TArrayView<FStateTreeDataView> OutDataViews) const
{
    return UStateTreeComponentSchema::CollectExternalData(Context, StateTree, Descs, OutDataViews);
}
```

`FOnCollectStateTreeExternalData` is `DECLARE_DELEGATE_RetVal_FourParams(bool, FOnCollectStateTreeExternalData, const FStateTreeExecutionContext&, const UStateTree*, TArrayView<const FStateTreeExternalDataDesc>, TArrayView<FStateTreeDataView>)`. Reusing `UStateTreeComponentSchema::CollectExternalData` requires `#include "Components/StateTreeComponentSchema.h"` and the `GameplayStateTreeModule` dependency.

---

## FStateTreeReference and runtime overrides

```cpp
void AMyAIController::ConfigureTree(UStateTree* TreeAsset, float AggroRange)
{
    FStateTreeReference TreeRef;
    TreeRef.SetStateTree(TreeAsset);  // also calls SyncParameters()
    TreeRef.GetMutableParameters().SetValueFloat(TEXT("AggroRange"), AggroRange);

    StateTreeComp->SetStateTreeReference(TreeRef);
}

void AMyAIController::SwapCombatSubtree(const FGameplayTag VariantTag, FStateTreeReference CombatRef)
{
    StateTreeComp->AddLinkedStateTreeOverrides(VariantTag, MoveTemp(CombatRef));
}
```

`GetParameters()` returns `const FInstancedPropertyBag&` — writing needs `GetMutableParameters()`. `GetGlobalParameters()` / `GetMutableGlobalParameters()` return `FConstStructView` / `FStructView` for the same data, which is what `FStartParameters::InitialGlobalParameters` expects. Overrides are matched by exact tag against `Linked Asset` states; remove one with `RemoveLinkedStateTreeOverrides(VariantTag)`.

---

## Consuming events inside a node

```cpp
USTRUCT()
struct FMyEventReaderTaskInstanceData
{
    GENERATED_BODY()

    UPROPERTY(EditAnywhere, Category = "Parameter")
    FGameplayTag ExpectedTag;

    UPROPERTY()
    FInstancedStruct PayloadCopy;
};

// inside FMyEventReaderTask : public FStateTreeTaskCommonBase,
// with using FInstanceDataType = FMyEventReaderTaskInstanceData; and the GetInstanceDataType() override
virtual EStateTreeRunStatus EnterState(FStateTreeExecutionContext& Context,
    const FStateTreeTransitionResult& Transition) const override
{
    FInstanceDataType& Data = Context.GetInstanceData(*this);

    FStateTreeSharedEvent Matched;
    Context.ForEachEvent([&Data, &Matched](const FStateTreeSharedEvent& SharedEvent)
    {
        const FStateTreeEvent& Event = *SharedEvent;
        if (Event.Tag == Data.ExpectedTag)
        {
            Data.PayloadCopy = Event.Payload;
            Matched = SharedEvent;   // consume after the loop, not while iterating the queue
            return EStateTreeLoopEvents::Break;
        }
        return EStateTreeLoopEvents::Next;
    });
    if (Matched.IsValid())
    {
        Context.ConsumeEvent(Matched);   // returns void
    }

    return EStateTreeRunStatus::Running;
}
```

`EStateTreeLoopEvents` is `Next`, `Break` or `Consume` (`StateTreeEvents.h:35`; returning `Consume` removes the current event via the iterator, `StateTreeEvents.h:244`). Do not call `Context.ConsumeEvent` inside the lambda — it `RemoveAllSwap`s the array being iterated (`StateTreeEvents.cpp:62`). `Context.HasEventToProcess(Tag)` is the cheap existence check. `FStateTreeEventQueue::ConsumeEvent` on the queue itself returns `bool`; the context method does not.

---

## Transition design

| Pattern | Trigger | Use for |
|---|---|---|
| Reactive | `OnEvent` + tag | External stimuli: perception, damage, gameplay messages. Leave `bConsumeEventOnSelect` true so one event fires one transition |
| Callback-driven | `OnDelegate` | A C++ system that knows exactly when the condition became true; cheaper than polling |
| Polled | `OnTick` + conditions | Continuous checks such as distance or a timer. Keep the conditions cheap |
| Sequential | `OnStateCompleted` / `OnStateSucceeded` / `OnStateFailed` with target `NextState` | Linear chains where the next state depends on the previous outcome |
| Interrupt | any trigger at `Critical` priority | Death, stagger, forced retreat — evaluated before lower-priority transitions regardless of depth |

`EStateTreeSelectionFallback` on `Context.RequestTransition(TargetState, Priority, Fallback)` controls what happens when the target state cannot be selected.

---

## Debugging

- Override `GetDebugInfo(const FStateTreeReadOnlyExecutionContext& Context) const` on tasks and evaluators; the string appears in the gameplay debugger and visual logger.
- `UStateTreeComponent::GetActiveStateNames()` and `GetDebugInfoString()` are compiled under `WITH_GAMEPLAY_DEBUGGER`.
- Implement `FStateTreeExecutionExtension::GetInstanceDescription` to label instances in traces, and pass it through `FStartParameters::ExecutionExtension`.
- `Context.GetStateTreeRunStatus()` and `Context.GetLastTickStatus()` report where execution stopped.
