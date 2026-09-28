# Data-Driven Design in Unreal Engine

Patterns for structuring game data with `UDataAsset`, `UPrimaryDataAsset`, `UDataTable` and `UDeveloperSettings` so designers control gameplay values without C++ changes. Target engine: **UE 5.8**.

---

## Core Principle

C++ defines the schema; designers populate instances in the editor or in spreadsheets.

| Layer | Owner | Tool |
|---|---|---|
| Data schema | Programmer | C++ `USTRUCT` / `UCLASS` |
| Data values | Designer | Data Asset editor, CSV/JSON |
| Data loading | Programmer | Asset Manager, soft references |
| Data consumption | Programmer | `FindRow`, `GetPrimaryAssetObject` |

---

## Pattern A: Item / Ability Definitions (`UPrimaryDataAsset`)

Best when each entry has unique properties, may need Blueprint extension, and should integrate with the Asset Manager for selective loading.

```cpp
// MyItemDefinition.h
#pragma once

#include "CoreMinimal.h"
#include "Engine/DataAsset.h"
#include "MyItemDefinition.generated.h"

class UStaticMesh;
class UTexture2D;   // TSoftObjectPtr<T> needs T declared; neither DataAsset.h nor CoreMinimal.h declares them

UENUM(BlueprintType)
enum class EMyItemRarity : uint8
{
    Common,
    Uncommon,
    Rare,
    Legendary
};

UCLASS(BlueprintType)
class MYGAME_API UMyItemDefinition : public UPrimaryDataAsset
{
    GENERATED_BODY()

public:
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Item")
    FText DisplayName;

    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Item")
    FText Description;

    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Item")
    EMyItemRarity Rarity = EMyItemRarity::Common;

    /** Searchable so the Asset Registry can filter categories without loading. */
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Item", AssetRegistrySearchable)
    FName ItemCategory;

    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Stats")
    float BaseDamage = 0.f;

    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Stats")
    int32 MaxStack = 1;

    /** UI bundle: loaded for the inventory screen. */
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Art", meta = (AssetBundles = "UI"))
    TSoftObjectPtr<UTexture2D> Icon;

    /** Game bundle: loaded when the item can appear in the world. */
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Art", meta = (AssetBundles = "Game"))
    TSoftObjectPtr<UStaticMesh> WorldMesh;

    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Spawning", meta = (AssetBundles = "Game"))
    TSoftClassPtr<AActor> DroppedActorClass;
};
```

### Registration

```ini
; DefaultGame.ini
[/Script/Engine.AssetManagerSettings]
+PrimaryAssetTypesToScan=(PrimaryAssetType="MyItemDefinition",AssetBaseClass="/Script/MyGame.MyItemDefinition",bHasBlueprintClasses=False,bIsEditorOnly=False,Directories=((Path="/Game/Data/Items")),Rules=(Priority=1,ChunkId=-1,bApplyRecursively=True,CookRule=AlwaysCook))
```

Set `bHasBlueprintClasses=True` and point `Directories` at the Blueprint folder when designers author Data Only Blueprints instead of Data Asset instances; the scanned objects are then `UClass` objects and you read them with `GetPrimaryAssetObjectClass<UMyItemDefinition>()`.

### Discovery and loading

```cpp
UAssetManager& AM = UAssetManager::Get();

// Ids only; nothing is loaded yet.
TArray<FPrimaryAssetId> AllItemIds;
AM.GetPrimaryAssetIdList(FPrimaryAssetType(TEXT("MyItemDefinition")), AllItemIds);

// Filter without loading, using the AssetRegistrySearchable tag.
IAssetRegistry& AR = IAssetRegistry::GetChecked();
FARFilter Filter;
Filter.ClassPaths.Add(FTopLevelAssetPath(TEXT("/Script/MyGame"), TEXT("MyItemDefinition")));
Filter.bRecursiveClasses = true;
Filter.TagsAndValues.Add(FName(TEXT("ItemCategory")), TOptional<FString>(TEXT("Melee")));

TArray<FAssetData> MeleeAssets;
AR.GetAssets(Filter, MeleeAssets);

// Load the UI bundle for the whole type.
AM.LoadPrimaryAssetsWithType(
    FPrimaryAssetType(TEXT("MyItemDefinition")),
    TArray<FName>{ TEXT("UI") });

// After the callback, resolve one definition.
const FPrimaryAssetId SwordId(FPrimaryAssetType(TEXT("MyItemDefinition")), FName(TEXT("DA_Sword")));
UMyItemDefinition* Sword = AM.GetPrimaryAssetObject<UMyItemDefinition>(SwordId);
```

---

## Pattern B: Stat Tables (`UDataTable`)

Best for large, flat, designer-authored datasets where every row has the same shape: XP curves, damage falloff, dialogue, loot weights.

```cpp
// MyXPTableRow.h
#pragma once

#include "CoreMinimal.h"
#include "Engine/DataTable.h"
#include "MyXPTableRow.generated.h"

USTRUCT(BlueprintType)
struct FMyXPTableRow : public FTableRowBase
{
    GENERATED_BODY()

    /** Player level. Also used as part of the row name. */
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Progression")
    int32 Level = 1;

    /** XP required to reach this level from the previous one. */
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Progression")
    int32 XPRequired = 0;

    /** Stat multiplier applied at this level. */
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Progression")
    float StatMultiplier = 1.f;

    virtual void OnPostDataImport(const UDataTable* InDataTable, const FName InRowName, TArray<FString>& OutCollectedImportProblems) override
    {
        if (XPRequired < 0)
        {
            OutCollectedImportProblems.Add(
                FString::Printf(TEXT("Row '%s': XPRequired cannot be negative."), *InRowName.ToString()));
        }
    }
};
```

`FTableRowBase` also declares `OnDataTableChanged(const UDataTable*, const FName)` — called for every row whenever the table is edited or `HandleDataTableChanged()` runs — and, inside `#if WITH_EDITOR`, `IsDataValid(FDataValidationContext&) const`.

### CSV format

```csv
---,Level,XPRequired,StatMultiplier
Level_1,1,0,1.0
Level_2,2,100,1.05
Level_3,3,250,1.10
Level_4,4,500,1.15
```

The first column (`---` or `Name`) is the row name. Import through the DataTable asset's Reimport button, or call `CreateTableFromCSVString` at runtime once `RowStruct` is set.

### Runtime lookup

```cpp
// MyLevelingComponent.h — member
UPROPERTY(EditDefaultsOnly, Category = "Data")
TObjectPtr<UDataTable> XPTable;
```

```cpp
const FMyXPTableRow* UMyLevelingComponent::GetRowForLevel(int32 Level) const
{
    if (!XPTable)
    {
        return nullptr;
    }
    const FName RowName = FName(*FString::Printf(TEXT("Level_%d"), Level));
    return XPTable->FindRow<FMyXPTableRow>(RowName, TEXT("GetRowForLevel"));
}

int32 UMyLevelingComponent::GetXPToNextLevel(int32 CurrentLevel) const
{
    const FMyXPTableRow* Row = GetRowForLevel(CurrentLevel + 1);
    return Row ? Row->XPRequired : 0;
}
```

The table is a hard `TObjectPtr` here because it is small and always needed. Use `TSoftObjectPtr<UDataTable>` when the table only matters in one mode or level.

---

## Pattern C: Config Objects (plain `UDataAsset`)

Use plain `UDataAsset` for configuration that is always loaded with whoever references it, is not addressable by id, and needs no Asset Manager integration.

```cpp
// MyGameBalanceConfig.h
#pragma once

#include "CoreMinimal.h"
#include "Engine/DataAsset.h"
#include "MyGameBalanceConfig.generated.h"

UCLASS(BlueprintType)
class MYGAME_API UMyGameBalanceConfig : public UDataAsset
{
    GENERATED_BODY()

public:
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Economy")
    int32 StartingGold = 100;

    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Economy")
    float SellPriceMultiplier = 0.5f;

    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Combat")
    float GlobalDamageScale = 1.f;
};
```

Reference it from the GameMode or GameInstance with a hard `TObjectPtr<UMyGameBalanceConfig>`.

---

## Pattern D: Hybrid — Data Asset plus Data Table

An item definition owns identity and art; a soft-referenced table carries per-tier scaling.

```cpp
// MyWeaponScalingRow.h
USTRUCT(BlueprintType)
struct FMyWeaponScalingRow : public FTableRowBase
{
    GENERATED_BODY()

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Scaling")
    float Damage = 0.f;

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Scaling")
    float CritChance = 0.f;
};
```

```cpp
// Inside UMyWeaponDefinition : public UPrimaryDataAsset
UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Stats", meta = (AssetBundles = "Game"))
TSoftObjectPtr<UDataTable> ScalingTable;

float UMyWeaponDefinition::GetDamageAtTier(int32 Tier) const
{
    // The Game bundle must already be loaded.
    UDataTable* Table = ScalingTable.Get();
    if (!Table)
    {
        return 0.f;
    }

    const FName RowName = FName(*FString::Printf(TEXT("Tier_%d"), Tier));
    const FMyWeaponScalingRow* Row = Table->FindRow<FMyWeaponScalingRow>(RowName, TEXT("GetDamageAtTier"));
    return Row ? Row->Damage : 0.f;
}
```

---

## Pattern E: Subsystem as Data Gateway

A `UGameInstanceSubsystem` scans once and caches, so the rest of the codebase never touches the Asset Manager directly.

```cpp
// MyItemSubsystem.h
#pragma once

#include "CoreMinimal.h"
#include "Engine/StreamableManager.h"
#include "Subsystems/GameInstanceSubsystem.h"
#include "MyItemSubsystem.generated.h"

class UMyItemDefinition;

UCLASS()
class MYGAME_API UMyItemSubsystem : public UGameInstanceSubsystem
{
    GENERATED_BODY()

public:
    virtual void Initialize(FSubsystemCollectionBase& Collection) override;
    virtual void Deinitialize() override;

    UMyItemDefinition* GetItemById(const FPrimaryAssetId& Id) const;

private:
    void OnItemsLoaded();

    UPROPERTY()
    TMap<FPrimaryAssetId, TObjectPtr<UMyItemDefinition>> ItemCache;

    TSharedPtr<FStreamableHandle> LoadHandle;
};
```

```cpp
// MyItemSubsystem.cpp
#include "MyItemSubsystem.h"

#include "Engine/AssetManager.h"
#include "MyItemDefinition.h"

void UMyItemSubsystem::Initialize(FSubsystemCollectionBase& Collection)
{
    Super::Initialize(Collection);

    UAssetManager& AM = UAssetManager::Get();
    TArray<FPrimaryAssetId> ItemIds;
    AM.GetPrimaryAssetIdList(FPrimaryAssetType(TEXT("MyItemDefinition")), ItemIds);

    LoadHandle = AM.LoadPrimaryAssets(
        ItemIds,
        TArray<FName>{ TEXT("UI") },
        FStreamableDelegate::CreateUObject(this, &UMyItemSubsystem::OnItemsLoaded));
}

void UMyItemSubsystem::Deinitialize()
{
    LoadHandle.Reset();
    ItemCache.Reset();

    Super::Deinitialize();
}

void UMyItemSubsystem::OnItemsLoaded()
{
    UAssetManager& AM = UAssetManager::Get();

    TArray<UObject*> LoadedObjects;
    AM.GetPrimaryAssetObjectList(FPrimaryAssetType(TEXT("MyItemDefinition")), LoadedObjects);

    for (UObject* Obj : LoadedObjects)
    {
        if (UMyItemDefinition* Item = Cast<UMyItemDefinition>(Obj))
        {
            ItemCache.Add(Item->GetPrimaryAssetId(), Item);
        }
    }
}

UMyItemDefinition* UMyItemSubsystem::GetItemById(const FPrimaryAssetId& Id) const
{
    const TObjectPtr<UMyItemDefinition>* Found = ItemCache.Find(Id);
    return Found ? Found->Get() : nullptr;
}
```

Primary assets stay resident until `UnloadPrimaryAssets`, so the cached pointers remain valid; the `UPROPERTY()` on `ItemCache` keeps them visible to the garbage collector regardless.

---

## Pattern F: Project Settings (`UDeveloperSettings`)

For a handful of global, programmer-owned tunables that belong in Project Settings rather than in a content asset.

```cpp
// MyGameSettings.h
#pragma once

#include "CoreMinimal.h"
#include "Engine/DeveloperSettings.h"
#include "MyGameSettings.generated.h"

class UDataTable;

UCLASS(config = Game, defaultconfig, meta = (DisplayName = "My Game"))
class MYGAME_API UMyGameSettings : public UDeveloperSettings
{
    GENERATED_BODY()

public:
    UPROPERTY(config, EditAnywhere, Category = "Economy")
    int32 StartingGold = 100;

    UPROPERTY(config, EditAnywhere, Category = "Data")
    TSoftObjectPtr<UDataTable> ItemTable;

    virtual FName GetCategoryName() const override;
};
```

```cpp
// MyGameSettings.cpp
#include "MyGameSettings.h"

FName UMyGameSettings::GetCategoryName() const
{
    return FName(TEXT("Game"));
}
```

```cpp
const UMyGameSettings* Settings = GetDefault<UMyGameSettings>();
const int32 Gold = Settings->StartingGold;
```

`config = Game` plus `defaultconfig` writes `DefaultGame.ini` under `[/Script/MyGame.MyGameSettings]`. The Build.cs dependency is `DeveloperSettings`.

---

## When to Use Each Approach

| Scenario | Approach |
|---|---|
| Per-item configs, designers work in the editor | `UPrimaryDataAsset` + Asset Manager |
| Large flat tables, designers work in a spreadsheet | `UDataTable` + CSV import |
| Global settings, always in memory | plain `UDataAsset`, hard reference |
| Global settings owned by programmers | `UDeveloperSettings` |
| Base rows plus DLC or platform overrides | `UCompositeDataTable` |
| Server-only data (drop rates, economy) | `UDataTable` with `bStripFromClientBuilds` |
| Items with per-tier scaling | Data asset + soft reference to a `UDataTable` |
| Runtime-generated content | `UDataTable::AddRow`, or `UAssetManager::AddDynamicAsset` |
| Querying without loading | `IAssetRegistry::GetAssets` + `FAssetData` |
| One id resolved across several sources | `UDataRegistrySubsystem` (Beta in 5.8) |

---

## Designer Workflow Checklist

1. **Schema compiled**: the row struct or data asset class is visible in the editor.
2. **Type registered**: `DefaultGame.ini` has the matching `PrimaryAssetTypesToScan` entry, or `ScanPathsForPrimaryAssets` runs at startup.
3. **Assets created**: instances live under one of the scanned `Directories`.
4. **Data populated**: fields filled in the editor, or CSV/JSON imported.
5. **Cook verified**: run a development cook and confirm the assets appear in the output; check the log for assets excluded by their `EPrimaryAssetCookRule`.
6. **Bundle states tested**: the UI bundle loads in menus and the Game bundle loads on gameplay entry without a hitch.
7. **Memory checked**: no `TObjectPtr` to heavy content on a definition that ships in every build.
