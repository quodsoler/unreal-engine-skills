---
name: ue-networking-replication
description: "Use when writing multiplayer C++: replicating a property, choosing an RPC type, tuning relevancy or dormancy, registering replicated subobjects, or turning on push model or Iris. Also use when the user mentions 'replicate a variable', 'OnRep', 'DOREPLIFETIME', 'GetLifetimeReplicatedProps', 'Server RPC', 'NetMulticast', 'WithValidation', 'COND_OwnerOnly', 'HasAuthority', 'ENetRole', 'FFastArraySerializer', 'MARK_PROPERTY_DIRTY_FROM_NAME', 'net dormancy', 'replication graph', 'dedicated server' or 'listen server'. For GAS prediction and attribute replication, see ue-gameplay-abilities; for movement replication and CMC prediction, see ue-character-movement."
metadata:
  version: "2.0.0"
  engine: "5.8"
---

# UE Networking & Replication

Target engine: **UE 5.8**. APIs below are verified against the 5.8 headers; older forms are listed under "Deprecated — do not use".

Server-authoritative replication for actors, components and subobjects. Core APIs live in the `Engine` module (`GameFramework/Actor.h`, `Engine/NetDriver.h`, `Net/UnrealNetwork.h`) and the `NetCore` module (`Net/Core/PushModel/PushModel.h`, `Net/Serialization/FastArraySerializer.h`). `ELifetimeCondition` and `REPNOTIFY_*` come from `CoreUObject` (`UObject/CoreNetTypes.h`). Optional systems: the Iris replication system (`IrisCore` module, `SetupIrisSupport(Target)` in Build.cs) and the ReplicationGraph plugin (Beta in 5.8, Build.cs module `ReplicationGraph`).

## Context

Read `.agents/ue-project-context.md` if it exists (module names, conventions, enabled plugins, GAS/networking setup). Do not stop if it is missing.

Identify the area from the request and the codebase. Ask only when two plausible readings would produce different code.

| Request is about… | Go to |
|---|---|
| Authority checks, roles, dedicated vs listen server | [Net Roles, Net Modes and Authority](#net-roles-net-modes-and-authority) |
| Replicating a variable, `OnRep_`, `GetLifetimeReplicatedProps` | [Property Replication](#property-replication) |
| Who receives a property (`COND_*`) | [Lifetime Conditions](#lifetime-conditions) |
| Cutting per-frame comparison cost | [Push Model](#push-model) |
| Replicated arrays / inventories | [Fast Array Serialization](#fast-array-serialization) |
| Server/Client/Multicast calls, validation | [Remote Procedure Calls](#remote-procedure-calls) |
| `SetOwner`, relevancy, culling, dormancy | [Ownership, Relevancy and Dormancy](#ownership-relevancy-and-dormancy) |
| Replicating `UObject`s owned by an actor or component | [Subobject Replication](#subobject-replication) |
| Simulated bodies, physics prediction | [Physics Replication](#physics-replication) |
| Iris, `net.Iris.UseIrisReplication` | [Iris Replication System](#iris-replication-system) |
| 50+ connections, spatial relevancy | [Replication Graph](#replication-graph) |
| `UNetDriver`, `UNetConnection`, disconnect detection | [Net Driver and Connections](#net-driver-and-connections) |

Worked end-to-end examples: [replication patterns](references/replication-patterns.md). RPC selection tables and verbatim engine RPC declarations: [RPC decision guide](references/rpc-decision-guide.md).

## Net Roles, Net Modes and Authority

`ENetRole` is declared in `Engine/Classes/Engine/EngineTypes.h`; the accessors are on `AActor`.

```cpp
ROLE_None            // not replicated
ROLE_SimulatedProxy  // remote copy driven by replicated state
ROLE_AutonomousProxy // remote copy of the locally controlled pawn/controller
ROLE_Authority       // owns and may mutate this actor
```

From `GameFramework/Actor.h`:

```cpp
ENetRole GetLocalRole() const;
ENetRole GetRemoteRole() const;
bool HasAuthority() const;
ENetMode GetNetMode() const;
bool IsNetMode(ENetMode Mode) const;
```

`ENetMode` (`Engine/Classes/Engine/EngineBaseTypes.h`): `NM_Standalone`, `NM_DedicatedServer`, `NM_ListenServer`, `NM_Client`. Every mode below `NM_Client` is a server.

| Machine | `GetLocalRole()` | `GetRemoteRole()` |
|---|---|---|
| Server | `ROLE_Authority` | `ROLE_AutonomousProxy` or `ROLE_SimulatedProxy` |
| Owning client | `ROLE_AutonomousProxy` | `ROLE_Authority` |
| Other clients | `ROLE_SimulatedProxy` | `ROLE_Authority` |

On a listen server the host is `ROLE_Authority` *and* locally controlled: use `APawn::IsLocallyControlled()` to skip host-only or client-only paths.

## Property Replication

```cpp
// MyReplicatedActor.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/Actor.h"
#include "MyReplicatedActor.generated.h"

UCLASS()
class MYGAME_API AMyReplicatedActor : public AActor
{
    GENERATED_BODY()

public:
    AMyReplicatedActor();

    virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override;

protected:
    UPROPERTY(ReplicatedUsing = OnRep_Health)
    float Health = 100.f;

    UPROPERTY(Replicated)
    int32 TeamId = 0;

    UFUNCTION()
    void OnRep_Health(float PreviousHealth);
};
```

```cpp
// MyReplicatedActor.cpp
#include "MyReplicatedActor.h"
#include "Net/UnrealNetwork.h"

AMyReplicatedActor::AMyReplicatedActor()
{
    bReplicates = true;                     // or SetReplicates(true) at runtime
    SetReplicateMovement(true);
    SetNetUpdateFrequency(10.f);            // replication considerations per second
    SetMinNetUpdateFrequency(2.f);          // throttled floor when nothing changes
    SetNetCullDistanceSquared(225000000.f); // 150 m squared
    NetPriority = 1.f;                      // still a public field in 5.8
}

void AMyReplicatedActor::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps); // never omit
    DOREPLIFETIME(AMyReplicatedActor, Health);
    DOREPLIFETIME_CONDITION(AMyReplicatedActor, TeamId, COND_InitialOnly);
}

void AMyReplicatedActor::OnRep_Health(float PreviousHealth)
{
    // Clients only. Must be idempotent: relevancy changes re-fire it.
}
```

Rules:

- `OnRep_` must be a `UFUNCTION()`. Its optional single parameter is the **previous** value and must match the property type.
- Only the authority may write a replicated property. `OnRep_` does not fire on the machine that made the change.
- `UPROPERTY(NotReplicated)` inside a replicated struct excludes a member.
- `FVector_NetQuantize` / `_NetQuantize10` / `_NetQuantize100` / `_NetQuantizeNormal` (`Engine/NetSerialization.h`) cut world-position bandwidth: 0, 1 and 2 decimal places, and 16 bits per component for normals.
- `FRepMovement` (`Engine/Classes/Engine/ReplicatedState.h`) carries location/rotation/velocity when `SetReplicateMovement(true)`; override `virtual void OnRep_ReplicatedMovement()` to post-process it. `UCharacterMovementComponent` replicates moves itself — see `ue-character-movement`.
- On the first bunch for a connection, every eligible property is sent at once. Keep initial state small and push spawn-only data behind `COND_InitialOnly`.

Macros (`Net/UnrealNetwork.h`): `DOREPLIFETIME`, `DOREPLIFETIME_CONDITION`, `DOREPLIFETIME_CONDITION_NOTIFY`, `DOREPLIFETIME_WITH_PARAMS`, `DOREPLIFETIME_WITH_PARAMS_FAST`, `DOREPLIFETIME_WITH_PARAMS_FAST_STATIC_ARRAY`, `DOREPLIFETIME_ACTIVE_OVERRIDE`, `DOREPLIFETIME_ACTIVE_OVERRIDE_FAST`, `DISABLE_REPLICATED_PROPERTY`.

`FDoRepLifetimeParams` (`Net/UnrealNetwork.h`) is the struct behind the `_WITH_PARAMS` forms:

```cpp
ELifetimeCondition Condition = COND_None;
ELifetimeRepNotifyCondition RepNotifyCondition = REPNOTIFY_OnChanged; // or REPNOTIFY_Always
bool bIsPushBased = false;
```

## Lifetime Conditions

Full `ELifetimeCondition` enum, `CoreUObject/Public/UObject/CoreNetTypes.h`:

| Value | Sends to |
|---|---|
| `COND_None` | every connection |
| `COND_InitialOnly` | initial bunch only |
| `COND_OwnerOnly` | the actor's owning connection |
| `COND_SkipOwner` | everyone except the owner |
| `COND_SimulatedOnly` | connections where the actor is a simulated proxy |
| `COND_AutonomousOnly` | connections where the actor is an autonomous proxy |
| `COND_SimulatedOrPhysics` | simulated proxies or `bRepPhysics` actors |
| `COND_InitialOrOwner` | initial bunch, or the owner |
| `COND_Custom` | always eligible; toggled at runtime via the active-override macros |
| `COND_ReplayOrOwner` | replay connection or the owner |
| `COND_ReplayOnly` | replay connection only |
| `COND_SimulatedOnlyNoReplay` | simulated proxies, never replay |
| `COND_SimulatedOrPhysicsNoReplay` | simulated or `bRepPhysics`, never replay |
| `COND_SkipReplay` | everything except the replay connection |
| `COND_Dynamic` | condition is overridden at runtime; replicates until you set one |
| `COND_Never` | never replicated |
| `COND_NetGroup` | **subobjects only** — connections in the same net condition group |

`COND_NetGroup` pairs with `UE::Net::FNetConditionGroupManager` (`Net/Core/Misc/NetConditionGroupManager.h`, module `NetCore`), reached through `UNetworkSubsystem::GetNetConditionGroupManager()` (`Net/Subsystems/NetworkSubsystem.h`):

```cpp
#include "Net/Core/Misc/NetConditionGroupManager.h"
#include "Net/Subsystems/NetworkSubsystem.h"

AddReplicatedSubObject(MySubObject, COND_NetGroup); // on the owning actor/component
UNetworkSubsystem* NetSubsystem = GetWorld()->GetSubsystem<UNetworkSubsystem>();
NetSubsystem->GetNetConditionGroupManager().RegisterSubObjectInGroup(MySubObject, FName("RedTeam"));
MyPlayerController->IncludeInNetConditionGroup(FName("RedTeam"));
```

`UE::Net::NetGroupOwner` and `UE::Net::NetGroupReplay` are engine-managed groups; do not add a player controller to them. `APlayerController` also exposes `RemoveFromNetConditionGroup`, `IsMemberOfNetConditionGroup` and `GetNetConditionGroups`.

## Push Model

Push model removes the per-frame memcmp of replicated properties: the property is only compared when you mark it dirty. It works with the **classic** net driver and with Iris — it is not an Iris feature.

```cpp
// MyPushActor.cpp
#include "MyPushActor.h"
#include "Net/UnrealNetwork.h"
#include "Net/Core/PushModel/PushModel.h"

void AMyPushActor::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    FDoRepLifetimeParams Params;
    Params.bIsPushBased = true;
    Params.Condition = COND_OwnerOnly;
    DOREPLIFETIME_WITH_PARAMS_FAST(AMyPushActor, Ammo, Params);
}

void AMyPushActor::ConsumeAmmo(int32 Count)
{
    if (!HasAuthority())
    {
        return;
    }
    Ammo -= Count;
    MARK_PROPERTY_DIRTY_FROM_NAME(AMyPushActor, Ammo, this);
}
```

- `MARK_PROPERTY_DIRTY_FROM_NAME(ClassName, PropertyName, Object)` is the preferred marker. Static arrays use `MARK_PROPERTY_DIRTY_FROM_NAME_STATIC_ARRAY_INDEX(ClassName, PropertyName, ArrayIndex, Object)` or `MARK_PROPERTY_DIRTY_FROM_NAME_STATIC_ARRAY(ClassName, PropertyName, Object)`.
- Opt in twice: `bWithPushModel = true;` in the game/server `*.Target.cs` (defaults to true only for Editor targets, `TargetRules.cs:1524`) and `Net.IsPushModelEnabled=1` (defaults to false, `PushModel.cpp:434`). Until both are on, properties are compared as usual.
- The macros compile to nothing when `WITH_PUSH_MODEL` is off, so the code is always safe to ship (exception: the no-push `MARK_PROPERTY_DIRTY_FROM_NAME_STATIC_ARRAY` stub takes 4 arguments, `PushModel.h:489`, so the 3-argument form fails to compile there).
- A struct property mutated by reference needs an accessor that marks it dirty before handing out the reference.
- With push model enabled, forgetting one marker means the property silently stops replicating — this is the main cost of opting in.

## Fast Array Serialization

`FFastArraySerializer` sends only changed items instead of the whole array.

```cpp
#include "Net/Serialization/FastArraySerializer.h"   // module NetCore
```

Item struct derives from `FFastArraySerializerItem` and may implement any of:

```cpp
void PreReplicatedRemove(const struct FMyItemArray& InArraySerializer);
void PostReplicatedAdd(const struct FMyItemArray& InArraySerializer);
void PostReplicatedChange(const struct FMyItemArray& InArraySerializer);
```

The array struct derives from `FFastArraySerializer`, implements `NetDeltaSerialize` via `FastArrayDeltaSerialize<ItemType, ArrayType>(Items, DeltaParms, *this)`, and needs `TStructOpsTypeTraits<...> { WithNetDeltaSerializer = true }`. It may also implement the array-level receive hook:

```cpp
void PostReplicatedReceive(const FFastArraySerializer::FPostReplicatedReceiveParameters& Parameters);
```

`FPostReplicatedReceiveParameters::OldArraySize` is the size before the update, which makes this the right place for "the whole list changed, rebuild the UI" work.

Dirty rules — both are mandatory, and both are common sources of "my array does not replicate":

- `MarkItemDirty(Item)` after adding or mutating an item.
- `MarkArrayDirty()` after **removing** an item (it also invalidates the ID map).

Full inventory implementation: [replication patterns, Pattern 3](references/replication-patterns.md).

## Remote Procedure Calls

`UFUNCTION` network specifiers (`UObject/ObjectMacros.h`): `Server`, `Client`, `NetMulticast`, `Reliable`, `Unreliable`, `WithValidation`, `BlueprintAuthorityOnly`, `BlueprintCosmetic`. `Reliable`/`Unreliable` are only valid together with `Server`, `Client` or `NetMulticast`.

```cpp
// Client calls, server executes. WithValidation generates the _Validate pair.
UFUNCTION(Server, Reliable, WithValidation)
void ServerFireWeapon(FVector_NetQuantize Origin, FVector_NetQuantizeNormal Direction);

// Server calls, the owning client executes.
UFUNCTION(Client, Reliable)
void ClientShowKillConfirm(int32 VictimId);

// Server calls, server and every client with this actor relevant execute.
UFUNCTION(NetMulticast, Unreliable)
void MulticastPlayHitEffect(FVector_NetQuantize ImpactPoint);
```

You write `_Implementation` bodies; with `WithValidation` you also write `_Validate`, and returning `false` disconnects the caller.

| Rule | Detail |
|---|---|
| Ownership | A `Server` RPC only routes if the calling client owns the actor through its `SetOwner` chain up to its `APlayerController`. Unowned actors drop the call; the only trace is a `LogNet` Warning "No owning connection for actor…" (`NetDriver.cpp:8274`). |
| Client RPC target | Needs an actor whose owning connection exists; `APlayerController` is the reliable choice. |
| Reliability | `Reliable` for state changes, `Unreliable` for high-frequency cosmetics. Do not send `Reliable` RPCs every tick: once a channel has more than `RELIABLE_BUFFER` (512, `NetConnection.h:82`) unacked reliable bunches the connection is closed with `ReliableBufferOverflow` (`DataChannel.cpp:1445`). |
| Late joiners | Multicasts are not replayed. Persistent state belongs in a replicated property. |
| Parameters | Net-addressable types only. A `UObject*` that is neither replicated nor stably named (asset, class, map-placed actor) arrives as `nullptr`; pass an id and resolve remotely. |
| Direction | `Client`/`NetMulticast` called on a client are never sent; they just run locally on that client (`Actor.cpp:5569-5573`, `5587-5592`). Guard call sites with `HasAuthority()`. |

Decision tables, reliability matrices and verbatim `PlayerController.h` declarations: [RPC decision guide](references/rpc-decision-guide.md).

## Ownership, Relevancy and Dormancy

```cpp
Weapon->SetOwner(OwningPlayerController);   // server only; drives RPC routing and COND_OwnerOnly
```

Relevancy fields and hooks on `AActor`:

```cpp
bAlwaysRelevant = true;       // ignore distance culling
bOnlyRelevantToOwner = true;  // owner connection only

virtual bool IsNetRelevantFor(const AActor* RealViewer, const AActor* ViewTarget, const FVector& SrcLocation) const override;
virtual float GetNetPriority(const FVector& ViewPos, const FVector& ViewDir, AActor* Viewer, AActor* ViewTarget, UActorChannel* InChannel, float Time, bool bLowBandwidth) override;
```

`SetNetCullDistanceSquared()` sets the squared radius; `NetPriority` breaks ties when bandwidth saturates; `GetReplayPriority()` does the same for replay recording.

Dormancy (`ENetDormancy` in `EngineTypes.h`: `DORM_Never`, `DORM_Awake`, `DORM_DormantAll`, `DORM_DormantPartial`, `DORM_Initial`):

```cpp
NetDormancy = DORM_Initial;        // constructor: map-placed actors start dormant
SetNetDormancy(DORM_DormantAll);   // runtime
FlushNetDormancy();                // push one update, then return to dormant
```

A waking dormant actor is diffed against the replicator the connection kept when it went dormant (`DataChannel.cpp:2510-2519`), so normally only changes are sent; an actor whose channel closed on relevancy gets a full initial bunch when it re-enters range, which fires its `OnRep_`s again. Write `OnRep_` handlers that tolerate repeats.

## Subobject Replication

`bReplicateUsingRegisteredSubObjectList` defaults to **false** in 5.8 (`GDefaultUseSubObjectReplicationList = false` in `Engine/Private/Components/ActorComponent.cpp`, project-wide CVar `net.SubObjects.DefaultUseSubObjectReplicationList`). Opt in per class, or nothing you register replicates:

```cpp
AMyQuestActor::AMyQuestActor()
{
    bReplicates = true;
    bReplicateUsingRegisteredSubObjectList = true;  // protected field; set in the constructor
}
```

`AActor` registration API (`GameFramework/Actor.h`); `UActorComponent` (`Components/ActorComponent.h`) mirrors all but the `ActorComponent` variants:

```cpp
void AddReplicatedSubObject(UObject* SubObject, ELifetimeCondition NetCondition = COND_None);
void RemoveReplicatedSubObject(UObject* SubObject);
void AddActorComponentReplicatedSubObject(UActorComponent* OwnerComponent, UObject* SubObject, ELifetimeCondition NetCondition = COND_None);
void RemoveActorComponentReplicatedSubObject(UActorComponent* OwnerComponent, UObject* SubObject);
void DestroyReplicatedSubObjectOnRemotePeers(UObject* SubObject);
void TearOffReplicatedSubObjectOnRemotePeers(UObject* SubObject);
bool IsUsingRegisteredSubObjectList() const;
```

`RemoveReplicatedSubObject` only stops future updates — the client keeps its copy. Use `DestroyReplicatedSubObjectOnRemotePeers` to delete it remotely, or `TearOffReplicatedSubObjectOnRemotePeers` to leave an orphaned local copy. The subobject must override `virtual bool IsSupportedForNetworking() const` to return `true` and declare its own `GetLifetimeReplicatedProps`.

When the flag is false, the engine instead calls the still-live virtual:

```cpp
virtual bool ReplicateSubobjects(class UActorChannel* Channel, class FOutBunch* Bunch, FReplicationFlags* RepFlags) override;
```

Inside it, forward to `Super::` and call `Channel->ReplicateSubobject(SubObject, *Bunch, *RepFlags)`. Registered subobjects: [replication patterns, Pattern 5](references/replication-patterns.md).

## Physics Replication

`EPhysicsReplicationMode` (`EngineTypes.h`) selects how a simulated body is corrected: `Default`, `PredictiveInterpolation`, `Resimulation`, `None`.

```cpp
SetPhysicsReplicationMode(EPhysicsReplicationMode::PredictiveInterpolation);
EPhysicsReplicationMode Mode = GetPhysicsReplicationMode();
```

`PredictiveInterpolation` and `Resimulation` require Project Settings → Physics → Physics Prediction. Keep every interacting body on the same mode. For rollback-style networked physics use `UNetworkPhysicsComponent` (`Engine/Public/Physics/NetworkPhysicsComponent.h`) and the `FAsyncPhysicsTimestamp` handshake on `APlayerController` (`ClientSetupNetworkPhysicsTimestamp`, `ServerSendLatestAsyncPhysicsTimestamp`); collision and body setup belong to `ue-physics-collision`.

## Iris Replication System

Iris is the alternative replication backend (plugin `Iris`, Beta flag in 5.8; Epic's 5.8 notes call it production-ready for licensees). Runtime module `IrisCore` under `Source/Runtime/Net/Iris`, engine glue under `Engine/Public/Net/Iris`.

- Enable in a module's Build.cs: `SetupIrisSupport(Target);`
- Enable the `Iris` plugin (`EnabledByDefault: false`) and set the runtime switch `net.Iris.UseIrisReplication=1`; it defaults to 0, the classic replication system (`IrisConfig.cpp:14`).
- Property declaration, RPCs, conditions and push model are unchanged. Iris changes the backend, not the gameplay-facing API.

Actor hooks when Iris is active (`GameFramework/Actor.h`):

```cpp
virtual void FillReplicationParams(const FFillReplicationParamsContext& Context, FActorReplicationParams& OutParams) override; // public
virtual void OnReplicationStartedForIris(const FOnReplicationStartedParams&) override;                                        // protected
virtual void OnStopReplicationForIris(const FOnStopReplicationParams&) override;                                              // protected
```

`UReplicationBridge` and `UObjectReplicationBridge` (`Iris/ReplicationSystem/`) are internal; gameplay code should not derive from them. `UReplicationBridge` no longer has the separate base class it used to; treat it as internal.

## Replication Graph

For high connection counts, replace per-connection relevancy scanning with the ReplicationGraph plugin (Beta in 5.8). Build.cs module: `ReplicationGraph`.

```ini
; DefaultEngine.ini
[/Script/OnlineSubsystemUtils.IpNetDriver]
ReplicationDriverClassName="/Script/MyGame.MyReplicationGraph"
```

```cpp
// MyReplicationGraph.h
#pragma once

#include "CoreMinimal.h"
#include "ReplicationGraph.h"
#include "MyReplicationGraph.generated.h"

UCLASS()
class MYGAME_API UMyReplicationGraph : public UReplicationGraph
{
    GENERATED_BODY()

public:
    virtual void InitGlobalActorClassSettings() override;
    virtual void InitGlobalGraphNodes() override;
    virtual void InitConnectionGraphNodes(UNetReplicationGraphConnection* ConnectionManager) override;
};
```

Node classes (`ReplicationGraph.h`): `UReplicationGraphNode_GridSpatialization2D` (world grid), `UReplicationGraphNode_AlwaysRelevant` (game state, managers), `UReplicationGraphNode_AlwaysRelevant_ForConnection` (controller, player state), `UReplicationGraphNode_ActorList` (hand-built lists), `UReplicationGraphNode_DormancyNode`, `UReplicationGraphNode_DynamicSpatialFrequency`.

## Net Driver and Connections

`UNetDriver` (`Engine/Classes/Engine/NetDriver.h`) owns the connections and drives replication for a world.

```cpp
if (UNetDriver* Driver = GetWorld()->GetNetDriver())
{
    // Driver->ClientConnections — server side, one per client
    // Driver->ServerConnection  — client side, the single upstream connection
}
```

`UNetConnection` (`Engine/Classes/Engine/NetConnection.h`) is one client's connection. Reach it with `AActor::GetNetConnection()` (overridden by `APlayerController`). Split-screen players get a `UChildConnection` (`Engine/ChildConnection.h`) hanging off the parent.

```cpp
UE::Net::FConnectionHandle Handle = Connection->GetConnectionHandle();
UNetConnection* Found = Driver->GetConnectionByHandle(Handle);
const bool bGone = Connection->IsClosingOrClosed();          // USOCK_Closing or USOCK_Closed
EConnectionState State = Connection->GetConnectionState();
```

Prefer `AGameModeBase::Logout` for disconnect handling over polling connection state.

`APlayerController` exists only on the server and its owning client. Data every client needs goes on `APawn`, `APlayerState` or `AGameStateBase` — see `ue-gameplay-framework`.

## Deprecated — do not use

| Do not emit | Use in 5.8 | Source |
|---|---|---|
| `NetUpdateFrequency = X;` | `SetNetUpdateFrequency(X)` / `GetNetUpdateFrequency()` | `UE_DEPRECATED(5.5)` in `GameFramework/Actor.h` |
| `MinNetUpdateFrequency = X;` | `SetMinNetUpdateFrequency(X)` / `GetMinNetUpdateFrequency()` | `UE_DEPRECATED(5.5)` in `GameFramework/Actor.h` |
| `NetCullDistanceSquared = X;` | `SetNetCullDistanceSquared(X)` / `GetNetCullDistanceSquared()` | `UE_DEPRECATED(5.5)` in `GameFramework/Actor.h` |
| `virtual void BeginReplication()` | `OnReplicationStartedForIris` or `FillReplicationParams` | `UE_DEPRECATED(5.7)` in `GameFramework/Actor.h` |
| `virtual void EndReplication(EEndPlayReason::Type)` | `OnStopReplicationForIris` | `UE_DEPRECATED(5.7)` in `GameFramework/Actor.h` |
| `BeginReplication(const FActorReplicationParams&)` | override `FillReplicationParams` | `UE_DEPRECATED(5.7)` in `GameFramework/Actor.h` |
| `UNetConnection::GetConnectionId()` | `GetConnectionHandle()` | `UE_DEPRECATED(5.6)` in `Engine/NetConnection.h` |
| `UNetConnection::SetConnectionId()` | nothing — external code must not set it (`SetConnectionHandle()` is protected, `NetConnection.h:2006`) | `UE_DEPRECATED(5.6)` in `Engine/NetConnection.h` |
| `UNetConnection::GetParentConnectionId()` | `GetConnectionHandle()` | `UE_DEPRECATED(5.6)` in `Engine/NetConnection.h` |
| `UNetDriver::GetConnectionById(uint32)` | `GetConnectionByHandle(FConnectionHandle)` | `UE_DEPRECATED(5.6)` in `Engine/NetDriver.h` |
| `UNetDriver::SetPendingDestruction()` | `RequestNetDriverDestruction()` | `UE_DEPRECATED(5.6)` in `Engine/NetDriver.h` |
| `FPostReplicatedReceiveParameters::bHasMoreUnmappedReferences` | read `OldArraySize` only | `UE_DEPRECATED(5.4)` in `Net/Serialization/FastArraySerializer.h` |
| `UObjectReplicationBridge::DetachInstanceFromRemote` | `DetachRootObjectFromRemote` / `DetachSubObjectFromRemote` | `UE_DEPRECATED(5.8)` in `Iris/ReplicationSystem/ObjectReplicationBridge.h` |
| `UNetObjectFactory::FDestroyedContext` | `FDetachContext` | `UE_DEPRECATED(5.8)` in `Iris/ReplicationSystem/NetObjectFactory.h` |

## Common Mistakes

**Assuming the registered subobject list is on:** it defaults to false, so `AddReplicatedSubObject` alone replicates nothing. Set `bReplicateUsingRegisteredSubObjectList = true` in the constructor (or flip `net.SubObjects.DefaultUseSubObjectReplicationList`) and confirm with `IsUsingRegisteredSubObjectList()`.

**Omitting `Super::GetLifetimeReplicatedProps`:** every inherited replicated property is dropped, including engine state on `AActor` and `APawn`. Call `Super` first, always.

**Mutating replicated state on a client:** the next server update overwrites it and no rollback happens. Guard with `if (!HasAuthority()) return;` and apply authoritative values in `OnRep_`.

**Push model without a marker:** a property declared with `Params.bIsPushBased = true` and never marked dirty simply stops replicating. Every write path needs `MARK_PROPERTY_DIRTY_FROM_NAME`.

**Removing from a fast array without `MarkArrayDirty()`:** removals do not reach clients. `MarkItemDirty` covers mutation only.

**Using `NetMulticast` for persistent state:** late joiners never receive it. Multicast is for one-shot cosmetics; state belongs in a replicated property.

**Server RPC on an unowned actor:** the call is dropped with only a `LogNet` Warning. Call `SetOwner()` on the server after spawning, and keep the chain intact up to the `APlayerController`.

**Non-`UFUNCTION` `OnRep_`:** UHT rejects `ReplicatedUsing` pointing at a plain member function. Declare `UFUNCTION()` above it.

**Server RPC without `WithValidation` that mutates state:** a client can send any parameter values. Validate in `_Validate`, or better, send an id and look up the authoritative data server-side.

**`OnRep_` that is not idempotent:** relevancy changes re-send the full state and re-fire callbacks. Do not treat `OnRep_` as an "exactly once" event.

## Related Skills

- `ue-cpp-foundations` — `UPROPERTY`/`UFUNCTION` specifier reference, module includes, reflection basics
- `ue-gameplay-framework` — `AGameModeBase` (server only), `AGameStateBase`/`APlayerState` replication scope, `APawn` possession
- `ue-gameplay-abilities` — GAS prediction keys, attribute set replication, gameplay effect replication modes
- `ue-character-movement` — CMC move replication, saved moves, server correction and network smoothing
- `ue-actor-component-architecture` — component lifecycle, `SetIsReplicated`, subobject ownership
- `ue-async-threading` — off-thread work feeding replicated state, task graph and async patterns
- `ue-serialization-savegames` — persisting state that outlives a session, `NetSerialize` vs `Serialize`
- `ue-gameplay-tags-messaging` — native and ini gameplay tags, containers, queries and the async message system
- `ue-mover` — the Mover plugin: movement modes, layered moves and rollback networking
- `ue-physics-collision` — collision channels, traces, overlaps and Chaos physics
- `ue-world-level-streaming` — World Partition, level streaming, data layers and travel
