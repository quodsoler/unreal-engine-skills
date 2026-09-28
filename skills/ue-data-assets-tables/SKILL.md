---
name: ue-data-assets-tables
description: "Use when modelling designer-authored game data in C++ and loading it: data assets, data tables, curve tables, soft references and asset streaming. Also use when the user mentions 'UDataAsset', 'UPrimaryDataAsset', 'UDataTable', 'FTableRowBase', 'FindRow', 'FDataTableRowHandle', 'TSoftObjectPtr', 'TSoftClassPtr', 'FSoftObjectPath', 'UAssetManager', 'LoadPrimaryAsset', 'FStreamableManager', 'RequestAsyncLoad', 'asset bundles', 'PrimaryAssetTypesToScan', 'IAssetRegistry', 'CSV import', or 'hard vs soft reference'. For save games, see ue-serialization-savegames; for UPROPERTY basics, see ue-cpp-foundations; for level streaming, see ue-world-level-streaming."
metadata:
  version: "2.0.0"
  engine: "5.8"
---

# UE Data Assets and Tables

Target engine: **UE 5.8**. APIs below are verified against the 5.8 headers; older forms are listed under "Deprecated — do not use".

This skill covers designer-authored game data and the loading paths that bring it into memory: `UDataAsset`/`UPrimaryDataAsset`, `UDataTable`/`UCurveTable`, soft references, `FStreamableManager`, `UAssetManager` primary assets and cook rules, `IAssetRegistry` queries, the DataRegistry plugin and `UDeveloperSettings`. Build.cs modules: `Engine` and `CoreUObject` for data assets, tables and soft pointers; `AssetRegistry` for registry queries; `DeveloperSettings` for settings classes; `DataRegistry` for data registries. Deep dives live in [asset loading patterns](references/asset-loading-patterns.md) and [data-driven design patterns](references/data-driven-design.md).

## Context

Read `.agents/ue-project-context.md` if it exists (module names, conventions, enabled plugins, custom `UAssetManager` subclass, registered primary asset types). Do not stop if it is missing.

Identify the area from the request and the codebase. Ask only when two plausible readings would produce different code.

| Request is about… | Go to |
|---|---|
| Picking a container for game data | [Choosing a Data Container](#choosing-a-data-container) |
| `UDataAsset`, `UPrimaryDataAsset`, asset bundles | [Data Assets](#data-assets) |
| Row structs, CSV/JSON import, `FindRow` | [Data Tables](#data-tables) |
| Curves, table composition | [Curve Tables and Composite Tables](#curve-tables-and-composite-tables) |
| `TSoftObjectPtr`, `FSoftObjectPath`, memory budget | [References: Hard vs Soft](#references-hard-vs-soft) |
| Async loading, handles, priorities | [Async Loading](#async-loading) |
| `FPrimaryAssetId`, scanning, bundle states | [Asset Manager](#asset-manager) |
| Cooking, chunking, `PrimaryAssetTypesToScan` | [Cook Rules](#cook-rules) |
| Finding assets without loading them | [Asset Registry Queries](#asset-registry-queries) |
| Sharing rows across sources at runtime | [Data Registry](#data-registry) |
| Project-wide tunables in Project Settings | [Project Settings](#project-settings) |

## Choosing a Data Container

| Need | Use | Header |
|---|---|---|
| Per-object config, editor-authored, one asset per entry | `UDataAsset` | `Engine/DataAsset.h` |
| Same, plus load/unload by id and cook rules | `UPrimaryDataAsset` | `Engine/DataAsset.h` |
| Flat rows of identical shape, spreadsheet authored | `UDataTable` + `FTableRowBase` | `Engine/DataTable.h` |
| Rows merged from several tables (DLC, overrides) | `UCompositeDataTable` | `Engine/CompositeDataTable.h` |
| Float curves keyed by row name | `UCurveTable`, `FCurveTableRowHandle` | `Engine/CurveTable.h` |
| One row id resolved across many sources at runtime | `UDataRegistrySubsystem` | `DataRegistrySubsystem.h` |
| Project-wide tunables in Project Settings | `UDeveloperSettings` | `Engine/DeveloperSettings.h` |

`UDataAsset` supports Blueprint subclasses and inheritance; `UDataTable` rows do not. `UDataTable` gives lookup by row name out of the box; data assets need an Asset Manager scan or an Asset Registry query to be discoverable.

## Data Assets

`UDataAsset` is a plain `UObject` you can instantiate from the Content Browser. `UPrimaryDataAsset` adds `GetPrimaryAssetId()` so `UAssetManager` can scan, load and unload it by id.

```cpp
// MyWeaponDefinition.h
#pragma once

#include "CoreMinimal.h"
#include "Engine/DataAsset.h"
#include "MyWeaponDefinition.generated.h"
class UTexture2D; class USkeletalMesh;   // TSoftObjectPtr<T> needs T declared

UCLASS(BlueprintType)
class MYGAME_API UMyWeaponDefinition : public UPrimaryDataAsset
{
    GENERATED_BODY()

public:
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Weapon")
    FText WeaponName;

    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Weapon")
    float FireRate = 1.f;

    /** Written into the Asset Registry so it can be filtered without loading. */
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Weapon", AssetRegistrySearchable)
    FName WeaponClassTag;

    /** Bundles group soft references so the Asset Manager can load a subset. */
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Art", meta = (AssetBundles = "UI"))
    TSoftObjectPtr<UTexture2D> Icon;

    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Art", meta = (AssetBundles = "Game"))
    TSoftObjectPtr<USkeletalMesh> WorldMesh;
};
```

`GetPrimaryAssetId()` rules (`Engine/DataAsset.h:36-53`):

- The default returns type = name of the **first native class** going up the hierarchy (or the highest-level Blueprint class), name = the asset FName.
- Blueprint subclasses should be authored as Data Only Blueprints, not Data Asset instances, so parent-class edits propagate.
- Override only when you need a different type/name scheme; inside the class body the declaration is `virtual FPrimaryAssetId GetPrimaryAssetId() const override;`.
- `UpdateAssetBundleData()` scans `meta=(AssetBundles="Name")` on the class and fills the `AssetBundleData` UPROPERTY during `PreSave`. Bundle names are free-form; `meta=(AssetBundles="Client,Server")` puts one property in two bundles.
- Plain `UDataAsset` has no primary asset id and loads only through whoever references it.

## Data Tables

Every row struct derives `FTableRowBase` (`Engine/DataTable.h:33`) and uses `GENERATED_BODY()`.

```cpp
// MyItemTableRow.h
#pragma once

#include "CoreMinimal.h"
#include "Engine/DataTable.h"
#include "MyItemTableRow.generated.h"
class UStaticMesh;

USTRUCT(BlueprintType)
struct FMyItemTableRow : public FTableRowBase
{
    GENERATED_BODY()

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Item")
    FText DisplayName;

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Item")
    int32 MaxStack = 1;

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Item")
    TSoftObjectPtr<UStaticMesh> PreviewMesh;

    /** Called after CSV/JSON import. Not editor-only. */
    virtual void OnPostDataImport(const UDataTable* InDataTable, const FName InRowName, TArray<FString>& OutCollectedImportProblems) override;

    /** Called for every row whenever the owning table is modified. Not editor-only. */
    virtual void OnDataTableChanged(const UDataTable* InDataTable, const FName InRowName) override;
};
```

`FTableRowBase::IsDataValid(FDataValidationContext& Context) const` is the third hook and **is** wrapped in `#if WITH_EDITOR` (`Engine/DataTable.h:60-68`); guard any override the same way and include `Misc/DataValidation.h`.

### Lookup

```cpp
UPROPERTY(EditDefaultsOnly, Category = "Data")
TObjectPtr<UDataTable> ItemTable;

// FindRow<T>(FName, const TCHAR*|const FString&, bool bWarnIfRowMissing = true) -> T* or nullptr.
const FMyItemTableRow* Row = ItemTable->FindRow<FMyItemTableRow>(RowName, TEXT("LookupItem"));

TArray<FMyItemTableRow*> AllRows;
ItemTable->GetAllRows<FMyItemTableRow>(TEXT("GetAllItems"), AllRows);

TArray<FName> RowNames = ItemTable->GetRowNames();

ItemTable->ForeachRow<FMyItemTableRow>(TEXT("Foreach"),
    [](const FName& Key, const FMyItemTableRow& Value)
    {
        // Key is the row name, Value the typed row.
    });

// GetRowMap() is the raw TMap<FName, uint8*>; use it only for generic/reflection code.
const int32 NumRows = ItemTable->GetRowMap().Num();
```

`FDataTableRowHandle` is the UPROPERTY-friendly reference (table plus row name) designers pick in the editor:

```cpp
UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Config")
FDataTableRowHandle StartingItemHandle;   // IsNull() when nothing is picked

const FMyItemTableRow* Starting = StartingItemHandle.GetRow<FMyItemTableRow>(TEXT("StartingItem"));
```

### Runtime mutation and import

```cpp
FMyItemTableRow NewRow;
NewRow.MaxStack = 5;
ItemTable->AddRow(FName(TEXT("RuntimeSword")), NewRow);   // In memory only, not saved to disk.
ItemTable->RemoveRow(FName(TEXT("ObsoleteItem")));
ItemTable->HandleDataTableChanged();                      // Fires the per-row OnDataTableChanged hooks.

// RowStruct must already be set. Available in all build configurations.
TArray<FString> Problems = ItemTable->CreateTableFromCSVString(CsvContent);
TArray<FString> JsonProblems = ItemTable->CreateTableFromJSONString(JsonContent);
```

`GetTableAsCSV()`, `GetTableAsJSON()` and `GetTableAsString()` sit inside `#if WITH_EDITOR` (`Engine/DataTable.h:318-345`) — they do not exist in a cooked build. `bStripFromClientBuilds` on the table asset makes `NeedsLoadForClient()` return false, so server-only tables never ship to clients (`Engine/DataTable.h:117,149`).

### Blueprint access

`UDataTableFunctionLibrary` (`Kismet/DataTableFunctionLibrary.h`) exposes tables to Blueprint: `GetDataTableRowFromName`, `DoesDataTableRowExist`, `GetDataTableRowNames`, `GetDataTableColumnNames`, `GetDataTableColumnAsString`, `GetDataTableRowStruct`, `EvaluateCurveTableRow`, `GetCurveTableRowNames`, plus the editor-scripting `FillDataTableFromCSVString` / `FillDataTableFromCSVFile`.

## Curve Tables and Composite Tables

```cpp
UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Scaling")
FCurveTableRowHandle DamageCurve;

float ScaledDamage = DamageCurve.Eval(CharacterLevel, TEXT("ScaleDamage"));

// Or take the curve once for repeated sampling.
const FRealCurve* Curve = DamageCurve.GetCurve(TEXT("ScaleDamage"));
```

`FCurveTableRowHandle` also offers `GetRichCurve()`, `GetSimpleCurve()`, `IsNull()` and `Eval(float XValue, float* YValue, const FString& ContextString)`, which reports whether the lookup succeeded.

`UCompositeDataTable` merges its `ParentTables` array into one row map; later tables override earlier ones by row name. Use it for base game plus DLC or per-platform overrides. `AppendParentTables()`, `RemoveParentTables()` and `GetParentTables()` manage the stack; `AddRow` and `RemoveRow` are overridden as no-ops (`Engine/CompositeDataTable.h:43-47`, `CompositeDataTable.cpp:191-204`), so edit the parent tables instead.

## References: Hard vs Soft

| Form | Loads target when owner loads | Use for |
|---|---|---|
| `TObjectPtr<UStaticMesh>` | Yes | Always-needed, small, or already resident assets |
| `TSubclassOf<AActor>` | Yes (the class) | Classes spawned constantly |
| `TSoftObjectPtr<UStaticMesh>` | No | Art, audio, VFX referenced by data |
| `TSoftClassPtr<AActor>` | No | Blueprint classes chosen by data |
| `FSoftObjectPath` | No | Untyped paths in generic systems |

```cpp
UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Art")
TSoftObjectPtr<UStaticMesh> MeshSoft;

UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Spawning")
TSoftClassPtr<AActor> SpawnableSoft;   // LoadSynchronous() returns UClass*

const FSoftObjectPath MeshPath = MeshSoft.ToSoftObjectPath();

// State checks, no loading.
const bool bNoPathSet   = MeshSoft.IsNull();
const bool bInMemory    = MeshSoft.IsValid();   // path set AND target resolved in memory
const bool bLoadPending = MeshSoft.IsPending(); // path set, target not in memory yet
UStaticMesh* Resident   = MeshSoft.Get();       // nullptr unless already loaded

// Blocking resolution. Avoid on the game thread outside loading screens.
UStaticMesh* Mesh = MeshSoft.LoadSynchronous();
UObject* Loaded   = MeshPath.TryLoad();         // nullptr if missing, does not assert
UObject* Existing = MeshPath.ResolveObject();   // in-memory only, never loads
```

A hard reference inside a `UPrimaryDataAsset` pulls its whole dependency chain into memory as soon as the definition loads — the single biggest cause of runaway memory in data-driven projects. Default to soft references for content, hard references for values and small structs.

## Async Loading

`FStreamableManager` (`Engine/StreamableManager.h`) is owned by the Asset Manager; reach it with `UAssetManager::GetStreamableManager()` and never construct your own in game code.

`RequestAsyncLoad` has two forms. The classic form takes the paths, a delegate, then `TAsyncLoadPriority Priority`, `bool bManageActiveHandle`, `bool bStartStalled`, `FString DebugName`. The params form takes a single `FStreamableAsyncLoadParams&&`.

```cpp
// AMyPickupActor members: the handle owns the load, so it must not be a local.
TSharedPtr<FStreamableHandle> LoadHandle;
UPROPERTY(EditDefaultsOnly, Category = "Art")
TSoftObjectPtr<UStaticMesh> MeshSoft;
UPROPERTY(VisibleAnywhere, Category = "Art")
TObjectPtr<UStaticMeshComponent> MeshComponent;

void AMyPickupActor::BeginLoadMesh()
{
    FStreamableManager& SM = UAssetManager::GetStreamableManager();

    LoadHandle = SM.RequestAsyncLoad(
        MeshSoft.ToSoftObjectPath(),
        FStreamableDelegate::CreateUObject(this, &AMyPickupActor::OnMeshLoaded),
        FStreamableManager::AsyncLoadHighPriority,
        /*bManageActiveHandle=*/false,
        /*bStartStalled=*/false,
        TEXT("Pickup mesh"));
}

void AMyPickupActor::OnMeshLoaded()
{
    if (UStaticMesh* Mesh = MeshSoft.Get())
    {
        MeshComponent->SetStaticMesh(Mesh);   // Component now holds a hard reference.
    }
    LoadHandle.Reset();
}
```

Params form, when you need the cancel/update delegates or JIT trickling:

```cpp
FStreamableAsyncLoadParams Params;
Params.TargetsToStream = { IconSoft.ToSoftObjectPath(), MeshSoft.ToSoftObjectPath() };
Params.Priority = FStreamableManager::AsyncLoadHighPriority;
Params.bUseJustInTimeAsyncLoader = true;
Params.OnComplete = FStreamableDelegateWithHandle::CreateWeakLambda(this,
    [this](TSharedPtr<FStreamableHandle> Handle)
    {
        // Both targets resolved.
    });

LoadHandle = SM.RequestAsyncLoad(MoveTemp(Params), TEXT("Pickup bundle"));
```

`RequestSyncLoad(TargetsToStream, bManageActiveHandle = false, DebugName = FString())` blocks until done — loading screens and one-time init only.

| `FStreamableHandle` member | Meaning |
|---|---|
| `IsActive()` | Request exists and has not been released or cancelled |
| `HasLoadCompleted()` | Every target finished (the delegate may not have run yet) |
| `IsLoadingInProgress()` | Still streaming |
| `WasCanceled()` / `HasError()` | Cancelled by you / one or more targets failed |
| `GetLoadProgress()` | 0.0 to 1.0 |
| `GetLoadedAsset()` / `GetLoadedAsset<T>()` | First target; `GetLoadedAssets(TArray<T*>&)` for all |
| `WaitUntilComplete(float Timeout = 0.f, bool bStartStalledHandles = true)` | Blocks the caller, returns `EAsyncPackageState::Type` |
| `BindCompleteDelegate` / `BindCancelDelegate` / `BindUpdateDelegate` | Attach callbacks after the request was made |
| `CancelHandle()` | Stops the load; the complete delegate never fires |
| `ReleaseHandle()` | Drops the streaming reference so the assets can be collected |

**The handle is the reference.** Assets loaded through `RequestAsyncLoad` stay alive only while a `TSharedPtr<FStreamableHandle>` to that request survives (or `bManageActiveHandle` was true). Dropping every pointer before completion does not cancel: the manager holds the handle until the completion delegate has run, then releases it (`Engine/StreamableManager.h:316-320`), so the assets are collectable at the next GC unless something else hard-references them.

Delegate factories: `FStreamableDelegate::CreateUObject(this, &UMyLoader::Fn)` (safe, checks the object), `CreateWeakLambda(this, Lambda)` (safe, drops if the owner died), `CreateLambda(Lambda)` (unsafe with a captured `this`).

`UE_ENABLE_STREAMABLE_JIT_ASYNC_LOADING` (`Engine/StreamableManager.h:20`) defaults to 0 and gates the just-in-time async loader at compile time. `s.StreamableEnableJITAsyncLoading` enables it for requests that opt in through `bUseJustInTimeAsyncLoader`; `s.StreamableEnableJITAsyncLoadingGlobally` forces it for every request. JIT trickles requests to the async loader instead of queueing them all at once, which keeps prioritisation and cancellation responsive under load. `FStreamableHandle::IsUsingJustInTimeAsyncLoader()` reports the resolved state.

More patterns — batching, combined handles, progress, cancellation — in [asset loading patterns](references/asset-loading-patterns.md).

## Asset Manager

`UAssetManager` is the global singleton registered through `[/Script/Engine.Engine] AssetManagerClassName`. Subclass it to override `StartInitialLoading()` and `PostInitialAssetScan()`.

```cpp
UAssetManager& AM = UAssetManager::Get();               // Engine/AssetManager.h:97

TArray<FPrimaryAssetId> WeaponIds;
AM.GetPrimaryAssetIdList(FPrimaryAssetType(TEXT("MyWeaponDefinition")), WeaponIds);

const FPrimaryAssetId WeaponId(FPrimaryAssetType(TEXT("MyWeaponDefinition")), FName(TEXT("DA_Sword")));
TSharedPtr<FStreamableHandle> Handle = AM.LoadPrimaryAsset(
    WeaponId,
    TArray<FName>{ TEXT("Game") },
    FStreamableDelegate::CreateUObject(this, &UMyItemSubsystem::OnWeaponLoaded));

UMyWeaponDefinition* Def = AM.GetPrimaryAssetObject<UMyWeaponDefinition>(WeaponId);
AM.ChangeBundleStateForPrimaryAssets(
    { WeaponId },
    TArray<FName>{ TEXT("Game") },   // AddBundles
    TArray<FName>{ TEXT("UI") });    // RemoveBundles
const int32 NumUnloaded = AM.UnloadPrimaryAsset(WeaponId);
```

Signatures (`Engine/AssetManager.h:333,340`):

```cpp
virtual TSharedPtr<FStreamableHandle> LoadPrimaryAsset(
    const FPrimaryAssetId& AssetToLoad,
    const TArray<FName>& LoadBundles = TArray<FName>(),
    FStreamableDelegate DelegateToCall = FStreamableDelegate(),
    TAsyncLoadPriority Priority = FStreamableManager::DefaultAsyncLoadPriority,
    UE::FSourceLocation Location = UE::FSourceLocation::Current());

virtual TSharedPtr<FStreamableHandle> LoadPrimaryAsset(
    const FPrimaryAssetId& AssetToLoad,
    const TArray<FName>& LoadBundles,
    FAssetManagerLoadParams&& LoadParams,
    UE::FSourceLocation Location = UE::FSourceLocation::Current());
```

`LoadPrimaryAssets` (plural), `LoadPrimaryAssetsWithType`, `ChangeBundleStateForPrimaryAssets` and `ChangeBundleStateForMatchingPrimaryAssets` all come in the same pair of shapes. `Location` is filled in by the compiler — never pass it. Prefer the `FAssetManagerLoadParams` overload for new code; it carries `OnComplete`, `OnCancel` (both `FStreamableDelegateWithHandle`), `OnUpdate` and `Priority`.

Unlike raw streamable requests, primary assets **stay loaded until you unload them**: you do not have to keep the returned handle alive, only to poll or wait on it.

`ScanPathsForPrimaryAssets(FPrimaryAssetType, const TArray<FString>& Paths, UClass* BaseClass, bool bHasBlueprintClasses, bool bIsEditorOnly = false, bool bForceSynchronousScan = true)` registers types from code when config-driven scanning is not enough. `AddDynamicAsset(const FPrimaryAssetId&, const FSoftObjectPath&, const FAssetBundleData&)` registers a runtime-generated primary asset.

## Cook Rules

```ini
; DefaultGame.ini
[/Script/Engine.AssetManagerSettings]
+PrimaryAssetTypesToScan=(PrimaryAssetType="MyWeaponDefinition",AssetBaseClass="/Script/MyGame.MyWeaponDefinition",bHasBlueprintClasses=False,bIsEditorOnly=False,Directories=((Path="/Game/Data/Weapons")),Rules=(Priority=1,ChunkId=-1,bApplyRecursively=True,CookRule=AlwaysCook))
```

`FPrimaryAssetTypeInfo` fields (`Engine/AssetManagerTypes.h:131`): `PrimaryAssetType`, `AssetBaseClass`, `bHasBlueprintClasses`, `bIsEditorOnly`, `Directories`, `SpecificAssets`, `Rules`. `FPrimaryAssetRules` (`Engine/AssetManagerTypes.h:66`): `Priority`, `ChunkId`, `bApplyRecursively`, `CookRule`.

`EPrimaryAssetCookRule` (`Engine/AssetManagerTypes.h:28`):

| Value | Meaning |
|---|---|
| `Unknown` | Cooks in development and production only if something references it |
| `NeverCook` | Never cooked; a dependency on it is an error |
| `ProductionNeverCook` | Cooks in development if referenced, never in production |
| `DevelopmentAlwaysProductionNeverCook` | Always cooked in development, never in production |
| `DevelopmentAlwaysProductionUnknownCook` | Always in development; production only if referenced |
| `AlwaysCook` | Always cooked in both |

`UAssetManagerSettings` (`config = Game`, `defaultconfig`) also exposes `DirectoriesToExclude`, `PrimaryAssetRules`, `CustomPrimaryAssetRules` and `bOnlyCookProductionAssets` — turn the last one on for shipping branches so `ProductionNeverCook` assets error instead of leaking into the build.

An asset reachable neither from a scanned primary asset nor from a hard reference is not cooked. A soft reference alone does not pull an asset into the cook; give it a primary asset type or a `UPrimaryAssetLabel`.

## Asset Registry Queries

`IAssetRegistry` reads asset metadata without loading the assets.

```cpp
#include "AssetRegistry/ARFilter.h"
#include "AssetRegistry/AssetData.h"
#include "AssetRegistry/IAssetRegistry.h"

IAssetRegistry& AR = IAssetRegistry::GetChecked();   // AssetRegistry/IAssetRegistry.h:272

FARFilter Filter;
Filter.PackagePaths.Add(FName(TEXT("/Game/Data/Weapons")));
Filter.bRecursivePaths = true;
Filter.ClassPaths.Add(FTopLevelAssetPath(TEXT("/Script/MyGame"), TEXT("MyWeaponDefinition")));
Filter.bRecursiveClasses = true;

TArray<FAssetData> Found;
AR.GetAssets(Filter, Found);

// Direct call; the class argument is FTopLevelAssetPath, never FName.
AR.GetAssetsByClass(FTopLevelAssetPath(TEXT("/Script/MyGame"), TEXT("MyWeaponDefinition")), Found, /*bSearchSubClasses=*/true);

for (const FAssetData& Data : Found)
{
    const FTopLevelAssetPath ClassPath = Data.AssetClassPath;   // Data.AssetName for the short name.

    FString TagValue;
    Data.GetTagValue(FName(TEXT("WeaponClassTag")), TagValue);  // Registry read, still no load.

    UObject* Asset = Data.GetAsset();                           // This one does load.
}
```

`IAssetRegistry::Get()` returns a pointer that can be null before the module is up; `GetChecked()` asserts instead, and `FAssetRegistryModule::GetRegistry()` is the module-level equivalent. `AssetRegistrySearchable` on a UPROPERTY writes that property into the registry so `FARFilter::TagsAndValues` and `GetAssetsByTagValues` can filter on it without loading. Use `ScanPathsSynchronous()` when assets may not have been discovered yet.

## Data Registry

DataRegistry (Beta in 5.8; plugin off by default, module `DataRegistry`) resolves one `FDataRegistryId` against a chain of sources — data tables, curve tables or custom ones — so gameplay code asks for an item by id without knowing which table or DLC provides it.

```cpp
#include "DataRegistryId.h"
#include "DataRegistrySubsystem.h"

UDataRegistrySubsystem* Registry = UDataRegistrySubsystem::Get();
const FDataRegistryId ItemId(FName(TEXT("ItemRegistry")), FName(TEXT("Sword")));

// Cached read: non-null only if the source is already resident.
const FMyItemTableRow* Row = Registry->GetCachedItem<FMyItemTableRow>(ItemId);

// Otherwise request it, then re-query the cache in the callback.
Registry->AcquireItem(ItemId, FDataRegistryItemAcquiredCallback::CreateWeakLambda(this,
    [ItemId](const FDataRegistryAcquireResult& Result)
    {
        const FMyItemTableRow* Acquired =
            UDataRegistrySubsystem::Get()->GetCachedItem<FMyItemTableRow>(ItemId);
    }));
```

`UDataRegistrySource_DataTable` points a registry at a `TSoftObjectPtr<UDataTable> SourceTable` with `FDataRegistrySource_DataTableRules` (`bPrecacheTable`, `CachedTableKeepSeconds`); `UDataRegistrySource_CurveTable` does the same for curves. Registry assets are configured through `UDataRegistrySettings`.

## Project Settings

`UDeveloperSettings` (module `DeveloperSettings`) auto-registers a class in Project Settings and reads from an ini section named after the class.

Declare the class `UCLASS(config = Game, defaultconfig, meta = (DisplayName = "My Game"))` deriving `UDeveloperSettings`, mark each field `UPROPERTY(config, EditAnywhere, Category = "Economy")`, and read it with `GetDefault<UMyGameSettings>()`. `config = Game` plus `defaultconfig` writes to `DefaultGame.ini` under `[/Script/MyGame.MyGameSettings]`; override `GetContainerName()`, `GetCategoryName()` or `GetSectionName()` to place the page. Use this for a handful of global tunables and data assets for anything designers iterate on per entry — the full class, with includes and the Build.cs dependency, is in [data-driven design patterns](references/data-driven-design.md).

## Deprecated — do not use

| Do not emit | Use in 5.8 | Source |
|---|---|---|
| `TAssetPtr<T>`, `TAssetSubclassOf<T>` | `TSoftObjectPtr<T>`, `TSoftClassPtr<T>` | Absent from 5.8 headers; `UObject/SoftObjectPtr.h:173,795` |
| `FStringAssetReference` | `FSoftObjectPath` | Absent from 5.8 headers; `UObject/SoftObjectPath.h` |
| `UAssetManager::IsValid()`, `GetIfValid()` | `IsInitialized()`, `GetIfInitialized()` | `UE_DEPRECATED(5.3)` in `Engine/AssetManager.h:91,99` |
| `OnAssetStateChangeCompleted(...)` override | `NotifyOnAssetStateChangeCompleted` | `UE_DEPRECATED(5.6)` in `Engine/AssetManager.h:1035` |
| `LoadAssetListInternal(...)` | `LoadAssetList(...)` taking `FAssetManagerLoadParams` | `UE_DEPRECATED(5.6)` in `Engine/AssetManager.h:1072` |
| `ShouldSetManager(Manager, Source, Target, ...)` | `ShouldSetManager(UE::AssetRegistry::FShouldSetManagerContext&)` | `UE_DEPRECATED(5.6)` in `Engine/AssetManager.h:1082` |
| `UpdateManagementDatabase(bool)` | `UpdateManagementDatabase(EUpdateManagementDatabaseFlags)` | `UE_DEPRECATED(5.6)` in `Engine/AssetManager.h:924` |
| `BuildChunkMap(PackagesToUpdateChunksFor)` | the 3-argument `BuildChunkMap` | `UE_DEPRECATED(5.6)` in `Engine/AssetManager.h:1129` |
| `FAssetData::AssetClass` (short name) | `FAssetData::AssetClassPath` (`FTopLevelAssetPath`) | `UE_DEPRECATED(5.1)` in `AssetRegistry/AssetData.h:198` |
| `GetAssetsByClass(FName ClassName, ...)` | `GetAssetsByClass(FTopLevelAssetPath, ...)` | `AssetRegistry/IAssetRegistry.h:335` |
| `UCurveTable::GetCurves()` returning an array | `GetCurves(TAdderReserverRef<FRichCurveEditInfoConst>)` | `UE_DEPRECATED(5.6)` in `Engine/CurveTable.h:106` |
| `UDataTable::RowStructName` | `GetRowStruct()` at runtime; `RowStructPathName` is `WITH_EDITORONLY_DATA` | `RowStructName_DEPRECATED` in `Engine/DataTable.h:168`; `GetRowStruct()` at `:110` |
| `GENERATED_USTRUCT_BODY()` in row structs | `GENERATED_BODY()` | Both compile; only one is current house style |
| `"AssetManager"` in a `Build.cs` dependency list | `"Engine"` | No such module; `Engine/Classes/Engine/AssetManager.h` |

## Common Mistakes

**Letting the streamable handle die:** loaded assets are referenced by the handle, not by the soft pointer. Store `TSharedPtr<FStreamableHandle>` as a member until a hard reference (component, `TObjectPtr`, array) owns the asset, then reset it.

**Hard-referencing content from a data asset:** `TObjectPtr<UNiagaraSystem>` in a `UPrimaryDataAsset` loads the effect with the definition. Use `TSoftObjectPtr` plus an asset bundle.

**`LoadSynchronous()` in `Tick` or on gameplay-critical paths:** it stalls the game thread whenever the asset is not already resident. Start one async load and act in the callback.

**Passing `FName` to `GetAssetsByClass`:** 5.8 takes `FTopLevelAssetPath(TEXT("/Script/Module"), TEXT("ClassName"))`. A short class name will not compile.

**Forgetting `PrimaryAssetTypesToScan`:** without a registered type, `GetPrimaryAssetIdList` returns nothing and `LoadPrimaryAsset` silently does nothing. Register the type in `DefaultGame.ini` or call `ScanPathsForPrimaryAssets`.

**Capturing `this` in a bare `CreateLambda`:** the callback can fire after the object is gone. Use `CreateUObject` or `CreateWeakLambda(this, Lambda)`.

**Calling `GetTableAsCSV()` at runtime:** it is `WITH_EDITOR` only and will not link in a cooked build. Serialise the rows yourself instead. Likewise, renaming or removing a row-struct property makes existing rows lose that column on the next import — export the table to CSV before changing the struct, then re-import.

## Related Skills

- `ue-cpp-foundations` — `UPROPERTY`/`USTRUCT` specifiers, `TObjectPtr`, UObject lifecycle, the subsystem table
- `ue-async-threading` — general async idioms (`UE::Tasks`, `Async`, `FTSTicker`) beyond streamable loading
- `ue-serialization-savegames` — persisting soft object paths and primary asset ids across sessions
- `ue-game-features` — shipping data assets inside plugins and game feature asset scanning
- `ue-gameplay-abilities` — ability and effect definitions that consume these data assets
- `ue-world-level-streaming` — level streaming, World Partition and Data Layers
- `ue-module-build-system` — adding `AssetRegistry`, `DeveloperSettings` and plugin modules to `Build.cs`
- `ue-audio-system` — UAudioComponent, MetaSounds, submixes, attenuation and concurrency
- `ue-blueprint-cpp-interop` — exposing C++ to Blueprint: UFUNCTION/UPROPERTY meta keys, latent actions and async nodes
- `ue-editor-tools` — detail customizations, editor utility widgets, UToolMenus and editor subsystems
- `ue-gameplay-tags-messaging` — native and ini gameplay tags, containers, queries and the async message system
- `ue-materials-rendering` — material instances, parameter collections, render targets and post process
- `ue-procedural-generation` — PCG graphs, procedural and dynamic meshes, instancing and splines
- `ue-ui-umg-slate` — UMG widgets, Slate, Common UI and MVVM
