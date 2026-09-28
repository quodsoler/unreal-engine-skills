---
name: ue-state-trees
description: "Use when writing or debugging State Tree logic in Unreal Engine C++ — tasks, conditions, evaluators, considerations, schemas, transitions, events or Mass behaviours. Also use when the user mentions 'StateTree', 'state tree', 'FStateTreeTaskBase', 'FStateTreeExecutionContext', 'GetInstanceDataType', 'FInstanceDataType', 'UStateTreeComponent', 'UStateTreeAIComponent', 'FStateTreeReference', 'SendStateTreeEvent', 'EStateTreeRunStatus', 'StateTree schema', 'utility selection', 'scheduled tick', 'FinishTask', or 'Mass StateTree'. For behaviour trees, perception and navigation, see ue-ai-navigation; for Mass processors and fragments, see ue-mass-entity; for USTRUCT basics, see ue-cpp-foundations."
metadata:
  version: "2.0.0"
  engine: "5.8"
---

# UE State Trees

Target engine: **UE 5.8**. APIs below are verified against the 5.8 headers; older forms are listed under "Deprecated — do not use".

State Tree is a data-driven hierarchical state machine authored as a `UStateTree` data asset and executed from C++ through an execution context. Runtime types live in the `StateTreeModule` (plugin `StateTree`); actor-facing components and schemas live in `GameplayStateTreeModule` (plugin `GameplayStateTree`). Mass-entity behaviours add `MassAIBehavior` (plugin `MassAI`, Experimental in 5.8) plus `MassEntity`, `MassCore` and `MassSignals`. Add the modules you use to `PublicDependencyModuleNames` in your `.Build.cs`.

## Context

Read `.agents/ue-project-context.md` if it exists (module names, conventions, enabled plugins, GAS/networking setup). Do not stop if it is missing.

Identify the area from the request and the codebase. Ask only when two plausible readings would produce different code.

| Request is about… | Go to |
|---|---|
| What the runtime looks like, instance data, contexts | [Architecture](#architecture), [Execution Contexts](#execution-contexts) |
| Writing a task | [Tasks](#tasks) |
| Writing a condition or gating a transition | [Conditions](#conditions) |
| Utility scoring, "pick the best child state" | [Considerations](#considerations) |
| Feeding world data into the tree | [Evaluators](#evaluators), [External Data](#external-data) |
| When and how states change | [Transitions](#transitions), [State Types and Selection](#state-types-and-selection) |
| Tag-driven signals into the tree | [Events](#events) |
| Reacting to a C++ callback instead of polling | [Delegates](#delegates) |
| Restricting which nodes an asset may use | [Schemas](#schemas) |
| Running a tree on an actor or AI controller | [Component and AI Setup](#component-and-ai-setup) |
| Reducing tick cost / sleeping trees | [Scheduled Tick](#scheduled-tick) |
| Thousands of entities | [Mass Entity Integration](#mass-entity-integration) |
| Finishing work from a callback or another thread | [Async Completion](#async-completion) |

## Architecture

```
UStateTree (UDataAsset)          IsReadyToRun() must be true before execution
  ├── UStateTreeSchema           which nodes and context data are allowed
  ├── States                     hierarchy of States/Groups/Linked/Subtree
  │     ├── Tasks                work performed while the state is active
  │     ├── EnterConditions      gate on selecting the state
  │     ├── Considerations       utility score used by selection behaviours
  │     └── Transitions          rules for leaving the state
  ├── Evaluators                 global data providers, tick before transitions
  └── Parameters                 FInstancedPropertyBag global parameters
```

Nodes are `USTRUCT`s, not `UObject`s, and every node virtual is `const`. Mutable per-instance state lives in a separate instance-data struct owned by `FStateTreeInstanceData`, which is what persists across frames. The execution context is a short-lived view constructed over that instance data.

| Type | Role |
|---|---|
| `UStateTree` | The compiled asset. `IsReadyToRun()` reports link success |
| `FStateTreeInstanceData` | Persistent runtime storage — hold this, not the context |
| `FStateTreeExecutionContext` | Full read/write context; construct per tick |
| `FStateTreeReference` | `UStateTree*` plus overridden global parameters |
| `EStateTreeRunStatus` | `Running`, `Stopped`, `Succeeded`, `Failed`, `Unset` |

```cpp
// MyTreeRunner.h — persistent storage on the owner
UPROPERTY(Transient)
FStateTreeInstanceData InstanceData;

// .cpp — one context per tick over the same instance data
FStateTreeExecutionContext Context(*this, *StateTreeAsset, InstanceData);
Context.SetCollectExternalDataCallback(
    FOnCollectStateTreeExternalData::CreateUObject(this, &UMyTreeRunner::CollectExternalData));

FStateTreeExecutionContext::FStartParameters StartParams;
StartParams.RandomSeed = 1234;
Context.Start(StartParams);
// later frames
Context.Tick(DeltaTime);
```

`Tick` can be split when transitions must be evaluated after other systems have run: call `TickUpdateTasks(DeltaTime)` first and `TickTriggerTransitions()` afterwards.

## Execution Contexts

5.8 splits the context by capability. Pick the narrowest one that compiles.

| Context | Gets you | Typical use |
|---|---|---|
| `FStateTreeReadOnlyExecutionContext` | `GetOwner`, `GetWorld`, `GetStateTree`, `HasEventToProcess`, `IsValid` | `GetDebugInfo` overrides, gameplay debugger |
| `FStateTreeMinimalExecutionContext` | adds `SendEvent`, `ScheduleNextTick`, scheduled-tick requests | sending an event from outside the tick |
| `FStateTreeExecutionContext` | adds `Start`/`Stop`/`Tick`, instance data, external data, `FinishTask`, `BindDelegate` | everything inside a node |
| `FStateTreeWeakExecutionContext` | weak handles captured for later | lambdas, latent actions, timers |
| `FStateTreeStrongExecutionContext` | resolved access from a weak context | inside the callback, via `MakeStrongExecutionContext()` |

`FStateTreeMinimalExecutionContext` derives from `FStateTreeReadOnlyExecutionContext`, and `FStateTreeExecutionContext` from `FStateTreeMinimalExecutionContext`, so a node method taking the full context can call anything above it.

### Async Completion

Capture `FStateTreeWeakExecutionContext` (constructed from the live context) and finish the task when the async work returns. See [state-tree-patterns.md](references/state-tree-patterns.md) for the full latent-task template.

```cpp
const FStateTreeWeakExecutionContext WeakContext = Context.MakeWeakExecutionContext();
OnRequestFinished.AddLambda([WeakContext]()
{
    WeakContext.FinishTask(EStateTreeFinishTaskType::Succeeded);
});
```

Inside a tick, a task finishes itself with `Context.FinishTask(*this, EStateTreeFinishTaskType::Succeeded)` or by returning a completion status from `Tick`.

## Schemas

A schema declares which node structs, which classes and which context data an asset may use. `IsStructAllowed` is what makes your nodes appear in the editor, so derive custom nodes from the `*CommonBase` structs the stock schemas accept: `FStateTreeTaskCommonBase`, `FStateTreeConditionCommonBase`, `FStateTreeEvaluatorCommonBase`, `FStateTreeConsiderationCommonBase`, `FStateTreePropertyFunctionCommonBase`.

| Schema | Context data it publishes |
|---|---|
| `UStateTreeComponentSchema` | `Actor` (class from `ContextActorClass`) |
| `UStateTreeAIComponentSchema` | `Actor` (defaults to `APawn`) plus `AIController` |
| `UMassStateTreeSchema` | Mass entity data, via `FMassStateTreeExecutionContext` |

```cpp
// MyGameStateTreeSchema.h
#pragma once
#include "StateTreeSchema.h"
#include "StateTreeExecutionTypes.h"
#include "MyGameStateTreeSchema.generated.h"

UCLASS(BlueprintType, EditInlineNew, CollapseCategories, meta = (DisplayName = "My Game Schema"))
class MYGAME_API UMyGameStateTreeSchema : public UStateTreeSchema
{
    GENERATED_BODY()

protected:
    virtual bool IsStructAllowed(const UScriptStruct* InScriptStruct) const override;
    virtual bool IsClassAllowed(const UClass* InScriptStruct) const override;
    virtual bool IsExternalItemAllowed(const UStruct& InStruct) const override;
    virtual TConstArrayView<FStateTreeExternalDataDesc> GetContextDataDescs() const override;

#if WITH_EDITOR
    virtual bool AllowEvaluators() const override { return true; }
    virtual bool AllowMultipleTasks() const override { return true; }
    virtual bool AllowGlobalParameters() const override { return true; }
    virtual bool AllowUtilityConsiderations() const override { return true; }
#endif

    UPROPERTY()
    TArray<FStateTreeExternalDataDesc> ContextDataDescs;
};
```

`AllowEnterConditions`, `AllowUtilityConsiderations`, `AllowEvaluators`, `AllowMultipleTasks`, `AllowGlobalParameters`, `AllowTasksCompletion` and `AllowQueuedCompilation` are declared inside `#if WITH_EDITOR` (`StateTreeSchema.h:96-138`) — they only drive the editor/compiler, so your overrides must be guarded the same way or game builds fail with "does not override". `IsStructAllowed`, `IsClassAllowed`, `IsExternalItemAllowed`, `IsScheduledTickAllowed`, `IsStateSelectionAllowed`, `IsStateTypeAllowed` and `GetContextDataDescs` are runtime virtuals (`StateTreeSchema.h:36-79`).

## Tasks

Every task overrides the virtuals it needs **and** `GetInstanceDataType()`. The `using FInstanceDataType = …;` alias alone allocates nothing; `FStateTreeNodeBase::GetInstanceDataType()` returns `nullptr` by default (`StateTreeNodeBase.h:94`) and the compiler reserves no storage, so `Context.GetInstanceData(*this)` then hits `check(Memory != nullptr)` (`PropertyBindingDataView.h:116`).

| Virtual (verbatim signature) | Returns |
|---|---|
| `EnterState(FStateTreeExecutionContext& Context, const FStateTreeTransitionResult& Transition) const` | `EStateTreeRunStatus` |
| `ExitState(FStateTreeExecutionContext& Context, const FStateTreeTransitionResult& Transition) const` | `void` |
| `StateCompleted(FStateTreeExecutionContext& Context, const EStateTreeRunStatus CompletionStatus, const FStateTreeActiveStates& CompletedActiveStates) const` | `void` |
| `Tick(FStateTreeExecutionContext& Context, const float DeltaTime) const` | `EStateTreeRunStatus` |
| `TriggerTransitions(FStateTreeExecutionContext& Context) const` | `void` |
| `GetDebugInfo(const FStateTreeReadOnlyExecutionContext& Context) const` | `FString` |

```cpp
// MyTimedTask.h
#pragma once
#include "StateTreeTaskBase.h"
#include "StateTreeExecutionContext.h"
#include "MyTimedTask.generated.h"

USTRUCT()
struct FMyTimedTaskInstanceData
{
    GENERATED_BODY()

    UPROPERTY(EditAnywhere, Category = "Parameter")
    float Duration = 2.0f;

    UPROPERTY()
    float ElapsedTime = 0.f;
};

USTRUCT(meta = (DisplayName = "My Timed Task"))
struct FMyTimedTask : public FStateTreeTaskCommonBase
{
    GENERATED_BODY()

    using FInstanceDataType = FMyTimedTaskInstanceData;

    virtual const UStruct* GetInstanceDataType() const override { return FInstanceDataType::StaticStruct(); }

    virtual EStateTreeRunStatus EnterState(FStateTreeExecutionContext& Context,
        const FStateTreeTransitionResult& Transition) const override
    {
        FInstanceDataType& Data = Context.GetInstanceData(*this);
        Data.ElapsedTime = 0.f;
        return EStateTreeRunStatus::Running;
    }

    virtual EStateTreeRunStatus Tick(FStateTreeExecutionContext& Context,
        const float DeltaTime) const override
    {
        FInstanceDataType& Data = Context.GetInstanceData(*this);
        Data.ElapsedTime += DeltaTime;
        return Data.ElapsedTime >= Data.Duration
            ? EStateTreeRunStatus::Succeeded
            : EStateTreeRunStatus::Running;
    }
};
```

Editable parameters usually live on the instance data so they can be bound in the editor; properties placed on the node struct itself are shared by every instance and cannot be bound.

### Behavioural flags

Set these in the task constructor.

| Flag | Default | Effect |
|---|---|---|
| `bShouldStateChangeOnReselect` | `true` | Re-run Exit/Enter when the same state is selected again |
| `bShouldCallTick` | `true` | Call `Tick()`; false also disables property copying |
| `bShouldCallTickOnlyOnEvents` | `false` | Tick only on frames with events (needs `bShouldCallTick` false) |
| `bShouldCopyBoundPropertiesOnTick` | `true` | Refresh bound properties before `Tick()` |
| `bShouldCopyBoundPropertiesOnExitState` | `true` | Refresh bound properties before `ExitState()` |
| `bShouldAffectTransitions` | `false` | Call `TriggerTransitions()` during transition handling |
| `bConsideredForScheduling` | `true` | Include this task when computing the scheduled tick rate |
| `bTaskEnabled` | `true` | Node not disabled in the asset |

`TransitionHandlingPriority` (`EStateTreeTransitionPriority`) orders `TriggerTransitions` across tasks of one state.

## Conditions

`FStateTreeConditionBase::TestCondition(FStateTreeExecutionContext& Context) const` returns `bool` and defaults to `false`. It must be pure: the selection pass may call it several times per frame.

| Member | Purpose |
|---|---|
| `EStateTreeExpressionOperand Operand` | `Copy`, `And`, `Or`, `Multiply` (`Multiply` is considerations only) |
| `int8 DeltaIndent` | Parenthesis level, gives `(A AND B) OR (C AND D)` without nesting |
| `EStateTreeConditionEvaluationMode EvaluationMode` | `Evaluated`, `ForcedTrue`, `ForcedFalse` |

Conditions also receive `EnterState`, `ExitState` and `StateCompleted` (all no-ops by default) so they can cache work across the state's lifetime.

Stock conditions: `FStateTreeCompareIntCondition`, `FStateTreeCompareFloatCondition`, `FStateTreeCompareBoolCondition`, `FStateTreeCompareEnumCondition`, `FStateTreeCompareNameCondition`, `FStateTreeCompareDistanceCondition`, `FStateTreeRandomCondition` (`Conditions/StateTreeCommonConditions.h`), `FStateTreeObjectIsValidCondition` (`Conditions/StateTreeObjectConditions.h`) and the gameplay-tag family in `Conditions/StateTreeGameplayTagConditions.h`. The comparison ones take `UE::StateTree::EComparisonOperator`.

## Considerations

Considerations score a state so a parent can choose between children. The API is marked experimental in the header, so keep custom considerations small.

```cpp
USTRUCT(meta = (DisplayName = "My Threat Score"))
struct FMyThreatConsideration : public FStateTreeConsiderationCommonBase
{
    GENERATED_BODY()

    using FInstanceDataType = FMyThreatConsiderationInstanceData;

    virtual const UStruct* GetInstanceDataType() const override { return FInstanceDataType::StaticStruct(); }

protected:
    virtual float GetScore(FStateTreeExecutionContext& Context) const override
    {
        const FInstanceDataType& Data = Context.GetInstanceData(*this);
        return FMath::Clamp(Data.ThreatLevel / 100.f, 0.f, 1.f);
    }
};
```

`GetScore` is `protected`; the framework calls the public `GetNormalizedScore`. `Operand` and `DeltaIndent` combine sibling considerations (`And` = min, `Or` = max, `Multiply` = product). Scores only matter for the two utility selection behaviours, and the schema must return true from `AllowUtilityConsiderations()`.

## Evaluators

Evaluators are global (not per-state) and tick before transitions and task ticks. Use them for data many nodes read; use external data for stable references.

```cpp
virtual void TreeStart(FStateTreeExecutionContext& Context) const;
virtual void TreeStop(FStateTreeExecutionContext& Context) const;
virtual void Tick(FStateTreeExecutionContext& Context, const float DeltaTime) const;
virtual FString GetDebugInfo(const FStateTreeReadOnlyExecutionContext& Context) const;
```

`DeltaTime` is `0` when the evaluator is ticked during state pre-selection. Evaluators need `GetInstanceDataType()` exactly like tasks — the full template is in [state-tree-patterns.md](references/state-tree-patterns.md).

## Transitions

`EStateTreeTransitionTrigger` is a bitmask (`ENUM_CLASS_FLAGS`).

| Trigger | Value | Fires when |
|---|---|---|
| `OnStateSucceeded` | `0x1` | The state completed with Succeeded |
| `OnStateFailed` | `0x2` | The state completed with Failed |
| `OnStateCompleted` | `0x1｜0x2` | Either of the above |
| `OnTick` | `0x4` | Every tick — always gate with conditions |
| `OnEvent` | `0x8` | A queued event matches the tag |
| `OnDelegate` | `0x10` | A bound delegate was broadcast |

`EStateTreeTransitionPriority`: `Low`, `Normal`, `Medium`, `High`, `Critical`. The first triggered transition of the highest priority wins. Transition types (`EStateTreeTransitionType`): `None`, `Succeeded`, `Failed`, `GotoState`, `Parent`, `NextState`, `NextSelectableState`, `NextParent`, `NextSelectableParent` (`StateTreeTypes.h:76`). Per-transition flags `bTransitionEnabled` and `bConsumeEventOnSelect` both default to `true`.

From C++, request a transition with `Context.RequestTransition(TargetState, Priority, Fallback)` where `Fallback` is an `EStateTreeSelectionFallback`.

## Delegates

Delegates replace polling when an external system can tell the tree exactly when to react. One node publishes an `FStateTreeDelegateDispatcher` on its instance data, another publishes an `FStateTreeDelegateListener`, and the editor connects the two.

```cpp
// On the listening node's instance data
UPROPERTY(EditAnywhere, Category = "Input")
FStateTreeDelegateListener ArrivedListener;

// EnterState — bind
Context.BindDelegate(Data.ArrivedListener, FSimpleDelegate::CreateLambda([]()
{
    UE_LOG(LogMyGame, Verbose, TEXT("Arrived"));
}));

// ExitState — always unbind
Context.UnbindDelegate(Data.ArrivedListener);

// On the dispatching node
Context.BroadcastDelegate(Data.ArrivedDispatcher);
```

A transition with the `OnDelegate` trigger fires when the bound dispatcher is broadcast, evaluated with the rest of the transitions.

## State Types and Selection

`EStateTreeStateType`: `State`, `Group`, `Linked`, `LinkedAsset`, `Subtree`. `LinkedAsset` states are the sharing mechanism — override them per instance with `FStateTreeReferenceOverrides`.

`EStateTreeStateSelectionBehavior`:

| Behaviour | Effect |
|---|---|
| `None` | State cannot be selected directly |
| `TryEnterState` | Enter this state even if it has children |
| `TrySelectChildrenInOrder` | First child whose enter conditions pass |
| `TrySelectChildrenAtRandom` | Shuffle children, take the first that passes |
| `TrySelectChildrenWithHighestUtility` | Highest consideration score, ties broken in order |
| `TrySelectChildrenAtRandomWeightedByUtility` | Random weighted by normalised score |
| `TryFollowTransitions` | Evaluate the state's transitions instead of entering |

`FStateTreeActiveStates::MaxStates = 8` caps the depth of the active state path. Flatten deeper designs with subtrees or linked assets.

## Events

Events are gameplay-tag messages with an optional payload: `FStateTreeEvent { FGameplayTag Tag; FInstancedStruct Payload; FName Origin; }`.

```cpp
// From outside the tree
StateTreeComp->SendStateTreeEvent(MyTag_AiAlert, FConstStructView::Make(AlertData), TEXT("Perception"));

// From inside a node (available on the minimal context and above)
Context.SendEvent(MyTag_AiAlert, FConstStructView::Make(ResultData), TEXT("MyTask"));
```

`FStateTreeEventQueue::MaxActiveEvents = 64` per instance. Iterate with `Context.ForEachEvent(Lambda)`, whose lambda returns `EStateTreeLoopEvents` (`Next`, `Break`, `Consume` — `StateTreeEvents.h:35`) and remove one with `Context.ConsumeEvent(SharedEvent)` (returns `void`). `Context.HasEventToProcess(Tag)` is a cheap existence check. For where gameplay tags are declared, see `ue-gameplay-tags-messaging`.

## External Data

External data gives a node a typed reference to an object or struct supplied by the schema's owner — subsystems, the owning actor, components.

```cpp
TStateTreeExternalDataHandle<UMyWorldSubsystem> SubsystemHandle;
TStateTreeExternalDataHandle<AActor, EStateTreeExternalDataRequirement::Optional> ActorHandle;

virtual bool Link(FStateTreeLinker& Linker) override
{
    Linker.LinkExternalData(SubsystemHandle);
    Linker.LinkExternalData(ActorHandle);
    return true;
}

// Required handles return a reference, Optional handles a pointer
UMyWorldSubsystem& Subsystem = Context.GetExternalData(SubsystemHandle);
AActor* Actor = Context.GetExternalDataPtr(ActorHandle);
```

`Link` returns `bool` (`[[nodiscard]]` on the base, `StateTreeNodeBase.h:116`); return `true` on success, `false` fails linking.

`UStateTreeComponentSchema::IsExternalItemAllowed` accepts `AActor`, `UActorComponent` and `UWorldSubsystem` subclasses only. The owning actor is published separately as *context data* named `Actor` (and `AIController` for the AI schema); the idiomatic way to reach it is an instance-data property the editor binds automatically:

```cpp
UPROPERTY(EditAnywhere, Category = "Context")
TObjectPtr<AActor> Actor = nullptr;
```

There is no `FStateTreeActorContext` type in the engine — use the above.

## Component and AI Setup

`UStateTreeComponent : UBrainComponent, IGameplayTaskOwnerInterface, IStateTreeSchemaProvider` runs one tree on an actor. `UStateTreeAIComponent` derives from it and returns `UStateTreeAIComponentSchema`, which guarantees an `AAIController` context value — use it on AI controllers.

```cpp
// MyAIController.h — inside the AMyAIController class body
UPROPERTY(VisibleAnywhere, Category = "AI")
TObjectPtr<UStateTreeAIComponent> StateTreeComp;

// MyAIController.cpp
AMyAIController::AMyAIController()
{
    StateTreeComp = CreateDefaultSubobject<UStateTreeAIComponent>(TEXT("StateTree"));
    StateTreeComp->SetStartLogicAutomatically(true);
}
```

`bStartLogicAutomatically` is `protected`; write it through `SetStartLogicAutomatically(const bool)`. Public surface: `SetStateTree`, `SetStateTreeReference`, `AddLinkedStateTreeOverrides(FGameplayTag, FStateTreeReference)`, `RemoveLinkedStateTreeOverrides(FGameplayTag)`, `SendStateTreeEvent`, `GetStateTreeRunStatus`, and the `BlueprintAssignable` `OnStateTreeRunStatusChanged`. `StartLogic`, `StopLogic`, `PauseLogic` and `ResumeLogic` come from `UBrainComponent`.

Subclass and override the protected `SetContextRequirements(FStateTreeExecutionContext& Context, bool bLogErrors)` / `CollectExternalData(const FStateTreeExecutionContext& Context, const UStateTree* StateTree, TArrayView<const FStateTreeExternalDataDesc> Descs, TArrayView<FStateTreeDataView> OutDataViews) const` to publish extra context or external data.

### FStateTreeReference

```cpp
FStateTreeReference TreeRef;
TreeRef.SetStateTree(MyStateTreeAsset);          // also calls SyncParameters()
TreeRef.GetMutableParameters().SetValueFloat(TEXT("AggroRange"), 1500.f);
StateTreeComp->SetStateTreeReference(TreeRef);
```

`GetParameters()` returns `const FInstancedPropertyBag&` — mutating needs `GetMutableParameters()`.

## Scheduled Tick

A tree whose active tasks all opt out of ticking can sleep instead of running every frame. `FStateTreeScheduledTick::MakeSleep()`, `MakeNextFrame()`, `MakeEveryFrames()` and `MakeCustomTickRate(DeltaTime)` describe the desired cadence; `FStateTreeMinimalExecutionContext::AddScheduledTickRequest` / `UpdateScheduledTickRequest` / `RemoveScheduledTickRequest` manage a request, and `ScheduleNextTick()` wakes the tree. `UStateTreeComponent` implements this through `FStateTreeComponentExecutionExtension`, gated by `UStateTreeComponentSchema::IsScheduledTickAllowed()` and the `StateTree.Component.DefaultScheduledTickAllowed` console variable. A task that must not be counted sets `bConsideredForScheduling = false`.

For per-instance descriptions and custom tick scheduling on your own runner, implement `FStateTreeExecutionExtension` (`GetInstanceDescription`, `ScheduleNextTick`, `OnLinkedStateTreeOverridesSet`, `OnBeginApplyTransition`) and pass it in `FStartParameters::ExecutionExtension`.

## Mass Entity Integration

Mass behaviours use `UMassStateTreeSchema` and the `FMassStateTreeTaskBase`, `FMassStateTreeConditionBase` and `FMassStateTreeEvaluatorBase` node types, which add `GetDependencies(UE::MassBehavior::FStateTreeDependencyBuilder&) const` so the dynamically created `UMassStateTreeProcessor` (a `UMassSignalProcessorBase`) can build the right fragment query. Full setup, fragment table, signal names and processor flow are in [state-tree-mass-integration.md](references/state-tree-mass-integration.md). For Mass architecture itself, see `ue-mass-entity`.

## Deprecated — do not use

| Do not emit | Use in 5.8 | Source |
|---|---|---|
| `Start(const FInstancedPropertyBag*, int32 RandomSeed)` | `Start(FStartParameters)` | `UE_DEPRECATED(5.8)` in `StateTreeExecutionContext.h:492` |
| `AppendDebugInfoString(FString&, const FStateTreeExecutionContext&)` | `GetDebugInfo(const FStateTreeReadOnlyExecutionContext&)` | `UE_DEPRECATED(5.8)` in `StateTreeEvaluatorBase.h:44` |
| `EGenericAICheck` operator on the compare conditions | `UE::StateTree::EComparisonOperator` | `UE_DEPRECATED(5.8)` in `Conditions/StateTreeCommonConditions.h:47` |
| `STATETREE_POD_INSTANCEDATA(Type)` | `UE_STATETREE_ZEROED_TRIVIALLY_COPIED_NO_DESTRUCTOR_INSTANCEDATA(Type)` (or the `_CONSTRUCTED_` variant) | `UE_DEPRECATED_MACRO(5.8)` in `StateTreeTypes.h:1388` |
| `FStateTreePropertyRefExternalHandle` | `FStateTreePropertyRef` read through the execution context | `UE_DEPRECATED(5.8)` in `StateTreePropertyRef.h:295` |
| `AddDelegateListener(Listener, Delegate)` | `BindDelegate(Listener, Delegate)` | `UE_DEPRECATED(5.6)` in `StateTreeExecutionContext.h:530` |
| `RemoveDelegateListener(Listener)` | `UnbindDelegate(Listener)` | `UE_DEPRECATED(5.6)` in `StateTreeExecutionContext.h:541` |
| `FinishTask(const UE::StateTree::FFinishedTask&, EStateTreeFinishTaskType)` | `FinishTask(const FStateTreeTaskBase&, EStateTreeFinishTaskType)` | `UE_DEPRECATED(5.6)` in `StateTreeExecutionContext.h:790` |
| `FStateTreeWeakTaskRef` / `FStateTreeStrongTaskRef` | `FStateTreeWeakExecutionContext` | `UE_DEPRECATED(5.6)` in `StateTreeNodeRef.h:15,51` |
| `OnBindingChanged(…, const FStateTreePropertyPath&, …)` | the `FPropertyBindingPath` overload | `UE_DEPRECATED(5.6)` in `StateTreeNodeBase.h:186` |
| `TrySelectChildrenAtUniformRandom` | `TrySelectChildrenAtRandom` | `UE_DEPRECATED(all)` in `StateTreeTypes.h:196` |
| `TrySelectChildrenBasedOnRelativeUtility` | `TrySelectChildrenAtRandomWeightedByUtility` | `UE_DEPRECATED(all)` in `StateTreeTypes.h:197` |
| `Compile(FStateTreeDataView, TArray<FText>&)` | `Compile(UE::StateTree::ICompileNodeContext&)` | `UE_DEPRECATED(5.6)` `final` stub in `StateTreeNodeBase.h:141` |
| `StateTreeComp->bStartLogicAutomatically = true;` | `StateTreeComp->SetStartLogicAutomatically(true);` | `protected` in `Components/StateTreeComponent.h:187` |
| `TreeRef.GetParameters().SetValueFloat(…)` | `TreeRef.GetMutableParameters().SetValueFloat(…)` | const accessor in `StateTreeReference.h:61` |
| `FStateTreeActorContext` | `TStateTreeExternalDataHandle<AActor>` or a `Category = "Context"` instance-data property | no such type in the engine |

## Common Mistakes

**Omitting `GetInstanceDataType()`:** the single most common State Tree bug. `using FInstanceDataType = …;` documents the type; only the override registers storage.
```cpp
using FInstanceDataType = FMyTaskInstanceData;
virtual const UStruct* GetInstanceDataType() const override { return FInstanceDataType::StaticStruct(); }
```

**Storing the execution context:** it is a per-tick view over `FStateTreeInstanceData`. Keep the instance data as a `UPROPERTY(Transient)` on the owner and rebuild the context each tick, or capture `FStateTreeWeakExecutionContext` if you need it later.

**Mutating the node struct:** every node virtual is `const`, so `Timer += DeltaTime;` on a member will not compile. Write to `Context.GetInstanceData(*this)`.

**Deriving from `FStateTreeTaskBase` directly:** stock schemas only allow `FStateTreeTaskCommonBase` and siblings, so the node never appears in the editor picker. Derive from the `*CommonBase` struct (or the Mass/AI base) that your schema accepts.

**Not guarding schema `Allow*` overrides with `#if WITH_EDITOR`:** the base declares them editor-only (`StateTreeSchema.h:96`), so an unguarded `override` compiles in the editor and fails in Game/Shipping targets.

**Side effects in `TestCondition`:** selection may test the same condition several times in a frame. Keep it pure; cache in the condition's `EnterState` instead.

**Leaving delegates bound:** `BindDelegate` in `EnterState` requires `UnbindDelegate` in `ExitState`, otherwise the listener fires against a state that is no longer active.

**Exceeding `MaxStates`:** the active state path is capped at 8. Deeper hierarchies fail selection rather than warning loudly.

**Assuming `Tick` runs:** with `bShouldCallTick = false`, or when a scheduled tick puts the tree to sleep, tasks do not tick and bound properties are not copied. Drive those tasks from events or delegates.

## Related Skills

- `ue-ai-navigation` — behaviour trees, perception, EQS, navigation and Smart Objects
- `ue-mass-entity` — Mass processors, fragments, traits, entity queries
- `ue-gameplay-abilities` — abilities and effects that drive or react to state changes
- `ue-gameplay-framework` — controllers, pawns and game-mode lifecycle around the tree
- `ue-actor-component-architecture` — component ownership, ticking and replication setup
- `ue-cpp-foundations` — `USTRUCT` rules, delegates, subsystem access
- `ue-gameplay-tags-messaging` — declaring gameplay tags used by State Tree events and transitions
- `ue-gameplay-cameras` — spring arms, view targets, camera modifiers, shakes and the Gameplay Camera System
