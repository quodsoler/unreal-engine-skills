---
name: ue-game-features
description: "Use when building, activating or debugging Game Feature plugins and modular gameplay. Also use when the user mentions 'GameFeatureAction', 'UGameFeatureData', 'GameFeaturesSubsystem', 'LoadAndActivateGameFeaturePlugin', 'BuiltInInitialFeatureState', 'ExplicitlyLoaded', '.uplugin', 'GameFrameworkComponentManager', 'AddReceiver', 'AddComponentRequest', 'AddExtensionHandler', 'NAME_GameActorReady', 'init state', 'UPawnComponent', 'ModularGameplay', 'experience system' or 'Lyra-style modular architecture'. For plugin modules and Build.cs wiring, see ue-module-build-system; for component fundamentals, see ue-actor-component-architecture."
metadata:
  version: "2.0.0"
  engine: "5.8"
---

# UE Game Features and Modular Gameplay

Target engine: **UE 5.8**. APIs below are verified against the 5.8 headers; older forms are listed under "Deprecated — do not use".

Game Features (Beta in 5.8) packages gameplay as self-contained plugins under `Plugins/GameFeatures/`. `UGameFeaturesSubsystem` (a `UEngineSubsystem`) drives each plugin through a state machine; a `UGameFeatureData` primary data asset inside the plugin holds an instanced list of `UGameFeatureAction` objects that run at registration, load, activation and deactivation. Modular Gameplay (Beta in 5.8) supplies `UGameFrameworkComponentManager` (a `UGameInstanceSubsystem`) that injects components into opted-in actors, plus the `UGameFrameworkComponent` bases and the init-state system that orders initialization across independently loaded features. Build.cs modules: `GameFeatures` and `ModularGameplay`. The `GameFeatures` module already depends publicly on `ModularGameplay` and `DataRegistry`, so code that only consumes actions needs `GameFeatures` alone.

## Context

Read `.agents/ue-project-context.md` if it exists (module names, conventions, enabled plugins, GAS/networking setup). Do not stop if it is missing.

Identify the area from the request and the codebase. Ask only when two plausible readings would produce different code.

| Request is about… | Go to |
|---|---|
| Creating a feature plugin, `.uplugin` keys, folder layout, `UGameFeatureData` | [Plugin Layout and Descriptor](#plugin-layout-and-descriptor) |
| Activating or unloading a feature at runtime, plugin URLs, plugin states | [Plugin States and the Subsystem](#plugin-states-and-the-subsystem) |
| Writing an action that runs when a feature turns on or off | [Writing a UGameFeatureAction](#writing-a-ugamefeatureaction) |
| Picking a ready-made action instead of writing one | [Engine-Provided Actions](#engine-provided-actions) |
| Adding components to actors from a feature, extension events | [Component Injection](#component-injection) |
| Ordering initialization between components that load separately | [Init State System](#init-state-system) |
| Choosing a base class for a Pawn/Controller/PlayerState/GameState component | [Modular Component Bases](#modular-component-bases) |
| Filtering which plugins load, client/server data, state observers | [Project Policies and Observers](#project-policies-and-observers) |
| Lyra-style experience composition (sample code, not engine API) | [references/experience-system.md](references/experience-system.md) |
| Full templates: custom action, receiver setup, policies subclass | [references/game-feature-patterns.md](references/game-feature-patterns.md) |

## Plugin Layout and Descriptor

A Game Feature plugin is an ordinary plugin that lives under `<Project>/Plugins/GameFeatures/` or under `<AdditionalPluginDirectory>/GameFeatures/`. **That location is what makes it a Game Feature plugin** — `UGameFeaturesSubsystemSettings::IsValidGameFeaturePlugin` matches the descriptor path against those folders. There is no plugin-level `"Type"` key in the `.uplugin` schema.

```
Plugins/GameFeatures/
├── MyFeature/
│   ├── MyFeature.uplugin
│   ├── Content/MyFeature.uasset        (UGameFeatureData, named after the plugin)
│   └── Source/MyFeatureRuntime/
└── MyFeatureRules/
    ├── MyFeatureRules.uplugin
    └── Content/MyFeatureRules.uasset
```

```json
{
	"FileVersion": 3,
	"Version": 1,
	"VersionName": "1.0",
	"FriendlyName": "My Feature",
	"Description": "Adds my gameplay feature",
	"Category": "Game Features",
	"CreatedBy": "",
	"CanContainContent": true,
	"ExplicitlyLoaded": true,
	"BuiltInInitialFeatureState": "Active",
	"Modules": [
		{ "Name": "MyFeatureRuntime", "Type": "Runtime", "LoadingPhase": "Default" }
	],
	"Plugins": [
		{ "Name": "GameFeatures", "Enabled": true },
		{ "Name": "ModularGameplay", "Enabled": true }
	]
}
```

| Key | Meaning |
|---|---|
| `ExplicitlyLoaded` | The plugin manager does not mount the plugin at startup; the feature's state machine mounts and unmounts it. Always `true` for a feature plugin. |
| `CanContainContent` | Plugin ships a `Content/` directory (needed for the `UGameFeatureData` asset). |
| `BuiltInInitialFeatureState` | How far a built-in plugin advances at startup: `"Installed"`, `"Registered"`, `"Loaded"` or `"Active"`. Unknown values log an error and fall back to `Active`. |
| `EnabledByDefault` | Plugin is enabled without a `.uproject` entry. |
| `Modules[].Type` | `ModuleHostType` name: `Runtime`, `RuntimeNoCommandlet`, `Editor`, `Developer`, `Program`, … |
| `Modules[].LoadingPhase` | `ModuleLoadingPhase` name: `Default`, `PreDefault`, `PostConfigInit`, `PostEngineInit`, … |

A content-only feature plugin omits `Modules` entirely. Adding a module means adding a `Build.cs`; see `ue-module-build-system`.

### UGameFeatureData

`UGameFeatureData : UPrimaryDataAsset` is the asset the subsystem loads for the plugin. Its two authored properties are protected; read them through accessors:

```cpp
const TArray<UGameFeatureAction*>& FeatureActions = FeatureData->GetActions();
const TArray<FPrimaryAssetTypeInfo>& ScanTypes = FeatureData->GetPrimaryAssetTypesToScan();
```

`Actions` is an `Instanced` array of `UGameFeatureAction` subclasses created inline in the asset. `PrimaryAssetTypesToScan` tells the Asset Manager which primary asset types inside the plugin to scan when the feature registers; see `ue-data-assets-tables`.

Subclass `UGameFeatureData` only to change `GetPrimaryAssetTypesToScan` or, in the editor, `GetDisallowedActions`.

## Plugin States and the Subsystem

Destination states are fully ordered; transition and error states sit between them. `EGameFeaturePluginState` is a plain enum generated from the `GAME_FEATURE_PLUGIN_STATE_LIST` macro.

| Destination state | Meaning |
|---|---|
| `Terminal` | Final state before the state machine is removed. |
| `UnknownStatus` | Only the URL is known. |
| `StatusKnown` | Availability confirmed (on disk or in a bundle). |
| `Uninstalled` | Local data for the plugin has been removed. |
| `Installed` | Files are in local storage, not registered. |
| `Registered` | Assets discovered and registered; actions received `OnGameFeatureRegistering`. |
| `Loaded` | Plugin code and content are in memory, not yet affecting the game. |
| `Active` | Actions applied; the feature affects the game. |

Transition states that show up in logs and in `GetPluginState` results include `CheckingStatus`, `Downloading`, `Mounting`, `Registering`, `Loading`, `ActivatingDependencies`, `Activating`, `Deactivating`, `Unloading`, `Unregistering`, `Unmounting`, `Releasing`, `Uninstalling`, plus the matching `Error*` states.

Two other enums: `EGameFeatureTargetState` (`Installed`, `Registered`, `Loaded`, `Active`) is what you pass to `ChangeGameFeatureTargetState`; `EBuiltInAutoState` (`Invalid`, `Installed`, `Registered`, `Loaded`, `Active`) is what `BuiltInInitialFeatureState` parses into.

### Driving a plugin from code

```cpp
#include "GameFeaturesSubsystem.h"

void UMyFeatureManager::EnableMyFeature()
{
    UGameFeaturesSubsystem& GFS = UGameFeaturesSubsystem::Get();

    FString PluginURL;
    if (!GFS.GetPluginURLByName(TEXT("MyFeature"), PluginURL))
    {
        UE_LOG(LogMyGame, Error, TEXT("MyFeature is not a discovered game feature plugin"));
        return;
    }

    GFS.LoadAndActivateGameFeaturePlugin(PluginURL,
        FGameFeaturePluginLoadComplete::CreateUObject(this, &UMyFeatureManager::HandleFeatureLoaded));
}

void UMyFeatureManager::HandleFeatureLoaded(const UE::GameFeatures::FResult& Result)
{
    if (Result.HasError())
    {
        UE_LOG(LogMyGame, Error, TEXT("MyFeature failed: %s"), *Result.GetError());
    }
}
```

`GetPluginURLByName(FStringView PluginName, FString& OutPluginURL)` is the safe way to build a URL for a built-in plugin. The static builders `UGameFeaturesSubsystem::GetPluginURL_FileProtocol(PluginDescriptorPath)` and `GetPluginURL_InstallBundleProtocol(PluginName, BundleName)` produce `file:` and `installbundle:` URLs (`EGameFeaturePluginProtocol::File`, `::InstallBundle`).

| Call | Effect |
|---|---|
| `LoadGameFeaturePlugin(URL, CompleteDelegate)` | Advance to `Loaded`. |
| `LoadAndActivateGameFeaturePlugin(URL, CompleteDelegate)` | Advance to `Active`. |
| `RegisterGameFeaturePlugin(URL, CompleteDelegate)` | Advance to `Registered`. |
| `ChangeGameFeatureTargetState(URL, EGameFeatureTargetState::Registered, CompleteDelegate)` | Move to any destination state. |
| `DeactivateGameFeaturePlugin(URL)` | Back down to `Loaded`. |
| `UnloadGameFeaturePlugin(URL, /*bKeepRegistered=*/ false)` | Back down to `Registered` or `Installed`. |
| `TerminateGameFeaturePlugin(URL)` | Tear the state machine down. |
| `CancelGameFeatureStateChange(URL)` | Abort an in-flight transition. |

Every completion delegate above is an alias of `FGameFeaturePluginChangeStateComplete`, i.e. `DECLARE_DELEGATE_OneParam(..., const UE::GameFeatures::FResult&)`. Overloads taking `TConstArrayView<FString> PluginURLs` report through `FMultipleGameFeaturePluginsLoaded`, a `TDelegate<void(const TMap<FString, UE::GameFeatures::FResult>&)>`.

Queries: `GetPluginState(URL)`, `IsGameFeaturePluginActive(URL, bCheckForActivating)`, `IsGameFeaturePluginActiveByName(PluginName, bCheckForActivating)`, `IsGameFeaturePluginRegistered(URL, bCheckForRegistering)`, `IsGameFeaturePluginLoaded(URL)`, `IsGameFeaturePluginInErrorState(URL)`, `GetActivePluginNames()`.

Console commands for testing (`GameFeaturesSubsystem.cpp:583-661`, take a plugin name or URL): `ListGameFeaturePlugins [-activeonly] [-csv]`, `LoadGameFeaturePlugin`, `DeactivateGameFeaturePlugin`, `UnloadGameFeaturePlugin`, `ReleaseGameFeaturePlugin`, `CancelGameFeaturePlugin`, `TerminateGameFeaturePlugin`. The `LoadGameFeaturePlugin` console command calls `LoadAndActivateGameFeaturePlugin` and is `ECVF_Cheat`, unlike the C++ function of the same name, which only reaches `Loaded`.

## Writing a UGameFeatureAction

`UGameFeatureAction` is `UCLASS(MinimalAPI, DefaultToInstanced, EditInlineNew, Abstract)` deriving from `UObject`. `DefaultToInstanced` and `EditInlineNew` are what let instances be created inline in the `Actions` array. The overridable virtuals, verbatim:

```cpp
virtual void OnGameFeatureRegistering() {}
virtual void OnGameFeatureUnregistering() {}
virtual void OnGameFeatureLoading() {}
virtual void OnGameFeatureUnloading() {}
virtual void OnGameFeatureActivating(FGameFeatureActivatingContext& Context);
virtual void OnGameFeatureActivating() {}
virtual void OnGameFeatureActivated() {}
virtual void OnGameFeatureDeactivating(FGameFeatureDeactivatingContext& Context) {}
```

Override the context version of `OnGameFeatureActivating`; the base implementation of that one calls the no-arg version, so overriding both runs your code twice. `IsDataValid` is a `UObject` override, not a `UGameFeatureAction` member, and is editor-only.

```cpp
// MyGameFeatureAction_GrantTag.h
#pragma once

#include "GameFeatureAction.h"
#include "GameplayTagContainer.h"
#include "MyGameFeatureAction_GrantTag.generated.h"

struct FGameFeatureActivatingContext;
struct FGameFeatureDeactivatingContext;
struct FWorldContext;

UCLASS(meta = (DisplayName = "My Grant Tag"))
class MYGAME_API UMyGameFeatureAction_GrantTag : public UGameFeatureAction
{
    GENERATED_BODY()

public:
    virtual void OnGameFeatureActivating(FGameFeatureActivatingContext& Context) override;
    virtual void OnGameFeatureDeactivating(FGameFeatureDeactivatingContext& Context) override;

#if WITH_EDITOR
    virtual EDataValidationResult IsDataValid(class FDataValidationContext& Context) const override;
#endif

    UPROPERTY(EditAnywhere, Category = "Tags")
    FGameplayTag TagToGrant;

private:
    void ApplyToWorld(const FWorldContext& WorldContext);
};
```

An action instance is shared by every world the feature applies to, so never keep per-world state in a bare member. Filter worlds with the context and key state by `FObjectKey`:

```cpp
void UMyGameFeatureAction_GrantTag::OnGameFeatureActivating(FGameFeatureActivatingContext& Context)
{
    for (const FWorldContext& WorldContext : GEngine->GetWorldContexts())
    {
        if (Context.ShouldApplyToWorldContext(WorldContext))
        {
            ApplyToWorld(WorldContext);
        }
    }
}
```

`FGameFeatureActivatingContext` and `FGameFeatureDeactivatingContext` both derive from `FGameFeatureStateChangeContext`, which supplies `ShouldApplyToWorldContext(const FWorldContext&)`, `ShouldApplyUsingOtherContext(const FGameFeatureStateChangeContext&)` and `SetRequiredWorldContextHandle(FName)`.

Deactivation that needs async work must hold the state machine open. `FGameFeatureDeactivatingContext::PauseDeactivationUntilComplete(FString InPauserTag)` returns an `FSimpleDelegate` you must execute on the game thread, on every path:

```cpp
void UMyGameFeatureAction_GrantTag::OnGameFeatureDeactivating(FGameFeatureDeactivatingContext& Context)
{
    FSimpleDelegate Resume = Context.PauseDeactivationUntilComplete(TEXT("MyGrantTag_Cleanup"));
    StartAsyncCleanup(Resume);   // must call Resume.ExecuteIfBound() when finished
}
```

Full templates, including world tracking and the RAII handle patterns, are in [references/game-feature-patterns.md](references/game-feature-patterns.md).

## Engine-Provided Actions

| Action | Purpose |
|---|---|
| `UGameFeatureAction_AddComponents` | Adds actor↔component spawn requests to `UGameFrameworkComponentManager`. |
| `UGameFeatureAction_AddCheats` | Adds cheat manager extensions for each player. |
| `UGameFeatureAction_AddActorFactory` | Registers an actor factory when the plugin registers. |
| `UGameFeatureAction_AddChunkOverride` | Cooks the plugin's assets into a specified chunk id. |
| `UGameFeatureAction_AddWPContent` | Adds World Partition content as a content bundle. |
| `UGameFeatureAction_AddWorldPartitionContent` | Adds World Partition content for the plugin. |
| `UGameFeatureAction_DataRegistry` | Loads and initializes a list of Data Registries. |
| `UGameFeatureAction_DataRegistrySource` | Adds source assets to existing Data Registries. |
| `UGameFeatureAction_AudioActionBase` | `Abstract` base for actions that affect the audio engine. |
| `UIrisFilterGameFeatureAction` | Creates an Iris network filter. |

`UGameFeatureAction_AddComponents` is the one most features need. It holds `TArray<FGameFeatureComponentEntry> ComponentList`, and each `FGameFeatureComponentEntry` carries `TSoftClassPtr<AActor> ActorClass` (`meta=(AllowAbstract="True")`), `TSoftClassPtr<UActorComponent> ComponentClass`, the bitfields `uint8 bClientComponent:1` and `uint8 bServerComponent:1`, and `uint8 AdditionFlags` (`meta=(Bitmask, BitmaskEnum="/Script/ModularGameplay.EGameFrameworkAddComponentFlags")`).

Set both `bClientComponent` and `bServerComponent` for components needed everywhere, server-only for authoritative gameplay logic, client-only for cosmetics. The action keeps a `TSharedPtr<FComponentRequestHandle>` per game instance and drops them on deactivation, which removes the injected components.

## Component Injection

`UGameFrameworkComponentManager` is a **`UGameInstanceSubsystem`**. Get it from an actor:

```cpp
#include "Components/GameFrameworkComponentManager.h"

UGameFrameworkComponentManager* CompMgr = UGameFrameworkComponentManager::GetForActor(MyActor);
```

`GetForActor(const AActor* Actor, bool bOnlyGameWorlds = true)` returns null outside game worlds. `GetGameInstance()->GetSubsystem<UGameFrameworkComponentManager>()` also works where a game instance is in hand.

### Receivers

An actor receives injected components only after it registers itself:

```cpp
void AMyCharacter::BeginPlay()
{
    Super::BeginPlay();
    UGameFrameworkComponentManager::AddGameFrameworkComponentReceiver(this);
}

void AMyCharacter::EndPlay(const EEndPlayReason::Type EndPlayReason)
{
    UGameFrameworkComponentManager::RemoveGameFrameworkComponentReceiver(this);
    Super::EndPlay(EndPlayReason);
}
```

The static helpers resolve the manager for the actor and forward to `AddReceiver(AActor* Receiver, bool bAddOnlyInGameWorlds = true)` and `RemoveReceiver(AActor* Receiver)`, which you can also call directly on the manager.

### Requests and extension handlers

```cpp
TSharedPtr<FComponentRequestHandle> RequestHandle = CompMgr->AddComponentRequest(
    TSoftClassPtr<AActor>(AMyCharacter::StaticClass()),
    UMyHealthComponent::StaticClass(),
    EGameFrameworkAddComponentFlags::AddUnique);
```

`FComponentRequestHandle` is RAII: keep it alive for as long as the injection should last, and destroy it to remove the request and the components it created. It also exposes `IsValid()`, which returns false once the owning manager is gone.

| `EGameFrameworkAddComponentFlags` | Value | Behavior |
|---|---|---|
| `None` | 0 | Always create the component on the receiver. |
| `AddUnique` | `0b00000001` | Skip if the receiver already has a component of that class. |
| `AddIfNotChild` | `0b00000010` | Skip if the requested class is a child of an existing component's class. |
| `UseAutoGeneratedName` | `0b00000100` | Generate a new name instead of reusing the class name. |

Extension handlers observe receivers instead of adding components. The delegate is declared inside the manager as `DECLARE_DELEGATE_TwoParams(FExtensionHandlerDelegate, AActor*, FName)`:

```cpp
ExtensionHandle = CompMgr->AddExtensionHandler(
    TSoftClassPtr<AActor>(AMyCharacter::StaticClass()),
    UGameFrameworkComponentManager::FExtensionHandlerDelegate::CreateUObject(
        this, &UMyFeatureManager::HandleActorExtension));
```

```cpp
void UMyFeatureManager::HandleActorExtension(AActor* Actor, FName EventName)
{
    if (EventName == UGameFrameworkComponentManager::NAME_GameActorReady)
    {
        // Actor has finished its default initialization
    }
}
```

The five `static FName` event constants are `NAME_ReceiverAdded`, `NAME_ReceiverRemoved`, `NAME_ExtensionAdded`, `NAME_ExtensionRemoved` and `NAME_GameActorReady`. Send your own with `CompMgr->SendExtensionEvent(Receiver, EventName, bOnlyInGameWorlds)` or the static `UGameFrameworkComponentManager::SendGameFrameworkComponentExtensionEvent(Receiver, EventName, bOnlyInGameWorlds)`.

## Init State System

Init states solve ordered initialization when components arrive from different plugins. States are `FGameplayTag` values registered in order on the manager; for how to declare the tags themselves see `ue-gameplay-tags-messaging`.

```cpp
CompMgr->RegisterInitState(TAG_InitState_Spawned, false, FGameplayTag());
CompMgr->RegisterInitState(TAG_InitState_DataAvailable, false, TAG_InitState_Spawned);
CompMgr->RegisterInitState(TAG_InitState_DataInitialized, false, TAG_InitState_DataAvailable);
CompMgr->RegisterInitState(TAG_InitState_GameplayReady, false, TAG_InitState_DataInitialized);
```

`RegisterInitState(FGameplayTag NewState, bool bAddBefore, FGameplayTag ExistingState)` inserts relative to an existing state, so the engine ships no fixed state list — the project owns it. Register the states once, for example from a `UGameInstanceSubsystem` or from the project's policies class.

| Manager call | Use |
|---|---|
| `ChangeFeatureInitState(Actor, FeatureName, Implementer, FeatureState)` | Advance one feature; returns `bool`. |
| `GetInitStateForFeature(Actor, FeatureName)` | Current state tag. |
| `HasFeatureReachedInitState(Actor, FeatureName, FeatureState)` | Test one feature. |
| `HaveAllFeaturesReachedInitState(Actor, RequiredState, ExcludingFeature)` | Test the whole actor. |
| `IsInitStateAfterOrEqual(FeatureState, RelativeState)` | Compare two tags in registration order. |
| `GetImplementerForFeature(Actor, FeatureName, RequiredState)` | Find the object implementing a feature. |
| `RegisterAndCallForActorInitState(Actor, FeatureName, RequiredState, Delegate, bCallImmediately)` | Wait on one actor; returns `FDelegateHandle`. |
| `RegisterAndCallForClassInitState(ActorClass, FeatureName, RequiredState, Delegate, bCallImmediately)` | Wait on every actor of a class; returns `FDelegateHandle`. |
| `UnregisterActorInitStateDelegate(Actor, Handle)` / `UnregisterClassInitStateDelegate(ActorClass, Handle)` | Remove a native binding (takes `FDelegateHandle&`). |

The native delegate is `DECLARE_DELEGATE_OneParam(FActorInitStateChangedDelegate, const FActorInitStateChangedParams&)`. `FActorInitStateChangedParams` is a `USTRUCT` with `OwningActor`, `FeatureName`, `Implementer` and `FeatureState`.

### IGameFrameworkInitStateInterface

Implement `IGameFrameworkInitStateInterface` (UINTERFACE `UGameFrameworkInitStateInterface`, `NotBlueprintable`) on a component to get the standard progression. The virtuals worth overriding, verbatim from the header:

```cpp
virtual FName GetFeatureName() const { return NAME_None; }
virtual bool CanChangeInitState(UGameFrameworkComponentManager* Manager, FGameplayTag CurrentState, FGameplayTag DesiredState) const { return true; }
virtual void HandleChangeInitState(UGameFrameworkComponentManager* Manager, FGameplayTag CurrentState, FGameplayTag DesiredState) {}
virtual void CheckDefaultInitialization() {}
virtual void OnActorInitStateChanged(const FActorInitStateChangedParams& Params) {}
```

The rest you call rather than override:

| Call | Effect |
|---|---|
| `RegisterInitStateFeature()` | Registers this implementer with the manager (call from `OnRegister`). |
| `UnregisterInitStateFeature()` | Removes the registration and the bound delegate (call from `EndPlay`). |
| `BindOnActorInitStateChanged(FeatureName, RequiredState, bCallIfReached)` | Binds `OnActorInitStateChanged`, storing the handle in `ActorInitStateChangedHandle`. |
| `TryToChangeInitState(DesiredState)` | Runs `CanChangeInitState`, then `HandleChangeInitState`, then notifies the manager. |
| `ContinueInitStateChain(const TArray<FGameplayTag>& InitStateChain)` | Walks the chain from the current state, running `CanChangeInitState` and `HandleChangeInitState` for each step and notifying the manager; returns the furthest state reached. |
| `CheckDefaultInitializationForImplementers()` | Calls `CheckDefaultInitialization` on every other feature of the same actor. |
| `HasReachedInitState(DesiredState)` / `GetInitState()` | Query this feature's state through the manager. |

A `CheckDefaultInitialization` override normally calls `CheckDefaultInitializationForImplementers()` and then `ContinueInitStateChain` with the project's ordered tag list; a full component implementation is in [references/game-feature-patterns.md](references/game-feature-patterns.md).

## Modular Component Bases

| Base class | Owner it expects | Typed accessors |
|---|---|---|
| `UGameFrameworkComponent` | any `AActor` | `GetGameInstance<T>()`, `GetGameInstanceChecked<T>()`, `HasAuthority()` |
| `UPawnComponent` | `APawn` | `GetPawn<T>()`, `GetPawnChecked<T>()`, `GetPlayerState<T>()`, `GetController<T>()` |
| `UControllerComponent` | `AController` | `GetController<T>()`, `GetControllerChecked<T>()`, `GetPawn<T>()`, `GetViewTarget<T>()`, `GetPawnOrViewTarget<T>()`, `GetPlayerState<T>()`, `GetPlayer<T>()` |
| `UPlayerStateComponent` | `APlayerState` | `GetPlayerState<T>()`, `GetPlayerStateChecked<T>()` |
| `UGameStateComponent` | `AGameStateBase` | `GetGameState<T>()`, `GetGameStateChecked<T>()`, `GetGameMode<T>()` |

All five live in `ModularGameplay` under `#include "Components/<Name>.h"`. `UGameFrameworkComponent` derives from `UActorComponent` and is `Blueprintable, BlueprintType`. Prefer these over raw `UActorComponent` for injected components: the accessors are `static_assert`-checked against the owner type and the init-state interface is designed around them.

## Project Policies and Observers

`UGameFeaturesProjectPolicies` decides which plugins may load and what data each build type loads. Select the subclass in project settings, which writes:

```ini
[/Script/GameFeatures.GameFeaturesSubsystemSettings]
GameFeaturesManagerClassName=/Script/MyGame.MyGameFeaturesProjectPolicies
```

Useful virtuals: `InitGameFeatureManager()`, `ShutdownGameFeatureManager()`, `IsPluginAllowed(const FString& PluginURL, FString* OutReason) const`, `GetGameFeatureLoadingMode(bool& bLoadClientData, bool& bLoadServerData) const`, `ShouldReadPluginDetails(const FString& PluginDescriptorFilename) const`, `GetPreloadBundleStateForPlugin(const FString& PluginName) const`, `IsLoadingStartupPlugins() const`. `UDefaultGameFeaturesProjectPolicies` is the fallback implementation. A subclass template is in [references/game-feature-patterns.md](references/game-feature-patterns.md).

`UGameFeaturesSubsystemSettings::LoadStateClient` and `::LoadStateServer` are the `FName` bundle states used for client/server asset filtering; the settings class also carries `EnabledPlugins`, `DisabledPlugins` and `AdditionalPluginMetadataKeys`.

For cross-cutting reactions that are not tied to one feature's data asset, implement `IGameFeatureStateChangeObserver` and register it:

```cpp
UGameFeaturesSubsystem::Get().AddObserver(MyObserver,
    UGameFeaturesSubsystem::EObserverPluginStateUpdateMode::CurrentAndFuture);
```

`EObserverPluginStateUpdateMode` is `FutureOnly` or `CurrentAndFuture`; the latter replays current plugin states at add time and is expensive when a project has many plugins. Remove with `RemoveObserver(MyObserver)`. Observer hooks include `OnGameFeatureRegistering(const UGameFeatureData*, const FString& PluginName, const FString& PluginURL)`, `OnGameFeatureLoading(const UGameFeatureData*, const FString& PluginURL)`, `OnGameFeatureActivating(const UGameFeatureData*, const FString& PluginURL)`, `OnGameFeatureDeactivating(const UGameFeatureData*, FGameFeatureDeactivatingContext& Context, const FString& PluginURL)` and the download and mounting hooks. Prefer a `UGameFeatureAction` on the feature's data asset whenever data is involved.

## Deprecated — do not use

| Do not emit | Use in 5.8 | Source |
|---|---|---|
| `"Type": "GameFeature"` in a `.uplugin` | Place the plugin under `Plugins/GameFeatures/` and set `"ExplicitlyLoaded": true` | No plugin-level `Type` key in `PluginDescriptor.cs` |
| `"BuiltInAutoRegister"` / `"BuiltInAutoLoad"` / `"BuiltInAutoActivate"` | `"BuiltInInitialFeatureState"` | Legacy fallback in `GameFeaturesSubsystem.cpp:5079-5090` |
| `AddObserver(Observer)` | `AddObserver(Observer, EObserverPluginStateUpdateMode::CurrentAndFuture)` | `UE_DEPRECATED(5.7)` in `GameFeaturesSubsystem.h:488` |
| `IsPluginAllowed(PluginURL)` | `IsPluginAllowed(PluginURL, OutReason)` | `UE_DEPRECATED(5.6)` in `GameFeaturesProjectPolicies.h:102` |
| `GetStreamingAssetInstallBundles(PluginURL)` | `GetStreamingAssetInstallModes(PluginURL, InstallBundleNames)` | `UE_DEPRECATED(5.6)` in `GameFeaturesProjectPolicies.h:154` |
| `FInstallBundlePluginProtocolOptions::ReleaseInstallBundleFlags` | Nothing — release flags are applied internally | `UE_DEPRECATED(5.6)` in `GameFeaturesSubsystem.h:409` |
| `FString`-based plugin details (`FPluginString`) | Set `UE_GAME_FEATURE_PLUGIN_DETAILS_USE_UTF8_STRING=1` and use the UTF8 forms | `UE_DEPRECATED(5.8)` in `GameFeaturesSubsystem.h:258` |
| Overriding `OnGameFeatureActivating()` with no argument | Override `OnGameFeatureActivating(FGameFeatureActivatingContext& Context)` | "Older-style activation function" in `GameFeatureAction.h:38-39` |
| `GetGameFeaturePluginState(URL)` | `GetPluginState(URL)` | Not an engine function; the real call is at `GameFeaturesSubsystem.h:731` |
| `UGameFeatureData::Actions` touched directly | `GetActions()` | `Actions` is protected in `GameFeatureData.h:120`; getter `:102` |
| `ULyraExperienceDefinition`, `ULyraExperienceManagerComponent`, `UGameFeatureAction_AddInputContextMapping` | Lyra sample code, not engine API — write your own equivalents, see [references/experience-system.md](references/experience-system.md) | Not in any 5.8 engine header |

## Common Mistakes

**Inventing a plugin `Type`:** a `.uplugin` has no plugin-level `Type` key, so `"Type": "GameFeature"` is silently ignored and the plugin never becomes a feature; put it under `Plugins/GameFeatures/` and set `"ExplicitlyLoaded": true`.

**Never registering the receiver:** components are injected only into registered actors, and nothing is logged when you forget.

```cpp
// WRONG — no components are ever added
void AMyCharacter::BeginPlay() { Super::BeginPlay(); }
// RIGHT
void AMyCharacter::BeginPlay()
{
    Super::BeginPlay();
    UGameFrameworkComponentManager::AddGameFrameworkComponentReceiver(this);
}
```

**Dropping `FComponentRequestHandle`:** the handle is RAII, so discarding it removes the request immediately.

```cpp
// WRONG — the temporary destructs at the end of the statement
CompMgr->AddComponentRequest(ActorClass, UMyHealthComponent::StaticClass(), AdditionFlags);
// RIGHT — store it for the lifetime of the injection
RequestHandle = CompMgr->AddComponentRequest(ActorClass, UMyHealthComponent::StaticClass(), AdditionFlags);
```

**Fetching the component manager as an engine subsystem:** it is a `UGameInstanceSubsystem`, so `GEngine->GetEngineSubsystem<UGameFrameworkComponentManager>()` returns null; use `UGameFrameworkComponentManager::GetForActor(MyActor)`.

**Not resuming a paused deactivation:** if `PauseDeactivationUntilComplete` is called and the returned `FSimpleDelegate` is never executed, the plugin stays in `Deactivating` forever; execute it on every path, including failures.

**Cross-component setup in `BeginPlay`:** a component injected by another feature may not exist yet.

```cpp
// WRONG — the other component may be added later
void UMyModularComponent::BeginPlay()
{
    Super::BeginPlay();
    GetOwner()->FindComponentByClass<UMyOtherComponent>()->Configure();
}
// RIGHT — react once the shared state is reached
void UMyModularComponent::HandleChangeInitState(UGameFrameworkComponentManager* Manager, FGameplayTag CurrentState, FGameplayTag DesiredState)
{
    if (DesiredState == TAG_InitState_DataInitialized)
    {
        GetOwner()->FindComponentByClass<UMyOtherComponent>()->Configure();
    }
}
```

**Overriding both `OnGameFeatureActivating` overloads:** the base context version calls the no-arg version, so activation code runs twice; override only the context version.

**Treating Lyra classes as engine API:** `ULyraExperienceDefinition`, `ULyraExperienceManagerComponent` and Lyra's own action subclasses ship with the Lyra sample and are absent from the engine; the engine pieces are `UGameFeatureData`, `UGameFeatureAction` and `UGameFrameworkComponentManager`.

## Related Skills

- `ue-gameplay-tags-messaging` — declaring the `FGameplayTag` values used as init states, tag containers and queries
- `ue-module-build-system` — plugin and module layout, `Build.cs` dependencies, loading phases
- `ue-actor-component-architecture` — component creation, registration, attachment and tick
- `ue-gameplay-abilities` — granting abilities and attribute sets from a feature's components
- `ue-data-assets-tables` — primary data assets, Asset Manager scanning and asset bundles
- `ue-input-system` — Enhanced Input mapping contexts a feature adds and removes
- `ue-gameplay-framework` — GameMode, GameState, PlayerController and PlayerState lifecycle
- `ue-world-level-streaming` — World Partition content bundles and data layers added by features
