# Game Feature Patterns Reference

Complete templates for Game Feature plugin setup, custom actions, component injection, init-state
components and project policies. All API is verified against UE 5.8 headers in
`Engine/Plugins/Runtime/GameFeatures` and `Engine/Plugins/Runtime/ModularGameplay` (both Beta in
5.8). Example types use the `AMy*` / `UMy*` / `FMy*` convention and the `MYGAME_API` export macro.

---

## GameFeature .uplugin template

The plugin must live under `<Project>/Plugins/GameFeatures/<Name>/`. There is no plugin-level
`"Type"` key; the folder plus `"ExplicitlyLoaded": true` is what makes it a Game Feature.

```json
{
	"FileVersion": 3,
	"Version": 1,
	"VersionName": "1.0",
	"FriendlyName": "My Feature",
	"Description": "Adds my gameplay feature",
	"Category": "Game Features",
	"CreatedBy": "",
	"CreatedByURL": "",
	"DocsURL": "",
	"SupportURL": "",
	"EnabledByDefault": false,
	"CanContainContent": true,
	"Installed": false,
	"ExplicitlyLoaded": true,
	"BuiltInInitialFeatureState": "Active",
	"Modules": [
		{
			"Name": "MyFeatureRuntime",
			"Type": "Runtime",
			"LoadingPhase": "Default"
		}
	],
	"Plugins": [
		{ "Name": "GameFeatures", "Enabled": true },
		{ "Name": "ModularGameplay", "Enabled": true }
	]
}
```

`BuiltInInitialFeatureState` accepts `"Installed"`, `"Registered"`, `"Loaded"` or `"Active"`; any
other value logs an error and falls back to `Active`. `Modules[].Type` takes a `ModuleHostType`
name (`Runtime`, `RuntimeNoCommandlet`, `Editor`, `Developer`, `Program`, …) and
`Modules[].LoadingPhase` a `ModuleLoadingPhase` name (`Default`, `PreDefault`, `PostConfigInit`,
`PostEngineInit`, …). Omit `Modules` entirely for a content-only feature.

The runtime module's `Build.cs` needs `GameFeatures` (which publicly re-exports `ModularGameplay`
and `DataRegistry`):

```csharp
PublicDependencyModuleNames.AddRange(new string[]
{
    "Core", "CoreUObject", "Engine", "GameFeatures", "ModularGameplay", "GameplayTags"
});
```

---

## Custom UGameFeatureAction

An action instance is shared across every world the feature applies to, so per-world state must be
keyed by `FObjectKey`. `FGameFeatureStateChangeContext::ShouldApplyToWorldContext` filters PIE
worlds.

```cpp
// MyGameFeatureAction_SpawnManagers.h
#pragma once

#include "GameFeatureAction.h"
#include "UObject/ObjectKey.h"
#include "MyGameFeatureAction_SpawnManagers.generated.h"

class AActor;
class UGameInstance;
struct FGameFeatureActivatingContext;
struct FGameFeatureDeactivatingContext;
struct FWorldContext;

UCLASS(meta = (DisplayName = "My Spawn Managers"))
class MYGAME_API UMyGameFeatureAction_SpawnManagers : public UGameFeatureAction
{
    GENERATED_BODY()

public:
    virtual void OnGameFeatureActivating(FGameFeatureActivatingContext& Context) override;
    virtual void OnGameFeatureDeactivating(FGameFeatureDeactivatingContext& Context) override;

    UPROPERTY(EditAnywhere, Category = "Spawn")
    TSubclassOf<AActor> ManagerClass;

private:
    void AddToWorld(const FWorldContext& WorldContext);
    void HandleGameInstanceStart(UGameInstance* GameInstance);

    TMap<FObjectKey, TArray<TWeakObjectPtr<AActor>>> SpawnedActorsPerWorld;
    FDelegateHandle GameInstanceStartHandle;
};
```

```cpp
// MyGameFeatureAction_SpawnManagers.cpp
#include "MyGameFeatureAction_SpawnManagers.h"
#include "GameFeaturesSubsystem.h"
#include "Engine/Engine.h"
#include "Engine/GameInstance.h"
#include "Engine/World.h"

void UMyGameFeatureAction_SpawnManagers::OnGameFeatureActivating(
    FGameFeatureActivatingContext& Context)
{
    // Worlds that already exist when the feature turns on
    for (const FWorldContext& WorldContext : GEngine->GetWorldContexts())
    {
        if (Context.ShouldApplyToWorldContext(WorldContext))
        {
            AddToWorld(WorldContext);
        }
    }

    // Worlds created afterwards (new PIE sessions, travel)
    GameInstanceStartHandle = FWorldDelegates::OnStartGameInstance.AddUObject(
        this, &UMyGameFeatureAction_SpawnManagers::HandleGameInstanceStart);
}

void UMyGameFeatureAction_SpawnManagers::OnGameFeatureDeactivating(
    FGameFeatureDeactivatingContext& Context)
{
    FWorldDelegates::OnStartGameInstance.Remove(GameInstanceStartHandle);
    GameInstanceStartHandle.Reset();

    for (TPair<FObjectKey, TArray<TWeakObjectPtr<AActor>>>& Pair : SpawnedActorsPerWorld)
    {
        for (TWeakObjectPtr<AActor>& ActorPtr : Pair.Value)
        {
            if (AActor* Actor = ActorPtr.Get())
            {
                Actor->Destroy();
            }
        }
    }
    SpawnedActorsPerWorld.Empty();
}

void UMyGameFeatureAction_SpawnManagers::AddToWorld(const FWorldContext& WorldContext)
{
    UWorld* World = WorldContext.World();
    if (!World || !World->IsGameWorld() || !ManagerClass)
    {
        return;
    }

    FActorSpawnParameters SpawnParams;
    SpawnParams.SpawnCollisionHandlingOverride =
        ESpawnActorCollisionHandlingMethod::AlwaysSpawn;

    if (AActor* Spawned = World->SpawnActor<AActor>(ManagerClass, FTransform::Identity, SpawnParams))
    {
        SpawnedActorsPerWorld.FindOrAdd(FObjectKey(World)).Add(Spawned);
    }
}

void UMyGameFeatureAction_SpawnManagers::HandleGameInstanceStart(UGameInstance* GameInstance)
{
    if (UWorld* World = GameInstance->GetWorld())
    {
        for (const FWorldContext& WorldContext : GEngine->GetWorldContexts())
        {
            if (WorldContext.World() == World)
            {
                AddToWorld(WorldContext);
                break;
            }
        }
    }
}
```

---

## Actor receiver setup

```cpp
// MyModularCharacter.cpp
#include "MyModularCharacter.h"
#include "Components/GameFrameworkComponentManager.h"

void AMyModularCharacter::BeginPlay()
{
    Super::BeginPlay();
    UGameFrameworkComponentManager::AddGameFrameworkComponentReceiver(this);
}

void AMyModularCharacter::EndPlay(const EEndPlayReason::Type EndPlayReason)
{
    UGameFrameworkComponentManager::RemoveGameFrameworkComponentReceiver(this);
    Super::EndPlay(EndPlayReason);
}
```

Call `RemoveGameFrameworkComponentReceiver` before `Super::EndPlay` so cleanup happens while the
actor is still fully valid. The static helpers forward to `AddReceiver(AActor* Receiver, bool
bAddOnlyInGameWorlds = true)` and `RemoveReceiver(AActor* Receiver)` on the manager for that actor.

---

## Component requests and extension handlers

`AddComponentRequest` and `AddExtensionHandler` both return `TSharedPtr<FComponentRequestHandle>`,
which removes the request when it is destroyed. `TSharedPtr` is not a reflected type, so these
members carry no `UPROPERTY()`.

```cpp
// MyGameFeatureAction_Inject.h (members)
TArray<TSharedPtr<FComponentRequestHandle>> ComponentRequests;
TArray<TSharedPtr<FComponentRequestHandle>> ExtensionHandles; // one per game instance
```

```cpp
// MyGameFeatureAction_Inject.cpp
#include "MyGameFeatureAction_Inject.h"
#include "Components/GameFrameworkComponentManager.h"
#include "GameFeaturesSubsystem.h"   // FGameFeatureActivatingContext / FGameFeatureDeactivatingContext
#include "Engine/Engine.h"
#include "Engine/GameInstance.h"
#include "Engine/World.h"
#include "MyCharacter.h"
#include "MyHealthComponent.h"
#include "MyInventoryComponent.h"

void UMyGameFeatureAction_Inject::OnGameFeatureActivating(
    FGameFeatureActivatingContext& Context)
{
    for (const FWorldContext& WorldContext : GEngine->GetWorldContexts())
    {
        if (!Context.ShouldApplyToWorldContext(WorldContext))
        {
            continue;
        }

        UGameInstance* GameInstance = WorldContext.OwningGameInstance;
        UGameFrameworkComponentManager* CompMgr =
            GameInstance ? GameInstance->GetSubsystem<UGameFrameworkComponentManager>() : nullptr;
        if (!CompMgr)
        {
            continue;
        }

        const TSoftClassPtr<AActor> ReceiverClass(AMyCharacter::StaticClass());

        ComponentRequests.Add(CompMgr->AddComponentRequest(ReceiverClass,
            UMyHealthComponent::StaticClass(),
            EGameFrameworkAddComponentFlags::AddUnique));

        ComponentRequests.Add(CompMgr->AddComponentRequest(ReceiverClass,
            UMyInventoryComponent::StaticClass(),
            EGameFrameworkAddComponentFlags::AddUnique));

        ExtensionHandles.Add(CompMgr->AddExtensionHandler(ReceiverClass,
            UGameFrameworkComponentManager::FExtensionHandlerDelegate::CreateUObject(
                this, &UMyGameFeatureAction_Inject::HandleActorExtension)));
    }
}

void UMyGameFeatureAction_Inject::HandleActorExtension(AActor* Actor, FName EventName)
{
    if (EventName == UGameFrameworkComponentManager::NAME_ReceiverAdded)
    {
        // Actor just registered — set up initial state
    }
    else if (EventName == UGameFrameworkComponentManager::NAME_GameActorReady)
    {
        // Default initialization finished — every injected component exists
    }
    else if (EventName == UGameFrameworkComponentManager::NAME_ReceiverRemoved)
    {
        // Actor unregistering — drop references to it
    }
}

void UMyGameFeatureAction_Inject::OnGameFeatureDeactivating(
    FGameFeatureDeactivatingContext& Context)
{
    // RAII: dropping the handles removes the requests and the injected components
    ComponentRequests.Empty();
    ExtensionHandles.Empty();
}
```

The five event names are `NAME_ReceiverAdded`, `NAME_ReceiverRemoved`, `NAME_ExtensionAdded`,
`NAME_ExtensionRemoved` and `NAME_GameActorReady`. Send custom ones with
`CompMgr->SendExtensionEvent(Receiver, EventName, bOnlyInGameWorlds)`.

---

## Init-state component

Full `IGameFrameworkInitStateInterface` implementation on a `UPawnComponent`. The tag variables
come from the project's native gameplay tags; see `ue-gameplay-tags-messaging`.

```cpp
// MyModularComponent.h
#pragma once

#include "Components/GameFrameworkInitStateInterface.h"
#include "Components/PawnComponent.h"
#include "GameplayTagContainer.h"
#include "MyModularComponent.generated.h"

class UGameFrameworkComponentManager;
struct FActorInitStateChangedParams;

UCLASS()
class MYGAME_API UMyModularComponent : public UPawnComponent,
    public IGameFrameworkInitStateInterface
{
    GENERATED_BODY()

public:
    virtual FName GetFeatureName() const override { return FName(TEXT("MyFeature")); }
    virtual bool CanChangeInitState(UGameFrameworkComponentManager* Manager, FGameplayTag CurrentState, FGameplayTag DesiredState) const override;
    virtual void HandleChangeInitState(UGameFrameworkComponentManager* Manager, FGameplayTag CurrentState, FGameplayTag DesiredState) override;
    virtual void CheckDefaultInitialization() override;
    virtual void OnActorInitStateChanged(const FActorInitStateChangedParams& Params) override;

    virtual void OnRegister() override;
    virtual void BeginPlay() override;
    virtual void EndPlay(const EEndPlayReason::Type EndPlayReason) override;
};
```

```cpp
// MyModularComponent.cpp
#include "MyModularComponent.h"
#include "Components/GameFrameworkComponentManager.h"
#include "GameFramework/Pawn.h"
#include "MyGameplayTags.h"

void UMyModularComponent::OnRegister()
{
    Super::OnRegister();
    RegisterInitStateFeature();
}

void UMyModularComponent::BeginPlay()
{
    Super::BeginPlay();

    // Listen to every feature on this actor so dependencies can unblock us
    BindOnActorInitStateChanged(NAME_None, FGameplayTag(), /*bCallIfReached=*/ false);
    CheckDefaultInitialization();
}

void UMyModularComponent::EndPlay(const EEndPlayReason::Type EndPlayReason)
{
    UnregisterInitStateFeature();
    Super::EndPlay(EndPlayReason);
}

bool UMyModularComponent::CanChangeInitState(UGameFrameworkComponentManager* Manager,
    FGameplayTag CurrentState, FGameplayTag DesiredState) const
{
    APawn* MyPawn = GetPawn<APawn>();

    if (!CurrentState.IsValid() && DesiredState == TAG_InitState_Spawned)
    {
        return MyPawn != nullptr;
    }
    if (CurrentState == TAG_InitState_Spawned && DesiredState == TAG_InitState_DataAvailable)
    {
        return MyPawn != nullptr && MyPawn->GetController() != nullptr;
    }
    if (CurrentState == TAG_InitState_DataAvailable && DesiredState == TAG_InitState_DataInitialized)
    {
        return Manager->HasFeatureReachedInitState(GetOwner(), FName(TEXT("MyOtherFeature")),
            TAG_InitState_DataAvailable);
    }
    if (CurrentState == TAG_InitState_DataInitialized && DesiredState == TAG_InitState_GameplayReady)
    {
        return true;
    }
    return false;
}

void UMyModularComponent::HandleChangeInitState(UGameFrameworkComponentManager* Manager,
    FGameplayTag CurrentState, FGameplayTag DesiredState)
{
    if (DesiredState == TAG_InitState_DataInitialized)
    {
        // Safe point to read other features' data on this actor
    }
}

void UMyModularComponent::CheckDefaultInitialization()
{
    // Give every other feature on this actor a chance to move first
    CheckDefaultInitializationForImplementers();

    static const TArray<FGameplayTag> StateChain = {
        TAG_InitState_Spawned,
        TAG_InitState_DataAvailable,
        TAG_InitState_DataInitialized,
        TAG_InitState_GameplayReady };

    ContinueInitStateChain(StateChain);
}

void UMyModularComponent::OnActorInitStateChanged(const FActorInitStateChangedParams& Params)
{
    if (Params.FeatureName != GetFeatureName())
    {
        CheckDefaultInitialization();
    }
}
```

Register the states themselves once, in registration order, before any actor spawns:

```cpp
void UMyInitStateSubsystem::Initialize(FSubsystemCollectionBase& Collection)
{
    Super::Initialize(Collection);

    if (UGameFrameworkComponentManager* CompMgr =
        GetGameInstance()->GetSubsystem<UGameFrameworkComponentManager>())
    {
        CompMgr->RegisterInitState(TAG_InitState_Spawned, false, FGameplayTag());
        CompMgr->RegisterInitState(TAG_InitState_DataAvailable, false, TAG_InitState_Spawned);
        CompMgr->RegisterInitState(TAG_InitState_DataInitialized, false, TAG_InitState_DataAvailable);
        CompMgr->RegisterInitState(TAG_InitState_GameplayReady, false, TAG_InitState_DataInitialized);
    }
}
```

To wait on a feature from outside the interface, bind on the manager:

```cpp
FDelegateHandle InitStateHandle = CompMgr->RegisterAndCallForActorInitState(
    MyActor, FName(TEXT("MyOtherFeature")), TAG_InitState_DataInitialized,
    FActorInitStateChangedDelegate::CreateUObject(this, &UMyListener::HandleInitStateChanged),
    /*bCallImmediately=*/ true);

// void UMyListener::HandleInitStateChanged(const FActorInitStateChangedParams& Params)
// Params carries OwningActor, FeatureName, Implementer and FeatureState.

CompMgr->UnregisterActorInitStateDelegate(MyActor, InitStateHandle);
```

---

## Action with async deactivation

`PauseDeactivationUntilComplete(FString InPauserTag)` returns an `FSimpleDelegate` that must be
executed on the game thread, on every path, or the plugin never leaves `Deactivating`.

```cpp
// MyGameFeatureAction_SaveProgress.h
#pragma once

#include "GameFeatureAction.h"
#include "MyGameFeatureAction_SaveProgress.generated.h"

class USaveGame;
struct FGameFeatureDeactivatingContext;

UCLASS(meta = (DisplayName = "My Save Progress On Deactivate"))
class MYGAME_API UMyGameFeatureAction_SaveProgress : public UGameFeatureAction
{
    GENERATED_BODY()

public:
    virtual void OnGameFeatureDeactivating(FGameFeatureDeactivatingContext& Context) override;

private:
    UPROPERTY()
    TObjectPtr<USaveGame> SaveGameObject;
};
```

```cpp
// MyGameFeatureAction_SaveProgress.cpp
#include "MyGameFeatureAction_SaveProgress.h"
#include "GameFeaturesSubsystem.h"
#include "GameFramework/SaveGame.h"
#include "Kismet/GameplayStatics.h"

void UMyGameFeatureAction_SaveProgress::OnGameFeatureDeactivating(
    FGameFeatureDeactivatingContext& Context)
{
    if (!SaveGameObject)
    {
        return;
    }

    FSimpleDelegate Resume = Context.PauseDeactivationUntilComplete(TEXT("MySaveProgress"));

    UGameplayStatics::AsyncSaveGameToSlot(SaveGameObject, TEXT("MyAutoSave"), 0,
        FAsyncSaveGameToSlotDelegate::CreateLambda(
            [Resume](const FString& SlotName, const int32 UserIndex, bool bSuccess)
            {
                UE_LOG(LogMyGame, Log, TEXT("Save to %s %s"), *SlotName,
                    bSuccess ? TEXT("succeeded") : TEXT("failed"));
                Resume.ExecuteIfBound();
            }));
}
```

`FAsyncSaveGameToSlotDelegate` is `DECLARE_DELEGATE_ThreeParams(..., const FString&, const int32,
bool)`. For save-game details see `ue-serialization-savegames`.

---

## Project policies subclass

```cpp
// MyGameFeaturesProjectPolicies.h
#pragma once

#include "GameFeaturesProjectPolicies.h"
#include "MyGameFeaturesProjectPolicies.generated.h"

UCLASS()
class MYGAME_API UMyGameFeaturesProjectPolicies : public UDefaultGameFeaturesProjectPolicies
{
    GENERATED_BODY()

public:
    virtual bool IsPluginAllowed(const FString& PluginURL, FString* OutReason) const override;
    virtual void GetGameFeatureLoadingMode(bool& bLoadClientData, bool& bLoadServerData) const override;
};
```

```cpp
// MyGameFeaturesProjectPolicies.cpp
#include "MyGameFeaturesProjectPolicies.h"
#include "Misc/CoreMisc.h"

bool UMyGameFeaturesProjectPolicies::IsPluginAllowed(const FString& PluginURL,
    FString* OutReason) const
{
    if (PluginURL.Contains(TEXT("MyDebugTools")) && UE_BUILD_SHIPPING)
    {
        if (OutReason)
        {
            *OutReason = TEXT("Debug tools are disabled in shipping builds");
        }
        return false;
    }

    return Super::IsPluginAllowed(PluginURL, OutReason);
}

void UMyGameFeaturesProjectPolicies::GetGameFeatureLoadingMode(bool& bLoadClientData,
    bool& bLoadServerData) const
{
    bLoadClientData = !IsRunningDedicatedServer();
    bLoadServerData = true;
}
```

Select the class in Project Settings → Game Features, which writes:

```ini
[/Script/GameFeatures.GameFeaturesSubsystemSettings]
GameFeaturesManagerClassName=/Script/MyGame.MyGameFeaturesProjectPolicies
```

The value is an `FSoftClassPath`, so it uses the class name without the `U` prefix.
`UDefaultGameFeaturesProjectPolicies` is the engine's fallback implementation; derive from it
unless the project replaces the startup behaviour entirely, in which case derive from
`UGameFeaturesProjectPolicies` and implement `InitGameFeatureManager()` yourself.

Other virtuals worth overriding: `ShouldReadPluginDetails(const FString& PluginDescriptorFilename)`
to skip descriptors quickly, `ShouldAllowMissingGameFeatureData(const FString& PluginName)` for
plugins without a `UGameFeatureData`, `GetPreloadBundleStateForPlugin(const FString& PluginName)`
to choose asset bundles, and `IsLoadingStartupPlugins()`.
