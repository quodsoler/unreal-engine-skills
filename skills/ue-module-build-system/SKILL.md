---
name: ue-module-build-system
description: "Use when writing or fixing Build.cs, Target.cs, .uproject or .uplugin files, creating a module or plugin, or resolving an Unreal build error. Also use when the user mentions 'unresolved external symbol', 'LNK2019', 'cannot open include file', 'C1083', 'MODULENAME_API', 'PublicDependencyModuleNames', 'IWYU', 'PCHUsage', 'IMPLEMENT_MODULE', 'IMPLEMENT_PRIMARY_GAME_MODULE', 'LoadingPhase', 'circular dependency', 'Missing precompiled manifest', 'EngineAssociation', 'IncludeOrderVersion', 'Live Coding'. For UCLASS/UPROPERTY rules see ue-cpp-foundations; for editor modules see ue-editor-tools; for Game Feature plugins see ue-game-features."
metadata:
  version: "2.0.0"
  engine: "5.8"
---

# UE Module & Build System

Target engine: **UE 5.8**. APIs below are verified against the 5.8 headers and UnrealBuildTool sources; older forms are listed under "Deprecated — do not use".

This skill covers UnrealBuildTool (UBT) `ModuleRules` (`*.Build.cs`) and `TargetRules` (`*.Target.cs`), the `.uproject` and `.uplugin` descriptors, module registration (`IModuleInterface` and `IMPLEMENT_MODULE` in `Modules/ModuleManager.h`, module `Core`), plugin discovery (`IPluginManager` in `Interfaces/IPluginManager.h`, module `Projects`), and the compiler, linker, UHT and UBT errors those files cause.

## Context

Read `.agents/ue-project-context.md` if it exists (module names, conventions, enabled plugins, GAS/networking setup). Do not stop if it is missing.

Identify the area from the request and the codebase. Ask only when two plausible readings would produce different code.

| Request is about… | Go to |
|---|---|
| Dependencies, include paths, PCH, warnings in an existing `Build.cs` | [Build.cs Anatomy](#buildcs-anatomy) |
| `Target.cs`, `BuildSettingsVersion`, `IncludeOrderVersion`, target types | [Target.cs](#targetcs) |
| `.uproject`, module `Type`, `LoadingPhase`, enabling plugins | [.uproject File](#uproject-file) |
| Adding a module (`IMPLEMENT_MODULE`, module interface, `StartupModule`) | [Creating a New Module](#creating-a-new-module) |
| Adding a plugin (`.uplugin`, runtime + editor modules, dependencies) | [Creating a Plugin](#creating-a-plugin) |
| "Which module do I add to use type X?" | [Which Module Provides This Type](#which-module-provides-this-type) |
| LNK2019, C1083, UHT, UBT or runtime module-load errors | [Resolving Build Errors](#resolving-build-errors) |

## Build.cs Anatomy

Every module has `<ModuleName>.Build.cs` next to its `Public/` and `Private/` folders. The class name must equal the module name.

```csharp
// Source/MyGame/MyGame.Build.cs
using UnrealBuildTool;

public class MyGame : ModuleRules
{
    public MyGame(ReadOnlyTargetRules Target) : base(Target)
    {
        PCHUsage = PCHUsageMode.UseExplicitOrSharedPCHs;

        // Types from these modules appear in this module's Public/ headers
        PublicDependencyModuleNames.AddRange(new string[]
        {
            "Core",
            "CoreUObject",
            "Engine",
            "InputCore",
            "EnhancedInput",
        });

        // Used only inside Private/ .cpp files; not re-exported downstream
        PrivateDependencyModuleNames.AddRange(new string[]
        {
            "Slate",
            "SlateCore",
            "UMG",
        });

        // Loaded through FModuleManager at runtime, never linked
        DynamicallyLoadedModuleNames.Add("OnlineSubsystem");
    }
}
```

### Public vs Private Dependencies

| Field | Use when |
|---|---|
| `PublicDependencyModuleNames` | A type from the module appears in one of your **Public/** headers (base class, member, parameter, return type, `UPROPERTY`). Downstream modules inherit the include paths. |
| `PrivateDependencyModuleNames` | The module is used only in **Private/** `.cpp` files or private headers. |
| `PublicIncludePathModuleNames` / `PrivateIncludePathModuleNames` | You need the module's headers (forward declarations, inline types) but link nothing from it. |
| `DynamicallyLoadedModuleNames` | You call `FModuleManager::Get().LoadModule(...)` yourself; UBT only ensures the module is built. |

Start private and promote to public only when a public header forces it. Everything-public inflates compile time for every downstream module.

### Include Paths and IWYU

UBT discovers `Public/`, `Internal/` and `Private/` automatically; `PublicIncludePaths`/`PrivateIncludePaths` exist for extra subfolders only. Under `BuildSettingsVersion.V2` and later, `bLegacyPublicIncludePaths` is false, so headers from other modules are written relative to that module's `Public/` (or `Classes/`) root:

```cpp
// Source/MyGame/Private/MyActor.cpp
#include "MyActor.h"                          // own header first
#include "GameFramework/Actor.h"              // Engine/Classes/GameFramework/Actor.h
#include "Engine/World.h"                     // only what this file uses directly
#include "GameFramework/PlayerController.h"
```

`IWYUSupport` defaults to `IWYUSupport.Full`, which lets `UnrealBuildTool -Mode=IWYU` rewrite the module's includes; set `IWYUSupport = IWYUSupport.KeepAsIs` for code you maintain by hand.

### API Export Macro

UBT defines `<MODULENAME>_API` (module name upper-cased plus `_API`) as `DLLEXPORT` while compiling the module and `DLLIMPORT` for importers; monolithic builds define it empty. Anything referenced from another module needs it:

```cpp
// Source/MyGame/Public/MyGameTypes.h
#pragma once
#include "CoreMinimal.h"

class MYGAME_API FMyGameTypes
{
public:
    void DoSomething();
};

MYGAME_API void MyGameFreeFunction();
```

Engine headers wrap this as `#define UE_API CORE_API` at the top of a file and `#undef UE_API` at the bottom (`Async/Mutex.h`). That is an engine-only convention; project code writes `MYGAME_API` directly. Missing `MYGAME_API` on a class another module references is the most common cause of LNK2019.

### Warnings and Compiler Flags

```csharp
bWarningsAsErrors = true;
CppCompileWarningSettings.ShadowVariableWarningLevel = WarningLevel.Error;   // MSVC C4456/C4458/C4459
CppCompileWarningSettings.UnsafeTypeCastWarningLevel = WarningLevel.Warning; // MSVC C4244/C4838
CppCompileWarningSettings.DeprecationWarningLevel = WarningLevel.Error;      // MSVC C4996 from UE_DEPRECATED

bEnableExceptions = false;               // default; enable only for third-party code that throws
bUseRTTI = false;                        // default; enable only for third-party dynamic_cast/typeid
CppStandard = CppStandardVersion.Cpp20;  // per-module override; the engine default is Cpp20
bUseUnity = false;                       // opt this module out of unity builds

// Engine-bundled third-party libraries (Engine/Source/ThirdParty/<Name>/<Name>.Build.cs)
AddEngineThirdPartyPrivateStaticDependencies(Target, "zlib", "OpenSSL", "libcurl");
```

Every `ModuleRules` field, the `RuntimeDependencies` API, platform conditionals and the full engine module table are in [references/build-cs-reference.md](references/build-cs-reference.md).

## Target.cs

`Source/MyGame.Target.cs` and `Source/MyGameEditor.Target.cs`. The class must be named `<FileName>Target`; UBT otherwise fails with "Expecting to find a type to be declared in a target rules named 'MyGameTarget'".

```csharp
// Source/MyGame.Target.cs
using UnrealBuildTool;
using System.Collections.Generic;

public class MyGameTarget : TargetRules
{
    public MyGameTarget(TargetInfo Target) : base(Target)
    {
        Type = TargetType.Game;
        DefaultBuildSettings = BuildSettingsVersion.V7;
        IncludeOrderVersion = EngineIncludeOrderVersion.Unreal5_8;
        ExtraModuleNames.Add("MyGame");
    }
}
```

`Source/MyGameEditor.Target.cs` declares `MyGameEditorTarget` with the same body except `Type = TargetType.Editor;` and `ExtraModuleNames.AddRange(new string[] { "MyGame", "MyGameEditor" });`.

- `BuildSettingsVersion.V7` is what 5.8 project templates emit (`Latest` also equals `V7`). Pin the explicit value; `Latest` changes meaning on every engine upgrade.
- `EngineIncludeOrderVersion.Unreal5_8` selects the `UE_ENABLE_INCLUDE_ORDER_DEPRECATED_IN_5_8` define that keeps removed implicit engine includes available to your code. `Unreal5_1`–`Unreal5_5` are `[Obsolete]` and unsupported; `Unreal5_6` is deprecated. A module may override with `ModuleRules.IncludeOrderVersion`.

| `BuildSettingsVersion` | Defaults it turns on |
|---|---|
| `V2` | `PCHUsage = UseExplicitOrSharedPCHs`, `bLegacyPublicIncludePaths = false` |
| `V3` | `bLegacyParentIncludePaths = false` |
| `V4` | `CppStandard` default becomes `Cpp20`, `WindowsPlatform.bStrictConformanceMode = true` |
| `V5` | `bValidateFormatStrings = true` |
| `V6` | `WindowsPlatform.bStrictInlineConformance = true`, `UndefinedIdentifierWarningLevel = Error` |
| `V7` (= `Latest`) | `ReturnTypeWarningLevel`, `DanglingWarningLevel`, `UnreachableCodeWarningLevel` = `Error` |

| `TargetType` | Builds |
|---|---|
| `Game` | Standalone game executable |
| `Editor` | Editor build; includes `Editor`-type modules |
| `Client` | Networked client without server-only code |
| `Server` | Dedicated server, no renderer |
| `Program` | Standalone tool (needs `bCompileAgainstEngine`/`bCompileAgainstCoreUObject` choices) |

Build configurations (`UnrealTargetConfiguration`): `Debug`, `DebugGame` (engine optimized, game unoptimized), `Development` (default), `Test`, `Shipping`. Useful target switches: `bUseUnityBuild`, `bUseLoggingInShipping`, `bUseChecksInShipping`, `bWithLiveCoding`, `LinkType = TargetLinkType.Monolithic`, `GlobalDefinitions`/`ProjectDefinitions`, and `bOverrideBuildEnvironment = true` to "ignore violations to the shared build environment (eg. editor targets modifying definitions)".

## .uproject File

```json
{
    "FileVersion": 3,
    "EngineAssociation": "5.8",
    "Category": "",
    "Description": "",
    "Modules": [
        {
            "Name": "MyGame",
            "Type": "Runtime",
            "LoadingPhase": "Default",
            "AdditionalDependencies": [ "Engine", "AIModule", "UMG" ]
        },
        { "Name": "MyGameEditor", "Type": "Editor", "LoadingPhase": "Default" }
    ],
    "Plugins": [
        { "Name": "ModelingToolsEditorMode", "Enabled": true, "TargetAllowList": [ "Editor" ] },
        { "Name": "StateTree", "Enabled": true },
        { "Name": "MyPlugin", "Enabled": true, "Optional": true }
    ]
}
```

`AdditionalDependencies` is the descriptor's "list of additional dependencies for building this module". Plugin references accept `Name`, `Enabled`, `Optional`, `TargetAllowList`/`TargetDenyList`, `PlatformAllowList`/`PlatformDenyList`, `TargetConfigurationAllowList`.

### Module Types (`ModuleHostType`)

| `Type` | Loaded by |
|---|---|
| `Runtime` | Any target using the UE runtime (the default for game code) |
| `RuntimeNoCommandlet` | Any target except commandlets |
| `RuntimeAndProgram` | Any target or program |
| `CookedOnly` / `UncookedOnly` | Cooked builds only / uncooked (editor, PIE) builds only — `UncookedOnly` is the type for Blueprint node (K2Node) modules |
| `Developer` | Only when developer-tool support is enabled (deprecated: use `DeveloperTool`) |
| `DeveloperTool` | Any target where `bBuildDeveloperTools` is enabled |
| `Editor` / `EditorNoCommandlet` | Editor only / editor but not commandlets |
| `EditorAndProgram` | Editor or program targets |
| `Program` | Program targets only |
| `ServerOnly` / `ClientOnly` / `ClientOnlyNoCommandlet` | Servers only / clients, commandlets and editor / clients and editor |
| `External` | Never loaded automatically; reference only |

### Loading Phases (`ModuleLoadingPhase`)

| `LoadingPhase` | When |
|---|---|
| `EarliestPossible` | As soon as plugins can load (`GConfig` exists) |
| `PostConfigInit` | Right after the config system; very low-level hooks only |
| `PostSplashScreen` | First screen after the system splash |
| `PreEarlyLoadingScreen` | After `PostConfigInit`, before CoreUObject initializes |
| `PreLoadingScreen` | Before the loading screen triggers |
| `PreDefault` | Right before `Default` — asset types, custom classes other modules need |
| `Default` | During engine init, after game modules load (most modules) |
| `PostDefault` | Right after `Default` |
| `PostEngineInit` | After the engine is fully initialized |
| `None` | Never auto-loaded; use `FModuleManager::Get().LoadModule()` |

## Creating a New Module

```
Source/
  MyModule/
    MyModule.Build.cs
    Public/
      IMyModule.h           (public module interface, optional)
      MyModuleTypes.h
    Private/
      MyModule.cpp          (IMPLEMENT_MODULE lives here)
      MyModuleTypes.cpp
```

```cpp
// Source/MyModule/Public/IMyModule.h
#pragma once
#include "Modules/ModuleManager.h"

class IMyModule : public IModuleInterface
{
public:
    static IMyModule& Get()
    {
        return FModuleManager::LoadModuleChecked<IMyModule>("MyModule");
    }

    static bool IsAvailable()
    {
        return FModuleManager::Get().IsModuleLoaded("MyModule");
    }
};
```

```cpp
// Source/MyModule/Private/MyModule.cpp
#include "IMyModule.h"

class FMyModule : public IMyModule
{
public:
    virtual void StartupModule() override
    {
        // Load dependent modules, register settings, delegates, asset types
    }

    virtual void ShutdownModule() override
    {
        // Unregister in reverse order; dependencies loaded in StartupModule are still alive
    }
};

IMPLEMENT_MODULE(FMyModule, MyModule)
```

| Macro (`Modules/ModuleManager.h`) | Use |
|---|---|
| `IMPLEMENT_MODULE(FMyModule, MyModule)` | Modules without gameplay classes (tools, editor, libraries) |
| `IMPLEMENT_GAME_MODULE(FMyModule, MyModule)` | Modules that contain gameplay `UCLASS`es; expands to `IMPLEMENT_MODULE` |
| `IMPLEMENT_PRIMARY_GAME_MODULE(FDefaultGameModuleImpl, MyGame, "MyGame")` | Exactly one per game; in monolithic builds it defines the project name from `UE_PROJECT_NAME`; the third argument is `DEPRECATED_GameName` (`ModuleManager.h:1100`) and unused |

`FDefaultModuleImpl` and `FDefaultGameModuleImpl` are ready-made classes for modules with no startup logic; `FDefaultGameModuleImpl::IsGameModule()` returns true so hot-reload tooling treats it as game code. The second macro argument must equal the `Build.cs` module name; both the monolithic and modular forms of the macro check it against `UE_MODULE_NAME` with `UE_STATIC_ASSERT_WARN` (`ModuleManager.h:950`, `:968`), which emits a "Module name mismatch" deprecation-style *warning*, an error only under `bWarningsAsErrors`. Other `IModuleInterface` hooks: `PreUnloadCallback()`, `PostLoadCallback()`, `SupportsDynamicReloading()`, `SupportsAutomaticShutdown()`.

Then register the module in three places: the `.uproject` (or `.uplugin`) `Modules` array, `ExtraModuleNames` in every `Target.cs` that should build it, and the dependency lists of modules that use it. Editor-only modules (`"Type": "Editor"`, `PrivateDependencyModuleNames.Add("UnrealEd")` inside `if (Target.bBuildEditor)`) are covered in `ue-editor-tools`.

## Creating a Plugin

```
Plugins/
  MyPlugin/
    MyPlugin.uplugin
    Resources/Icon128.png
    Content/                    (only when CanContainContent is true)
    Source/
      MyPlugin/
        MyPlugin.Build.cs
        Public/  Private/MyPluginModule.cpp
      MyPluginEditor/
        MyPluginEditor.Build.cs
        Public/  Private/MyPluginEditorModule.cpp
```

```json
{
    "FileVersion": 3,
    "Version": 1,
    "VersionName": "1.0",
    "FriendlyName": "My Plugin",
    "Description": "Does something useful.",
    "Category": "Gameplay",
    "CreatedBy": "MyStudio",
    "CreatedByURL": "",
    "DocsURL": "",
    "MarketplaceURL": "",
    "SupportURL": "",
    "EnabledByDefault": false,
    "CanContainContent": true,
    "IsBetaVersion": false,
    "IsExperimentalVersion": false,
    "Installed": false,
    "Modules": [
        { "Name": "MyPlugin", "Type": "Runtime", "LoadingPhase": "PreDefault" },
        { "Name": "MyPluginEditor", "Type": "Editor", "LoadingPhase": "Default", "PlatformAllowList": [ "Win64", "Mac", "Linux" ] }
    ],
    "Plugins": [
        { "Name": "GameplayAbilities", "Enabled": true },
        { "Name": "StateTree", "Enabled": true, "Optional": true }
    ]
}
```

- Declare every plugin whose modules you depend on in `Plugins`; UBT warns "Plugin 'MyPlugin' does not list plugin 'X' as a dependency, but module 'MyPlugin' depends on module 'Y'" otherwise.
- `IsExperimentalVersion` / `IsBetaVersion` are what the editor shows as maturity; `EnabledByDefault: true` enables the plugin in every project without a `.uproject` entry.
- Content-only plugins omit `Modules` entirely and keep `"CanContainContent": true`.
- Game Feature plugins (`UGameFeatureData`, actions) are owned by `ue-game-features`.

```csharp
// Plugins/MyPlugin/Source/MyPlugin/MyPlugin.Build.cs — runtime module
using UnrealBuildTool;

public class MyPlugin : ModuleRules
{
    public MyPlugin(ReadOnlyTargetRules Target) : base(Target)
    {
        PCHUsage = PCHUsageMode.UseExplicitOrSharedPCHs;
        PublicDependencyModuleNames.AddRange(new string[] { "Core", "CoreUObject", "Engine", "GameplayAbilities" });
    }
}
```

```csharp
// Plugins/MyPlugin/Source/MyPluginEditor/MyPluginEditor.Build.cs — editor module
using UnrealBuildTool;

public class MyPluginEditor : ModuleRules
{
    public MyPluginEditor(ReadOnlyTargetRules Target) : base(Target)
    {
        PCHUsage = PCHUsageMode.UseExplicitOrSharedPCHs;
        PublicDependencyModuleNames.AddRange(new string[] { "Core", "CoreUObject", "Engine" });
        PrivateDependencyModuleNames.AddRange(new string[] { "MyPlugin", "UnrealEd", "ToolMenus", "PropertyEditor" });
    }
}
```

The runtime module never includes editor headers; gate editor code with `#if WITH_EDITOR` and editor dependencies with `if (Target.bBuildEditor)`.

**Engine vs project plugins:** `Engine/Plugins/` is shared by every project on that install; `<Project>/Plugins/` travels with the project and wins on a name clash. **Installed (Launcher) engines** ship only precompiled engine modules: depending on an engine module that was not precompiled fails with "Missing precompiled manifest for 'X'. This module can not be referenced in a monolithic precompiled build, remove this reference or migrate to a fully compiled source build."

## Which Module Provides This Type

| Add to `*DependencyModuleNames` | For | Plugin to enable |
|---|---|---|
| `Core` | `FString`, `TArray`, `FName`, `UE_LOG`, `FModuleManager` | — |
| `CoreUObject` | `UObject`, `UClass`, `FInstancedStruct`, reflection | — |
| `Engine` | `AActor`, `UActorComponent`, `UWorld`, `UGameInstance`, `UAudioComponent` | — |
| `InputCore` | `FKey`, `EKeys` | — |
| `EnhancedInput` | `UInputAction`, `UInputMappingContext`, `UEnhancedInputComponent` | `EnhancedInput` |
| `GameplayTags` | `FGameplayTag`, `FGameplayTagContainer`, `UGameplayTagsManager` | — |
| `GameplayTasks` | `UGameplayTask`, `UGameplayTasksComponent` | — |
| `GameplayAbilities` | `UAbilitySystemComponent`, `UGameplayAbility`, `UGameplayEffect` | `GameplayAbilities` |
| `NetCore` | `FFastArraySerializer`, push-model macros | — |
| `DeveloperSettings` | `UDeveloperSettings` | — |
| `Slate`, `SlateCore` | `SWidget`, `SCompoundWidget` / `FSlateBrush`, `FSlateColor` | — |
| `UMG` | `UUserWidget`, `UButton`, `UTextBlock` | — |
| `CommonUI`, `CommonInput` | `UCommonActivatableWidget`, `UCommonButtonBase` | `CommonUI` |
| `ModelViewViewModel` | `UMVVMViewModelBase`, `UMVVMView` | `ModelViewViewModel` (Beta in 5.8) |
| `AIModule`, `NavigationSystem` | `AAIController`, `UBehaviorTree` / `UNavigationSystemV1` | — |
| `StateTreeModule`, `GameplayStateTreeModule` | `UStateTree`, `FStateTreeTaskBase` / `UStateTreeComponent` | `StateTree`, `GameplayStateTree` |
| `MassEntity`, `MassCore`, `MassSignals` | `FMassEntityManager`, `UMassProcessor` / `FTransformFragment` / `UMassSignalSubsystem` | — (core modules) |
| `Niagara` | `UNiagaraComponent`, `UNiagaraSystem`, `UNiagaraFunctionLibrary` | `Niagara` |
| `PCG` | `UPCGComponent`, `UPCGGraph`, `UPCGSettings` | `PCG` |
| `Mover` | `UMoverComponent`, `UCharacterMoverComponent` | `Mover` (Experimental in 5.8) |
| `GameplayCameras` | `UGameplayCameraComponent`, `UCameraAsset` | `GameplayCameras` (Experimental in 5.8) |
| `LevelSequence`, `MovieScene` | `ULevelSequence`, `ULevelSequencePlayer` / `UMovieScene` | — |
| `PhysicsCore`, `Chaos` | `UPhysicalMaterial` / Chaos solver types | — |
| `ProceduralMeshComponent` | `UProceduralMeshComponent` | `ProceduralMeshComponent` |
| `GeometryScriptingCore`, `GeometryFramework` | `UGeometryScriptLibrary_MeshPrimitiveFunctions` / `UDynamicMesh` | `GeometryScripting` |
| `AudioExtensions` | `IAudioParameterControllerInterface`, `FAudioParameter` | — |

The full table, including editor and networking modules, is in [references/build-cs-reference.md](references/build-cs-reference.md).

## Resolving Build Errors

| Error text | First thing to check |
|---|---|
| `LNK2019 unresolved external symbol` / `LNK2001` | Module missing from `*DependencyModuleNames`, or missing `MYGAME_API` on the referenced class |
| `C1083: Cannot open include file` | Module missing from dependencies, or include not written relative to the module's `Public/` root |
| `#include found after .generated.h file` | Move `#include "MyClass.generated.h"` to be the last include |
| `Expected a GENERATED_BODY() at the start of the class` | Add `GENERATED_BODY()` as the first line inside the `UCLASS`/`USTRUCT` body |
| `Could not find definition for module 'X'` | No `X.Build.cs` found, or wrong module name in `.uproject`/`ExtraModuleNames` |
| `Circular dependency for 'X.Build.cs' detected` | Extract a shared module or switch one side to `DynamicallyLoadedModuleNames` |
| `Missing precompiled manifest for 'X'` | Installed engine cannot compile that engine module; remove the dependency or use a source build |
| `Plugin 'X' failed to load because module 'Y' could not be found` | Rebuild the plugin; module not compiled for this target |
| `Module 'X' not found - its StaticallyLinkedModuleInitializers function is null` | Monolithic build did not link the module; add it to `ExtraModuleNames` or a `.uplugin` |

Full messages, causes, MSVC warning-level mapping, Live Coding limits and cooking failures are in [references/common-build-errors.md](references/common-build-errors.md).

## Deprecated — do not use

| Do not emit | Use in 5.8 | Source |
|---|---|---|
| `bEnforceIWYU = true` | `IWYUSupport = IWYUSupport.Full` (already the default) | `[Obsolete("Deprecated in UE5.2 - Use IWYUSupport instead.")]` in `Configuration/Rules/ModuleRules.Obsolete.cs` |
| `bUseAVX = true` | `MinCpuArchX64 = MinimumCpuArchitectureX64.AVX2` | `[Obsolete("bUseAVX is obsolete ... replace with MinCpuArchX64")]` in `ModuleRules.Obsolete.cs` |
| `ShadowVariableWarningLevel = WarningLevel.Error` on `ModuleRules`/`TargetRules` | `CppCompileWarningSettings.ShadowVariableWarningLevel` | `[Obsolete("Deprecated in UE5.6 ...")]` in `ModuleRules.Obsolete.cs`, `TargetRules.Obsolete.cs` |
| `UnsafeTypeCastWarningLevel = ...` directly | `CppCompileWarningSettings.UnsafeTypeCastWarningLevel` | `[Obsolete("Deprecated in UE5.6 ...")]` in `ModuleRules.Obsolete.cs`, `TargetRules.Obsolete.cs` |
| `bEnableUndefinedIdentifierWarnings` / `bUndefinedIdentifierErrors` | `CppCompileWarningSettings.UndefinedIdentifierWarningLevel` | Two steps: 5.5 moved them to `UndefinedIdentifierWarningLevel` on the rules class, 5.6 moved that under `CppCompileWarningSettings`. `[Obsolete(...)]` in `ModuleRules.Obsolete.cs`, `TargetRules.Obsolete.cs` |
| `DeprecationWarningLevel = ...` on `TargetRules` | `CppCompileWarningSettings.DeprecationWarningLevel` | `[Obsolete("Deprecated in UE5.6 ...")]` in `TargetRules.Obsolete.cs` |
| `bUsesSteam = true` | Remove; the flag does nothing | `[Obsolete("Deprecated in UE5.5 - No longer used in engine.")]` in `TargetRules.Obsolete.cs` |
| `bCompileChaos`, `bUseChaos`, `bCompilePhysX`, `bCompileAPEX`, `bCompileNvCloth` | Remove; Chaos is always enabled | `[Obsolete("Deprecated in UE5.1 ...")]` in `TargetRules.Obsolete.cs` |
| `bDisableDebugInfo = true` | `DebugInfo = DebugInfoMode.None` | `[Obsolete("Deprecated in UE5.4 - Replace with TargetRules.DebugInfo")]` in `TargetRules.Obsolete.cs` |
| `bUseFastPDBLinking = true` | Remove | `[Obsolete("Deprecated in UE5.7 - No longer recommended")]` in `TargetRules.Obsolete.cs` |
| `IncludeOrderVersion = EngineIncludeOrderVersion.Unreal5_1` … `Unreal5_5` | `EngineIncludeOrderVersion.Unreal5_8` | `[Obsolete("The Unreal 5.5 include order is unsupported.")]` in `Configuration/Rules/TargetRules.cs` |
| `EngineIncludeOrderVersion.Unreal5_6` | `Unreal5_8` | `[Obsolete("... deprecated and will be unsupported in 5.9.")]` in `TargetRules.cs` |
| `"WhitelistPlatforms"`, `"BlacklistPlatforms"`, `"WhitelistTargets"` in descriptors | `"PlatformAllowList"`, `"PlatformDenyList"`, `"TargetAllowList"` | deprecated-fallback readers in `Configuration/Descriptors/ModuleDescriptor.cs` |
| `"Type": "Developer"` | `"Type": "DeveloperTool"` | "The 'Developer' module type has been deprecated in 4.24" in `ModuleDescriptor.cs` |
| `EngineIncludeOrderVersion.Latest`, `BuildSettingsVersion.Latest` in a shipped project | Pin `Unreal5_8` / `V7` | UBT doc: "Using Latest comes with a high risk of introducing compile errors" (`TargetRules.cs`) |

## Common Mistakes

**Everything in `PublicDependencyModuleNames`:** every downstream module inherits the include paths and recompiles more often. Keep a dependency private until one of your `Public/` headers needs it.

**Missing `MYGAME_API`:** the class compiles inside its own module and fails with LNK2019 from any other module. Add the macro to the class or free-function declaration, not the definition.

**`IMPLEMENT_MODULE` name mismatch:** `IMPLEMENT_MODULE(FMyModule, MyGameplay)` in a module whose `Build.cs` is `MyModule` emits the "Module name mismatch" warning (`UE_STATIC_ASSERT_WARN`, not a hard error) and, separately, fails to load in monolithic builds because the linker looks for `IMPLEMENT_MODULE_MyModule`. The second argument is the module name, not the class.

**Editor code in a runtime module:** `#include "Editor.h"` or `UnrealEd` in `PublicDependencyModuleNames` breaks `Game`, `Client` and `Server` targets. Wrap with `#if WITH_EDITOR` and `if (Target.bBuildEditor)`, or move the code to an `Editor`-type module.

**Module registered in one place only:** a module must appear in `.uproject`/`.uplugin` `Modules` *and* in `ExtraModuleNames` of each `Target.cs`; missing either gives "Could not find definition for module" or a module that never loads.

**Wrong `LoadingPhase` for registration:** asset types, custom `UClass` registrations and console variables that other modules read at `Default` must load at `PreDefault`.

**Plugin dependency not declared:** a `.uplugin` module depending on another plugin's module without a `Plugins` entry produces the UBT dependency warning and breaks when the other plugin is disabled.

**Relying on transitive includes:** a header that compiled because `Engine/World.h` happened to pull it in breaks when that engine header trims its includes under a newer `IncludeOrderVersion`. Include what you use.

**Wrong header path:** including `Actor.h` or `Engine/Actor.h` fails with C1083; the file is `GameFramework/Actor.h` under `Engine/Classes`.

## Related Skills

- `ue-cpp-foundations` — `UCLASS`/`UPROPERTY`/`UFUNCTION` specifiers, `GENERATED_BODY`, what UHT needs in public headers
- `ue-editor-tools` — editor-only modules, `UnrealEd`/`ToolMenus`/`PropertyEditor` dependencies, `UEditorSubsystem`
- `ue-testing-debugging` — `UE_LOG` categories and `bUseLoggingInShipping`, automation test modules and targets
- `ue-game-features` — Game Feature plugins, `UGameFeatureData`, `UGameFeatureAction`, Modular Gameplay
- `ue-project-context` — the `.agents/ue-project-context.md` file that records module names, targets and enabled plugins
- `ue-data-assets-tables` — UDataAsset, UDataTable, soft references, Asset Manager and async loading
