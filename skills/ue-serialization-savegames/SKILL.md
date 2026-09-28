---
name: ue-serialization-savegames
description: "Use when implementing save/load, player progress persistence, slot management, or binary serialization in Unreal Engine C++. Also use when the user mentions 'save game', 'USaveGame', 'SaveGameToSlot', 'AsyncSaveGameToSlot', 'LoadGameFromSlot', 'DoesSaveGameExist', 'DeleteGameInSlot', 'ULocalPlayerSaveGame', 'UPROPERTY(SaveGame)', 'FArchive', 'FMemoryWriter', 'FCustomVersion', 'save versioning', 'migrate old saves', 'save file corrupted', 'UDeveloperSettings', or 'GConfig'. For asset references, see ue-data-assets-tables; for work off the game thread, see ue-async-threading; for streamed level state, see ue-world-level-streaming."
metadata:
  version: "2.0.0"
  engine: "5.8"
---

# UE Serialization & Save Games

Target engine: **UE 5.8**. APIs below are verified against the 5.8 headers; older forms are listed under "Deprecated — do not use".

This skill covers persisting game state: `USaveGame` objects through `UGameplayStatics`, the platform `ISaveGameSystem`, `FArchive` binary serialization, `UPROPERTY(SaveGame)` actor snapshots, save versioning, and config/settings storage. Build.cs modules: `Core`, `CoreUObject`, `Engine`; add `DeveloperSettings` to `PublicDependencyModuleNames` for `UDeveloperSettings`, and `Json` + `JsonUtilities` to `PrivateDependencyModuleNames` for `FJsonObjectConverter`.

## Context

Read `.agents/ue-project-context.md` if it exists (module names, conventions, enabled plugins, GAS/networking setup). Do not stop if it is missing.

Identify the area from the request and the codebase. Ask only when two plausible readings would produce different code.

| Request is about… | Go to |
|---|---|
| A save object, slot names, save/load/delete | [USaveGame and the Slot API](#usavegame-and-the-slot-api) |
| Not blocking the game thread, callbacks | [Async Save and Load](#async-save-and-load) |
| Per-player saves, settings profiles, split-screen | [ULocalPlayerSaveGame](#ulocalplayersavegame) |
| Raw bytes, `FArchive`, memory buffers | [FArchive and Binary Serialization](#farchive-and-binary-serialization) |
| Snapshotting actors, world state, transforms | [Actor State with UPROPERTY(SaveGame)](#actor-state-with-upropertysavegame) |
| A `USTRUCT` needing its own `Serialize`; old saves breaking after a patch | [USTRUCT Custom Serialization](#ustruct-custom-serialization), [Versioning](#versioning) |
| Console/cloud storage, slot enumeration, save UI | [ISaveGameSystem and Platform Storage](#isavegamesystem-and-platform-storage) |
| Project settings, user options, `.ini`, JSON, compression | [Config, Settings, and File Formats](#config-settings-and-file-formats) |
| Slot manager subsystem, metadata, thumbnails, checksums, encryption | [Save system architecture](references/save-system-architecture.md) |

## USaveGame and the Slot API

`USaveGame` is an abstract `UObject` declared in `GameFramework/SaveGame.h:23`. Subclass it, add `UPROPERTY` fields, and route everything through `UGameplayStatics`.

**`UGameplayStatics::SaveGameToSlot` writes every non-transient `UPROPERTY`** — it does not filter on the `SaveGame` flag (`Kismet/GameplayStatics.h:1159`). Mark fields `Transient` to exclude them. The `SaveGame` specifier only matters for archives with `ArIsSaveGame` set — see [Actor State](#actor-state-with-upropertysavegame).

```cpp
// MySaveGame.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/SaveGame.h"
#include "UObject/SoftObjectPath.h"
#include "MySaveGame.generated.h"

USTRUCT(BlueprintType)
struct FMyInventoryItem
{
    GENERATED_BODY()

    UPROPERTY(SaveGame) FName ItemId;
    UPROPERTY(SaveGame) int32 Quantity = 0;
};

UCLASS(BlueprintType)
class MYGAME_API UMySaveGame : public USaveGame
{
    GENERATED_BODY()

public:
    UPROPERTY(SaveGame) int32 SaveVersion = 0;   // bump alongside MySaveStage::Latest
    UPROPERTY(SaveGame) float PlayerHealth = 100.f;
    UPROPERTY(SaveGame) FVector LastCheckpoint = FVector::ZeroVector;
    UPROPERTY(SaveGame) TArray<FMyInventoryItem> InventoryItems;
    UPROPERTY(SaveGame) TMap<FName, int32> AbilityLevels;

    /** Store asset references as paths; a hard pointer cannot round-trip through a file. */
    UPROPERTY(SaveGame) FSoftObjectPath LastEquippedWeapon;

    /** Excluded from SaveGameToSlot because it is Transient. */
    UPROPERTY(Transient) float RuntimeOnlyScratch = 0.f;
};
```

Slot functions, all static on `UGameplayStatics` (`Kismet/GameplayStatics.h`):

| Call | Signature |
|---|---|
| `CreateSaveGameObject` | `USaveGame* (TSubclassOf<USaveGame> SaveGameClass)` (`:1124`) |
| `SaveGameToSlot` | `bool (USaveGame*, const FString& SlotName, const int32 UserIndex)` (`:1167`) |
| `LoadGameFromSlot` | `USaveGame* (const FString& SlotName, const int32 UserIndex)` (`:1211`) |
| `DoesSaveGameExist` | `bool (const FString& SlotName, const int32 UserIndex)` (`:1175`) |
| `DeleteGameInSlot` | `bool (const FString& SlotName, const int32 UserIndex)` (`:1231`) |
| `SaveGameToMemory` | `bool (USaveGame*, TArray<uint8>& OutSaveData)` (`:1134`) |
| `LoadGameFromMemory` | `USaveGame* (const TArray<uint8>& InSaveData)` (`:1182`) |
| `SaveDataToSlot` | `bool (const TArray<uint8>&, const FString& SlotName, const int32 UserIndex)` (`:1143`) |
| `LoadDataFromSlot` | `bool (TArray<uint8>&, const FString& SlotName, const int32 UserIndex)` (`:1191`) |

## Async Save and Load

The async entry points take plain (non-dynamic) delegates, so the bound function must **not** be a `UFUNCTION`:

- `DECLARE_DELEGATE_ThreeParams(FAsyncSaveGameToSlotDelegate, const FString&, const int32, bool)` (`GameplayStatics.h:44`)
- `DECLARE_DELEGATE_ThreeParams(FAsyncLoadGameFromSlotDelegate, const FString&, const int32, USaveGame*)` (`GameplayStatics.h:47`)

```cpp
// MySaveSubsystem.h
#pragma once

#include "CoreMinimal.h"
#include "Subsystems/GameInstanceSubsystem.h"
#include "MySaveSubsystem.generated.h"

class UMySaveGame; class USaveGame;

UCLASS()
class MYGAME_API UMySaveSubsystem : public UGameInstanceSubsystem
{
    GENERATED_BODY()

public:
    void RequestSave(float PlayerHealth);
    void RequestLoad();

private:
    void HandleSaveComplete(const FString& SlotName, const int32 UserIndex, bool bSuccess);
    void HandleLoadComplete(const FString& SlotName, const int32 UserIndex, USaveGame* LoadedSave);
    void RunMigrations(UMySaveGame* Save);

    UPROPERTY()
    TObjectPtr<UMySaveGame> CurrentSave = nullptr;
    bool bSaveInProgress = false;
};
```

```cpp
// MySaveSubsystem.cpp
#include "MySaveSubsystem.h"
#include "MySaveGame.h"
#include "Kismet/GameplayStatics.h"

static const FString GMySaveSlot = TEXT("MainSave");
static constexpr int32 GMyUserIndex = 0; // see ULocalPlayerSaveGame for per-user consoles

void UMySaveSubsystem::RequestSave(float PlayerHealth)
{
    if (bSaveInProgress) { return; } // overlapping writes to one slot can truncate the file
    if (!CurrentSave)
    {
        CurrentSave = Cast<UMySaveGame>(
            UGameplayStatics::CreateSaveGameObject(UMySaveGame::StaticClass()));
    }
    if (!CurrentSave) { return; }
    CurrentSave->PlayerHealth = PlayerHealth;
    CurrentSave->SaveVersion = MySaveStage::Latest; // in-memory data is current; see Versioning

    // Synchronous form blocks the game thread until the platform write finishes:
    // UGameplayStatics::SaveGameToSlot(CurrentSave, GMySaveSlot, GMyUserIndex);
    bSaveInProgress = true;
    UGameplayStatics::AsyncSaveGameToSlot(CurrentSave, GMySaveSlot, GMyUserIndex,
        FAsyncSaveGameToSlotDelegate::CreateUObject(this, &UMySaveSubsystem::HandleSaveComplete));
}

void UMySaveSubsystem::HandleSaveComplete(const FString& SlotName, const int32 UserIndex, bool bSuccess)
{
    bSaveInProgress = false;
    UE_LOG(LogMyGame, Log, TEXT("Save '%s' (user %d) ok=%d"), *SlotName, UserIndex, bSuccess ? 1 : 0);
}

void UMySaveSubsystem::RequestLoad()
{
    UGameplayStatics::AsyncLoadGameFromSlot(GMySaveSlot, GMyUserIndex,
        FAsyncLoadGameFromSlotDelegate::CreateUObject(this, &UMySaveSubsystem::HandleLoadComplete));
}

void UMySaveSubsystem::HandleLoadComplete(const FString& SlotName, const int32 UserIndex, USaveGame* LoadedSave)
{
    CurrentSave = Cast<UMySaveGame>(LoadedSave); // null when the slot is missing or unreadable
    if (!CurrentSave)
    {
        CurrentSave = Cast<UMySaveGame>(
            UGameplayStatics::CreateSaveGameObject(UMySaveGame::StaticClass()));
        return;
    }
    RunMigrations(CurrentSave);
}
```

`CreateLambda` works when there is no owning `UObject` to keep alive; capture by value, because the call returns before the write finishes. The delegate runs on the game thread (`check(IsInGameThread())`, `GameplayStatics.cpp:2417`), and runs synchronously inside the call when the slot name is empty or serialization fails (`:2424`). `SaveGameToMemory` itself always runs on the game thread (`:2410`); only the platform write is async.

## ULocalPlayerSaveGame

`ULocalPlayerSaveGame` is declared in **`GameFramework/SaveGame.h:47`** (there is no `LocalPlayerSaveGame.h`). It binds a save to one local player, resolves the platform user index automatically, and provides versioning hooks for subclasses to override. `GetLatestDataVersion()` returns the current schema number. `HandlePostLoad()` compares it with `GetSavedDataVersion()`, which is the value stored at the last save, and migrates old data. `HandlePreSave()` sanitises fields before they are written, and `HandlePostSave(bool bSuccess)` is where save results arrive. Call `Super::` in each. A complete subclass with a two-step migration is in [references/save-system-architecture.md](references/save-system-architecture.md#7-per-player-saves-and-thumbnails).

The **native** delegate overload takes a `ULocalPlayer*`; the `APlayerController*` overload takes the dynamic `FOnLocalPlayerSaveGameLoaded` instead.

```cpp
// Synchronous: returns null only for invalid parameters, otherwise creates a fresh instance.
UMyLocalPlayerSave* Save = Cast<UMyLocalPlayerSave>(
    ULocalPlayerSaveGame::LoadOrCreateSaveGameForLocalPlayer(
        UMyLocalPlayerSave::StaticClass(), PlayerController, TEXT("PlayerSlot0")));

// Asynchronous, native delegate: pass the ULocalPlayer, not the controller.
const ULocalPlayer* LocalPlayer = PlayerController ? PlayerController->GetLocalPlayer() : nullptr;
const bool bScheduled = ULocalPlayerSaveGame::AsyncLoadOrCreateSaveGameForLocalPlayer(
    UMyLocalPlayerSave::StaticClass(), LocalPlayer, TEXT("PlayerSlot0"),
    FOnLocalPlayerSaveGameLoadedNative::CreateUObject(this, &AMyPlayerController::HandleSaveLoaded));

// Writing back; results arrive through HandlePostSave, not the return value.
Save->AsyncSaveGameToSlotForLocalPlayer(); // bool: the save was requested
Save->SaveGameToSlotForLocalPlayer();      // synchronous
```

Other verified members (`GameFramework/SaveGame.h:50-224`): `CreateNewSaveGameForLocalPlayer`, `GetLocalPlayerController`, `GetLocalPlayer`, `SetLocalPlayer`, `GetPlatformUserId`, `GetPlatformUserIndex`, `GetSaveSlotName`, `SetSaveSlotName`, `GetSavedDataVersion`, `GetInvalidDataVersion`, `WasLoaded`, `IsSaveInProgress`, `WasLastSaveSuccessful`, `WasSaveRequested`, `InitializeSaveGame`, `ResetToDefault`.

## FArchive and Binary Serialization

`FArchive` (`Serialization/Archive.h`) is bidirectional: one `operator<<` body handles both read and write.

```cpp
Ar.IsLoading()                 // Archive.h:272
Ar.IsSaving()                  // Archive.h:284
Ar.IsError()                   // Archive.h:398 — check after every block
Ar.Tell()                      // Archive.h:185 — int64 position
Ar.IsSaveGame()                // Archive.h:659 — reads the ArIsSaveGame field
Ar.ArIsSaveGame = true;        // Archive.h:942 — public bitfield, no setter exists
Ar.UsingCustomVersion(Guid);   // Archive.h:2006
Ar.CustomVer(Guid);            // Archive.h:269 — int32
```

`FMemoryWriter` / `FMemoryReader` move bytes in and out of a `TArray<uint8>`. Both take `bIsPersistent` (`MemoryWriter.h:26`, `MemoryReader.h:52`); pass `true` so the archive behaves like an on-disk write rather than a transient in-memory one.

```cpp
#include "Serialization/MemoryWriter.h"
#include "Serialization/MemoryReader.h"

bool UMyObject::WriteBlob(TArray<uint8>& OutBytes)
{
    FMemoryWriter Writer(OutBytes, /*bIsPersistent=*/true);
    int32 Magic = 0x4D595347;   // 'MYSG'
    int32 Version = 2;
    Writer << Magic << Version << BinaryBlob;
    return !Writer.IsError();
}

bool UMyObject::ReadBlob(const TArray<uint8>& InBytes)
{
    FMemoryReader Reader(InBytes, /*bIsPersistent=*/true);
    int32 Magic = 0;
    int32 Version = 0;
    Reader << Magic << Version;
    if (Reader.IsError() || Magic != 0x4D595347 || Version < 1) { return false; }
    Reader << BinaryBlob;
    return !Reader.IsError();
}
```

Overriding `virtual void UMyObject::Serialize(FArchive& Ar)` gives byte-level control on a `UObject`; call `Super::Serialize(Ar)` first so the tagged property block is written. Define a free `FArchive& operator<<(FArchive& Ar, FMyCustomData& Data)` to make a plain struct archive-serializable. `FBufferArchive` (`Serialization/BufferArchive.h:47`) derives from both the memory writer and `TArray<uint8>`, so the archive *is* the buffer.

## Actor State with UPROPERTY(SaveGame)

`UPROPERTY(SaveGame)` sets `CPF_SaveGame` (`UObject/ObjectMacros.h:458`, specifier at `:1194`), which is **only** honoured by archives with `ArIsSaveGame` set. That is how you snapshot live actors: wrap a memory archive in `FObjectAndNameAsStringProxyArchive` (`Serialization/ObjectAndNameAsStringProxyArchive.h:21`) so `UObject` and `FName` references survive as strings instead of load-order-dependent indices.

```cpp
// MyActorRecord.cpp — include MemoryWriter.h, MemoryReader.h,
// ObjectAndNameAsStringProxyArchive.h and GameFramework/Actor.h.
void FMyActorRecord::SaveActor(AActor* Actor)
{
    if (!Actor) { return; }
    ActorClass = Actor->GetClass();
    ActorTransform = Actor->GetActorTransform();
    ByteData.Reset();

    FMemoryWriter MemWriter(ByteData, /*bIsPersistent=*/true);
    FObjectAndNameAsStringProxyArchive Ar(MemWriter, /*bInLoadIfFindFails=*/true);
    Ar.ArIsSaveGame = true; // only UPROPERTY(SaveGame) fields are written
    Actor->Serialize(Ar);
}

void FMyActorRecord::RestoreActor(AActor* Actor) const
{
    if (!Actor || ByteData.Num() == 0) { return; }
    Actor->SetActorTransform(ActorTransform);

    FMemoryReader MemReader(ByteData, /*bIsPersistent=*/true);
    FObjectAndNameAsStringProxyArchive Ar(MemReader, /*bInLoadIfFindFails=*/true);
    Ar.ArIsSaveGame = true;
    Actor->Serialize(Ar);
}
```

`FObjectAndNameAsStringProxyArchive` also exposes `bResolveRedirectors` and `bResolveCoreRedirects` (both default `false`); set them when assets may have been renamed between builds. `FNameAsStringProxyArchive` (`Serialization/NameAsStringProxyArchive.h:11`) handles `FName` only. Full world-capture pipeline, per-actor `FGuid` identity, the actor save interface and respawn ordering: [Save system architecture](references/save-system-architecture.md).

## USTRUCT Custom Serialization

A `USTRUCT` hook is `bool Serialize(FArchive& Ar)` **plus** a `TStructOpsTypeTraits` specialization with `WithSerializer = true`. Without the traits specialization the function is never called — the engine gates on `TStructOpsTypeTraits<CppStruct>::WithSerializer` (`UObject/Class.h:1307`; trait declared in `UObject/StructOpsTypeTraits.h:24`). Return `true` to mean "fully handled; skip the default tagged-property path".

```cpp
// MyStatBlock.h — include "CoreMinimal.h" then "MyStatBlock.generated.h".
USTRUCT(BlueprintType)
struct MYGAME_API FMyStatBlock
{
    GENERATED_BODY()

    UPROPERTY(SaveGame) float HP = 100.f;
    UPROPERTY(SaveGame) float Stamina = 100.f;

    bool Serialize(FArchive& Ar);
};

template<>
struct TStructOpsTypeTraits<FMyStatBlock> : public TStructOpsTypeTraitsBase2<FMyStatBlock>
{
    enum
    {
        WithSerializer = true
    };
};
```

```cpp
// MyStatBlock.cpp — include "MyStatBlock.h" and "MySaveVersion.h".
bool FMyStatBlock::Serialize(FArchive& Ar)
{
    Ar.UsingCustomVersion(FMySaveVersion::GUID);
    if (Ar.CustomVer(FMySaveVersion::GUID) < FMySaveVersion::SplitStamina)
    {
        float LegacyHealth = 0.f; // the old layout stored a single float
        Ar << LegacyHealth;
        HP = LegacyHealth;
        Stamina = 100.f;
    }
    else
    {
        Ar << HP << Stamina;
    }
    return true;
}
```

Related traits on the same base (`StructOpsTypeTraits.h`): `WithPostSerialize`, `WithStructuredSerializer`, `WithSerializeFromMismatchedTag`, `WithNetSerializer`, `WithIdentical`.

## Versioning

**Explicit version field** — the simplest option for a `USaveGame` written through `UGameplayStatics`, because the field is just another `UPROPERTY`:

```cpp
namespace MySaveStage
{
    constexpr int32 Initial = 0, AddedInventory = 1, SoftRefForWeapon = 2, Latest = SoftRefForWeapon;
}

void UMySaveSubsystem::RunMigrations(UMySaveGame* Save)
{
    if (!Save || Save->SaveVersion == MySaveStage::Latest) { return; }
    if (Save->SaveVersion < MySaveStage::AddedInventory) { Save->InventoryItems.Reset(); }
    if (Save->SaveVersion < MySaveStage::SoftRefForWeapon) { Save->LastEquippedWeapon.Reset(); }
    Save->SaveVersion = MySaveStage::Latest; // stamp only after every step ran
}
```

**`FCustomVersion`** — per-archive versions (`Serialization/CustomVersion.h`: `FCustomVersion` at `:39`, `FCustomVersionRegistration` at `:211`). `SaveGameToSlot`/`SaveGameToMemory` write every registered custom version into the file header and restore them on load (`GameplayStatics.cpp:233,207`), so `CustomVer` works inside a `USaveGame`. A bare `FMemoryWriter`/`FMemoryReader` pair carries no versions: serialize `Ar.GetCustomVersions()` yourself (`Archive.h:555,562`) or `CustomVer` returns `-1` on load.

```cpp
// MySaveVersion.h — include "CoreMinimal.h" and "Misc/Guid.h".
struct FMySaveVersion
{
    enum Type { Initial = 0, AddedQuestData = 1, SplitStamina = 2, VersionPlusOne, Latest = VersionPlusOne - 1 };

    /** Generate once with FGuid::NewGuid(), then hardcode forever. */
    static const FGuid GUID;
};

// MySaveVersion.cpp — include "MySaveVersion.h" and "Serialization/CustomVersion.h".
const FGuid FMySaveVersion::GUID(0xA1B2C3D4, 0xE5F60718, 0x293A4B5C, 0x6D7E8F90);

// Registers the version with every archive for the module's lifetime.
FCustomVersionRegistration GRegisterMySaveVersion(
    FMySaveVersion::GUID, FMySaveVersion::Latest, TEXT("MySaveVersion"));

// Inside any Serialize(); UsingCustomVersion is required when saving (CustomVer asserts
// otherwise, Archive.cpp:646) and a harmless no-op when loading (Archive.cpp:631).
// CoreData and QuestData are UPROPERTY members of UMyObject.
void UMyObject::SerializeVersioned(FArchive& Ar)
{
    Ar.UsingCustomVersion(FMySaveVersion::GUID);
    const int32 Version = Ar.CustomVer(FMySaveVersion::GUID);
    Ar << CoreData;
    if (Version >= FMySaveVersion::AddedQuestData) { Ar << QuestData; }
    else if (Ar.IsLoading())                       { QuestData.Reset(); }
}
```

## ISaveGameSystem and Platform Storage

`UGameplayStatics` routes through the platform's `ISaveGameSystem` (`Engine/Public/SaveGameSystem.h:19`), reached via `IPlatformFeaturesModule::Get().GetSaveGameSystem()` (`Engine/Public/PlatformFeatures.h:41`). Use it directly only for capabilities `UGameplayStatics` does not expose — enumerating slots, native save UI, or multi-user checks.

```cpp
#include "SaveGameSystem.h"
#include "PlatformFeatures.h"

ISaveGameSystem* SaveSystem = IPlatformFeaturesModule::Get().GetSaveGameSystem();
if (!SaveSystem) { return; }
const bool bPerUser = SaveSystem->DoesSaveSystemSupportMultipleUsers();
TArray<FString> FoundSlots;
SaveSystem->GetSaveGameNames(FoundSlots, /*UserIndex=*/0);
const ISaveGameSystem::ESaveExistsResult R = SaveSystem->DoesSaveGameExistWithResult(TEXT("MainSave"), 0);
```

Interface members (`SaveGameSystem.h:34-58`): `PlatformHasNativeUI`, `DoesSaveSystemSupportMultipleUsers`, `DoesSaveGameExist`, `DoesSaveGameExistWithResult`, `GetSaveGameNames`, `SaveGame`, `LoadGame`, `DeleteGame`. `ISaveGameSystem` also declares the async set `DoesSaveGameExistAsync`, `SaveGameAsync`, `LoadGameAsync`, `LoadGameIfExistsAsync`, `DeleteGameAsync`, `GetSaveGameNamesAsync` and `InitAsync`, all keyed by `FPlatformUserId` (`:78-97`); `FBaseAsyncSaveGameSystem` (`:178`) is the helper base that implements them on `UE::Tasks`. Never build save paths by hand: `FGenericSaveGameSystem::GetSaveGamePath` (`:169`) is the desktop fallback only, and console and cloud backends ignore the filesystem entirely.

## Config, Settings, and File Formats

Settings are a separate channel from save slots: `.ini` for options and tuning, `USaveGame` for progress.

| Need | API | Header |
|---|---|---|
| Project Settings page, `Config=Game`, `DefaultConfig` | `UDeveloperSettings` + `GetDefault<T>()`; override `GetContainerName` / `GetCategoryName` / `GetSectionName` | `Engine/DeveloperSettings.h:23,31-35` (module `DeveloperSettings`) |
| Player options (resolution, audio, keybinds) | `UGameUserSettings::ApplySettings(bool bCheckForCommandLineOverrides)`, `SaveSettings()`, `LoadSettings(bool bForceReload)` | `GameFramework/GameUserSettings.h:49,311,307` |
| Write/read an object's `UPROPERTY(Config)` fields | `UObject::SaveConfig()`, `LoadConfig()`, `OverrideConfigSection(FString&)` | `UObject/Object.h:1283,1389,1370` |
| Raw `.ini` keys | `GConfig->GetString/SetString(Section, Key, Value, GGameIni)`, `GConfig->Flush(bRemoveFromCache, GGameIni)` | `Misc/ConfigCacheIni.h:1402,1408,1395`; `GGameIni` at `CoreGlobals.h:439` |
| `USTRUCT` to/from JSON text; byte array to/from a file (tooling only) | `FJsonObjectConverter::UStructToJsonObjectString` / `JsonObjectStringToUStruct`; `FFileHelper::SaveArrayToFile` / `LoadFileToArray` under `FPaths::ProjectSavedDir()` | `JsonObjectConverter.h:156,313`; `Misc/FileHelper.h:185,79`; `Misc/Paths.h:290` |

Section `[/Script/ModuleName.ClassName]` maps to the class CDO; register a `UGameUserSettings` subclass with `GameUserSettingsClassName=/Script/MyGame.MyGameUserSettings` under `[/Script/Engine.Engine]` in `DefaultEngine.ini`. Compression uses `FCompression::CompressMemoryBound` / `CompressMemory` / `UncompressMemory` (`Misc/Compression.h:73,108,143`) with an `FName` format — `NAME_Zlib`, `NAME_Oodle`, `NAME_Gzip`, `NAME_LZ4` (`UObject/UnrealNames.inl:210-214`) — and must store the uncompressed size alongside the blob; `FArchiveSaveCompressedProxy(TArray<uint8>&, FName, ECompressionFlags)` (`Serialization/ArchiveSaveCompressedProxy.h:28`, `Flush()` at `:36`) and `FArchiveLoadCompressedProxy` (`ArchiveLoadCompressedProxy.h:22`) wrap the same thing as archives. Worked `UDeveloperSettings` class, `GConfig` calls, JSON round-trip, compression, checksum (`FCrc::MemCrc32`, `Misc/Crc.h:29`) and `FAES` encryption code: [Save system architecture](references/save-system-architecture.md).

## Deprecated — do not use

| Do not emit | Use in 5.8 | Source |
|---|---|---|
| `#include "GameFramework/LocalPlayerSaveGame.h"` | `#include "GameFramework/SaveGame.h"` | `ULocalPlayerSaveGame` declared at `GameFramework/SaveGame.h:47`; no such header exists |
| `Ar.SetIsSaveGame(true)` | `Ar.ArIsSaveGame = true;` | public bitfield at `Serialization/Archive.h:942`; no setter in the header |
| `void FMyStruct::Serialize(FArchive&)` | `bool Serialize(FArchive&)` + `TStructOpsTypeTraits` with `WithSerializer = true` | gate at `UObject/Class.h:1307`, trait at `UObject/StructOpsTypeTraits.h:24` |
| `COMPRESS_ZLIB` | `NAME_Zlib` | `COMPRESS_ZLIB_DEPRECATED` in `Misc/CompressionFlags.h:18` |
| `FCompression::GetCompressionFormatFromDeprecatedFlags(Flags)` | pass `NAME_Zlib` / `NAME_Oodle` directly | `UE_DEPRECATED(5.5)` in `Misc/Compression.h:173` |
| `UGameUserSettings::WindowPosX` / `WindowPosY` | the `WindowPositions` array | `UE_DEPRECATED(5.6)` in `GameFramework/GameUserSettings.h:480,485` |
| `UFUNCTION()` on an `AsyncSaveGameToSlot` callback | a plain member function | `FAsyncSaveGameToSlotDelegate` is `DECLARE_DELEGATE_ThreeParams` (`GameplayStatics.h:44`), not dynamic |
| `ISteamRemoteStorage`, `SteamRemoteStorage()`, `IOnlineTitleFileInterface` | `ISaveGameSystem` via `IPlatformFeaturesModule::Get().GetSaveGameSystem()` | none of these appear in any 5.8 engine header |

## Common Mistakes

**Assuming `UPROPERTY(SaveGame)` filters `SaveGameToSlot`:** it does not — `SaveGameToSlot` writes all non-transient properties (`GameplayStatics.h:1159`). Use `Transient` to exclude a field; `SaveGame` matters only for archives with `ArIsSaveGame = true`.

**Serializing an actor without the proxy archive:** a bare `FMemoryWriter` stores `UObject` and `FName` references as indices that do not survive a restart. Always wrap it in `FObjectAndNameAsStringProxyArchive`.

**Passing an `APlayerController*` with the native local-player delegate:** the `FOnLocalPlayerSaveGameLoadedNative` overload takes `const ULocalPlayer*`. Call `PlayerController->GetLocalPlayer()` first.

**Getting versioning backwards:** set `SaveVersion = Latest` only after every migration step has run (and on every save, so fresh saves are not migrated next load). `Ar.UsingCustomVersion()` is mandatory on the *saving* archive — `CustomVer` `check()`s otherwise — while on a loading archive it is a no-op and `CustomVer` returns the file's version, or `-1` if the archive holds none (`Archive.cpp:631,646-648`).

**Hardcoding `Saved/SaveGames/*.sav` paths:** consoles and cloud backends never touch that directory; use `UGameplayStatics` or `ISaveGameSystem`.

## Related Skills

- `ue-cpp-foundations` — `UPROPERTY`/`USTRUCT` specifiers, `UObject` lifetime, subsystem types
- `ue-data-assets-tables` — `FSoftObjectPath`, `FPrimaryAssetId`, async loading behind saved references
- `ue-gameplay-framework` — `UGameInstance` as the save manager host, GameMode-driven autosave points
- `ue-world-level-streaming` — persisting streamed-level and World Partition actor state
- `ue-async-threading` — running serialization or compression off the game thread
- `ue-networking-replication` — server-authoritative saves versus client-local profiles
