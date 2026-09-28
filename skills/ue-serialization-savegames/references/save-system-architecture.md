# Save System Architecture (UE 5.8)

Production patterns for a complete save system: a slot manager subsystem, slot metadata with thumbnails and checksums, world-state capture through a save interface, migration, and the config/JSON/compression/encryption code that the main skill file links to. Every engine call here is verified against the UE 5.8 headers.

---

## Architecture Overview

```
Game code (GameMode, PlayerController, subsystems)
        |  calls the save manager
Save manager  (UGameInstanceSubsystem)
        |  slot routing, async coordination, migration
UGameplayStatics  /  ISaveGameSystem
        |  platform-agnostic byte I/O
```

The manager lives on the `UGameInstance` as a `UGameInstanceSubsystem` (`Subsystems/GameInstanceSubsystem.h:16`) so it survives level transitions. `USubsystem::Initialize(FSubsystemCollectionBase&)` and `Deinitialize()` are declared in `Subsystems/Subsystem.h:59,62`.

---

## 1. Slot Metadata

Keep a small metadata object in a fixed slot so the load menu can list slots without deserializing every payload.

```cpp
// MySaveMetadata.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/SaveGame.h"
#include "Misc/DateTime.h"
#include "MySaveMetadata.generated.h"

USTRUCT(BlueprintType)
struct FMySaveSlotInfo
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadOnly) int32 SlotIndex = -1;
    UPROPERTY(BlueprintReadOnly) bool bIsValid = false;
    UPROPERTY(BlueprintReadOnly) FString PlayerDisplayName;
    UPROPERTY(BlueprintReadOnly) FString MapName;
    UPROPERTY(BlueprintReadOnly) int32 PlayerLevel = 0;
    UPROPERTY(BlueprintReadOnly) FDateTime LastSaveTime;
    UPROPERTY(BlueprintReadOnly) float TotalPlayTimeSeconds = 0.f;

    /** Screenshot written next to the save; see section 7. */
    UPROPERTY(BlueprintReadOnly) FString ThumbnailPath;

    /** FCrc::MemCrc32 of the serialized payload; see section 8. */
    UPROPERTY(BlueprintReadOnly) int64 PayloadChecksum = 0;
};

UCLASS()
class MYGAME_API UMySaveMetadata : public USaveGame
{
    GENERATED_BODY()

public:
    static constexpr int32 MaxSlots = 10;
    static constexpr int32 UserIndex = 0;
    static const FString SlotName;

    UPROPERTY()
    TArray<FMySaveSlotInfo> Slots;

    void Init()
    {
        Slots.SetNum(MaxSlots);
        for (int32 Index = 0; Index < MaxSlots; ++Index)
        {
            Slots[Index].SlotIndex = Index;
            Slots[Index].bIsValid = false;
        }
    }

    void UpdateSlot(int32 Index, const FMySaveSlotInfo& Info)
    {
        if (!Slots.IsValidIndex(Index)) { return; }
        Slots[Index] = Info;
        Slots[Index].SlotIndex = Index;
        Slots[Index].bIsValid = true;
    }

    void ClearSlot(int32 Index)
    {
        if (!Slots.IsValidIndex(Index)) { return; }
        Slots[Index] = FMySaveSlotInfo();
        Slots[Index].SlotIndex = Index;
    }
};
```

```cpp
// MySaveMetadata.cpp
#include "MySaveMetadata.h"

const FString UMySaveMetadata::SlotName = TEXT("MySaveMetadata");
```

`UGameplayStatics::SaveGameToSlot` writes every non-transient `UPROPERTY`, so the `SaveGame` specifier is optional on a metadata object that is saved as a whole.

---

## 2. Save Manager Subsystem

The async completion delegates are plain (non-dynamic) delegates — `DECLARE_DELEGATE_ThreeParams` at `Kismet/GameplayStatics.h:44,47` — so their handlers must **not** carry `UFUNCTION()`. Blueprint-facing events use dynamic multicast delegates instead.

The manager below reads two fields beyond the `UMySaveGame` shown in the main skill file; add them to that class:

```cpp
// In UMySaveGame:
UPROPERTY(SaveGame) int32 PlayerLevel = 1;
UPROPERTY(SaveGame) float TotalPlayTimeSeconds = 0.f;
```

```cpp
// MySaveManager.h
#pragma once

#include "CoreMinimal.h"
#include "Subsystems/GameInstanceSubsystem.h"
#include "MySaveMetadata.h"
#include "MySaveGame.h"
#include "MySaveManager.generated.h"

DECLARE_DYNAMIC_MULTICAST_DELEGATE_TwoParams(FMyOnSaveComplete, int32, SlotIndex, bool, bSuccess);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_TwoParams(FMyOnLoadComplete, int32, SlotIndex, bool, bSuccess);

UCLASS()
class MYGAME_API UMySaveManager : public UGameInstanceSubsystem
{
    GENERATED_BODY()

public:
    virtual void Initialize(FSubsystemCollectionBase& Collection) override;
    virtual void Deinitialize() override;

    /** Loads the metadata object; called from Initialize. */
    UFUNCTION(BlueprintCallable, Category="Save")
    void LoadMetadata();

    UFUNCTION(BlueprintCallable, Category="Save")
    TArray<FMySaveSlotInfo> GetAllSlotInfo() const;

    UFUNCTION(BlueprintCallable, Category="Save")
    void AsyncSaveToSlot(int32 SlotIndex);

    UFUNCTION(BlueprintCallable, Category="Save")
    void AsyncLoadFromSlot(int32 SlotIndex);

    /** Blocks the game thread; only safe before the first frame. */
    UFUNCTION(BlueprintCallable, Category="Save")
    bool SyncLoadFromSlot(int32 SlotIndex);

    UFUNCTION(BlueprintCallable, Category="Save")
    void DeleteSlot(int32 SlotIndex);

    UFUNCTION(BlueprintPure, Category="Save")
    bool IsSaveInProgress() const { return bSaveInProgress; }

    UFUNCTION(BlueprintPure, Category="Save")
    UMySaveGame* GetCurrentSave() const { return CurrentSave; }

    UFUNCTION(BlueprintCallable, Category="Save")
    void StartAutoSave(int32 SlotIndex, float IntervalSeconds = 300.f);

    UFUNCTION(BlueprintCallable, Category="Save")
    void StopAutoSave();

    UPROPERTY(BlueprintAssignable, Category="Save")
    FMyOnSaveComplete OnSaveComplete;

    UPROPERTY(BlueprintAssignable, Category="Save")
    FMyOnLoadComplete OnLoadComplete;

private:
    static FString MakeSlotName(int32 SlotIndex);

    void PopulateSaveData(UMySaveGame* Save);
    void ApplySaveData(UMySaveGame* Save);
    void RunMigrations(UMySaveGame* Save);
    void WriteMetadata();
    FMySaveSlotInfo BuildSlotInfo(int32 SlotIndex, const UMySaveGame* Save) const;

    // Bound to plain delegates — no UFUNCTION here.
    void HandleSaveComplete(const FString& SlotName, const int32 UserIndex, bool bSuccess);
    void HandleLoadComplete(const FString& SlotName, const int32 UserIndex, USaveGame* LoadedSave);

    UPROPERTY()
    TObjectPtr<UMySaveMetadata> Metadata = nullptr;

    UPROPERTY()
    TObjectPtr<UMySaveGame> CurrentSave = nullptr;

    int32 ActiveSlotIndex = -1;
    bool bSaveInProgress = false;

    FTimerHandle AutoSaveTimerHandle;
};
```

```cpp
// MySaveManager.cpp
#include "MySaveManager.h"
#include "Kismet/GameplayStatics.h"
#include "Engine/World.h"
#include "TimerManager.h"

void UMySaveManager::Initialize(FSubsystemCollectionBase& Collection)
{
    Super::Initialize(Collection);
    LoadMetadata();
}

void UMySaveManager::Deinitialize()
{
    StopAutoSave();
    Super::Deinitialize();
}

FString UMySaveManager::MakeSlotName(int32 SlotIndex)
{
    return FString::Printf(TEXT("MyGameSave_%02d"), SlotIndex);
}

void UMySaveManager::LoadMetadata()
{
    if (UGameplayStatics::DoesSaveGameExist(UMySaveMetadata::SlotName, UMySaveMetadata::UserIndex))
    {
        Metadata = Cast<UMySaveMetadata>(UGameplayStatics::LoadGameFromSlot(
            UMySaveMetadata::SlotName, UMySaveMetadata::UserIndex));
    }

    if (!Metadata)
    {
        Metadata = Cast<UMySaveMetadata>(
            UGameplayStatics::CreateSaveGameObject(UMySaveMetadata::StaticClass()));
        if (Metadata) { Metadata->Init(); }
    }
}

TArray<FMySaveSlotInfo> UMySaveManager::GetAllSlotInfo() const
{
    return Metadata ? Metadata->Slots : TArray<FMySaveSlotInfo>();
}

void UMySaveManager::AsyncSaveToSlot(int32 SlotIndex)
{
    if (bSaveInProgress)
    {
        UE_LOG(LogMyGame, Warning, TEXT("Save already in progress; request ignored."));
        return;
    }

    if (!CurrentSave)
    {
        CurrentSave = Cast<UMySaveGame>(
            UGameplayStatics::CreateSaveGameObject(UMySaveGame::StaticClass()));
    }
    if (!CurrentSave) { return; }

    PopulateSaveData(CurrentSave);
    CurrentSave->SaveVersion = MySaveStage::Latest; // MySaveStage: section 5 (declare it above this point)
    ActiveSlotIndex = SlotIndex;
    bSaveInProgress = true;

    UGameplayStatics::AsyncSaveGameToSlot(
        CurrentSave, MakeSlotName(SlotIndex), UMySaveMetadata::UserIndex,
        FAsyncSaveGameToSlotDelegate::CreateUObject(this, &UMySaveManager::HandleSaveComplete));
}

void UMySaveManager::HandleSaveComplete(const FString& SlotName, const int32 UserIndex, bool bSuccess)
{
    bSaveInProgress = false;

    if (bSuccess && Metadata)
    {
        Metadata->UpdateSlot(ActiveSlotIndex, BuildSlotInfo(ActiveSlotIndex, CurrentSave));
        WriteMetadata();
    }
    else if (!bSuccess)
    {
        UE_LOG(LogMyGame, Error, TEXT("Async save failed for slot '%s'."), *SlotName);
    }

    OnSaveComplete.Broadcast(ActiveSlotIndex, bSuccess);
}

void UMySaveManager::AsyncLoadFromSlot(int32 SlotIndex)
{
    ActiveSlotIndex = SlotIndex;
    UGameplayStatics::AsyncLoadGameFromSlot(
        MakeSlotName(SlotIndex), UMySaveMetadata::UserIndex,
        FAsyncLoadGameFromSlotDelegate::CreateUObject(this, &UMySaveManager::HandleLoadComplete));
}

void UMySaveManager::HandleLoadComplete(const FString& SlotName, const int32 UserIndex, USaveGame* LoadedSave)
{
    CurrentSave = Cast<UMySaveGame>(LoadedSave);
    const bool bLoaded = CurrentSave != nullptr;

    if (!bLoaded)
    {
        UE_LOG(LogMyGame, Warning, TEXT("Slot '%s' missing or unreadable; using a fresh save."), *SlotName);
        CurrentSave = Cast<UMySaveGame>(
            UGameplayStatics::CreateSaveGameObject(UMySaveGame::StaticClass()));
    }

    RunMigrations(CurrentSave);
    ApplySaveData(CurrentSave);
    OnLoadComplete.Broadcast(ActiveSlotIndex, bLoaded);
}

bool UMySaveManager::SyncLoadFromSlot(int32 SlotIndex)
{
    USaveGame* Loaded = UGameplayStatics::LoadGameFromSlot(
        MakeSlotName(SlotIndex), UMySaveMetadata::UserIndex);

    CurrentSave = Cast<UMySaveGame>(Loaded);
    ActiveSlotIndex = SlotIndex;

    if (!CurrentSave)
    {
        CurrentSave = Cast<UMySaveGame>(
            UGameplayStatics::CreateSaveGameObject(UMySaveGame::StaticClass()));
        return false;
    }

    RunMigrations(CurrentSave);
    ApplySaveData(CurrentSave);
    return true;
}

void UMySaveManager::DeleteSlot(int32 SlotIndex)
{
    UGameplayStatics::DeleteGameInSlot(MakeSlotName(SlotIndex), UMySaveMetadata::UserIndex);

    if (Metadata)
    {
        Metadata->ClearSlot(SlotIndex);
        WriteMetadata();
    }
}

void UMySaveManager::WriteMetadata()
{
    if (Metadata)
    {
        UGameplayStatics::SaveGameToSlot(
            Metadata, UMySaveMetadata::SlotName, UMySaveMetadata::UserIndex);
    }
}

FMySaveSlotInfo UMySaveManager::BuildSlotInfo(int32 SlotIndex, const UMySaveGame* Save) const
{
    FMySaveSlotInfo Info;
    Info.SlotIndex = SlotIndex;
    Info.bIsValid = true;
    Info.LastSaveTime = FDateTime::Now();

    if (Save)
    {
        Info.PlayerLevel = Save->PlayerLevel;
        Info.TotalPlayTimeSeconds = Save->TotalPlayTimeSeconds;
    }

    if (const UGameInstance* GI = GetGameInstance())
    {
        if (const UWorld* World = GI->GetWorld())
        {
            Info.MapName = World->GetMapName();
        }
    }

    return Info;
}

void UMySaveManager::StartAutoSave(int32 SlotIndex, float IntervalSeconds)
{
    UGameInstance* GI = GetGameInstance();
    UWorld* World = GI ? GI->GetWorld() : nullptr;
    if (!World) { return; }

    World->GetTimerManager().SetTimer(
        AutoSaveTimerHandle,
        FTimerDelegate::CreateUObject(this, &UMySaveManager::AsyncSaveToSlot, SlotIndex),
        IntervalSeconds,
        /*InbLoop=*/true);
}

void UMySaveManager::StopAutoSave()
{
    UGameInstance* GI = GetGameInstance();
    if (UWorld* World = GI ? GI->GetWorld() : nullptr)
    {
        World->GetTimerManager().ClearTimer(AutoSaveTimerHandle);
    }
}
```

`FTimerManager::SetTimer(FTimerHandle&, FTimerDelegate const&, float InRate, bool InbLoop, float InFirstDelay = -1.f)` is declared at `Engine/Public/TimerManager.h:178`. Because `UMySaveManager` is a `UGameInstanceSubsystem`, the engine constructs it automatically when the `GameInstance` initializes; game code reaches it with `GetGameInstance()->GetSubsystem<UMySaveManager>()`.

---

## 3. Actor Save Interface and FGuid Identity

Give every persistent actor a stable `FGuid` and a pair of hooks so subsystems can prepare their own state before the snapshot is taken.

```cpp
// MySaveable.h
#pragma once

#include "CoreMinimal.h"
#include "UObject/Interface.h"
#include "Misc/Guid.h"
#include "MySaveable.generated.h"

UINTERFACE(MinimalAPI, BlueprintType)
class UMySaveable : public UInterface
{
    GENERATED_BODY()
};

class MYGAME_API IMySaveable
{
    GENERATED_BODY()

public:
    /** Stable identity across sessions; generate once with FGuid::NewGuid() and store it. */
    UFUNCTION(BlueprintNativeEvent, BlueprintCallable, Category="Save")
    FGuid GetSaveId() const;

    UFUNCTION(BlueprintNativeEvent, BlueprintCallable, Category="Save")
    void OnBeforeSave();

    UFUNCTION(BlueprintNativeEvent, BlueprintCallable, Category="Save")
    void OnAfterRestore();
};
```

```cpp
// MyPersistentActor.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/Actor.h"
#include "MySaveable.h"
#include "MyPersistentActor.generated.h"

UCLASS()
class MYGAME_API AMyPersistentActor : public AActor, public IMySaveable
{
    GENERATED_BODY()

public:
    AMyPersistentActor();

    virtual FGuid GetSaveId_Implementation() const override { return SaveId; }
    virtual void OnBeforeSave_Implementation() override;
    virtual void OnAfterRestore_Implementation() override;

protected:
    /** Assigned in the constructor; VisibleAnywhere so designers can see it but not retype it. */
    UPROPERTY(VisibleAnywhere, SaveGame, Category="Save")
    FGuid SaveId;

    UPROPERTY(SaveGame) float CurrentHealth = 100.f;
    UPROPERTY(SaveGame) bool bHasBeenLooted = false;
};
```

```cpp
// MyPersistentActor.cpp
#include "MyPersistentActor.h"

AMyPersistentActor::AMyPersistentActor()
{
    PrimaryActorTick.bCanEverTick = false;
    SaveId = FGuid::NewGuid();
}

void AMyPersistentActor::OnBeforeSave_Implementation()
{
    // Flush any runtime-only state into UPROPERTY(SaveGame) fields here.
}

void AMyPersistentActor::OnAfterRestore_Implementation()
{
    // Re-apply restored values to components, materials, AI state, and so on.
}
```

---

## 4. World-State Capture

One record per actor, serialized through `FObjectAndNameAsStringProxyArchive` with `ArIsSaveGame = true` so only `UPROPERTY(SaveGame)` fields are written (`UObject/ObjectMacros.h:458`, `Serialization/Archive.h:942`).

```cpp
// MyWorldState.h
#pragma once

#include "CoreMinimal.h"
#include "Misc/Guid.h"
#include "UObject/SoftObjectPtr.h"
#include "MyWorldState.generated.h"

class AActor;

USTRUCT()
struct FMyActorRecord
{
    GENERATED_BODY()

    UPROPERTY() FGuid SaveId;
    UPROPERTY() TSoftClassPtr<AActor> ActorClass;
    UPROPERTY() FTransform ActorTransform;
    UPROPERTY() TArray<uint8> ByteData;
};

USTRUCT()
struct FMyWorldState
{
    GENERATED_BODY()

    UPROPERTY() FString MapName;
    UPROPERTY() TArray<FMyActorRecord> Records;
};
```

```cpp
// MyWorldState.cpp
#include "MyWorldState.h"
#include "MySaveable.h"
#include "Serialization/MemoryWriter.h"
#include "Serialization/MemoryReader.h"
#include "Serialization/ObjectAndNameAsStringProxyArchive.h"
#include "Kismet/GameplayStatics.h"
#include "GameFramework/Actor.h"
#include "Engine/Engine.h"
#include "Engine/World.h"

void CaptureWorldState(UObject* WorldContextObject, FMyWorldState& OutState)
{
    TArray<AActor*> Saveables;
    UGameplayStatics::GetAllActorsWithInterface(
        WorldContextObject, UMySaveable::StaticClass(), Saveables); // GameplayStatics.h:107

    OutState.Records.Reset();
    OutState.MapName = UGameplayStatics::GetCurrentLevelName(WorldContextObject); // :358

    for (AActor* Actor : Saveables)
    {
        if (!IsValid(Actor)) { continue; }

        IMySaveable::Execute_OnBeforeSave(Actor);

        FMyActorRecord& Record = OutState.Records.AddDefaulted_GetRef();
        Record.SaveId = IMySaveable::Execute_GetSaveId(Actor);
        Record.ActorClass = Actor->GetClass();
        Record.ActorTransform = Actor->GetActorTransform();

        FMemoryWriter MemWriter(Record.ByteData, /*bIsPersistent=*/true);
        FObjectAndNameAsStringProxyArchive Ar(MemWriter, /*bInLoadIfFindFails=*/true);
        Ar.ArIsSaveGame = true;
        Actor->Serialize(Ar);
    }
}

void RestoreWorldState(UObject* WorldContextObject, const FMyWorldState& State)
{
    TArray<AActor*> Saveables;
    UGameplayStatics::GetAllActorsWithInterface(
        WorldContextObject, UMySaveable::StaticClass(), Saveables);

    TMap<FGuid, AActor*> ById;
    for (AActor* Actor : Saveables)
    {
        if (IsValid(Actor))
        {
            ById.Add(IMySaveable::Execute_GetSaveId(Actor), Actor);
        }
    }

    UWorld* World = GEngine ? GEngine->GetWorldFromContextObject(
        WorldContextObject, EGetWorldErrorMode::LogAndReturnNull) : nullptr;

    for (const FMyActorRecord& Record : State.Records)
    {
        AActor* Target = ById.FindRef(Record.SaveId);

        // Actors placed in the level already exist; runtime-spawned ones must be respawned
        // before their bytes are applied, so the class is stored in the record.
        if (!Target && World)
        {
            if (UClass* Class = Record.ActorClass.LoadSynchronous())
            {
                FActorSpawnParameters Params;
                Params.SpawnCollisionHandlingOverride =
                    ESpawnActorCollisionHandlingMethod::AlwaysSpawn;
                Target = World->SpawnActor<AActor>(Class, Record.ActorTransform, Params);
            }
        }
        if (!Target) { continue; }

        Target->SetActorTransform(Record.ActorTransform);

        FMemoryReader MemReader(Record.ByteData, /*bIsPersistent=*/true);
        FObjectAndNameAsStringProxyArchive Ar(MemReader, /*bInLoadIfFindFails=*/true);
        Ar.ArIsSaveGame = true;
        Target->Serialize(Ar);

        IMySaveable::Execute_OnAfterRestore(Target);
    }
}
```

Ordering matters: restore the transform before the byte block if the actor's `UPROPERTY(SaveGame)` fields depend on it, and always call `OnAfterRestore` last so components see final values. `TActorIterator<AActor>` from `EngineUtils.h` is the alternative when you need a class filter instead of an interface filter.

---

## 5. Migration Pipeline

Run migrations from exactly one place so the sync and async load paths cannot diverge.

```cpp
// MySaveManager.cpp (continued)
namespace MySaveStage
{
    constexpr int32 Initial = 0, AddedInventory = 1, AddedAbilities = 2, SoftRefForWeapon = 3;
    constexpr int32 Latest = SoftRefForWeapon;
}

void UMySaveManager::RunMigrations(UMySaveGame* Save)
{
    if (!Save || Save->SaveVersion == MySaveStage::Latest) { return; }

    UE_LOG(LogMyGame, Log, TEXT("Migrating save %d -> %d"), Save->SaveVersion, MySaveStage::Latest);

    if (Save->SaveVersion < MySaveStage::AddedInventory)
    {
        Save->InventoryItems.Reset();
    }
    if (Save->SaveVersion < MySaveStage::AddedAbilities)
    {
        Save->AbilityLevels.Reset();
    }
    if (Save->SaveVersion < MySaveStage::SoftRefForWeapon)
    {
        // The old build stored a weapon FName; the new one stores a path.
        Save->LastEquippedWeapon.Reset();
    }

    Save->SaveVersion = MySaveStage::Latest; // stamp only after every step has run
}
```

For raw `FArchive` payloads use `FCustomVersionRegistration` instead — see the Versioning section of the main skill file. The version only rides inside a `USaveGame` file (whose header stores all registered custom versions, `GameplayStatics.cpp:233`); a standalone blob must serialize `Ar.GetCustomVersions()` itself.

---

## 6. Binary Blob Inside a USaveGame

When a subsystem owns a large, self-describing payload, store it as `TArray<uint8>` on the save object and serialize it manually.

```cpp
// In UMySaveGame:
//     UPROPERTY() TArray<uint8> QuestBlob;

bool UMyQuestSubsystem::WriteBlob(TArray<uint8>& OutBlob) const
{
    FMemoryWriter Writer(OutBlob, /*bIsPersistent=*/true);

    int32 BlobVersion = 2;
    int32 NumQuests = ActiveQuests.Num();
    Writer << BlobVersion << NumQuests;

    for (const FMyQuestState& Quest : ActiveQuests)
    {
        FName QuestId = Quest.QuestId;
        int32 Stage = Quest.CurrentStage;
        Writer << QuestId << Stage;
    }

    return !Writer.IsError();
}

bool UMyQuestSubsystem::ReadBlob(const TArray<uint8>& Blob)
{
    if (Blob.IsEmpty()) { return true; } // nothing saved yet

    FMemoryReader Reader(Blob, /*bIsPersistent=*/true);

    int32 BlobVersion = 0;
    int32 NumQuests = 0;
    Reader << BlobVersion << NumQuests;
    if (Reader.IsError() || BlobVersion < 1) { return false; }

    ActiveQuests.Empty(NumQuests);
    for (int32 Index = 0; Index < NumQuests; ++Index)
    {
        FName QuestId;
        int32 Stage = 0;
        Reader << QuestId << Stage;
        if (Reader.IsError()) { return false; }

        FMyQuestState& Quest = ActiveQuests.AddDefaulted_GetRef();
        Quest.QuestId = QuestId;
        Quest.CurrentStage = Stage;
    }

    return !Reader.IsError();
}
```

---

## 7. Per-Player Saves and Thumbnails

`ULocalPlayerSaveGame` (`GameFramework/SaveGame.h:47`) resolves the platform user itself. Use it for keybinds, accessibility and per-profile progress; keep shared world state in the manager's slots.

A subclass with versioned migration and pre-save sanitising:

```cpp
// MyLocalPlayerSave.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/SaveGame.h"
#include "MyLocalPlayerSave.generated.h"

UCLASS()
class MYGAME_API UMyLocalPlayerSave : public ULocalPlayerSaveGame
{
    GENERATED_BODY()

public:
    virtual int32 GetLatestDataVersion() const override;
    virtual void HandlePreSave() override;
    virtual void HandlePostLoad() override;
    virtual void HandlePostSave(bool bSuccess) override;

    UPROPERTY(SaveGame) TMap<FName, int32> UnlockedAbilities;
    UPROPERTY(SaveGame) float MouseSensitivity = 1.f;
};
```

```cpp
// MyLocalPlayerSave.cpp — include "MyLocalPlayerSave.h".
int32 UMyLocalPlayerSave::GetLatestDataVersion() const { return 3; }

void UMyLocalPlayerSave::HandlePreSave()
{
    Super::HandlePreSave();
    MouseSensitivity = FMath::Clamp(MouseSensitivity, 0.1f, 10.f);
}

void UMyLocalPlayerSave::HandlePostLoad()
{
    Super::HandlePostLoad();
    // GetSavedDataVersion() is the GetLatestDataVersion() value at the time of the last save.
    const int32 LoadedVersion = GetSavedDataVersion();
    if (LoadedVersion < 2) { UnlockedAbilities.Add(TEXT("Dash"), 1); }
    if (LoadedVersion < 3) { MouseSensitivity = 1.f; }
}

void UMyLocalPlayerSave::HandlePostSave(bool bSuccess)
{
    Super::HandlePostSave(bSuccess);
    if (!bSuccess) { UE_LOG(LogMyGame, Error, TEXT("Local player save failed.")); }
}
```

Loading it asynchronously from the player controller:

```cpp
// MyPlayerController.h (the members used below)
//     void LoadPlayerProfile();
//     void HandleProfileLoaded(ULocalPlayerSaveGame* SaveGame);
//     UPROPERTY() TObjectPtr<UMyLocalPlayerSave> Profile = nullptr;

// MyPlayerController.cpp
void AMyPlayerController::LoadPlayerProfile()
{
    const ULocalPlayer* LocalPlayer = GetLocalPlayer();
    if (!LocalPlayer) { return; }

    const bool bScheduled = ULocalPlayerSaveGame::AsyncLoadOrCreateSaveGameForLocalPlayer(
        UMyLocalPlayerSave::StaticClass(),
        LocalPlayer,
        FString::Printf(TEXT("MyPlayerProfile_%d"), LocalPlayer->GetControllerId()),
        FOnLocalPlayerSaveGameLoadedNative::CreateUObject(
            this, &AMyPlayerController::HandleProfileLoaded));

    if (!bScheduled)
    {
        UE_LOG(LogMyGame, Error, TEXT("Could not schedule the local player save load."));
    }
}

void AMyPlayerController::HandleProfileLoaded(ULocalPlayerSaveGame* SaveGame)
{
    Profile = Cast<UMyLocalPlayerSave>(SaveGame);
    if (!Profile)
    {
        UE_LOG(LogMyGame, Error, TEXT("Local player save cast failed."));
        return;
    }
    // Apply Profile->MouseSensitivity, Profile->UnlockedAbilities, and so on.
}
```

`ULocalPlayer::GetControllerId()` (`Engine/LocalPlayer.h:522`) and `ULocalPlayerSaveGame::GetPlatformUserIndex()` / `GetPlatformUserId()` are the two supported ways to key per-user data; never derive slot names from a display name.

A slot thumbnail is a screenshot written beside the save. Capture it with the `HighResShot` console command or your own render-target readback, save the bytes with `FFileHelper::SaveArrayToFile` (`Misc/FileHelper.h:185`) under `FPaths::ProjectSavedDir()` (`Misc/Paths.h:290`), and store the resulting path in `FMySaveSlotInfo::ThumbnailPath`.

---

## 8. Checksums, Compression, and Encryption

```cpp
#include "Misc/Crc.h"
#include "Misc/Compression.h"
#include "Misc/AES.h"
#include "Kismet/GameplayStatics.h"

// Checksum: FCrc::MemCrc32(const void* Data, int32 Length, uint32 CRC = 0) — Misc/Crc.h:29
uint32 ComputeChecksum(const TArray<uint8>& Payload)
{
    return FCrc::MemCrc32(Payload.GetData(), Payload.Num());
}

// Compression: FCompression takes an FName format — NAME_Zlib, NAME_Oodle, NAME_Gzip, NAME_LZ4.
bool CompressPayload(const TArray<uint8>& Raw, TArray<uint8>& OutCompressed)
{
    int32 CompressedSize = FCompression::CompressMemoryBound(NAME_Zlib, Raw.Num()); // :73
    OutCompressed.SetNumUninitialized(CompressedSize);

    if (!FCompression::CompressMemory(NAME_Zlib, OutCompressed.GetData(), CompressedSize,
                                      Raw.GetData(), Raw.Num()))                    // :108
    {
        return false;
    }

    OutCompressed.SetNum(CompressedSize, EAllowShrinking::No);
    return true;
}

bool DecompressPayload(const TArray<uint8>& Compressed, int32 UncompressedSize, TArray<uint8>& OutRaw)
{
    OutRaw.SetNumUninitialized(UncompressedSize);
    return FCompression::UncompressMemory(NAME_Zlib, OutRaw.GetData(), UncompressedSize,
                                          Compressed.GetData(), Compressed.Num());  // :143
}
```

`UncompressMemory` needs the exact uncompressed size, so write it into your header alongside the checksum before the compressed block.

```cpp
// FAES::FAESKey::KeySize is 32 and FAES::AESBlockSize is 16 (Misc/AES.h:21,28).
// Never truncate a shorter passphrase with Left(32): zero-pad it instead.
static FAES::FAESKey MakeAESKey(const FString& Passphrase)
{
    FAES::FAESKey Key;
    FMemory::Memzero(Key.Key, FAES::FAESKey::KeySize);
    const FTCHARToUTF8 Utf8(*Passphrase);
    FMemory::Memcpy(Key.Key, Utf8.Get(), FMath::Min<int32>(Utf8.Length(), FAES::FAESKey::KeySize));
    return Key;
}

void EncryptPayload(TArray<uint8>& Data, const FString& Passphrase)
{
    const int32 PaddedSize = Align(Data.Num(), FAES::AESBlockSize);
    Data.SetNumZeroed(PaddedSize);
    FAES::EncryptData(Data.GetData(), PaddedSize, MakeAESKey(Passphrase)); // Misc/AES.h:68
}

void DecryptPayload(TArray<uint8>& Data, const FString& Passphrase)
{
    FAES::DecryptData(Data.GetData(), Data.Num(), MakeAESKey(Passphrase));  // Misc/AES.h:96
}
```

To apply any of this to a `USaveGame`, serialize it to bytes yourself and write the bytes to the slot: `UGameplayStatics::SaveGameToMemory(USaveGame*, TArray<uint8>&)` (`GameplayStatics.h:1134`) then `SaveDataToSlot(const TArray<uint8>&, SlotName, UserIndex)` (`:1143`); on load, `LoadDataFromSlot` (`:1191`) then `LoadGameFromMemory` (`:1182`). Encryption only raises the cost of casual save editing — anything competitive needs server authority.

---

## 9. Config and Settings

```cpp
// MyProjectSettings.h
#pragma once

#include "CoreMinimal.h"
#include "Engine/DeveloperSettings.h"
#include "MyProjectSettings.generated.h"

UCLASS(Config=Game, DefaultConfig, meta=(DisplayName="My Game Save Settings"))
class MYGAME_API UMyProjectSettings : public UDeveloperSettings
{
    GENERATED_BODY()

public:
    UPROPERTY(Config, EditAnywhere, Category="Save") int32 MaxSaveSlots = 5;
    UPROPERTY(Config, EditAnywhere, Category="Save") bool bEnableAutoSave = true;
    UPROPERTY(Config, EditAnywhere, Category="Save") float AutoSaveIntervalSeconds = 300.f;

    static const UMyProjectSettings* Get() { return GetDefault<UMyProjectSettings>(); }

    /** Optional: move the page in Project Settings. */
    virtual FName GetCategoryName() const override { return TEXT("Game"); }
};
```

`UDeveloperSettings` is declared at `Engine/DeveloperSettings.h:23` with `GetContainerName`, `GetCategoryName` and `GetSectionName` at `:31-35`; the module is `DeveloperSettings`. `Config=Game` selects `DefaultGame.ini` and `DefaultConfig` writes edits back to the project default rather than the per-user file.

```cpp
// MyGameUserSettings.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/GameUserSettings.h"
#include "MyGameUserSettings.generated.h"

UCLASS()
class MYGAME_API UMyGameUserSettings : public UGameUserSettings
{
    GENERATED_BODY()

public:
    UPROPERTY(Config, BlueprintReadWrite, Category="Audio") float MasterVolume = 1.f;
    UPROPERTY(Config, BlueprintReadWrite, Category="UI") bool bSubtitlesEnabled = true;
};
```

```cpp
// Applying and persisting them.
#include "Engine/Engine.h"
#include "Misc/ConfigCacheIni.h"

void ApplyMySettings(float NewVolume)
{
    UMyGameUserSettings* Settings =
        Cast<UMyGameUserSettings>(GEngine ? GEngine->GetGameUserSettings() : nullptr); // Engine.h:3811
    if (!Settings) { return; }

    Settings->MasterVolume = NewVolume;
    Settings->ApplySettings(/*bCheckForCommandLineOverrides=*/false); // GameUserSettings.h:49
    Settings->SaveSettings();                                        // GameUserSettings.h:311
}

void ReadRawIniValue()
{
    FString Value;
    GConfig->GetString(TEXT("/Script/MyGame.MyProjectSettings"), TEXT("MaxSaveSlots"),
                       Value, GGameIni);                                   // ConfigCacheIni.h:1402
    GConfig->SetString(TEXT("/Script/MyGame.MyProjectSettings"), TEXT("MaxSaveSlots"),
                       TEXT("8"), GGameIni);                               // ConfigCacheIni.h:1408
    GConfig->Flush(/*bRemoveFromCache=*/false, GGameIni);                  // ConfigCacheIni.h:1395
}
```

Register the settings subclass in `DefaultEngine.ini`:

```ini
[/Script/Engine.Engine]
GameUserSettingsClassName=/Script/MyGame.MyGameUserSettings
```

`UObject::SaveConfig()` (`UObject/Object.h:1283`) and `LoadConfig()` (`:1389`) move `UPROPERTY(Config)` fields between an object and its `[/Script/ModuleName.ClassName]` section; override `OverrideConfigSection(FString& SectionName)` (`:1370`) for a custom section.

---

## 10. JSON Saves

Useful for server-authoritative saves, debug dumps and hand-editable test data. Modules: `Json` and `JsonUtilities`.

```cpp
#include "JsonObjectConverter.h"
#include "Misc/FileHelper.h"
#include "Misc/Paths.h"

bool DumpStateToJson(const FMyWorldState& State)
{
    FString Json;
    if (!FJsonObjectConverter::UStructToJsonObjectString(State, Json)) // JsonObjectConverter.h:156
    {
        return false;
    }
    return FFileHelper::SaveStringToFile(
        Json, *(FPaths::ProjectSavedDir() / TEXT("Debug/WorldState.json"))); // FileHelper.h:196
}

bool LoadStateFromJson(FMyWorldState& OutState)
{
    FString Json;
    if (!FFileHelper::LoadFileToString(
            Json, *(FPaths::ProjectSavedDir() / TEXT("Debug/WorldState.json")))) // :130
    {
        return false;
    }
    return FJsonObjectConverter::JsonObjectStringToUStruct(Json, &OutState); // :313
}
```

JSON drops `TArray<uint8>` readability and roughly triples file size, so keep it for tooling and ship the binary `USaveGame` path.

---

## 11. Platform Notes

| Platform | What changes |
|---|---|
| PC / Mac | `UGameplayStatics` writes under the project's `Saved/SaveGames/`. `UserIndex` is usually 0. |
| Console | The user index maps to an account. Use `ULocalPlayerSaveGame::GetPlatformUserIndex()` or `GetPlatformUserId()`; never hardcode 0. |
| Mobile | A sandboxed app directory managed by the platform backend; watch total save size. |
| PIE | Saves land in the project's `Saved/SaveGames/`. Clear them between runs with `UGameplayStatics::DeleteGameInSlot`. |

`FGenericSaveGameSystem::GetSaveGamePath` (`Engine/Public/SaveGameSystem.h:169`) exists only on the generic desktop backend and is for diagnostics. For slot enumeration and existence checks with error detail, go through `ISaveGameSystem::GetSaveGameNames` and `DoesSaveGameExistWithResult` (`:46,43`) via `IPlatformFeaturesModule::Get().GetSaveGameSystem()` (`Engine/Public/PlatformFeatures.h:41`).
