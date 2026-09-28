# Blueprint Node Patterns — Libraries, Latent Actions, Async Actions

Complete implementations for the node shapes summarised in the skill body. Verified against UE 5.8 (`Runtime/Engine/Classes/Kismet/`, `Runtime/Engine/Public/LatentActions.h`, `Runtime/Engine/Classes/Engine/LatentActionManager.h`, `Runtime/Engine/Classes/Engine/CancellableAsyncAction.h`).

## Choosing a node shape

| The node must… | Use |
|---|---|
| Return immediately | `UFUNCTION(BlueprintCallable)` / `BlueprintPure` |
| Hold one exec pin until a condition is met, tick-driven | Latent action (`FPendingLatentAction`) |
| Fire one of several exec pins when events arrive, possibly more than once | `UBlueprintAsyncActionBase` |
| Do the above and be cancellable from Blueprint | `UCancellableAsyncAction` |
| Run every frame while active | Latent action; its `UpdateOperation` is called once per tick |

A latent node can only live in an event graph: the Blueprint compiler rejects one anywhere else with "contains a latent call, which cannot exist outside of the event graph". Async action nodes have no such restriction.

---

## Function Library Implementation

```cpp
// MyBlueprintLibrary.cpp
#include "MyBlueprintLibrary.h"
#include "Engine/Engine.h"
#include "Engine/World.h"
#include "EngineUtils.h"
#include "GameFramework/Actor.h"
#include "GameFramework/Pawn.h"

int32 UMyBlueprintLibrary::CountPawns(const UObject* WorldContextObject)
{
    // meta=(WorldContext="WorldContextObject") hides this pin; the graph fills it in.
    UWorld* World = GEngine->GetWorldFromContextObject(WorldContextObject, EGetWorldErrorMode::LogAndReturnNull);
    if (!World) { return 0; }

    int32 Count = 0;
    for (TActorIterator<APawn> It(World); It; ++It) { ++Count; }
    return Count;
}

AActor* UMyBlueprintLibrary::FindAttachedActorOfClass(AActor* Target, TSubclassOf<AActor> ActorClass)
{
    if (!Target || !ActorClass) { return nullptr; }

    TArray<AActor*> Attached;
    Target->GetAttachedActors(Attached);
    for (AActor* Actor : Attached)
    {
        if (Actor->IsA(ActorClass)) { return Actor; }
    }
    return nullptr;
}

void UMyBlueprintLibrary::TryConsume(int32 Available, int32 Amount, EMyConsumeResult& Result)
{
    Result = (Available >= Amount) ? EMyConsumeResult::Consumed : EMyConsumeResult::NotEnough;
}
```

`EGetWorldErrorMode` (`Engine/Engine.h:62`) has three values: `ReturnNull`, `LogAndReturnNull` and `Assert`. Use `LogAndReturnNull` in library code so a misused node logs instead of crashing.

Other world-context resolutions used by the engine: `GEngine->GetWorldFromContextObjectChecked(Object)` when null is a bug, and taking an `AActor*`/`UActorComponent*` parameter with `meta=(DefaultToSelf="Target")` and calling `Target->GetWorld()`.

---

## Latent Actions

### The complete action class

```cpp
// MyWaitAction.h
#pragma once
#include "CoreMinimal.h"
#include "Engine/LatentActionManager.h"
#include "LatentActions.h"

/** Counts down and triggers its output link when the time remaining reaches zero. */
class FMyWaitAction : public FPendingLatentAction
{
public:
    float TimeRemaining;
    FName ExecutionFunction;
    int32 OutputLink;
    FWeakObjectPtr CallbackTarget;

    FMyWaitAction(float Duration, const FLatentActionInfo& LatentInfo)
        : TimeRemaining(Duration)
        , ExecutionFunction(LatentInfo.ExecutionFunction)
        , OutputLink(LatentInfo.Linkage)
        , CallbackTarget(LatentInfo.CallbackTarget)
    {
    }

    /** Called once per tick of the owning object while the action is pending. */
    virtual void UpdateOperation(FLatentResponse& Response) override
    {
        TimeRemaining -= Response.ElapsedTime();
        Response.FinishAndTriggerIf(TimeRemaining <= 0.f, ExecutionFunction, OutputLink, CallbackTarget);
    }

    /** The object that started the action was collected; no more UpdateOperation calls follow. */
    virtual void NotifyObjectDestroyed() override {}

    /** The graph abandoned the action (for example the owning object was torn down). */
    virtual void NotifyActionAborted() override {}

#if WITH_EDITOR
    /** Shown in the Blueprint debugger's pending-latent-action list. */
    virtual FString GetDescription() const override
    {
        return FString::Printf(TEXT("Wait (%.2f seconds left)"), TimeRemaining);
    }
#endif
};
```

`FLatentActionInfo` is `USTRUCT(BlueprintInternalUseOnly)` (`Engine/LatentActionManager.h:17`), so the parameter never shows as a user-facing pin — the graph fills it in. It carries four fields the action must copy: `Linkage` (the output exec link index), `UUID` (identifies this node instance), `ExecutionFunction` (the graph function to resume) and `CallbackTarget` (the object that owns the graph). Store `CallbackTarget` as an `FWeakObjectPtr`, never as a raw pointer.

`FLatentResponse` (`LatentActions.h:9`) offers:

| Call | Effect |
|---|---|
| `ElapsedTime()` | Delta time for this update |
| `DoneIf(bool)` | Removes the action without firing any link |
| `TriggerLink(ExecutionFunction, LinkID, CallbackTarget)` | Fires one link and keeps running |
| `FinishAndTriggerIf(bool, ExecutionFunction, LinkID, CallbackTarget)` | Fires the link and removes the action when the condition is true |

A node with several output exec pins calls `TriggerLink` with different `LinkID` values (the `Linkage` values the graph assigned) and finishes with `DoneIf(true)`.

### Registering the action

```cpp
// MyLatentLibrary.cpp
#include "MyLatentLibrary.h"
#include "MyWaitAction.h"
#include "Engine/Engine.h"
#include "Engine/World.h"

void UMyLatentLibrary::WaitSeconds(const UObject* WorldContextObject, float Duration, FLatentActionInfo LatentInfo)
{
    if (UWorld* World = GEngine->GetWorldFromContextObject(WorldContextObject, EGetWorldErrorMode::LogAndReturnNull))
    {
        FLatentActionManager& Manager = World->GetLatentActionManager();
        if (Manager.FindExistingAction<FMyWaitAction>(LatentInfo.CallbackTarget, LatentInfo.UUID) == nullptr)
        {
            Manager.AddNewAction(LatentInfo.CallbackTarget, LatentInfo.UUID, new FMyWaitAction(Duration, LatentInfo));
        }
    }
}
```

The manager owns the action and deletes it; never delete it yourself and never keep the pointer past `AddNewAction`.

### Retriggerable variant

`UKismetSystemLibrary::RetriggerableDelay` reuses the existing action instead of ignoring the call:

```cpp
FLatentActionManager& Manager = World->GetLatentActionManager();
if (FMyWaitAction* Existing = Manager.FindExistingAction<FMyWaitAction>(LatentInfo.CallbackTarget, LatentInfo.UUID))
{
    Existing->TimeRemaining = Duration;      // reset instead of stacking a second action
}
else
{
    Manager.AddNewAction(LatentInfo.CallbackTarget, LatentInfo.UUID, new FMyWaitAction(Duration, LatentInfo));
}
```

### The rest of `FLatentActionManager`

| Member | Use |
|---|---|
| `FindExistingAction<ActionType>(UObject* InActionObject, int32 UUID)` | The UUID guard; returns null when nothing is pending |
| `FindExistingActionWithPredicate<ActionType, PredicateType>(UObject*, int32, const PredicateType&)` | Same, filtered when several actions share a UUID |
| `AddNewAction(UObject* InActionObject, int32 UUID, FPendingLatentAction* NewAction)` | Takes ownership of the action |
| `RemoveActionsForObject(TWeakObjectPtr<UObject> InObject)` | Cancels every pending action for an object |
| `GetActiveUUIDs(UObject* InObject, TSet<int32>& UUIDList) const` | Which nodes on an object are pending |
| `GetDescription(UObject* InObject, int32 UUID) const` | Editor-facing text from the action's `GetDescription` |

The `Latent`/`LatentInfo` UFUNCTION metadata is what makes the graph pass a filled-in `FLatentActionInfo`; without both keys the parameter arrives zeroed and the output pin never fires.

---

## Async Actions

### The complete action

Header: see the skill body. Implementation:

```cpp
// MyWaitForHealthAction.cpp
#include "MyWaitForHealthAction.h"
#include "MyStatsComponent.h"

UMyWaitForHealthAction* UMyWaitForHealthAction::WaitForHealthBelow(UObject* WorldContextObject, UMyStatsComponent* Stats, double Threshold)
{
    UMyWaitForHealthAction* Action = NewObject<UMyWaitForHealthAction>();
    Action->TrackedStats = Stats;
    Action->ThresholdValue = Threshold;
    Action->RegisterWithGameInstance(WorldContextObject);
    return Action;
}

void UMyWaitForHealthAction::Activate()
{
    if (!TrackedStats)
    {
        OnFailed.Broadcast(0.0);
        SetReadyToDestroy();
        return;
    }
    TrackedStats->OnHealthChanged.AddDynamic(this, &UMyWaitForHealthAction::HandleHealthChanged);
}

void UMyWaitForHealthAction::HandleHealthChanged(double NewHealth)
{
    if (NewHealth <= ThresholdValue)
    {
        TrackedStats->OnHealthChanged.RemoveDynamic(this, &UMyWaitForHealthAction::HandleHealthChanged);
        OnReached.Broadcast(NewHealth);
        SetReadyToDestroy();
    }
}
```

Order of operations for the generated node:

1. The graph calls the static factory. It does `NewObject`, stores the inputs, calls `RegisterWithGameInstance(WorldContextObject)` and returns the proxy.
2. The node binds every `BlueprintAssignable` delegate to its output exec pins.
3. The node calls `Activate()`. Start the work there, not in the factory — delegates are not bound yet when the factory runs.
4. Each completion path broadcasts and calls `SetReadyToDestroy()`.

`UBlueprintAsyncActionBase` API (`Kismet/BlueprintAsyncActionBase.h`):

| Member | Behaviour |
|---|---|
| `virtual void Activate()` | `UFUNCTION(BlueprintCallable, meta=(BlueprintInternalUseOnly="true"))`; base body is empty |
| `virtual void RegisterWithGameInstance(const UObject* WorldContextObject)` | Resolves the world, then registers with its game instance |
| `virtual void RegisterWithGameInstance(UGameInstance* GameInstance)` | Protected overload when the game instance is already known |
| `virtual void SetReadyToDestroy()` | Clears `RF_StrongRefOnFrame` and unregisters from the game instance |
| `virtual void BeginDestroy()` | Base override; call `Super::BeginDestroy()` from any override |

The constructor sets `RF_StrongRefOnFrame` on every non-CDO instance, which is what keeps an unregistered action alive for exactly one frame. `bp.MaxAsyncActionCount` (default 10000, `Private/Kismet/BlueprintAsyncActionBase.cpp`) ensures when too many actions are alive at once — a stuck action that never calls `SetReadyToDestroy()` shows up there first.

`meta=(BlueprintInternalUseOnly="true")` on the factory is what makes Blueprint offer only the proxy node. `UCLASS(meta=(HasDedicatedAsyncNode))` suppresses the generic `UK2Node_AsyncAction` node when a custom K2 node exists, and `UCLASS(meta=(ExposedAsyncProxy=AsyncAction))` adds an output pin carrying the proxy object.

### Cancellable Async Actions

```cpp
// MyTimedAction.h
#pragma once
#include "CoreMinimal.h"
#include "Engine/CancellableAsyncAction.h"
#include "Engine/TimerHandle.h"
#include "MyTimedAction.generated.h"

DECLARE_DYNAMIC_MULTICAST_DELEGATE(FMyTimedActionSignature);

UCLASS()
class MYGAME_API UMyTimedAction : public UCancellableAsyncAction
{
    GENERATED_BODY()
public:
    UFUNCTION(BlueprintCallable, Category="MyGame|Async", meta=(BlueprintInternalUseOnly="true", WorldContext="WorldContextObject", DisplayName="Repeat Every"))
    static UMyTimedAction* RepeatEvery(UObject* WorldContextObject, float Interval);

    virtual void Activate() override;
    virtual void Cancel() override;

    UPROPERTY(BlueprintAssignable)
    FMyTimedActionSignature OnTick;

private:
    void HandleTimer();

    FTimerHandle TimerHandle;
    float IntervalSeconds = 1.f;
};
```

```cpp
// MyTimedAction.cpp
#include "MyTimedAction.h"
#include "Engine/GameInstance.h"
#include "TimerManager.h"

UMyTimedAction* UMyTimedAction::RepeatEvery(UObject* WorldContextObject, float Interval)
{
    UMyTimedAction* Action = NewObject<UMyTimedAction>();
    Action->IntervalSeconds = Interval;
    Action->RegisterWithGameInstance(WorldContextObject);
    return Action;
}

void UMyTimedAction::Activate()
{
    if (FTimerManager* TimerManager = GetTimerManager())
    {
        TimerManager->SetTimer(TimerHandle, this, &UMyTimedAction::HandleTimer, IntervalSeconds, true);
    }
}

void UMyTimedAction::Cancel()
{
    if (FTimerManager* TimerManager = GetTimerManager())
    {
        TimerManager->ClearTimer(TimerHandle);
    }
    Super::Cancel();          // the base implementation calls SetReadyToDestroy()
}

void UMyTimedAction::HandleTimer()
{
    if (ShouldBroadcastDelegates())
    {
        OnTick.Broadcast();
    }
}
```

`UCancellableAsyncAction` (`Engine/CancellableAsyncAction.h`) adds:

| Member | Behaviour |
|---|---|
| `virtual void Cancel()` | `UFUNCTION(BlueprintCallable, Category="Async Action")`; the base body calls `SetReadyToDestroy()` |
| `virtual bool IsActive() const` | `UFUNCTION(BlueprintCallable)`; returns `ShouldBroadcastDelegates()` |
| `virtual bool ShouldBroadcastDelegates() const` | Returns `IsRegistered()` — false once `SetReadyToDestroy()` has run |
| `bool IsRegistered() const` | True while the action is registered with a valid game instance |
| `FTimerManager* GetTimerManager() const` | Timer manager of the registered game instance, or null |
| `virtual void BeginDestroy()` | Calls `Cancel()` first, so a collected action always cleans up |

Because `BeginDestroy` calls `Cancel`, a `Cancel` override must be safe to run twice and safe to run during garbage collection: clear handles and unbind delegates, do not spawn or broadcast.

Always guard a broadcast with `ShouldBroadcastDelegates()`. After `Cancel()` or `SetReadyToDestroy()` the action is unregistered, so broadcasting would drive a graph the designer already stopped.
