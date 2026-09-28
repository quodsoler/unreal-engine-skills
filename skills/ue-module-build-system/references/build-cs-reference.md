# Build.cs and Target.cs Field Reference (UE 5.8)

Every field below exists in `Engine/Source/Programs/UnrealBuildTool/Configuration/Rules/ModuleRules.cs` or `TargetRules.cs` in UE 5.8. Fields are set inside the constructor of a class deriving from `ModuleRules` (one per module) or `TargetRules` (one per target).

---

## Class Structure

```csharp
using UnrealBuildTool;
using System.IO;                 // Path.Combine, File.Exists

public class MyModule : ModuleRules
{
    public MyModule(ReadOnlyTargetRules Target) : base(Target)
    {
        // All fields are set here
    }
}
```

`ReadOnlyTargetRules Target` exposes the target being built: `Target.Platform`, `Target.Configuration`, `Target.Type`, `Target.bBuildEditor`, `Target.bBuildDeveloperTools`, `Target.bWithLiveCoding`, `Target.LinkType`, `Target.ProjectFile`, `Target.IncludeOrderVersion`. Inside the module, `ModuleDirectory` is the folder containing the `Build.cs` and `PluginDirectory` is the owning plugin's root (or null for project modules).

---

## Dependency Fields

### PublicDependencyModuleNames

```csharp
PublicDependencyModuleNames.AddRange(new string[]
{
    "Core",           // FString, TArray, TMap, FName, UE_LOG
    "CoreUObject",    // UObject, UClass, reflection, FInstancedStruct
    "Engine",         // AActor, UWorld, UGameInstance, UActorComponent
    "InputCore",      // FKey, EKeys
});
```

"Modules that are required by our public source files." Their include paths and link dependencies are inherited by every module that depends on yours. Promote a module here only when a type from it appears in one of your `Public/` headers.

### PrivateDependencyModuleNames

```csharp
PrivateDependencyModuleNames.AddRange(new string[]
{
    "Slate",          // SWidget, SCompoundWidget
    "SlateCore",      // FSlateColor, FSlateBrush
    "RenderCore",     // FRenderCommandFence
    "RHI",            // FRHITexture, FRHICommandList
    "Json",           // FJsonObject, FJsonSerializer
    "HTTP",           // FHttpModule, IHttpRequest
});
```

"Modules that our private code depends on but nothing in our public include files depend on." Not inherited downstream. Default to this list.

### PublicIncludePathModuleNames / PrivateIncludePathModuleNames

```csharp
// Header access only — no link dependency, no exported symbols
PrivateIncludePathModuleNames.Add("AssetRegistry");
```

Use for forward declarations and header-only types. Calling an exported function from such a module still fails with LNK2019.

### DynamicallyLoadedModuleNames

```csharp
DynamicallyLoadedModuleNames.AddRange(new string[] { "OnlineSubsystem", "OnlineSubsystemSteam" });
```

"Additional modules this module may require at run-time." UBT builds them but does not link them; load with `FModuleManager::Get().LoadModule(...)` or `FModuleManager::LoadModulePtr<T>(...)`.

### CircularlyReferencedDependentModules

```csharp
// Legacy escape hatch only — the module must already be in a dependency list
CircularlyReferencedDependentModules.Add("MyOtherModule");
```

"Only for legacy reason, should not be used in new code." `bValidateCircularDependencies` (default true) validates cycles against UBT's allow list; disabling it is "strongly discouraged".

---

## Include Path Fields

UBT discovers `Public/`, `Internal/` and `Private/` automatically (`PublicIncludePaths` is documented as "currently not needed as we discover all files from the 'Public' folder"). Use these only for extra folders.

```csharp
// Extra folders exposed to dependents (rarely needed)
PublicIncludePaths.Add(Path.Combine(ModuleDirectory, "Public", "Interfaces"));

// Extra folders visible only inside this module
PrivateIncludePaths.Add(Path.Combine(ModuleDirectory, "Private", "Helpers"));

// Third-party headers; not checked when resolving header dependencies
string ThirdPartyDir = Path.Combine(ModuleDirectory, "ThirdParty");
PublicSystemIncludePaths.Add(Path.Combine(ThirdPartyDir, "include"));
```

`bLegacyPublicIncludePaths` and `bLegacyParentIncludePaths` (both false under `BuildSettingsVersion.V3` and later) control whether other modules' headers may be included by bare file name or relative to their parent folder. Leave them false and write `#include "GameFramework/Actor.h"`.

---

## PCH and IWYU Fields

### PCHUsage

| `PCHUsageMode` | Meaning (from `ModuleRules.cs`) |
|---|---|
| `Default` | Engine modules use shared PCHs, game modules do not |
| `NoPCHs` | Never use any PCHs |
| `NoSharedPCHs` | Always generate a unique PCH for this module |
| `UseSharedPCHs` | Shared PCHs are OK |
| `UseExplicitOrSharedPCHs` | Shared PCHs unless `PrivatePCHHeaderFile` is set; no source file includes a module PCH by hand |

```csharp
PCHUsage = PCHUsageMode.UseExplicitOrSharedPCHs;   // what BuildSettingsVersion.V2+ defaults to
PrivatePCHHeaderFile = "Private/MyModulePCH.h";    // explicit private PCH (optional)
SharedPCHHeaderFile = "Public/MyModuleSharedPCH.h"; // offer a shared PCH to dependents (engine-style)
```

### IWYUSupport

| `IWYUSupport` | Meaning |
|---|---|
| `None` | Code does not compile under IWYU; skipped entirely |
| `KeepAsIs` | Could be processed, but stays as written; changes handled manually |
| `KeepAsIsForNow` | Parsed and stripped of includes from the outside, files not modified |
| `KeepPublicAsIsForNow` | As above, but private headers and `.cpp` files may be updated |
| `Full` (default) | `UnrealBuildTool -Mode=IWYU` may rewrite this module's includes |

```csharp
IWYUSupport = IWYUSupport.Full;
```

---

## Compiler Flags

```csharp
bEnableExceptions = true;                     // C++ exceptions (default false)
bUseRTTI = true;                              // typeid / dynamic_cast (default false)
MinCpuArchX64 = MinimumCpuArchitectureX64.AVX2; // None, AVX, AVX2, AVX512; null = target default
bEnableBufferSecurityChecks = true;           // default true
bDisableStaticAnalysis = true;                // skip this module when -StaticAnalyzer is on
bUseUnity = false;                            // per-module unity override
bTreatAsEngineModule = true;                  // apply engine warning/include rules to a project module
```

### OptimizeCode

```csharp
OptimizeCode = CodeOptimization.Always;              // even in Debug
OptimizeCode = CodeOptimization.Never;               // even in Shipping (debugging a module)
OptimizeCode = CodeOptimization.InNonDebugBuilds;    // Development/Test/Shipping
OptimizeCode = CodeOptimization.InShippingBuildsOnly;
OptimizeCode = CodeOptimization.Default;             // follow the build configuration
```

### CppStandard

```csharp
CppStandard = CppStandardVersion.Cpp20;   // Cpp20, Cpp23, Latest; Default, EngineDefault and Minimum equal Cpp20 (Cpp14/Cpp17 are [Obsolete] in 5.8)
```

`ModuleRules.CppStandard` is nullable and overrides `TargetRules.CppStandard` for one module.

### Warnings

`bWarningsAsErrors` promotes every warning. Individual categories live on `CppCompileWarningSettings` (type `CppCompileWarnings`, `Configuration/Rules/CppCompileWarnings.cs`); each accepts `WarningLevel.Default`, `Off`, `Warning` or `Error`. Module settings override the target's.

| `CppCompileWarningSettings.` | MSVC flags UBT emits | Default |
|---|---|---|
| `ShadowVariableWarningLevel` | C4456, C4458, C4459 | Keyed on `BuildSettingsVersion`, not engine vs project: Error at target scope from `V2`, Warning for V1 and module-constructor scope (`CppCompileWarnings.cs:91-93`) |
| `UnsafeTypeCastWarningLevel` | C4244, C4838 | Off for projects |
| `UndefinedIdentifierWarningLevel` | C4668 | Error from `BuildSettingsVersion.V6` |
| `DeprecationWarningLevel` | C4996 (`UE_DEPRECATED`) | Warning |
| `ReturnTypeWarningLevel`, `DanglingWarningLevel`, `UnreachableCodeWarningLevel` | — | Error from `BuildSettingsVersion.V7` |

```csharp
bWarningsAsErrors = true;
CppCompileWarningSettings.ShadowVariableWarningLevel = WarningLevel.Error;
CppCompileWarningSettings.UnsafeTypeCastWarningLevel = WarningLevel.Warning;
CppCompileWarningSettings.DeprecationWarningLevel = WarningLevel.Error;
```

---

## Third-Party Libraries

### AddEngineThirdPartyPrivateStaticDependencies

```csharp
// Static, private link to libraries under Engine/Source/ThirdParty/<Name>/<Name>.Build.cs
AddEngineThirdPartyPrivateStaticDependencies(Target, "zlib", "OpenSSL", "libcurl");

// Dynamic (DLL) variant
AddEngineThirdPartyPrivateDynamicDependencies(Target, "OpenSSL");
```

UBT notes: "There is no AddThirdPartyPublicStaticDependencies function." Third-party modules are always private.

### PublicAdditionalLibraries / PublicSystemLibraries / PublicDelayLoadDLLs / PublicFrameworks

```csharp
string ThirdPartyDir = Path.Combine(ModuleDirectory, "ThirdParty");

PublicAdditionalLibraries.Add(Path.Combine(ThirdPartyDir, "lib", "Win64", "MyLib.lib")); // full path to a .lib/.a
PublicSystemLibraries.Add("ws2_32.lib");                                                  // system library by name
PublicDelayLoadDLLs.Add("MyLib.dll");                                                     // Windows delay-load
// PublicFrameworks: Apple frameworks by name (Mac, IOS, TVOS, VisionOS)
```

### RuntimeDependencies

"List of files which this module depends on at runtime. These files will be staged along with the target."

```csharp
RuntimeDependencies.Add("$(PluginDir)/Binaries/ThirdParty/MyLib/Win64/MyLib.dll");             // staged next to binaries
RuntimeDependencies.Add("$(ProjectDir)/Config/MyRuntimeData.json", StagedFileType.NonUFS);     // explicit type
RuntimeDependencies.Add("$(BinaryOutputDir)/MyLib.dll", Path.Combine(ThirdPartyDir, "bin", "Win64", "MyLib.dll"), StagedFileType.NonUFS); // copy from source path
```

`StagedFileType`: `UFS` (inside the pak/IoStore), `NonUFS` (loose file, default), `DebugNonUFS`, `SystemNonUFS`. Path variables UBT expands: `$(EngineDir)`, `$(ProjectDir)`, `$(PluginDir)`, `$(BinaryOutputDir)`.

---

## Preprocessor Definitions

```csharp
PublicDefinitions.Add("MYGAME_WITH_DEBUG_MENU=1");   // visible to this module and its dependents
PrivateDefinitions.Add("MYGAME_WITH_DEBUG_MENU=0");  // visible only inside this module
```

Target-wide equivalents: `TargetRules.GlobalDefinitions` (every module including engine) and `ProjectDefinitions` (project modules only).

---

## Helper Methods on ModuleRules

```csharp
SetupIrisSupport(Target);                 // adds Iris replication dependencies based on target settings
SetupGameplayDebuggerSupport(Target);     // adds GameplayDebugger when the target supports it
SetupModulePhysicsSupport(Target);        // adds physics module dependencies
```

---

## Platform-Conditional Patterns

```csharp
public MyModule(ReadOnlyTargetRules Target) : base(Target)
{
    PCHUsage = PCHUsageMode.UseExplicitOrSharedPCHs;

    PublicDependencyModuleNames.AddRange(new string[] { "Core", "CoreUObject", "Engine" });

    // Editor-only dependencies
    if (Target.bBuildEditor)
    {
        PrivateDependencyModuleNames.AddRange(new string[] { "UnrealEd", "ToolMenus", "EditorFramework" });
    }

    // Platform-specific libraries
    if (Target.Platform == UnrealTargetPlatform.Win64)
    {
        PublicAdditionalLibraries.Add(Path.Combine(ModuleDirectory, "lib", "Win64", "MyLib.lib"));
        PublicDelayLoadDLLs.Add("MyLib.dll");
    }
    else if (Target.Platform == UnrealTargetPlatform.Mac)
    {
        PublicAdditionalLibraries.Add(Path.Combine(ModuleDirectory, "lib", "Mac", "libMyLib.a"));
    }

    // Configuration-specific code
    if (Target.Configuration != UnrealTargetConfiguration.Shipping)
    {
        PrivateDependencyModuleNames.Add("EngineSettings");
        PublicDefinitions.Add("MYGAME_WITH_DEBUG_MENU=1");
    }
    else
    {
        PublicDefinitions.Add("MYGAME_WITH_DEBUG_MENU=0");
    }

    // Server-only / client-only code paths
    if (Target.Type == TargetType.Server)
    {
        PrivateDefinitions.Add("MYGAME_WITH_DEBUG_MENU=0");
    }
}
```

### Target.Platform Values (`UnrealTargetPlatform`)

`Win64`, `Mac`, `Linux`, `LinuxArm64`, `IOS`, `TVOS`, `VisionOS`, `Android`. Console platforms come from platform extensions and are not part of the public enum.

### Target.Configuration Values (`UnrealTargetConfiguration`)

| Value | Description |
|---|---|
| `Debug` | Engine and game unoptimized, full symbols |
| `DebugGame` | Engine optimized, game modules unoptimized |
| `Development` | Default; optimized with checks and logging |
| `Test` | Shipping optimizations with stats and console |
| `Shipping` | Final release; `check`/`UE_LOG` compiled out unless `bUseChecksInShipping`/`bUseLoggingInShipping` |

### Target.LinkType Values (`TargetLinkType`)

`Default` (Modular for Editor, Monolithic for Game/Client/Server), `Monolithic` (one executable, `IS_MONOLITHIC`), `Modular` (one DLL per module).

---

## TargetRules Reference (Target.cs)

```csharp
using UnrealBuildTool;
using System.Collections.Generic;

public class MyGameTarget : TargetRules
{
    public MyGameTarget(TargetInfo Target) : base(Target)
    {
        Type = TargetType.Game;
        DefaultBuildSettings = BuildSettingsVersion.V7;              // 5.8 defaults; Latest == V7
        IncludeOrderVersion = EngineIncludeOrderVersion.Unreal5_8;   // Latest == Unreal5_8

        ExtraModuleNames.AddRange(new string[] { "MyGame", "MyGameCore" });

        bUseUnityBuild = false;              // faster incremental builds, slower full builds
        bUseAdaptiveUnityBuild = true;       // default: files you edit leave the unity blob
        bAllowLTCG = Target.Configuration == UnrealTargetConfiguration.Shipping;
        bUseLoggingInShipping = false;       // keep UE_LOG in Shipping when true
        bUseChecksInShipping = false;        // keep check()/ensure() in Shipping when true
        bWithLiveCoding = true;              // Live Coding support (editor/development)
        CppStandard = CppStandardVersion.Cpp20;
        bWarningsAsErrors = false;
        CppCompileWarningSettings.ShadowVariableWarningLevel = WarningLevel.Error;
        GlobalDefinitions.Add("MYGAME_WITH_DEBUG_MENU=1");
    }
}
```

| Field | Purpose |
|---|---|
| `Type` | `TargetType.Game`, `Editor`, `Client`, `Server`, `Program` |
| `DefaultBuildSettings` | `BuildSettingsVersion.V1`–`V7`; `Latest = V7` |
| `IncludeOrderVersion` | `EngineIncludeOrderVersion.Unreal5_7`, `Unreal5_8`; `Latest = Unreal5_8`, `Oldest = Unreal5_6` (deprecated) |
| `ExtraModuleNames` | Project modules UBT compiles and links into this target |
| `LaunchModuleName` | Module containing the entry point (Programs) |
| `LinkType` | `TargetLinkType.Default`, `Monolithic`, `Modular` |
| `BuildEnvironment` | `TargetBuildEnvironment.Shared`, `Unique`, `UniqueIfNeeded` |
| `bOverrideBuildEnvironment` | "Ignore violations to the shared build environment (eg. editor targets modifying definitions)" |
| `bUseUnityBuild`, `bForceUnityBuild`, `bUseAdaptiveUnityBuild` | Unity build control |
| `bUsePCHFiles`, `bUseSharedPCHs` | Global PCH switches |
| `bUseLoggingInShipping`, `bUseChecksInShipping` | Keep logging / checks in Shipping |
| `bWithServerCode`, `bWithPushModel` | Compile `WITH_SERVER_CODE`; enable push-model replication defines |
| `bBuildDeveloperTools`, `bBuildWithEditorOnlyData`, `bBuildRequiresCookedData` | Feature gates for developer tools and editor-only data |
| `bCompileAgainstEngine`, `bCompileAgainstCoreUObject`, `bCompileAgainstApplicationCore`, `bUsesSlate` | Program-target composition |
| `bWithLiveCoding` | Live Coding support |
| `CppStandard`, `bWarningsAsErrors`, `DefaultWarningLevel`, `CppCompileWarningSettings` | Language and warning policy |
| `OptimizationLevel`, `DebugInfo`, `bAllowLTCG` | `OptimizationMode`, `DebugInfoMode`, link-time codegen |
| `StaticAnalyzer` | `StaticAnalyzer.None` or a supported analyzer |
| `GlobalDefinitions`, `ProjectDefinitions` | Target-wide preprocessor definitions |
| `bBuildAllModules`, `bCompileWithPluginSupport` | Build every module / enable plugin loading in Programs |
| `WindowsPlatform` | `WindowsTargetRules` (toolchain version, conformance flags) |

### BuildSettingsVersion

| Version | Defaults introduced |
|---|---|
| `V1` | Legacy defaults |
| `V2` | `ModuleRules.PCHUsage = UseExplicitOrSharedPCHs`, `bLegacyPublicIncludePaths = false` |
| `V3` | `ModuleRules.bLegacyParentIncludePaths = false` |
| `V4` | `TargetRules.CppStandard` default `Cpp20`, `WindowsPlatform.bStrictConformanceMode = true` |
| `V5` | `TargetRules.bValidateFormatStrings = true` |
| `V6` | `WindowsPlatform.bStrictInlineConformance = true`, `CppCompileWarningSettings.UndefinedIdentifierWarningLevel = Error` |
| `V7` | `ReturnTypeWarningLevel`, `DanglingWarningLevel`, `UnreachableCodeWarningLevel` = `Error` |

---

## UBT Command-Line Modes

`UnrealBuildTool -Mode=<Name>` selects a tool mode; `Modes/*.cs` in UBT define these names: `Build` (default), `Clean`, `GenerateProjectFiles`, `IWYU`, `FixIncludePaths`, `InlineGeneratedCpps`, `QueryTargets`, `Query`, `JsonExport`, `WriteMetadata`, `GenerateClangDatabase`, `Analyze`, `Test`, `UnrealHeaderTool`, `ValidatePlatforms`, `PrintBuildGraphInfo`, `ProfileUnitySizes`, `Deploy`. Compile one translation unit with `-SingleFile=<path>`.

---

## Common Engine Module Reference

Every module name below is a 5.8 module. The "Plugin" column names the `.uplugin` that must be enabled in `.uproject` (blank for engine modules that are always available).

| Module | Provides | Plugin |
|---|---|---|
| `Core` | `FString`, `TArray`, `TMap`, `FName`, `FText`, `UE_LOG`, `check`, `FModuleManager`, `FCoreDelegates` | — |
| `CoreUObject` | `UObject`, `UClass`, `UPROPERTY`/`UFUNCTION` reflection, `FInstancedStruct`, `FStructView` | — |
| `Engine` | `AActor`, `UActorComponent`, `UPrimitiveComponent`, `UWorld`, `UGameInstance`, `UWorldSubsystem`, `UAssetManager`, `FStreamableManager`, `UDataTable`, `USaveGame`, `USoundBase`, `UAudioComponent`, `FBodyInstance`, `UGameplayStatics`, `UKismetSystemLibrary` | — |
| `InputCore` | `FKey`, `EKeys` | — |
| `EnhancedInput` | `UInputAction`, `UInputMappingContext`, `UEnhancedInputComponent`, `UEnhancedInputLocalPlayerSubsystem` | `EnhancedInput` |
| `GameplayTags` | `FGameplayTag`, `FGameplayTagContainer`, `UGameplayTagsManager` | — |
| `GameplayTasks` | `UGameplayTask`, `UGameplayTasksComponent` | — |
| `GameplayAbilities` | `UAbilitySystemComponent`, `UGameplayAbility`, `UGameplayEffect` | `GameplayAbilities` |
| `NetCore` | `FFastArraySerializer`, `FFastArraySerializerItem`, push model (`Net/Core/PushModel/PushModel.h`) | — |
| `DeveloperSettings` | `UDeveloperSettings` (`Engine/DeveloperSettings.h`) | — |
| `Slate` | `SWidget`, `SCompoundWidget`, `SWindow` | — |
| `SlateCore` | `FSlateColor`, `FSlateBrush`, `FGeometry` | — |
| `UMG` | `UUserWidget`, `UWidget`, `UButton`, `UTextBlock` | — |
| `CommonUI` | `UCommonActivatableWidget`, `UCommonButtonBase`, `UCommonUserWidget` | `CommonUI` |
| `CommonInput` | `UCommonInputSubsystem` | `CommonUI` |
| `ModelViewViewModel` | `UMVVMViewModelBase`, `UMVVMView`, `UMVVMSubsystem` | `ModelViewViewModel` (Beta in 5.8) |
| `AIModule` | `AAIController`, `UBehaviorTree`, `UBehaviorTreeComponent` | — |
| `NavigationSystem` | `UNavigationSystemV1`, navmesh queries | — |
| `StateTreeModule` | `UStateTree`, `FStateTreeTaskBase`, `FStateTreeConditionBase` | `StateTree` |
| `GameplayStateTreeModule` | `UStateTreeComponent`, `UStateTreeAIComponent` | `GameplayStateTree` |
| `MassEntity` | `FMassEntityManager`, `UMassProcessor`, `FMassEntityQuery`, `FMassExecutionContext`, `UMassEntitySubsystem`, `FMassEntityHandle` | — (core module) |
| `MassCore` | `FTransformFragment` (`Mass/EntityFragments.h`), `FMassFragment`, external subsystem traits | — (core module) |
| `MassSignals` | `UMassSignalSubsystem`, `UMassSignalProcessorBase` | — (core module) |
| `MassSpawner`, `MassRepresentation`, `MassAIBehavior` | Spawning, visualization and StateTree-driven Mass agents | `MassGameplay`, `MassAI` (Experimental in 5.8) |
| `Niagara` | `UNiagaraComponent`, `UNiagaraSystem`, `UNiagaraFunctionLibrary` | `Niagara` |
| `PCG` | `UPCGComponent`, `UPCGGraph`, `UPCGSettings`, `IPCGElement` | `PCG` |
| `Mover` | `UMoverComponent`, `UCharacterMoverComponent`, `UBaseMovementMode` | `Mover` (Experimental in 5.8) |
| `GameplayCameras` | `UGameplayCameraComponent`, `UCameraAsset`, `UCameraRigAsset` | `GameplayCameras` (Experimental in 5.8) |
| `LevelSequence` | `ULevelSequence`, `ULevelSequencePlayer`, `ALevelSequenceActor` | — |
| `MovieScene`, `MovieSceneTracks` | `UMovieScene`, `UMovieSceneSequence`, `UMovieSceneTrack` and built-in tracks | — |
| `CinematicCamera` | `ACineCameraActor`, `UCineCameraComponent` | — |
| `PhysicsCore` | `UPhysicalMaterial`, physics settings shared with Chaos | — |
| `Chaos` | Chaos solver types (`Chaos::` namespace); needed only when calling the solver directly | — |
| `ProceduralMeshComponent` | `UProceduralMeshComponent` | `ProceduralMeshComponent` |
| `GeometryScriptingCore` | `UGeometryScriptLibrary_MeshPrimitiveFunctions` and the other Geometry Script function libraries | `GeometryScripting` |
| `GeometryFramework` | `UDynamicMesh`, `UDynamicMeshComponent` | — |
| `AudioExtensions` | `IAudioParameterControllerInterface`, `FAudioParameter` | — |
| `AudioMixer` | Submix effects, `USoundSubmix` runtime, `UAudioBus` mixing | — |
| `GameFeatures` | `UGameFeaturesSubsystem`, `UGameFeatureData`, `UGameFeatureAction` | `GameFeatures` (Beta in 5.8) |
| `ModularGameplay` | `UGameFrameworkComponentManager` | `ModularGameplay` (Beta in 5.8) |
| `RenderCore` | `FRenderCommandFence`, render-thread utilities | — |
| `RHI` | `FRHITexture`, `FRHICommandList` | — |
| `Json`, `JsonUtilities` | `FJsonObject`, `FJsonSerializer` / `FJsonObjectConverter` | — |
| `HTTP` | `FHttpModule`, `IHttpRequest` | — |
| `AssetRegistry` | `IAssetRegistry`, `FAssetData` | — |
| `Projects` | `IPluginManager`, `IPlugin` | — |
| `OnlineSubsystem` | `IOnlineSubsystem` (legacy OSS) | `OnlineSubsystem` |
| `UnrealEd` | `UEditorEngine`, editor framework (editor-only) | — |
| `EditorSubsystem` | `UEditorSubsystem` (editor-only) | — |
| `EditorFramework` | Toolkit host interfaces (editor-only) | — |
| `ToolMenus` | `UToolMenus` (editor-only) | — |
| `PropertyEditor` | `IDetailCustomization`, `FPropertyEditorModule` (editor-only) | — |
| `Blutility` | `UEditorUtilitySubsystem` (editor-only) | — |

---

## API Macro Generation Rules

UBT builds the export macro by upper-casing the module name and appending `_API` (`Configuration/UEBuildModule.cs`) and defines it per compile as one of empty, `DLLEXPORT` or `DLLIMPORT`:

- Module `MyPlugin` → `MYPLUGIN_API`
- Module `MyPlugin_Runtime` → `MYPLUGIN_RUNTIME_API`
- Module `Core` → `CORE_API`

`DLLEXPORT`/`DLLIMPORT` expand to `__declspec(dllexport)`/`__declspec(dllimport)` on Windows (`Windows/WindowsPlatform.h`) and to `__attribute__((visibility("default")))` on Unix platforms (`Unix/UnixPlatform.h`). In monolithic builds the macro is empty. Engine headers wrap the module macro in a file-local `#define UE_API CORE_API` … `#undef UE_API` pair; project headers write `MYGAME_API` directly.
