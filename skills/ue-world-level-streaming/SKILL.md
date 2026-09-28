---
name: ue-world-level-streaming
description: "Use when streaming levels in and out, setting up World Partition, or moving between maps. Also use when the user mentions 'level streaming', 'World Partition', 'data layer', 'UDataLayerManager', 'SetDataLayerRuntimeState', 'LoadStreamLevel', 'UnloadStreamLevel', 'ULevelStreamingDynamic', 'LoadLevelInstance', 'ALevelInstance', 'streaming source', 'streaming volume', 'HLOD', 'OpenLevel', 'ServerTravel', 'seamless travel', 'sub-level', 'open world', or 'UWorldSubsystem'. For replication and net relevancy, see ue-networking-replication; for GameMode travel callbacks, see ue-gameplay-framework; for saving world state to disk, see ue-serialization-savegames."
metadata:
  version: "2.0.0"
  engine: "5.8"
---

# UE World & Level Streaming

Target engine: **UE 5.8**. APIs below are verified against the 5.8 headers; older forms are listed under "Deprecated — do not use".

This skill covers how a `UWorld` is composed and swapped at runtime: World Partition grids, data layers, HLOD, level instances, classic sub-level streaming, streaming volumes, and map travel. Everything here lives in the `Engine` module (`Engine/Source/Runtime/Engine`), so `Core`, `CoreUObject` and `Engine` in `PublicDependencyModuleNames` is all that is needed. The one exception is the Level Streaming Persistence plugin, which adds the `LevelStreamingPersistence` module.

For end-to-end configurations per game type (open world, hub-and-spoke, procedural, chapter-based, volume-driven, dedicated server), see [streaming patterns](references/streaming-patterns.md).

## Context

Read `.agents/ue-project-context.md` if it exists (module names, conventions, enabled plugins, GAS/networking setup). Do not stop if it is missing.

Identify the area from the request and the codebase. Ask only when two plausible readings would produce different code.

| Request is about… | Go to |
|---|---|
| Picking between World Partition and sub-levels | [Choosing a Streaming Model](#choosing-a-streaming-model) |
| Grid streaming, streaming sources, blocking loads, CVars | [World Partition Runtime](#world-partition-runtime) |
| Turning content on and off inside a partitioned world | [Data Layers](#data-layers) |
| Reusable level chunks placed in the editor, packed actors | [Level Instances](#level-instances) |
| `LoadStreamLevel`, `ULevelStreaming`, volumes, runtime instancing | [Sub-Level Streaming](#sub-level-streaming) |
| `OpenLevel`, `ServerTravel`, `ClientTravel`, seamless travel | [Level Travel](#level-travel) |
| Distant proxies for streamed-out cells | [HLOD](#hlod) |
| Keeping actor changes across a streaming cycle | [Level Streaming Persistence](#level-streaming-persistence) |
| Per-world managers, state that must survive travel | [World Subsystems and Cross-Level State](#world-subsystems-and-cross-level-state) |

## Choosing a Streaming Model

| Situation | Use | Why |
|---|---|---|
| One large continuous map | World Partition grid | Automatic spatial cells, no manual sub-level bookkeeping |
| Optional content inside a partitioned map | Data layers | `EDataLayerRuntimeState` toggles without travel |
| The same level asset placed many times in the editor | `ALevelInstance` / `APackedLevelActor` | Edited as a unit, packed into instanced meshes |
| The same level package spawned many times at runtime | `ULevelStreamingDynamic::LoadLevelInstance` | Creates a uniquely named package per instance |
| Discrete hand-authored zones in a non-partitioned map | `ULevelStreaming` sub-levels | Explicit load and visibility control per zone |
| Proximity-driven interiors in a non-partitioned map | `ALevelStreamingVolume` | Camera-position driven, no gameplay code |
| A different map entirely | `OpenLevel` / `ServerTravel` | Tears down and rebuilds the world |

World Partition and sub-level streaming are mutually exclusive on the same persistent level. `UWorld::IsPartitionedWorld()` (`Engine/World.h:2968`) reports which one is in play; `AWorldSettings::GetWorldPartition()` (`GameFramework/WorldSettings.h:837`) returns the partition object (the `WorldPartition` member is protected).

## World Partition Runtime

`UWorldPartition` (`WorldPartition/WorldPartition.h`) owns the runtime hash that turns actors into cells. `UWorldPartitionSubsystem` is the per-world entry point and is a `UTickableWorldSubsystem`.

```cpp
#include "WorldPartition/WorldPartition.h"
#include "WorldPartition/WorldPartitionSubsystem.h"

UWorldPartitionSubsystem* Subsystem = GetWorld()->GetSubsystem<UWorldPartitionSubsystem>();
if (Subsystem && Subsystem->IsAllStreamingCompleted())
{
    // Every cell has finished loading, adding to or removing from the world.
}

Subsystem->ForEachWorldPartition([](UWorldPartition* Partition)
{
    return Partition->IsStreamingEnabled(); // return false to stop iterating
});
```

`UWorldPartitionRuntimeCell` is the unit the hash produces. Query readiness at an arbitrary point with a query source instead of polling actors:

```cpp
#include "WorldPartition/WorldPartitionRuntimeCell.h"
#include "WorldPartition/WorldPartitionStreamingSource.h"

TArray<FWorldPartitionStreamingQuerySource> QuerySources;
QuerySources.Emplace(TargetLocation); // FVector

const bool bReady = Subsystem->IsStreamingCompleted(
    EWorldPartitionRuntimeCellState::Activated, QuerySources, /*bExactState=*/false);
```

`EWorldPartitionRuntimeCellState` is `Unloaded`, `Loaded`, `Activated` (`WorldPartitionRuntimeCell.h:202`).

### Streaming sources

Any object can drive streaming by implementing `IWorldPartitionStreamingSourceProvider` (a plain interface struct in `WorldPartitionStreamingSource.h`) and registering it with the subsystem. For the common "this actor pulls cells in" case, add a `UWorldPartitionStreamingSourceComponent` instead — it already implements the provider and registers itself.

The two virtuals to override are `virtual bool GetStreamingSources(TArray<FWorldPartitionStreamingSource>& StreamingSources) const` and `virtual const UObject* GetStreamingSourceOwner() const`. Register in `BeginPlay` with `UWorldPartitionSubsystem::RegisterStreamingSourceProvider(this)` and release in `EndPlay` with `UnregisterStreamingSourceProvider(this)`. A complete actor implementation is in [streaming patterns](references/streaming-patterns.md#custom-streaming-source-provider).

- `EStreamingSourceTargetState`: `Loaded` (in memory, not added to the world) or `Activated`.
- `EStreamingSourcePriority`: `Highest`, `High`, `Normal`, `Low`, `Lowest`, `Default` (equals `Normal`).
- `TargetGrids` plus `EStreamingSourceTargetBehavior` (`Include` / `Exclude`) restrict a source to named runtime grids.
- `bBlockOnSlowLoading` opts the source into the engine's blocking path when streaming falls behind. Leave it false for cosmetic sources.
- `UWorldPartitionSubsystem::GetStreamingSources(const UWorldPartition*, TArray<FWorldPartitionStreamingSource>&)` reads back everything currently registered.

### Runtime CVars

| CVar | Effect |
|---|---|
| `wp.Runtime.EnableServerStreaming` | Server-side cell streaming on dedicated and listen servers |
| `wp.Runtime.EnableServerStreamingOut` | Lets the server stream cells back out |
| `wp.Runtime.BlockOnSlowStreaming` | Blocks the game thread when a blocking source falls behind |
| `wp.Runtime.MaxLoadingStreamingCells` | Caps concurrent cell loads |
| `wp.Runtime.OverrideRuntimeLoadingRange` | Overrides a grid's loading range for testing |
| `wp.Runtime.ToggleDrawRuntimeHash2D` | Draws the cell grid on screen |
| `wp.Runtime.DumpStreamingSources` | Logs every registered streaming source |
| `wp.Runtime.HLOD` | Enables and disables HLOD proxies at runtime |

## Data Layers

Actors reference data layers through `AActor::DataLayerAssets` (`TArray<TSoftObjectPtr<UDataLayerAsset>>`, `GameFramework/Actor.h:1123`). `UDataLayerAsset` is the authored asset, `UDataLayerInstance` is its per-world instance, and `UDataLayerManager` is the runtime accessor, reachable from any object inside a partitioned world.

```cpp
// MyDataLayerController.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/Actor.h"
#include "MyDataLayerController.generated.h"

class UDataLayerAsset;

UCLASS()
class MYGAME_API AMyDataLayerController : public AActor
{
    GENERATED_BODY()

public:
    UFUNCTION(BlueprintCallable, Category = "MyGame|Streaming")
    void SetDungeonActive(bool bActive);

    UFUNCTION(BlueprintCallable, Category = "MyGame|Streaming")
    bool IsDungeonActivated() const;

protected:
    UPROPERTY(EditAnywhere, Category = "MyGame|Streaming")
    TObjectPtr<const UDataLayerAsset> DungeonLayer;
};
```

```cpp
// MyDataLayerController.cpp
#include "MyDataLayerController.h"
#include "WorldPartition/DataLayer/DataLayerAsset.h"
#include "WorldPartition/DataLayer/DataLayerInstance.h"
#include "WorldPartition/DataLayer/DataLayerManager.h"

void AMyDataLayerController::SetDungeonActive(bool bActive)
{
    UDataLayerManager* Manager = UDataLayerManager::GetDataLayerManager(this);
    if (!Manager || !DungeonLayer)
    {
        return;
    }

    const EDataLayerRuntimeState NewState =
        bActive ? EDataLayerRuntimeState::Activated : EDataLayerRuntimeState::Unloaded;

    Manager->SetDataLayerRuntimeState(DungeonLayer, NewState, /*bInIsRecursive=*/true);
}

bool AMyDataLayerController::IsDungeonActivated() const
{
    UDataLayerManager* Manager = UDataLayerManager::GetDataLayerManager(this);
    if (!Manager || !DungeonLayer)
    {
        return false;
    }

    const UDataLayerInstance* Instance = Manager->GetDataLayerInstanceFromAsset(DungeonLayer);
    return Instance && Manager->GetDataLayerInstanceRuntimeState(Instance) == EDataLayerRuntimeState::Activated;
}
```

`EDataLayerRuntimeState` (`DataLayerInstance.h:23`): `Unloaded`, `Loaded` (in memory but not visible — use it to pre-warm), `Activated` (loaded and visible).

Other manager entry points: `SetDataLayerInstanceRuntimeState(const UDataLayerInstance*, EDataLayerRuntimeState, bool bInIsRecursive)`, `GetDataLayerInstanceEffectiveRuntimeState`, `GetDataLayerInstances()`, `GetDataLayerInstanceFromName(const FName&)`, and the `BlueprintAssignable` delegate `OnDataLayerInstanceRuntimeStateChanged`. To sweep every instance:

```cpp
Manager->ForEachDataLayerInstance([](UDataLayerInstance* Instance)
{
    return Instance->IsRuntime(); // return false to stop early
});
```

**Authority:** a runtime data layer with no load filter only responds to state changes made on the server, and the resulting state replicates. Client-only layers must be changed on the client, server-only layers on the server. The old world-subsystem accessor still exists as a deprecated shim — see [Deprecated — do not use](#deprecated--do-not-use); route everything through `UDataLayerManager`.

## Level Instances

`ALevelInstance` (`LevelInstance/LevelInstanceActor.h`) places a `.umap` as a reusable chunk and implements `ILevelInstanceInterface`. `ULevelInstanceSubsystem` is the `UWorldSubsystem` that tracks them.

```cpp
#include "Engine/World.h"
#include "LevelInstance/LevelInstanceActor.h"
#include "LevelInstance/LevelInstanceSubsystem.h"

void AMyLevelInstanceDriver::ShowRoom(ALevelInstance* RoomInstance)
{
    if (!RoomInstance)
    {
        return;
    }

    RoomInstance->LoadLevelInstance(); // queued; loads on the subsystem's next streaming update, so IsLoaded() is false this frame

    ULevelInstanceSubsystem* Subsystem = GetWorld()->GetSubsystem<ULevelInstanceSubsystem>();
    if (Subsystem && Subsystem->IsLoaded(RoomInstance))
    {
        Subsystem->ForEachActorInLevelInstance(RoomInstance, [](AActor* LevelActor)
        {
            LevelActor->SetActorTickEnabled(true);
            return true;
        });
    }
}
```

- `ILevelInstanceInterface` runtime members: `LoadLevelInstance()`, `UnloadLevelInstance()`, `IsLoaded()`, `GetLoadedLevel()`, `GetLevelStreaming()`, `GetWorldAsset()`, `GetLevelInstanceID()`, `IsInitiallyVisible()`.
- `ULevelInstanceSubsystem` runtime members: `GetLevelInstance(const FLevelInstanceID&)`, `GetOwningLevelInstance(const ULevel*)`, `RequestLoadLevelInstance(ILevelInstanceInterface*, bool bUpdate)`, `RequestUnloadLevelInstance(ILevelInstanceInterface*)`, `IsLoaded`, `IsLoading`, `ForEachActorInLevelInstance`, `GetLevelInstanceLevel`. The editing entry points are `WITH_EDITOR` only.
- `ELevelInstanceRuntimeBehavior` (`LevelInstanceTypes.h:56`) picks how the instance reaches the runtime: `Partitioned` (actors move into the main partition grid; shown as "Embedded") or `LevelStreaming` (its own streaming level; shown as "Standalone").
- `APackedLevelActor` (`PackedLevelActor/PackedLevelActor.h`) derives from `ALevelInstance` and bakes its static meshes into instanced-mesh components. It reports `ELevelInstanceRuntimeBehavior::None` and loads no level at runtime.

## Sub-Level Streaming

### ULevelStreaming state

`ELevelStreamingState` (`Engine/LevelStreaming.h:110`) is `Removed`, `Unloaded`, `FailedToLoad`, `Loading`, `LoadedNotVisible`, `MakingVisible`, `LoadedVisible`, `MakingInvisible`. Read it with `GetLevelStreamingState()`. The requested side is `ELevelStreamingTargetState` (`Unloaded`, `UnloadedAndRemoved`, `LoadedNotVisible`, `LoadedVisible`).

```cpp
#include "Engine/LevelStreaming.h"
#include "Kismet/GameplayStatics.h"

ULevelStreaming* Streaming = UGameplayStatics::GetStreamingLevel(this, FName("/Game/Levels/L_Zone_A"));
if (Streaming)
{
    Streaming->SetShouldBeLoaded(true);
    Streaming->SetShouldBeVisible(true);

    const bool bReady = Streaming->IsLevelLoaded() && Streaming->IsLevelVisible();
    ULevel* Loaded = Streaming->GetLoadedLevel();
}
```

Four `BlueprintAssignable` delegates, all with no parameters: `OnLevelLoaded`, `OnLevelUnloaded`, `OnLevelShown`, `OnLevelHidden`. Bind with `AddDynamic` to a no-argument `UFUNCTION()`.

Use `SetIsRequestingUnloadAndRemoval(true)` to drop the streaming level object entirely. Never mutate `UWorld::StreamingLevels` directly — use `AddStreamingLevel`, `AddStreamingLevels`, `RemoveStreamingLevel`, `RemoveStreamingLevels` and `UpdateStreamingLevelShouldBeConsidered` (`Engine/World.h:1073-1109`), which maintain `StreamingLevelsToConsider`.

### Latent load and unload

`UGameplayStatics::LoadStreamLevel(const UObject* WorldContextObject, FName LevelName, bool bMakeVisibleAfterLoad, bool bShouldBlockOnLoad, FLatentActionInfo LatentInfo)` and `UnloadStreamLevel(const UObject* WorldContextObject, FName LevelName, FLatentActionInfo LatentInfo, bool bShouldBlockOnUnload)` resume through `FLatentActionInfo::ExecutionFunction`, which is resolved **by name through the reflection system**. The callback therefore has to be a `UFUNCTION()` declared inside a class body in a header.

```cpp
// MyStreamingActor.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/Actor.h"
#include "MyStreamingActor.generated.h"

UCLASS()
class MYGAME_API AMyStreamingActor : public AActor
{
    GENERATED_BODY()

public:
    void StreamInRoom(FName LevelName);
    void StreamOutRoom(FName LevelName);

    // UFUNCTION on the declaration is mandatory: FLatentActionInfo::ExecutionFunction
    // is looked up by name on CallbackTarget's UClass.
    UFUNCTION()
    void OnRoomLoaded();

    UFUNCTION()
    void OnRoomUnloaded();

protected:
    bool bRoomReady = false;
};
```

```cpp
// MyStreamingActor.cpp
#include "MyStreamingActor.h"
#include "Engine/LatentActionManager.h"
#include "Kismet/GameplayStatics.h"

void AMyStreamingActor::StreamInRoom(FName LevelName)
{
    FLatentActionInfo LatentInfo;
    LatentInfo.CallbackTarget = this;
    LatentInfo.ExecutionFunction = FName("OnRoomLoaded");
    LatentInfo.Linkage = 0;
    LatentInfo.UUID = 1; // unique per pending latent action on this object

    UGameplayStatics::LoadStreamLevel(this, LevelName,
        /*bMakeVisibleAfterLoad=*/true, /*bShouldBlockOnLoad=*/false, LatentInfo);
}

void AMyStreamingActor::StreamOutRoom(FName LevelName)
{
    FLatentActionInfo LatentInfo;
    LatentInfo.CallbackTarget = this;
    LatentInfo.ExecutionFunction = FName("OnRoomUnloaded");
    LatentInfo.Linkage = 0;
    LatentInfo.UUID = 2;

    UGameplayStatics::UnloadStreamLevel(this, LevelName, LatentInfo, /*bShouldBlockOnUnload=*/false);
}

void AMyStreamingActor::OnRoomLoaded()
{
    bRoomReady = true;
}

void AMyStreamingActor::OnRoomUnloaded()
{
    bRoomReady = false;
}
```

`LoadStreamLevelBySoftObjectPtr` and `UnloadStreamLevelBySoftObjectPtr` take a `TSoftObjectPtr<UWorld>` in place of the `FName` and are the packaging-safe form. `UGameplayStatics::FlushLevelStreaming(const UObject*)` blocks until pending work completes; `UWorld::FlushLevelStreaming(EFlushLevelStreamingType)` gives finer control (`None`, `Full`, `Visibility`).

### Runtime level instancing

`ULevelStreamingDynamic` loads one package repeatedly at different transforms. Prefer the params-struct overload, because the by-name overload cannot express a full `FTransform`.

```cpp
#include "Engine/LevelStreamingDynamic.h"
#include "Engine/World.h"

ULevelStreamingDynamic* AMyDungeonGenerator::SpawnRoom(const FTransform& RoomTransform, const FString& InstanceName)
{
    ULevelStreamingDynamic::FLoadLevelInstanceParams Params(
        GetWorld(), TEXT("/Game/Levels/L_Room_Corridor"), RoomTransform);

    Params.OptionalLevelNameOverride = &InstanceName; // identical on server and clients
    Params.bInitiallyVisible = true;

    bool bSuccess = false;
    ULevelStreamingDynamic* Level = ULevelStreamingDynamic::LoadLevelInstance(Params, bSuccess);
    return bSuccess ? Level : nullptr;
}
```

Tear an instance down with `SetShouldBeVisible(false)`, `SetShouldBeLoaded(false)`, then `SetIsRequestingUnloadAndRemoval(true)`. A generator that tracks its instances and binds `OnLevelShown` is in [streaming patterns](references/streaming-patterns.md#pattern-3-procedural-and-instanced-level-streaming).

`FLoadLevelInstanceParams` members: `World`, `LongPackageName`, `LevelTransform`, `OptionalLevelNameOverride` (`const FString*`), `OptionalLevelStreamingClass`, `bLoadAsTempPackage`, `bInitiallyVisible`, `bAllowReuseExitingLevelStreaming`, `EditorPathOwner`, `LevelStreamingCreatedCallback`.

The Blueprint-exposed overloads are `LoadLevelInstance(UObject* WorldContextObject, FString LevelName, FVector Location, FRotator Rotation, bool& bOutSuccess, const FString& OptionalLevelNameOverride, TSubclassOf<ULevelStreamingDynamic> OptionalLevelStreamingClass, bool bLoadAsTempPackage)` and `LoadLevelInstanceBySoftObjectPtr`, which takes a `TSoftObjectPtr<UWorld>` in place of `LevelName` and the same tail parameters; a C++-only `BySoftObjectPtr` overload takes an `FTransform` (`Engine/LevelStreamingDynamic.h:94`).

### Streaming volumes

`ALevelStreamingVolume` drives sub-levels from camera position. `EStreamingVolumeUsage` values: `SVB_Loading`, `SVB_LoadingAndVisibility`, `SVB_VisibilityBlockingOnLoad`, `SVB_BlockingOnLoad`, `SVB_LoadingNotVisible`. Set `StreamingUsage` on the volume; `StreamingLevelNames` lists what it affects and a streaming level's `EditorStreamingVolumes` array is the editor-side link. Set `ULevelStreaming::bDisableDistanceStreaming = true` to take a level off volume control, and `MinTimeBetweenVolumeUnloadRequests` to damp flicker at volume boundaries. Streaming volumes do nothing in a partitioned world.

## Level Travel

| Call | Header | Effect |
|---|---|---|
| `UGameplayStatics::OpenLevel(const UObject*, FName LevelName, bool bAbsolute, FString Options)` | `Kismet/GameplayStatics.h:340` | Tears down the world; clients disconnect |
| `UGameplayStatics::OpenLevelBySoftObjectPtr(const UObject*, TSoftObjectPtr<UWorld>, bool, FString)` | `Kismet/GameplayStatics.h:350` | Same, packaging-safe reference |
| `UWorld::ServerTravel(const FString& InURL, bool bAbsolute, bool bShouldSkipGameNotify)` | `Engine/World.h:4231` | Server moves, clients follow |
| `APlayerController::ClientTravel(const FString& URL, ETravelType, bool bSeamless, FGuid)` | `GameFramework/PlayerController.h:1413` | Client-initiated; `TRAVEL_Absolute`, `TRAVEL_Partial`, `TRAVEL_Relative` |
| `UWorld::SeamlessTravel(const FString& InURL, bool bAbsolute)` | `Engine/World.h:4243` | Background transition, connections kept |
| `UGameplayStatics::GetCurrentLevelName(const UObject*, bool bRemovePrefixString)` | `Kismet/GameplayStatics.h:358` | Current map name |

### Seamless travel

1. Set `bUseSeamlessTravel = true` in the GameMode constructor. It is already a `UPROPERTY` on `AGameModeBase` (`GameModeBase.h:579`) — do not redeclare it.
2. Set a transition map. Without one the engine transitions through an empty dummy world (`World.cpp:8367`). In PIE, seamless travel falls back to hard travel unless `net.AllowPIESeamlessTravel=1` (`GameModeBase.cpp:506`).

```ini
[/Script/EngineSettings.GameMapsSettings]
TransitionMap=/Game/Maps/L_Transition.L_Transition
```

3. Override the persistence hooks. Signatures are verbatim from `GameFramework/GameModeBase.h:239-268`:

```cpp
// MyGameMode.cpp
void AMyGameMode::GetSeamlessTravelActorList(bool bToTransition, TArray<AActor*>& ActorList)
{
    Super::GetSeamlessTravelActorList(bToTransition, ActorList);

    if (MyPersistentManager)
    {
        ActorList.Add(MyPersistentManager); // called for both legs (to transition, then to destination); add on both
    }
}

void AMyGameMode::PostSeamlessTravel()
{
    Super::PostSeamlessTravel();
}

void AMyGameMode::HandleSeamlessTravelPlayer(AController*& C)
{
    Super::HandleSeamlessTravelPlayer(C);
}
```

`APlayerController::GetSeamlessTravelActorList(bool bToEntry, TArray<AActor*>& ActorList)` is the client-side counterpart. `APlayerController::SeamlessTravelTo(APlayerController* NewPC)` / `SeamlessTravelFrom(APlayerController* OldPC)` and `APlayerState::SeamlessTravelTo(APlayerState* NewPlayerState)` copy per-player state onto the new objects. `UWorld::IsInSeamlessTravel()` and `UWorld::SetSeamlessTravelMidpointPause(bool bNowPaused)` control the midpoint; `UWorld::OnSeamlessTravelStart` and `UWorld::OnSeamlessTravelTransition` are static multicast delegates for observers.

Replication behaviour across travel (channel teardown, actor re-creation, net relevancy) belongs to `ue-networking-replication`; `UNetDriver::NotifyActorLevelUnloaded(AActor*)` is the hook it uses when a streamed level goes away.

## HLOD

HLOD produces stand-in proxies for cells that are streamed out. `UHLODLayer` (`WorldPartition/HLOD/HLODLayer.h`) configures a tier; grid placement comes from the runtime partition's settings rather than the layer. Hash and rebuild bookkeeping runs through `UHLODRebuildPolicy` and `UHLODRebuildPolicyData`, and builder settings (`UHLODBuilderSettings`) implement `ComputeHLODHash(FHLODHashBuilder&)`.

At runtime, `UWorldPartitionHLODRuntimeSubsystem` (a `UWorldSubsystem`) tracks proxies as `IWorldPartitionHLODObject`, not as actors: `RegisterHLODObject`, `UnregisterHLODObject`, `GetHLODObjectsForCell(const UWorldPartitionRuntimeCell*)`, `OnHLODObjectRegisteredEvent()`, `OnHLODObjectUnregisteredEvent()`, `IsHLODEnabled()`, `IsWarmupEnabled()`. `AWorldPartitionHLOD` and `AWorldPartitionCustomHLOD` (`WorldPartition/HLOD/CustomHLODActor.h`) both implement that interface, and `AWorldPartitionHLODSourceCellPlaceholder` stands in for a source cell during builds.

Build HLODs before shipping. Without them, everything past the loading range is simply absent.

## Level Streaming Persistence

The Level Streaming Persistence plugin (Experimental in 5.8) keeps property values, destroyed-actor records and respawn data attached to streaming levels, so a level that streams out and back in returns changed. Add the `LevelStreamingPersistence` module to `Build.cs`.

`ULevelStreamingPersistenceManager` is a `UWorldSubsystem`. Blueprint-callable: `SerializeTo(TArray<uint8>& OutPayload, bool bForceUpdate)`, `InitializeFrom(const TArray<uint8>& InPayload)`, `EjectPlacedActor(AActor*)`, `RecreateActorInLevel(AActor*, ULevel* NewOwningLevel)`, `RecreateActorInPersistentLevel(AActor*)`. Templated accessors `SetPropertyValue`, `TrySetPropertyValue` and `GetPropertyValue` read and write individual persisted properties by object path and property name. Which properties persist is declared in `ULevelStreamingPersistenceSettings`, a `UDeveloperSettings` with config `Engine`.

Call `InitializeFrom` once per map and early — from a custom `UWorldSubsystem::Initialize` — so values land before actors begin play. The payload it produces is what a `USaveGame` stores; see `ue-serialization-savegames`.

## World Subsystems and Cross-Level State

`UWorldSubsystem` (`Subsystems/WorldSubsystem.h`) is created once per `UWorld` and dies with it, including on travel — the right home for a streaming manager or zone tracker. The full subsystem type table lives in `ue-cpp-foundations`.

Overridable hooks, verbatim from `Subsystems/WorldSubsystem.h:34-66`: `virtual bool ShouldCreateSubsystem(UObject* Outer) const`, `virtual void PostInitialize()`, `virtual void OnWorldBeginPlay(UWorld& InWorld)`, `virtual void OnWorldEndPlay(UWorld& InWorld)`, `virtual void OnWorldComponentsUpdated(UWorld& World)`, `virtual void PreDeinitialize()`, `virtual bool DoesSupportWorldType(const EWorldType::Type WorldType) const`. Reach it with `GetWorld()->GetSubsystem<UMyStreamingManager>()`. A worked zone-manager subsystem is in [streaming patterns](references/streaming-patterns.md#pattern-2-hub-and-spoke-with-manual-sub-level-streaming).

For per-frame work derive from `UTickableWorldSubsystem` and implement `Tick(float DeltaTime)` plus `GetStatId() const` (pure virtual). Its `Initialize` and `Deinitialize` overrides are what arm and disarm ticking, so always call `Super::` in both.

| Mechanism | Lifetime | Use for |
|---|---|---|
| `UGameInstance` | Whole application session | Cross-map player and session data |
| `UGameInstanceSubsystem` | Whole application session | Services that outlive every world |
| Seamless travel actor list | Transition only | Actors that physically cross |
| `UWorldSubsystem` | One world | World-scoped caches; push to `UGameInstance` before travel |
| `USaveGame` | Disk | Progression — see `ue-serialization-savegames` |

## Deprecated — do not use

| Do not emit | Use in 5.8 | Source |
|---|---|---|
| `AActor::DataLayers`, `FActorDataLayer` | `AActor::DataLayerAssets` | `UE_DEPRECATED(5.8)` in `GameFramework/Actor.h:1115` |
| `UDataLayerSubsystem` | `UDataLayerManager` | `UE_DEPRECATED(5.3)` in `WorldPartition/DataLayer/DataLayerSubsystem.h:37` |
| `AWorldDataLayers::OverwriteDataLayerRuntimeStates` | `UDataLayerManager::SetDataLayerRuntimeState` | `UE_DEPRECATED(5.8)` in `WorldPartition/DataLayer/WorldDataLayers.h:135` |
| `UDataLayerInstance::GetDataLayerFName` | `UObject::GetFName` | `UE_DEPRECATED(5.8)` in `WorldPartition/DataLayer/DataLayerInstance.h:190` |
| `UDataLayerManager::GetDataLayerInstanceNames` outside the editor | Pass asset or instance pointers at runtime | `UE_DEPRECATED(5.8)` in `WorldPartition/DataLayer/DataLayerManager.h:121` |
| `ULevelStreaming::ECurrentState`, `GetCurrentState()` | `ELevelStreamingState`, `GetLevelStreamingState()` | `UE_DEPRECATED(5.2)` in `Engine/LevelStreaming.h:367` |
| `UWorldPartitionLevelStreamingDynamic::LoadInEditor` / `UnloadFromEditor` | No supported replacement | `UE_DEPRECATED(5.7)` in `WorldPartition/WorldPartitionLevelStreamingDynamic.h:63` |
| `UHLODBuilderSettings::GetCRC()` | `ComputeHLODHash(FHLODHashBuilder&)` | `UE_DEPRECATED(5.7)` in `WorldPartition/HLOD/HLODBuilder.h:43` |
| `UWorldPartitionHLODSourceActors::GetHLODHash()` | `ComputeHLODHash(FHLODHashBuilder&)` | `UE_DEPRECATED(5.7)` in `WorldPartition/HLOD/HLODSourceActors.h:27` |
| `AWorldPartitionHLOD::GetHLODHash` / `ComputeHLODHash` / `SetHLODHash` | `GetHLODRebuildPolicyDataSet` / `SetHLODRebuildPolicyDataSet` with `UHLODRebuildPolicyData` | `UE_DEPRECATED(5.8)` in `WorldPartition/HLOD/HLODActor.h:146` |
| `UHLODLayer::GetCellSize` / `GetLoadingRange` / `IsSpatiallyLoaded` | Runtime partition settings | `UE_DEPRECATED(5.7)` in `WorldPartition/HLOD/HLODLayer.h:82` |
| `UWorldPartitionHLODRuntimeSubsystem::RegisterHLODActor` / `GetHLODActorsForCell` / `GetNumOutdatedHLODActors` | `RegisterHLODObject` / `GetHLODObjectsForCell` / `GetNumOutdatedHLODObjects` | `UE_DEPRECATED(5.6)` in `WorldPartition/HLOD/HLODRuntimeSubsystem.h:77` |
| `AWorldPartitionCustomHLODPlaceholder` | `AWorldPartitionHLODSourceCellPlaceholder` | `UE_DEPRECATED(5.8)` in `WorldPartition/HLOD/CustomHLODPlaceholderActor.h:9` |
| `UWorldPartitionStreamingSourceComponent::TargetGrid` / `TargetHLODLayers` | `TargetGrids` with `TargetBehavior` | `DeprecatedProperty` in `Components/WorldPartitionStreamingSourceComponent.h:63` |

## Common Mistakes

**`UFUNCTION()` above an out-of-line definition.** A latent callback written as `UFUNCTION() void AMyActor::OnRoomLoaded() {}` in a `.cpp` is invisible to UHT, so `FLatentActionInfo::ExecutionFunction` never resolves and the continuation silently never fires. Declare it inside the class body in the header.

**Reusing one `FLatentActionInfo::UUID`.** Two pending actions on the same `CallbackTarget` that share a UUID collide and one is dropped. Give each call site its own constant.

**Blocking loads outside a loading screen.** `bShouldBlockOnLoad = true`, `FlushLevelStreaming` and `wp.Runtime.BlockOnSlowStreaming` all stall the game thread. On a server they stall every client.

**Setting data-layer state on the client.** A runtime data layer with no load filter only responds on the server; the state then replicates. Client-side calls appear to do nothing.

**Mismatched dynamic level names in multiplayer.** `ULevelStreamingDynamic::LoadLevelInstance` generates a unique package name per process. Without `OptionalLevelNameOverride` set to the same string on server and clients, the instances are different packages and nothing lines up.

**Hard references across level boundaries.** Unloading marks every object in the level package as garbage (`LevelStreamingGCHelper.cpp:201-208`), so a `UPROPERTY()` pointer from a persistent actor into a streamed level is nulled (a non-`UPROPERTY` raw pointer dangles) and does not re-point when the level streams back in. Use `TSoftObjectPtr` or `TWeakObjectPtr` across boundaries.

**Expecting streaming volumes to work in World Partition.** They do not. Use data layers or a streaming source instead.

**Assuming the dedicated server streams like a client.** Server cell streaming is off unless enabled, and its sources are the ones you register. Check `UWorldPartition::IsServerStreamingEnabled()` before debugging "missing" actors on the server.

**Mutating `UWorld::StreamingLevels` directly.** It bypasses `StreamingLevelsToConsider`, so the level is never evaluated. Use the `AddStreamingLevel` / `RemoveStreamingLevel` family.

**Skipping `Super::Initialize` / `Super::Deinitialize` in a `UTickableWorldSubsystem`.** Those calls are what enable and disable ticking.

## Related Skills

- `ue-cpp-foundations` — the full subsystem type table, `TObjectPtr`, `TSoftObjectPtr`, GC and object lifetime rules.
- `ue-networking-replication` — replication across travel, net relevancy, `DOREPLIFETIME`, dedicated-server authority.
- `ue-gameplay-framework` — GameMode and GameState travel callbacks, login flow, `UGameInstance` role.
- `ue-data-assets-tables` — `UDataAsset` (the base of `UDataLayerAsset`), async asset loading, Asset Manager.
- `ue-procedural-generation` — PCG-driven content that populates partitioned levels.
- `ue-serialization-savegames` — `USaveGame`, writing the Level Streaming Persistence payload to disk.
- `ue-actor-component-architecture` — component design for streaming-source actors and world managers.
- `ue-game-features` — Game Feature plugins, GameFeatureAction and the modular component manager
- `ue-sequencer-cinematics` — Level Sequences, playback, cine cameras and Movie Render Graph
