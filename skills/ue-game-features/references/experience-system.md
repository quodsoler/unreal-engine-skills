# Experience System Reference

> **Lyra sample, not engine.** The "experience system" is a pattern implemented in Epic's Lyra
> sample project. `ULyraExperienceDefinition`, `ULyraExperienceManagerComponent`,
> `ULyraPawnData` and Lyra's own action subclasses (for example
> `UGameFeatureAction_AddInputContextMapping`) exist **only in Lyra**, not in any UE 5.8 engine
> header. Do not `#include` them, do not reference them from a project that has not copied them,
> and do not tell a user they ship with the engine. Everything below is a template for classes
> **you write in your own module**, named `UMy*` here.

The engine pieces this pattern builds on, all real UE 5.8 API:

| Engine type | Module | Role |
|---|---|---|
| `UGameFeaturesSubsystem` | `GameFeatures` | Loads, activates and deactivates feature plugins. |
| `UGameFeatureData` | `GameFeatures` | Per-plugin data asset holding the action list. |
| `UGameFeatureAction` | `GameFeatures` | Base class for work done on activation/deactivation. |
| `UGameFrameworkComponentManager` | `ModularGameplay` | Injects components into registered actors. |
| `UGameStateComponent` | `ModularGameplay` | Base for a component living on `AGameStateBase`. |
| `UPrimaryDataAsset`, `UAssetManager` | `Engine` | Primary asset definition and async loading. |

Both `GameFeatures` and `ModularGameplay` are Beta in 5.8.

---

## Why compose a game mode from features

A monolithic `AGameMode` subclass per game type duplicates shared systems. Instead, define a
lightweight primary data asset per mode that lists which Game Feature plugins to activate, and let
each plugin bring its own components, input, abilities and UI.

```
B_MyDeathmatch (UMyExperienceDefinition)
├── GameFeaturesToEnable:
│   ├── "MyShooterCore"      → health, weapons, HUD, hit detection
│   ├── "MyDeathmatchRules"  → score tracking, kill feed, respawn timer
│   └── "MyTeamSystem"       → team assignment, team colors, team HUD
├── Actions:
│   └── UGameFeatureAction_AddComponents: UMyScoreComponent → AGameStateBase
└── DefaultPawnData: BP_MyShooterCharacter
```

Switching to another mode replaces `MyDeathmatchRules` with a different rules plugin while keeping
the shared ones. (Lyra ships the same idea under the names `ULyraExperienceDefinition`,
`B_ShooterCore` and so on — Lyra sample, not engine.)

---

## Experience definition asset

`UGameFeatureAction` is an engine class, so an experience asset can carry a list of actions that run
without a dedicated plugin. `GameFeaturesToEnable` holds plugin **names**; the manager resolves each
one to a URL with `UGameFeaturesSubsystem::GetPluginURLByName` before loading it.

```cpp
// MyExperienceDefinition.h
#pragma once

#include "Engine/DataAsset.h"
#include "MyExperienceDefinition.generated.h"

class UGameFeatureAction;
class UMyPawnData;

UCLASS(BlueprintType, Const)
class MYGAME_API UMyExperienceDefinition : public UPrimaryDataAsset
{
    GENERATED_BODY()

public:
    /** Names of the Game Feature plugins this experience turns on */
    UPROPERTY(EditDefaultsOnly, Category = "Experience")
    TArray<FString> GameFeaturesToEnable;

    /** Actions run directly for this experience, without a dedicated plugin */
    UPROPERTY(EditDefaultsOnly, Instanced, Category = "Experience")
    TArray<TObjectPtr<UGameFeatureAction>> Actions;

    /** Pawn configuration this experience hands to the GameMode */
    UPROPERTY(EditDefaultsOnly, Category = "Experience")
    TObjectPtr<const UMyPawnData> DefaultPawnData;

    virtual FPrimaryAssetId GetPrimaryAssetId() const override;
};
```

```cpp
// MyExperienceDefinition.cpp
#include "MyExperienceDefinition.h"

FPrimaryAssetId UMyExperienceDefinition::GetPrimaryAssetId() const
{
    return FPrimaryAssetId(FPrimaryAssetType(TEXT("MyExperience")), GetFName());
}
```

Register `MyExperience` as a primary asset type in Project Settings → Asset Manager, or through a
`UGameFeatureData`'s `PrimaryAssetTypesToScan` when the experiences live inside a feature plugin.
See `ue-data-assets-tables`.

---

## Experience manager component

A `UGameStateComponent` (engine class, `ModularGameplay`) on the game state is the natural owner:
the game state exists on server and clients. The component replicates the chosen experience, and
both sides run the same loading flow when it arrives.

```cpp
// MyExperienceManagerComponent.h
#pragma once

#include "Components/GameStateComponent.h"
#include "MyExperienceManagerComponent.generated.h"

class UMyExperienceDefinition;
namespace UE::GameFeatures { struct FResult; }

DECLARE_MULTICAST_DELEGATE_OneParam(FMyOnExperienceLoaded, const UMyExperienceDefinition*);

UCLASS()
class MYGAME_API UMyExperienceManagerComponent : public UGameStateComponent
{
    GENERATED_BODY()

public:
    UMyExperienceManagerComponent(const FObjectInitializer& ObjectInitializer);

    /** Server only: called from the GameMode once the experience id is known */
    void SetCurrentExperience(FPrimaryAssetId ExperienceId);

    bool IsExperienceLoaded() const { return bExperienceLoaded; }

    const UMyExperienceDefinition* GetCurrentExperience() const { return CurrentExperience; }

    /** Fires immediately when the experience is already loaded, otherwise on load */
    void CallOrRegister_OnExperienceLoaded(FMyOnExperienceLoaded::FDelegate&& Delegate);

    virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override;
    virtual void EndPlay(const EEndPlayReason::Type EndPlayReason) override;

private:
    void OnExperienceAssetLoaded();
    void StartExperienceLoad();
    void OnGameFeaturePluginLoadComplete(const UE::GameFeatures::FResult& Result);
    void BroadcastExperienceLoaded();

    UFUNCTION()
    void OnRep_CurrentExperience();

    UPROPERTY(ReplicatedUsing = OnRep_CurrentExperience)
    TObjectPtr<const UMyExperienceDefinition> CurrentExperience;

    FMyOnExperienceLoaded OnExperienceLoaded;
    FPrimaryAssetId PendingExperienceId;
    int32 NumFeaturePluginsLoading = 0;
    bool bExperienceLoaded = false;
};
```

---

## Loading flow

```cpp
// MyExperienceManagerComponent.cpp
#include "MyExperienceManagerComponent.h"
#include "MyExperienceDefinition.h"
#include "Engine/AssetManager.h"
#include "Engine/Engine.h"
#include "GameFeatureAction.h"
#include "GameFeaturesSubsystem.h"
#include "Net/UnrealNetwork.h"

UMyExperienceManagerComponent::UMyExperienceManagerComponent(const FObjectInitializer& ObjectInitializer)
    : Super(ObjectInitializer)
{
    SetIsReplicatedByDefault(true);
}

void UMyExperienceManagerComponent::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);
    DOREPLIFETIME(UMyExperienceManagerComponent, CurrentExperience);
}

void UMyExperienceManagerComponent::SetCurrentExperience(FPrimaryAssetId ExperienceId)
{
    PendingExperienceId = ExperienceId;

    UAssetManager::Get().LoadPrimaryAsset(ExperienceId, TArray<FName>(),
        FStreamableDelegate::CreateUObject(
            this, &UMyExperienceManagerComponent::OnExperienceAssetLoaded));
}

void UMyExperienceManagerComponent::OnExperienceAssetLoaded()
{
    CurrentExperience = Cast<UMyExperienceDefinition>(
        UAssetManager::Get().GetPrimaryAssetObject(PendingExperienceId));
    if (!CurrentExperience)
    {
        UE_LOG(LogMyGame, Error, TEXT("Experience asset failed to load"));
        return;
    }

    // Setting CurrentExperience replicates it; clients continue in OnRep_CurrentExperience.
    StartExperienceLoad();
}

void UMyExperienceManagerComponent::OnRep_CurrentExperience()
{
    if (CurrentExperience)
    {
        StartExperienceLoad();
    }
}

void UMyExperienceManagerComponent::StartExperienceLoad()
{
    UGameFeaturesSubsystem& GFS = UGameFeaturesSubsystem::Get();
    NumFeaturePluginsLoading = CurrentExperience->GameFeaturesToEnable.Num();

    if (NumFeaturePluginsLoading == 0)
    {
        BroadcastExperienceLoaded();
        return;
    }

    for (const FString& PluginName : CurrentExperience->GameFeaturesToEnable)
    {
        FString PluginURL;
        if (!GFS.GetPluginURLByName(PluginName, PluginURL))
        {
            UE_LOG(LogMyGame, Error, TEXT("Unknown game feature plugin %s"), *PluginName);
            --NumFeaturePluginsLoading;
            continue;
        }

        GFS.LoadAndActivateGameFeaturePlugin(PluginURL,
            FGameFeaturePluginLoadComplete::CreateUObject(
                this, &UMyExperienceManagerComponent::OnGameFeaturePluginLoadComplete));
    }

    if (NumFeaturePluginsLoading == 0)
    {
        BroadcastExperienceLoaded();
    }
}

void UMyExperienceManagerComponent::OnGameFeaturePluginLoadComplete(
    const UE::GameFeatures::FResult& Result)
{
    if (Result.HasError())
    {
        UE_LOG(LogMyGame, Error, TEXT("Feature plugin failed: %s"), *Result.GetError());
    }

    --NumFeaturePluginsLoading;
    if (NumFeaturePluginsLoading == 0)
    {
        BroadcastExperienceLoaded();
    }
}

void UMyExperienceManagerComponent::BroadcastExperienceLoaded()
{
    // Run the experience's own actions, limited to this world so other PIE instances are untouched.
    FGameFeatureActivatingContext Context;
    if (const FWorldContext* WorldContext = GEngine->GetWorldContextFromWorld(GetWorld()))
    {
        Context.SetRequiredWorldContextHandle(WorldContext->ContextHandle);
    }
    for (UGameFeatureAction* Action : CurrentExperience->Actions)
    {
        if (Action)
        {
            Action->OnGameFeatureRegistering();
            Action->OnGameFeatureLoading();
            Action->OnGameFeatureActivating(Context);
        }
    }

    bExperienceLoaded = true;
    OnExperienceLoaded.Broadcast(CurrentExperience);
    OnExperienceLoaded.Clear();
}

void UMyExperienceManagerComponent::CallOrRegister_OnExperienceLoaded(
    FMyOnExperienceLoaded::FDelegate&& Delegate)
{
    if (bExperienceLoaded)
    {
        Delegate.Execute(CurrentExperience);
    }
    else
    {
        OnExperienceLoaded.Add(MoveTemp(Delegate));
    }
}

void UMyExperienceManagerComponent::EndPlay(const EEndPlayReason::Type EndPlayReason)
{
    if (bExperienceLoaded && CurrentExperience)
    {
        // Undo the actions in the same world; no action here pauses deactivation.
        FGameFeatureDeactivatingContext Context(TEXT(""), [](FStringView) {});
        if (const FWorldContext* WorldContext = GEngine->GetWorldContextFromWorld(GetWorld()))
        {
            Context.SetRequiredWorldContextHandle(WorldContext->ContextHandle);
        }
        for (UGameFeatureAction* Action : CurrentExperience->Actions)
        {
            if (Action)
            {
                Action->OnGameFeatureDeactivating(Context);
                Action->OnGameFeatureUnregistering();
            }
        }
    }

    Super::EndPlay(EndPlayReason);
}
```

The GameMode picks the experience and hands it over. `AGameModeBase::InitGame` runs before the game
state exists, so resolve the id there and push it to the component once `InitGameState` has run:

```cpp
void AMyGameMode::InitGameState()
{
    Super::InitGameState();

    const FPrimaryAssetId ExperienceId(FPrimaryAssetType(TEXT("MyExperience")),
        FName(TEXT("B_MyDeathmatch")));

    if (UMyExperienceManagerComponent* ExperienceManager =
        GameState->FindComponentByClass<UMyExperienceManagerComponent>())
    {
        ExperienceManager->SetCurrentExperience(ExperienceId);
    }
}
```

---

## Consuming the experience

Anything that depends on feature-provided components must wait rather than assume they exist at
`BeginPlay`:

```cpp
void UMyPawnComponent::BeginPlay()
{
    Super::BeginPlay();

    if (AGameStateBase* GameStateBase = GetWorld()->GetGameState())
    {
        if (UMyExperienceManagerComponent* ExperienceManager =
            GameStateBase->FindComponentByClass<UMyExperienceManagerComponent>())
        {
            ExperienceManager->CallOrRegister_OnExperienceLoaded(
                FMyOnExperienceLoaded::FDelegate::CreateUObject(
                    this, &UMyPawnComponent::OnExperienceReady));
        }
    }
}

void UMyPawnComponent::OnExperienceReady(const UMyExperienceDefinition* Experience)
{
    // All feature plugins are active: injected components exist and can be configured.
}
```

For per-actor ordering inside one experience, prefer the engine's init-state system
(`UGameFrameworkComponentManager::ChangeFeatureInitState` and
`IGameFrameworkInitStateInterface`) over experience-level waiting; see the Init State System section
of this skill.
