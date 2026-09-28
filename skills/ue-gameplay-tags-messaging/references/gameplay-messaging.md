# Decoupled Gameplay Messaging

How one gameplay system tells another that something happened, without a hard reference in either
direction. Target engine: UE 5.8.

## Choosing a mechanism

| Mechanism | Ships in | Addressing | Threading | Payload |
|---|---|---|---|---|
| Multicast delegate on a subsystem | Core engine | Direct — listeners hold the subsystem | Game thread, synchronous | Delegate parameters |
| `AsyncMessageSystem` plugin (Experimental in 5.8) | `Engine/Plugins/Experimental/AsyncMessageSystem` | `FAsyncMessageId`, a gameplay tag, with parent fan-out | Tick group, named thread or task priority; deferred | `FInstancedStruct` |
| GAS gameplay events | `GameplayAbilities` plugin | `FGameplayTag` on an ability system component | Game thread | `FGameplayEventData` |

Start with the delegate. Move to `AsyncMessageSystem` when you need tag hierarchy, off-thread
listeners or a payload type the sender does not want to expose. Use GAS events when both ends already
have an `UAbilitySystemComponent` — that path belongs to `ue-gameplay-abilities`
(`UAbilitySystemBlueprintLibrary::SendGameplayEventToActor`,
`UAbilitySystemComponent::RegisterGameplayTagEvent`, `UAbilityTask_WaitGameplayEvent`).

## Option 1 — a delegate on a subsystem

Enough for most games, costs nothing, and the Blueprint side gets a red event node for free.

```cpp
// MyScoreSubsystem.h
#pragma once

#include "Subsystems/WorldSubsystem.h"
#include "MyScoreSubsystem.generated.h"

DECLARE_DYNAMIC_MULTICAST_DELEGATE_TwoParams(FMyScoreChanged, AActor*, Scorer, int32, NewScore);

UCLASS()
class MYGAME_API UMyScoreSubsystem : public UWorldSubsystem
{
    GENERATED_BODY()

public:
    UPROPERTY(BlueprintAssignable, Category = "Score")
    FMyScoreChanged OnScoreChanged;

    UFUNCTION(BlueprintCallable, Category = "Score")
    void ReportScore(AActor* Scorer, int32 NewScore);
};
```

```cpp
// MyScoreSubsystem.cpp
#include "MyScoreSubsystem.h"

void UMyScoreSubsystem::ReportScore(AActor* Scorer, int32 NewScore)
{
    OnScoreChanged.Broadcast(Scorer, NewScore);
}
```

Listeners do `GetWorld()->GetSubsystem<UMyScoreSubsystem>()->OnScoreChanged.AddDynamic(this, &AMyHud::HandleScoreChanged)`
and `RemoveDynamic` on teardown. Use a non-dynamic `TMulticastDelegate` if no Blueprint needs to bind.

Limits, and why the next option exists: every listener must know the subsystem type, the broadcast is
synchronous on the calling thread, and there is no hierarchy — a listener cannot say "any score event".

## Option 2 — the AsyncMessageSystem plugin

Experimental in 5.8 (`"IsExperimentalVersion": true` in `AsyncMessageSystem.uplugin`), disabled by
default. Enable it in the `.uproject` and add the module:

```csharp
PublicDependencyModuleNames.AddRange(new string[] { "GameplayTags", "AsyncMessageSystem" });
```

The plugin's own dependencies are `Core`, `CoreUObject`, `DeveloperSettings`, `Engine`, `GameplayTags`,
and its module loads at `PreDefault`.

### The pieces

| Type | Header | Role |
|---|---|---|
| `FAsyncMessageSystemBase` | `AsyncMessageSystemBase.h` | Abstract system: bind, unbind, queue, process |
| `FAsyncGameplayMessageSystem` | `AsyncGameplayMessageSystem.h` | Concrete system driven by tick groups and task-graph threads |
| `UAsyncMessageWorldSubsystem` | `AsyncMessageWorldSubsystem.h` | One message system per `UWorld` |
| `FAsyncMessageId` | `AsyncMessageId.h` | Message identity — internally an `FGameplayTag` |
| `FAsyncMessageHandle` | `AsyncMessageHandle.h` | Identifies one bound listener |
| `FAsyncMessage` | `AsyncMessage.h` | One queued message plus its `FInstancedStruct` payload copy |
| `FAsyncMessageBindingOptions` | `AsyncMessageBindingOptions.h` | Where and when a listener is called |
| `FAsyncMessageBindingEndpoint` | `AsyncMessageBindingEndpoint.h` | A private channel; listeners only hear their own endpoint |
| `UAsyncMessageBindingComponent` | `AsyncMessageBindingComponent.h` | Actor component that owns an endpoint |
| `FAsyncMessageStore` | `AsyncMessageStore.h` | Per-binding message queues |

### Verbatim signatures

```cpp
// UAsyncMessageWorldSubsystem
template<class TMessageSystemType = FAsyncMessageSystemBase>
static TSharedPtr<TMessageSystemType> GetSharedMessageSystem(const UWorld* InWorld);

template<class TMessageSystemType = FAsyncMessageSystemBase>
TSharedPtr<TMessageSystemType> GetSharedMessageSystem() const;

TMulticastDelegate<void()> OnShutdownMessageSystem;
```

```cpp
// FAsyncMessageSystemBase
using FMessageCallbackFunc = TFunction<void(const FAsyncMessage&)>;

FAsyncMessageHandle BindListener(
    const FAsyncMessageId MessageId,
    FMessageCallbackFunc&& Callback,
    const FAsyncMessageBindingOptions& Options = {},
    TWeakPtr<FAsyncMessageBindingEndpoint> BindingEndpoint = nullptr);

template <typename TOwner = UObject>
FAsyncMessageHandle BindListener(
    const FAsyncMessageId MessageId,
    TWeakObjectPtr<TOwner> WeakOwnerObject,
    void(TOwner::* Callback)(const FAsyncMessage&),
    const FAsyncMessageBindingOptions& Options = {},
    TWeakPtr<FAsyncMessageBindingEndpoint> BindingEndpoint = nullptr);

template <typename TOwner>
FAsyncMessageHandle BindListener(
    const FAsyncMessageId MessageId,
    TWeakPtr<TOwner> WeakObject,
    void(TOwner::* Callback)(const FAsyncMessage&),
    const FAsyncMessageBindingOptions& Options = {},
    TWeakPtr<FAsyncMessageBindingEndpoint> BindingEndpoint = nullptr);

void UnbindListener(const FAsyncMessageHandle& HandleToUnbind);

bool QueueMessageForBroadcast(
    const FAsyncMessageId MessageId,
    FConstStructView PayloadData = nullptr,
    TWeakPtr<FAsyncMessageBindingEndpoint> BindingEndpoint = nullptr);

void ProcessMessageQueueForBinding(const FAsyncMessageBindingOptions& Options);
void Shutdown();
```

```cpp
// FAsyncMessageId
FAsyncMessageId(const FName MessageName);
FAsyncMessageId(const FGameplayTag& MessageTag);
bool IsValid() const;
FName GetMessageName() const;
FString ToString() const;
FAsyncMessageId GetParentMessageId() const;
static const FAsyncMessageId Invalid;
static void WalkMessageHierarchy(const FAsyncMessageId StartingMessage,
    TFunctionRef<void(const FAsyncMessageId MessageId)> ForEachMessageFunc);
```

```cpp
// FAsyncMessage
FAsyncMessageId GetMessageId() const;       // the message this listener bound to
FAsyncMessageId GetMessageSourceId() const; // the childmost message that actually fired
double GetQueueTimestamp() const;
uint64 GetQueueFrame() const;
uint32 GetThreadQueuedFromThreadId() const;
uint32 GetSequenceId() const;
FStructView GetPayloadView();
FConstStructView GetPayloadView() const;
template<typename TDataType> TDataType* GetPayloadData() const;
TSharedPtr<FAsyncMessageBindingEndpoint> GetBindingEndpoint() const;
```

```cpp
// FAsyncMessageHandle
bool IsValid() const;
uint32 GetId() const;
FAsyncMessageId GetBoundMessageId() const;
FString ToString() const;
TSharedPtr<FAsyncMessageBindingEndpoint> GetBindingEndpoint() const;
static const FAsyncMessageHandle Invalid;
```

```cpp
// FAsyncMessageBindingOptions
FAsyncMessageBindingOptions();                                   // defaults to UseTickGroup
FAsyncMessageBindingOptions(const ETickingGroup DesiredTickGroup);
FAsyncMessageBindingOptions(const ENamedThreads::Type NamedThreads);
FAsyncMessageBindingOptions(const UE::Tasks::ETaskPriority InTaskPriority,
    const UE::Tasks::EExtendedTaskPriority InExtendedTaskPriority);

EBindingType GetType() const;                 // UseTickGroup | UseNamedThreads | UseTaskPriorities
void SetTickGroup(const ETickingGroup DesiredTickGroup);
ETickingGroup GetTickGroup() const;
void SetNamedThreads(const ENamedThreads::Type NamedThreads);
ENamedThreads::Type GetNamedThreads() const;
void SetTaskPriorities(const UE::Tasks::ETaskPriority InTaskPriority,
    const UE::Tasks::EExtendedTaskPriority InExtendedTaskPriority = UE::Tasks::EExtendedTaskPriority::None);
UE::Tasks::ETaskPriority GetTaskPriority() const;
UE::Tasks::EExtendedTaskPriority GetExtendedTaskPriority() const;
```

`FAsyncGameplayMessageSystem` supports tick groups from `TG_PrePhysics` to `TG_PostUpdateWork`.

### Full recipe

Message ids are gameplay tags, so declare them with the native tag macros:

```cpp
// MyMessageTags.h
#pragma once

#include "NativeGameplayTags.h"

MYGAME_API UE_DECLARE_GAMEPLAY_TAG_EXTERN(TAG_MyGame_Message_ScoreChanged);
```

```cpp
// MyMessageTags.cpp
#include "MyMessageTags.h"

UE_DEFINE_GAMEPLAY_TAG_COMMENT(TAG_MyGame_Message_ScoreChanged, "MyGame.Message.ScoreChanged", "Score changed broadcast");
```

The payload is any `USTRUCT`:

```cpp
// MyScorePayload.h
#pragma once

#include "StructUtils/InstancedStruct.h"
#include "MyScorePayload.generated.h"

class AActor;

USTRUCT()
struct FMyScorePayload
{
    GENERATED_BODY()

    UPROPERTY()
    TObjectPtr<AActor> Scorer = nullptr;

    UPROPERTY()
    int32 NewScore = 0;
};
```

Listener — bind in `BeginPlay`, unbind in `EndPlay`:

```cpp
// MyScoreWidgetActor.h
#pragma once

#include "GameFramework/Actor.h"
#include "AsyncMessageHandle.h"
#include "MyScoreWidgetActor.generated.h"

struct FAsyncMessage;

UCLASS()
class MYGAME_API AMyScoreWidgetActor : public AActor
{
    GENERATED_BODY()

public:
    virtual void BeginPlay() override;
    virtual void EndPlay(const EEndPlayReason::Type EndPlayReason) override;

private:
    void HandleScoreChanged(const FAsyncMessage& Message);

    FAsyncMessageHandle ScoreHandle;
};
```

```cpp
// MyScoreWidgetActor.cpp
#include "MyScoreWidgetActor.h"
#include "AsyncMessageSystemBase.h"
#include "AsyncMessageWorldSubsystem.h"
#include "MyMessageTags.h"
#include "MyScorePayload.h"

void AMyScoreWidgetActor::BeginPlay()
{
    Super::BeginPlay();

    TSharedPtr<FAsyncMessageSystemBase> Sys = UAsyncMessageWorldSubsystem::GetSharedMessageSystem(GetWorld());
    if (Sys.IsValid())
    {
        const FAsyncMessageBindingOptions Options(TG_PostUpdateWork);
        ScoreHandle = Sys->BindListener(
            FAsyncMessageId(TAG_MyGame_Message_ScoreChanged.GetTag()),
            TWeakObjectPtr<AMyScoreWidgetActor>(this),
            &AMyScoreWidgetActor::HandleScoreChanged,
            Options);
    }
}

void AMyScoreWidgetActor::EndPlay(const EEndPlayReason::Type EndPlayReason)
{
    TSharedPtr<FAsyncMessageSystemBase> Sys = UAsyncMessageWorldSubsystem::GetSharedMessageSystem(GetWorld());
    if (Sys.IsValid())
    {
        Sys->UnbindListener(ScoreHandle);
    }

    Super::EndPlay(EndPlayReason);
}

void AMyScoreWidgetActor::HandleScoreChanged(const FAsyncMessage& Message)
{
    if (const FMyScorePayload* Payload = Message.GetPayloadData<const FMyScorePayload>())
    {
        UE_LOG(LogMyGame, Log, TEXT("Score %d from %s"), Payload->NewScore, *Message.GetMessageSourceId().ToString());
    }
}
```

Sender:

```cpp
// MyScoreComponent.cpp
#include "MyScoreComponent.h"
#include "AsyncMessageSystemBase.h"
#include "AsyncMessageWorldSubsystem.h"
#include "MyMessageTags.h"
#include "MyScorePayload.h"

void UMyScoreComponent::BroadcastScore(AActor* Scorer, int32 NewScore)
{
    TSharedPtr<FAsyncMessageSystemBase> Sys = UAsyncMessageWorldSubsystem::GetSharedMessageSystem(GetWorld());
    if (!Sys.IsValid())
    {
        return;
    }

    FMyScorePayload Payload;
    Payload.Scorer = Scorer;
    Payload.NewScore = NewScore;

    Sys->QueueMessageForBroadcast(
        FAsyncMessageId(TAG_MyGame_Message_ScoreChanged.GetTag()),
        FInstancedStruct::Make<FMyScorePayload>(Payload));
}
```

### Behaviour worth knowing

- **The payload is copied.** `QueueMessageForBroadcast` copies the struct into the message so listeners
  on other threads can read it safely. Keep payloads small; for expensive data, put a pointer in the
  payload and synchronise it yourself.
- **Delivery is deferred.** Messages sit in `FAsyncMessageStore` until the system processes the queue
  for that binding — the next matching tick group, or an async task. Never assume a listener has run by
  the time `QueueMessageForBroadcast` returns.
- **It returns false when nobody is listening.** `QueueMessageForBroadcast` returns true only if the
  message had listeners and was queued.
- **Parent fan-out.** A listener bound to `MyGame.Message` also receives `MyGame.Message.ScoreChanged`.
  `GetMessageId()` is what you bound to; `GetMessageSourceId()` is the childmost message that fired.
  `FAsyncMessageId::WalkMessageHierarchy` walks that chain explicitly.
- **Binding and unbinding are deferred too.** New listeners go into a pending queue so a callback can
  bind another listener without mutating the array being iterated.
- **Weak ownership is handled.** The `TWeakObjectPtr` and `TWeakPtr` overloads unbind themselves when
  the owner dies, so a missed `UnbindListener` leaks a handle, not a crash. Unbind anyway.
- **Same-frame ordering.** `GetSequenceId()` distinguishes several instances of the same message queued
  in one frame; `GetQueueFrame()` and `GetQueueTimestamp()` record when.
- **Endpoints isolate traffic.** Pass a `TWeakPtr<FAsyncMessageBindingEndpoint>` to both `BindListener`
  and `QueueMessageForBroadcast` and only listeners on that endpoint hear it. Add a
  `UAsyncMessageBindingComponent` to an actor to get a per-actor endpoint; it creates the endpoint in
  `InitializeComponent` and destroys it in `UninitializeComponent`, and exposes it through
  `IAsyncMessageBindingEndpointInterface::GetEndpoint()`. The stock component never sets
  `bWantsInitializeComponent`, and `AActor::InitializeComponents` skips components without it
  (`Actor.cpp:6404`), so as shipped its endpoint stays null: subclass it and set
  `bWantsInitializeComponent = true` in the constructor. Omit the endpoint (or pass a null one) and
  the world's default one is used.
- **Lifetime.** `UAsyncMessageWorldSubsystem::OnShutdownMessageSystem` fires when the world's system is
  torn down — bind it if you cache the shared pointer anywhere.
- **Debugging.** `ENABLE_ASYNC_MESSAGES_DEBUG` (on by default outside Shipping) enables recording the
  callstack at queue time on `FAsyncMessage`. Force it on in Shipping with
  `PublicDefinitions.Add("ENABLE_ASYNC_MESSAGES_DEBUG=1");` in your `Build.cs`.
- **Do not build your own system directly.** Use `UAsyncMessageWorldSubsystem::GetSharedMessageSystem`.
  If you genuinely need a separate one, `FAsyncMessageSystemBase::CreateMessageSystem<T>(Args...)`
  constructs and starts it, and you must call `Shutdown()` before it is destroyed.

## Not in the engine: Lyra's message router

`UGameplayMessageSubsystem` and its listener handle and match-type enum are Lyra sample code
(`LyraGame/GameplayMessageRuntime`), not engine code. No such header exists under `Engine/Source` or
`Engine/Plugins`, so `#include "GameFramework/GameplayMessageSubsystem.h"` fails to compile in a clean
project, and the Blueprint "Listen For Gameplay Messages" async node is part of that same sample.
Either copy those files into your own module from the Lyra sample, or use `AsyncMessageSystem` — it is
the engine-side answer to the same problem, with the same tag-addressed, hierarchical model.

## Messaging and the network

None of these mechanisms replicate. A message is local to the process that queued it. To cross the
wire, replicate a property or call an RPC, and broadcast the message from the `OnRep` or the RPC body
on each machine — see `ue-networking-replication`.
