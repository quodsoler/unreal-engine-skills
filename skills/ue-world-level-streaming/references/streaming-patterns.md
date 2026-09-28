# Level Streaming Configuration Patterns

Target engine: **UE 5.8**. Reference configurations for common game shapes, plus the long code examples that do not fit in the main skill file. Pick the pattern that matches the project's world structure and adapt the specifics to scale and multiplayer needs.

---

## Pattern 1: Open World with World Partition

**Best for:** large continuous worlds — survival games, open-world RPGs, exploration games.

### Configuration

Enable World Partition on the persistent level. Every actor is managed by the runtime grid, and the level cannot also carry sub-levels. Loading range is a property of the runtime partition (`URuntimePartition::LoadingRange`, `WorldPartition/RuntimeHashSet/RuntimePartition.h:104`), configured in the World Partition editor, not in an ini file. `UWorldPartitionRuntimeHashSet::GetLoadingRange()` reads the effective value back at runtime.

One File Per Actor stores each actor as its own package under `__ExternalActors__`, which is what makes concurrent editing of a single map practical.

### Data layers for optional content

```cpp
// Interiors, quest states and time-of-day sets live on runtime data layers
// and toggle without traveling to another map.
#include "WorldPartition/DataLayer/DataLayerAsset.h"
#include "WorldPartition/DataLayer/DataLayerManager.h"

void AMyWorldDirector::SetInteriorState(const UDataLayerAsset* InteriorLayer, bool bVisible)
{
    UDataLayerManager* Manager = UDataLayerManager::GetDataLayerManager(this);
    if (!Manager || !InteriorLayer)
    {
        return;
    }

    // Activated = loaded and visible. Loaded = in memory, invisible (pre-warm).
    Manager->SetDataLayerRuntimeState(InteriorLayer,
        bVisible ? EDataLayerRuntimeState::Activated : EDataLayerRuntimeState::Loaded);
}
```

Call this on the server for ordinary runtime layers; the resulting state replicates to clients. `UDataLayerManager::GetDataLayerInstanceEffectiveRuntimeState` is the query that accounts for parent layers.

### HLOD setup

1. Create one `UHLODLayer` asset per proxy tier and assign it in the World Partition settings.
2. Set the runtime grid properties on the partition, not on the HLOD layer.
3. Build HLODs before cooking. Cells past the loading range have no representation otherwise.
4. Verify at runtime with the `wp.Runtime.HLOD` CVar and `UWorldPartitionHLODRuntimeSubsystem::IsHLODEnabled()`.

### Anti-patterns for this setup

- Streaming volumes have no effect in a partitioned world.
- `UGameplayStatics::LoadStreamLevel` has nothing to target — there are no named sub-levels.
- Actors marked not spatially loaded are always resident; use that sparingly.

---

## Custom Streaming Source Provider

A cinematic camera, an AI director or a spectator proxy can pull cells in without being a player. Implement `IWorldPartitionStreamingSourceProvider` and register with `UWorldPartitionSubsystem`.

```cpp
// MyDirectorCamera.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/Actor.h"
#include "WorldPartition/WorldPartitionStreamingSource.h"
#include "MyDirectorCamera.generated.h"

UCLASS()
class MYGAME_API AMyDirectorCamera : public AActor, public IWorldPartitionStreamingSourceProvider
{
    GENERATED_BODY()

public:
    virtual void BeginPlay() override;
    virtual void EndPlay(const EEndPlayReason::Type EndPlayReason) override;

    virtual bool GetStreamingSources(TArray<FWorldPartitionStreamingSource>& StreamingSources) const override;
    virtual const UObject* GetStreamingSourceOwner() const override;
};
```

```cpp
// MyDirectorCamera.cpp
#include "MyDirectorCamera.h"
#include "Engine/World.h"
#include "WorldPartition/WorldPartitionSubsystem.h"

void AMyDirectorCamera::BeginPlay()
{
    Super::BeginPlay();

    if (UWorldPartitionSubsystem* Subsystem = GetWorld()->GetSubsystem<UWorldPartitionSubsystem>())
    {
        Subsystem->RegisterStreamingSourceProvider(this);
    }
}

void AMyDirectorCamera::EndPlay(const EEndPlayReason::Type EndPlayReason)
{
    if (UWorldPartitionSubsystem* Subsystem = GetWorld()->GetSubsystem<UWorldPartitionSubsystem>())
    {
        Subsystem->UnregisterStreamingSourceProvider(this);
    }

    Super::EndPlay(EndPlayReason);
}

bool AMyDirectorCamera::GetStreamingSources(TArray<FWorldPartitionStreamingSource>& StreamingSources) const
{
    FWorldPartitionStreamingSource Source;
    Source.Name = GetFName();
    Source.Location = GetActorLocation();
    Source.Rotation = GetActorRotation();
    Source.TargetState = EStreamingSourceTargetState::Activated;
    Source.Priority = EStreamingSourcePriority::High;
    Source.bBlockOnSlowLoading = false;
    StreamingSources.Add(MoveTemp(Source));
    return true;
}

const UObject* AMyDirectorCamera::GetStreamingSourceOwner() const
{
    return this;
}
```

To restrict the source to specific runtime grids, fill `Source.TargetGrids` and set `Source.TargetBehavior` to `EStreamingSourceTargetBehavior::Include` or `Exclude`. To shape the query volume beyond the default sphere, push `FStreamingSourceShape` entries into `Source.Shapes`.

Confirm registration with the `wp.Runtime.DumpStreamingSources` console command, or `UWorldPartitionSubsystem::GetStreamingSourceProviders()` in code.

---

## Pattern 2: Hub-and-Spoke with Manual Sub-Level Streaming

**Best for:** discrete zones — MMO-style areas, dungeon crawlers, hub worlds with portal travel.

The persistent level holds GameMode, GameState, spawn points, UI actors and global managers, and never unloads. Each zone is a separate `.umap` in the persistent level's streaming list.

```cpp
// MyZoneManager.h
#pragma once

#include "CoreMinimal.h"
#include "Subsystems/WorldSubsystem.h"
#include "MyZoneManager.generated.h"

UCLASS()
class MYGAME_API UMyZoneManager : public UWorldSubsystem
{
    GENERATED_BODY()

public:
    void LoadZone(FName ZonePackageName);
    void UnloadZone(FName ZonePackageName);
    void PreloadAdjacentZone(FName ZonePackageName);
    bool IsZoneLoaded(FName ZonePackageName) const;

    // Latent continuations must be UFUNCTIONs declared in the class body.
    UFUNCTION()
    void OnZoneLoaded();

    UFUNCTION()
    void OnZoneUnloaded();

private:
    TSet<FName> LoadedZones;
    FName PendingZone;
};
```

```cpp
// MyZoneManager.cpp
#include "MyZoneManager.h"
#include "Engine/LatentActionManager.h"
#include "Engine/LevelStreaming.h"
#include "Kismet/GameplayStatics.h"

void UMyZoneManager::LoadZone(FName ZonePackageName)
{
    PendingZone = ZonePackageName;

    FLatentActionInfo LatentInfo;
    LatentInfo.CallbackTarget = this;
    LatentInfo.ExecutionFunction = FName("OnZoneLoaded");
    LatentInfo.Linkage = 0;
    LatentInfo.UUID = 1; // one load in flight: a second call with a pending UUID is ignored (FindExistingAction)

    UGameplayStatics::LoadStreamLevel(this, ZonePackageName,
        /*bMakeVisibleAfterLoad=*/true, /*bShouldBlockOnLoad=*/false, LatentInfo);
}

void UMyZoneManager::UnloadZone(FName ZonePackageName)
{
    PendingZone = ZonePackageName;

    FLatentActionInfo LatentInfo;
    LatentInfo.CallbackTarget = this;
    LatentInfo.ExecutionFunction = FName("OnZoneUnloaded");
    LatentInfo.Linkage = 0;
    LatentInfo.UUID = 2;

    UGameplayStatics::UnloadStreamLevel(this, ZonePackageName, LatentInfo, /*bShouldBlockOnUnload=*/false);
}

void UMyZoneManager::PreloadAdjacentZone(FName ZonePackageName)
{
    // Bring the neighbour into memory without adding it to the world.
    if (ULevelStreaming* Streaming = UGameplayStatics::GetStreamingLevel(this, ZonePackageName))
    {
        Streaming->SetShouldBeLoaded(true);
        Streaming->SetShouldBeVisible(false);
    }
}

void UMyZoneManager::OnZoneLoaded()
{
    LoadedZones.Add(PendingZone);
}

void UMyZoneManager::OnZoneUnloaded()
{
    LoadedZones.Remove(PendingZone);
}

bool UMyZoneManager::IsZoneLoaded(FName ZonePackageName) const
{
    return LoadedZones.Contains(ZonePackageName);
}
```

### Loading screen handoff

Hold the loading screen until the target zone reports `LoadedVisible`.

```cpp
// MyLoadingScreenActor.h
#pragma once

#include "CoreMinimal.h"
#include "Engine/TimerHandle.h"
#include "GameFramework/Actor.h"
#include "MyLoadingScreenActor.generated.h"

UCLASS()
class MYGAME_API AMyLoadingScreenActor : public AActor
{
    GENERATED_BODY()

public:
    void WaitForZoneVisible(FName ZonePackageName);

protected:
    void DismissLoadingScreen();

private:
    FTimerHandle PollTimerHandle;
};
```

```cpp
// MyLoadingScreenActor.cpp
#include "MyLoadingScreenActor.h"
#include "Engine/LevelStreaming.h"
#include "Engine/World.h"
#include "Kismet/GameplayStatics.h"
#include "TimerManager.h"

void AMyLoadingScreenActor::WaitForZoneVisible(FName ZonePackageName)
{
    GetWorld()->GetTimerManager().SetTimer(PollTimerHandle, FTimerDelegate::CreateWeakLambda(this, [this, ZonePackageName]()
    {
        ULevelStreaming* Streaming = UGameplayStatics::GetStreamingLevel(this, ZonePackageName);
        if (Streaming && Streaming->GetLevelStreamingState() == ELevelStreamingState::LoadedVisible)
        {
            DismissLoadingScreen();
            GetWorld()->GetTimerManager().ClearTimer(PollTimerHandle);
        }
    }),
    0.1f,   // poll interval
    true);  // looping
}
```

Binding `ULevelStreaming::OnLevelShown` is cheaper than polling when the streaming object already exists; poll only when the level may not have been created yet.

---

## Pattern 3: Procedural and Instanced Level Streaming

**Best for:** roguelikes, procedural dungeons, instanced arenas, modular buildings assembled at runtime.

```cpp
// MyDungeonGenerator.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/Actor.h"
#include "MyDungeonGenerator.generated.h"

class ULevelStreamingDynamic;

UCLASS()
class MYGAME_API AMyDungeonGenerator : public AActor
{
    GENERATED_BODY()

public:
    void SpawnRoom(const FTransform& RoomTransform, const FString& InstanceName);
    void DespawnRoom(const FString& InstanceName);

    UFUNCTION()
    void HandleRoomShown();

protected:
    UPROPERTY(EditDefaultsOnly, Category = "MyGame|Streaming")
    TSoftObjectPtr<UWorld> RoomTemplate;

private:
    UPROPERTY()
    TMap<FString, TObjectPtr<ULevelStreamingDynamic>> SpawnedRooms;
};
```

```cpp
// MyDungeonGenerator.cpp
#include "MyDungeonGenerator.h"
#include "Engine/LevelStreamingDynamic.h"
#include "Engine/World.h"

void AMyDungeonGenerator::SpawnRoom(const FTransform& RoomTransform, const FString& InstanceName)
{
    ULevelStreamingDynamic::FLoadLevelInstanceParams Params(
        GetWorld(), RoomTemplate.GetLongPackageName(), RoomTransform);

    // Deterministic package name so server and clients agree.
    Params.OptionalLevelNameOverride = &InstanceName;
    Params.bInitiallyVisible = true;

    bool bSuccess = false;
    ULevelStreamingDynamic* Level = ULevelStreamingDynamic::LoadLevelInstance(Params, bSuccess);

    if (bSuccess && Level)
    {
        Level->OnLevelShown.AddDynamic(this, &AMyDungeonGenerator::HandleRoomShown);
        SpawnedRooms.Add(InstanceName, Level);
    }
}

void AMyDungeonGenerator::DespawnRoom(const FString& InstanceName)
{
    if (TObjectPtr<ULevelStreamingDynamic>* Found = SpawnedRooms.Find(InstanceName))
    {
        ULevelStreamingDynamic* Level = *Found;
        Level->SetShouldBeVisible(false);
        Level->SetShouldBeLoaded(false);
        Level->SetIsRequestingUnloadAndRemoval(true);
        SpawnedRooms.Remove(InstanceName);
    }
}

void AMyDungeonGenerator::HandleRoomShown()
{
    // OnLevelShown is a parameterless dynamic multicast delegate.
}
```

### Multiplayer instance names

The server decides the instance names (`Room_0001`, `Room_0002`, …) and replicates them, typically as an array on the GameState. Clients call `LoadLevelInstance` with the same `OptionalLevelNameOverride`, so every connection ends up with identically named packages. Without that, the server and each client build different package names and actor references across the boundary never resolve. For the replication side — `GetLifetimeReplicatedProps`, `ReplicatedUsing`, conditions — see `ue-networking-replication`.

Reserve `Params.bAllowReuseExitingLevelStreaming` for cases where the same instance name is intentionally re-requested, and `Params.OptionalLevelStreamingClass` when a `ULevelStreamingDynamic` subclass should own the instance.

---

## Pattern 4: Linear Chapter Structure

**Best for:** narrative games and linear action games with distinct chapters or missions.

Each chapter is its own `.umap`. Single-player uses `OpenLevel`; multiplayer uses seamless travel so connections survive.

```cpp
// Single player: stash progress in the GameInstance, then hard-travel.
void AMyGameMode::TravelToChapter(FName ChapterMapName)
{
    if (UMyGameInstance* GameInstance = GetGameInstance<UMyGameInstance>())
    {
        GameInstance->LastCompletedChapter = CurrentChapter;
    }

    UGameplayStatics::OpenLevel(this, ChapterMapName, /*bAbsolute=*/true);
}
```

```ini
; DefaultEngine.ini — optional; without it the engine transitions through an empty dummy world
[/Script/EngineSettings.GameMapsSettings]
TransitionMap=/Game/Maps/L_Transition.L_Transition
```

```cpp
// Multiplayer: bUseSeamlessTravel is already a UPROPERTY on AGameModeBase.
// Set it in the constructor; never redeclare it.
AMyGameMode::AMyGameMode()
{
    bUseSeamlessTravel = true;
}

void AMyGameMode::TravelToChapter(const FString& ChapterURL)
{
    GetWorld()->ServerTravel(ChapterURL); // clients follow automatically
}

void AMyGameMode::GetSeamlessTravelActorList(bool bToTransition, TArray<AActor*>& ActorList)
{
    Super::GetSeamlessTravelActorList(bToTransition, ActorList);

    // Super keeps PlayerStates both ways; GameMode, GameState and GameSession only reach the transition map.
    // Called for both legs: an actor left out when bToTransition is true is destroyed with the old world.
    // Carry only actors you own (PersistentManager is a member of AMyGameMode).
    if (PersistentManager)
    {
        ActorList.Add(PersistentManager);
    }
}

void AMyGameMode::HandleSeamlessTravelPlayer(AController*& C)
{
    Super::HandleSeamlessTravelPlayer(C);
    RestorePlayerState(C);
}
```

Anything not in the actor list is destroyed with the old world. Put data that must outlive every map in `UGameInstance` or a `UGameInstanceSubsystem`.

---

## Pattern 5: Streaming Volume-Driven Interiors

**Best for:** non-partitioned maps with buildings or interiors that appear as the player approaches.

1. One sub-level per interior, for example `L_Building_Interior_01`.
2. Place an `ALevelStreamingVolume` covering the approach.
3. Set `StreamingUsage = SVB_LoadingAndVisibility` on the volume.
4. Link the sub-level through the streaming level's `EditorStreamingVolumes` array.
5. Set `MinTimeBetweenVolumeUnloadRequests` on the sub-level (a few seconds) so standing on the boundary does not thrash.

### Taking a level off volume control

```cpp
// MyBossRoomTrigger.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/Actor.h"
#include "MyBossRoomTrigger.generated.h"

UCLASS()
class MYGAME_API AMyBossRoomTrigger : public AActor
{
    GENERATED_BODY()

public:
    virtual void BeginPlay() override;
    virtual void NotifyActorBeginOverlap(AActor* OtherActor) override;

protected:
    UPROPERTY(EditAnywhere, Category = "MyGame|Streaming")
    FName BossArenaPackageName;
};
```

```cpp
// MyBossRoomTrigger.cpp
#include "MyBossRoomTrigger.h"
#include "Engine/LevelStreaming.h"
#include "GameFramework/Pawn.h"
#include "Kismet/GameplayStatics.h"

void AMyBossRoomTrigger::BeginPlay()
{
    Super::BeginPlay();

    if (ULevelStreaming* Streaming = UGameplayStatics::GetStreamingLevel(this, BossArenaPackageName))
    {
        Streaming->bDisableDistanceStreaming = true; // code owns this level from now on
    }
}

void AMyBossRoomTrigger::NotifyActorBeginOverlap(AActor* OtherActor)
{
    Super::NotifyActorBeginOverlap(OtherActor);

    if (!OtherActor || !OtherActor->IsA<APawn>())
    {
        return;
    }

    if (ULevelStreaming* Streaming = UGameplayStatics::GetStreamingLevel(this, BossArenaPackageName))
    {
        Streaming->SetShouldBeLoaded(true);
        Streaming->SetShouldBeVisible(true);
    }
}
```

`EStreamingVolumeUsage` in full: `SVB_Loading` (load, stay invisible), `SVB_LoadingAndVisibility` (the common case), `SVB_VisibilityBlockingOnLoad`, `SVB_BlockingOnLoad`, `SVB_LoadingNotVisible`.

---

## Dedicated Server Streaming

A dedicated server has no rendering, so nothing about streaming is camera-driven there.

- **World Partition:** server cell streaming is controlled by `wp.Runtime.EnableServerStreaming`, and streaming out by `wp.Runtime.EnableServerStreamingOut`. Query the effective setting with `UWorldPartition::IsServerStreamingEnabled()`. Sources on the server are the ones you register. `AServerStreamingLevelsVisibility` (`Streaming/ServerStreamingLevelsVisibility.h`) is the engine actor that tracks per-level server visibility, reachable through `UWorld::GetServerStreamingLevelsVisibility()`.
- **Sub-levels:** call `SetShouldBeLoaded` and `SetShouldBeVisible` explicitly. Volume streaming does not apply.
- **Blocking:** never set `bShouldBlockOnLoad` or `wp.Runtime.BlockOnSlowStreaming` on a server outside a map change — a stalled server stalls every client.

```cpp
// MyServerZoneSubsystem.h
#pragma once

#include "CoreMinimal.h"
#include "Subsystems/WorldSubsystem.h"
#include "MyServerZoneSubsystem.generated.h"

UCLASS()
class MYGAME_API UMyServerZoneSubsystem : public UTickableWorldSubsystem
{
    GENERATED_BODY()

public:
    virtual void Tick(float DeltaTime) override;
    virtual TStatId GetStatId() const override;

protected:
    void UpdateZonesForLocation(const FVector& Location);
};
```

```cpp
// MyServerZoneSubsystem.cpp
#include "MyServerZoneSubsystem.h"
#include "Engine/World.h"
#include "EngineUtils.h"
#include "GameFramework/Pawn.h"
#include "GameFramework/PlayerController.h"

TStatId UMyServerZoneSubsystem::GetStatId() const
{
    RETURN_QUICK_DECLARE_CYCLE_STAT(UMyServerZoneSubsystem, STATGROUP_Tickables);
}

void UMyServerZoneSubsystem::Tick(float DeltaTime)
{
    Super::Tick(DeltaTime);

    UWorld* World = GetWorld();
    if (World->GetNetMode() != NM_DedicatedServer)
    {
        return;
    }

    for (APlayerController* PlayerController : TActorRange<APlayerController>(World))
    {
        if (const APawn* Pawn = PlayerController->GetPawn())
        {
            UpdateZonesForLocation(Pawn->GetActorLocation());
        }
    }
}
```

`UTickableWorldSubsystem::Initialize` and `Deinitialize` are what enable and disable ticking, so any override of them must call `Super::`.

---

## Budget Guidelines

| Scenario | Cell or zone sizing | Simultaneously resident |
|---|---|---|
| Open world (World Partition) | Loading range tuned per runtime partition | Grid-driven; cap with `wp.Runtime.MaxLoadingStreamingCells` |
| Hub-and-spoke zones | One level per zone | Current zone plus immediate neighbours |
| Procedural rooms | One level per room | Tens, depending on actor counts per room |
| Interior streaming | One level per building | A handful around the player |

Level streaming work is spread across frames by design. `bShouldBlockOnLoad`, `UGameplayStatics::FlushLevelStreaming` and `UWorld::FlushLevelStreaming(EFlushLevelStreamingType::Full)` collapse it into one frame — acceptable only behind a loading screen. Keep dynamic actor counts per cell low; static geometry in distant cells is represented by HLOD proxies instead.
