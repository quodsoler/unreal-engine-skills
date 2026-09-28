# Common UE Build Errors: Causes and Fixes (UE 5.8)

Error texts below are the literal MSVC, UnrealHeaderTool (UHT), UnrealBuildTool (UBT) and runtime `FModuleManager`/`IPluginManager` messages emitted by 5.8. Placeholders such as `MyModule` stand for your names.

---

## Linker Errors (LNK)

### LNK2019 — unresolved external symbol

```
error LNK2019: unresolved external symbol "public: void __cdecl FMyGameTypes::DoSomething(void)"
    referenced in function "public: virtual void __cdecl AMyActor::BeginPlay(void)"
```

**Cause A: module not a dependency.** Add the owning module to `PrivateDependencyModuleNames` (or `PublicDependencyModuleNames` if a public header needs it). Header access through `PrivateIncludePathModuleNames` alone does not link.

**Cause B: missing `MYGAME_API`.** The symbol lives in another module but is not exported.

```cpp
// Before
class FMyGameTypes { public: void DoSomething(); };

// After
class MYGAME_API FMyGameTypes { public: void DoSomething(); };
```

**Cause C: free function exported at the definition instead of the declaration.**

```cpp
// MyGameTypes.h — export here
MYGAME_API void MyGameFreeFunction();

// MyGameTypes.cpp — no macro on the definition
void MyGameFreeFunction()
{
}
```

**Cause D: body never compiled for this target.** A member declared in the header but defined only in a `.cpp` that is excluded for this target (for example an editor-only `.cpp` guarded by `#if WITH_EDITOR` while the declaration is not).

### LNK2001 — unresolved external symbol (dllimport variant)

```
error LNK2001: unresolved external symbol "__declspec(dllimport) public: static class UClass * __cdecl UMyObject::StaticClass(void)"
```

Same causes as LNK2019. `StaticClass` variants mean the `UCLASS` itself lacks `MYGAME_API`.

### LNK1104 — cannot open file

```
fatal error LNK1104: cannot open file 'MyLib.lib'
```

A path in `PublicAdditionalLibraries` does not exist for this platform/architecture. Build paths with `Path.Combine` and guard them:

```csharp
string LibPath = Path.Combine(ModuleDirectory, "lib", "Win64", "MyLib.lib");
if (File.Exists(LibPath))
{
    PublicAdditionalLibraries.Add(LibPath);
}
```

### LNK4099 — PDB not found

```
warning LNK4099: PDB 'MyLib.pdb' was not found with 'MyLib.lib'
```

UBT already passes `/ignore:4099` to the Windows linker (`Platform/Windows/VCToolChain.cs`), so this only appears from custom link steps or third-party build scripts. It is harmless.

### Missing precompiled manifest (installed engine builds)

```
Missing precompiled manifest for 'MyEngineModule', '.../MyEngineModule.precompiled'. This module can not be referenced in a monolithic precompiled build, remove this reference or migrate to a fully compiled source build.
```

Launcher (installed) engines ship only the engine modules Epic precompiled. Remove the dependency, depend on a module that is precompiled, or build from a source engine.

---

## Compiler Errors and Warnings (MSVC)

### C1083 — cannot open include file

```
fatal error C1083: Cannot open include file: 'MyPluginTypes.h': No such file or directory
```

**Cause A: module not a dependency.** Add it to `PublicDependencyModuleNames`, `PrivateDependencyModuleNames`, or (headers only) `PrivateIncludePathModuleNames`.

**Cause B: path not relative to the module's `Public/` root.** With `bLegacyPublicIncludePaths = false` (every `BuildSettingsVersion` since `V2`) headers from other modules need the full path from that module's `Public/` or `Classes/` folder.

```cpp
// Wrong: "Actor.h" or "Engine/Actor.h" — neither path exists relative to a Public/ or Classes/ root

// Correct — Engine/Source/Runtime/Engine/Classes/GameFramework/Actor.h
#include "GameFramework/Actor.h"
```

**Cause C: third-party header.** Add its folder to `PublicSystemIncludePaths`.

**Cause D: stale `.generated.h`.** Regenerate project files (`UnrealBuildTool -Mode=GenerateProjectFiles` or the `.uproject` context menu) and rebuild.

### C2039 — is not a member of

```
error C2039: 'GetSubsystem': is not a member of 'UWorld'
```

Usually a forward-declared type used through an incomplete definition, or a header that used to be pulled in transitively. Include the header that defines the member in the `.cpp` (`Engine/World.h` for `UWorld`), and check whether the engine renamed or moved the member (search `UE_DEPRECATED` in the header).

### C2027 / C2371 — undefined type / redefinition

```
error C2027: use of undefined type 'FMyGameTypes'
error C2371: 'FMyGameTypes': redefinition; different basic types
```

**Undefined type:** forward declaration where the full definition is needed. Keep the forward declaration in the header for pointers and references; include the header in the `.cpp`.

```cpp
// MyActor.h
#pragma once
#include "CoreMinimal.h"
#include "GameFramework/Actor.h"
#include "MyActor.generated.h"

class FMyGameTypes;

UCLASS()
class MYGAME_API AMyActor : public AActor
{
    GENERATED_BODY()
public:
    void UseTypes(FMyGameTypes* Types);
};
```

```cpp
// MyActor.cpp
#include "MyActor.h"
#include "MyGameTypes.h"

void AMyActor::UseTypes(FMyGameTypes* Types)
{
    Types->DoSomething();
}
```

**Redefinition:** the same header reached through two different include paths (`MyGameTypes.h` and `Types/MyGameTypes.h`) or a `class`/`struct` keyword mismatch between declaration and definition. Use one path from the module root everywhere.

### C4996 — deprecated (UE_DEPRECATED)

```
warning C4996: 'AActor::NetUpdateFrequency': Public access to NetUpdateFrequency has been deprecated. Use SetNetUpdateFrequency() and GetNetUpdateFrequency() instead. - Please update your code to the new API before upgrading to the next release, otherwise your project will no longer compile.
```

The engine marks the old form with `UE_DEPRECATED(5.x, "...")`; the message names the replacement. Fix the call. `CppCompileWarningSettings.DeprecationWarningLevel = WarningLevel.Error` turns these into build breaks so they cannot accumulate.

### C4456 / C4458 / C4459 — declaration hides

```
error C4458: declaration of 'Owner' hides class member
```

Controlled by `CppCompileWarningSettings.ShadowVariableWarningLevel` (Error by default for targets on `BuildSettingsVersion.V2` and later). Rename the local variable or parameter.

### C4244 / C4838 — conversion, possible loss of data

```
warning C4244: 'argument': conversion from 'double' to 'float', possible loss of data
```

Controlled by `CppCompileWarningSettings.UnsafeTypeCastWarningLevel`. Use explicit `static_cast<float>(...)` or keep the wider type; large world coordinates use `double` in `FVector`.

### C4668 — undefined identifier in `#if`

```
error C4668: 'MYGAME_WITH_DEBUG_MENU' is not defined as a preprocessor macro, replacing with '0' for '#if/#elif'
```

Error from `BuildSettingsVersion.V6`. Define the macro in every configuration (`PublicDefinitions.Add("MYGAME_WITH_DEBUG_MENU=0")` in the `else` branch) instead of relying on the undefined-is-zero behaviour.

---

## UnrealHeaderTool Errors

### `.generated.h` must be the last include

```
error: #include found after .generated.h file - the .generated.h file should always be the last #include in a header
```

```cpp
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/Actor.h"
#include "MyActor.generated.h"   // last include, always
```

### GENERATED_BODY missing or misplaced

```
error: Expected a GENERATED_BODY() at the start of the class
```

`GENERATED_BODY()` must be the first statement inside every `UCLASS`, `USTRUCT`, `UINTERFACE` body and the generated interface class:

```cpp
#pragma once
#include "CoreMinimal.h"
#include "MyGameStats.generated.h"

USTRUCT(BlueprintType)
struct MYGAME_API FMyGameStats
{
    GENERATED_BODY()

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    float Health = 100.f;
};
```

### Missing `#pragma once`

```
error: MyActor.generated.h already included, missing '#pragma once' in MyActor.h
```

The `#error` is emitted by the generated header itself. Add `#pragma once` as the first line of the header.

### Missing `MYGAME_API` on a UCLASS

```cpp
#pragma once
#include "CoreMinimal.h"
#include "UObject/Object.h"
#include "MyObject.generated.h"

UCLASS()
class MYGAME_API UMyObject : public UObject
{
    GENERATED_BODY()
};
```

Omitting the macro compiles inside the module and fails with LNK2001 on `StaticClass()` the first time another module (or Blueprint-generated code in the editor target) references the class.

---

## UnrealBuildTool Errors

### Could not find definition for module

```
Could not find definition for module 'MyGameplay', (referenced via MyGame.uproject -> MyGame.Build.cs)
```

No `MyGameplay.Build.cs` was found under `Source/` or an enabled plugin's `Source/`, or the class inside the `Build.cs` is not named `MyGameplay`. Check spelling in `PublicDependencyModuleNames`, `.uproject` `Modules` and `ExtraModuleNames`.

### Unable to find module referenced by a plugin

```
Unable to find module 'MyPluginEditor' referenced by D:/Project/Plugins/MyPlugin/MyPlugin.uplugin
```

The `.uplugin` lists a module with no matching `Source/<Name>/<Name>.Build.cs`.

### Plugin not found or dependency undeclared

```
Unable to find plugin 'MyPlugin' (referenced via MyGame.uproject). Install it and try again, or remove it from the required plugin list.
Plugin 'MyPlugin' (referenced via MyGame.uproject) does not contain the 'MyPluginEditor' module, but lists it in 'D:/Project/Plugins/MyPlugin/MyPlugin.uplugin'.
Warning: Plugin 'MyPlugin' does not list plugin 'GameplayAbilities' as a dependency, but module 'MyPlugin' depends on module 'GameplayAbilities'.
```

Add the plugin to `Plugins` in the `.uplugin` (dependency) or `.uproject` (enable), mark it `"Optional": true` if it may be absent, and make the `Modules` array match the `Source/` folders.

### Target rules class name

```
Expecting to find a type to be declared in a target rules named 'MyGameTarget'.  This type must derive from the 'TargetRules' type defined by UnrealBuildTool.
```

`Source/MyGame.Target.cs` must declare `public class MyGameTarget : TargetRules`.

### Circular dependency

```
Circular dependency for 'MyModuleA.Build.cs' detected:
```

UBT prints each cycle as `* A -> B -> A`, then `Circular dependency in 'MyModuleA' possibly due to '<file>.Build.cs'` and suggests moving dependencies into a separate module or using `Private/PublicIncludePathModuleNames` (`Configuration/UEBuildModule.cs`). Resolve, in preference order:

1. **Extract a shared module.** Move the interfaces both sides need (`IMyServiceA`, `IMyServiceB`) into `MyInterfaces`; both modules depend on it.
2. **Break the link with dynamic loading.** The lower-level module lists the higher one in `DynamicallyLoadedModuleNames` and resolves it at runtime:

```cpp
// MyModuleB.cpp — no compile-time dependency on MyModuleA
#include "Modules/ModuleManager.h"
#include "IMyModuleA.h"

void UseModuleA()
{
    if (IMyModuleA* ModuleA = FModuleManager::GetModulePtr<IMyModuleA>("MyModuleA"))
    {
        ModuleA->DoThing();
    }
}
```

3. **Forward declare** in headers and include only in `.cpp` files.

`CircularlyReferencedDependentModules` exists for legacy engine cases only; `bValidateCircularDependencies` should stay true.

### Unknown mode

```
No mode named 'GenerateProjectFile'. Available modes are:
```

Valid names include `Build`, `Clean`, `GenerateProjectFiles`, `IWYU`, `FixIncludePaths`, `QueryTargets`, `JsonExport`, `GenerateClangDatabase`.

---

## Runtime Module-Load Failures

```
LogModuleManager: Warning: ModuleManager: Module 'MyModule' not found - its StaticallyLinkedModuleInitializers function is null.
LogModuleManager: Warning: ModuleManager: Unable to load module 'MyModule'  - 0 instances of that module name found.
LogModuleManager: Warning: ModuleManager: Unable to load module 'MyModule' because InitializeModule function was not found.
LogModuleManager: Warning: ModuleManager: Unable to load module 'MyModule' because InitializeModule function failed (returned nullptr.)
Plugin 'MyPlugin' failed to load because module 'MyPlugin' could not be found.  Please ensure the plugin is properly installed, otherwise consider disabling the plugin for this project.
Plugin 'MyPlugin' failed to load because module 'MyPlugin' does not appear to be compatible with the current version of the engine.  The plugin may need to be recompiled.
```

`FModuleManager::LoadModuleWithFailureReason` reports one of `EModuleLoadResult::Success`, `FileNotFound`, `FileIncompatible`, `CouldNotBeLoadedByOS`, `FailedToInitialize`, `NotLoadedByGameThread`.

| Message | Fix |
|---|---|
| `StaticallyLinkedModuleInitializers function is null` | Monolithic target did not link the module: add it to `ExtraModuleNames` or list it in an enabled `.uplugin` |
| `0 instances of that module name found` | The DLL was never built for this target/configuration: rebuild, check `Type` and `PlatformAllowList` |
| `InitializeModule function was not found` | The module has no `IMPLEMENT_MODULE` (or `bRequiresImplementModule = false` was set wrongly) |
| `does not appear to be compatible` | Plugin binaries built against another engine version: recompile the plugin |
| `Module name mismatch` compile *warning* (`UE_STATIC_ASSERT_WARN`; error only with warnings-as-errors), plus a load failure in monolithic builds | Second argument of `IMPLEMENT_MODULE` differs from the `Build.cs` module name |
| Module loads too late | Move `LoadingPhase` to `PreDefault` or load explicitly with `FModuleManager::Get().LoadModule(...)` |

---

## Live Coding and Hot Reload

Live Coding (`bWithLiveCoding`, Windows editor and development targets) patches functions in place. Hot Reload (`WITH_HOT_RELOAD`, modular non-shipping builds) swaps a module DLL. Neither can safely change the layout of a live `UObject`:

- Adding, removing or reordering `UPROPERTY` members, changing a base class, or adding `UFUNCTION`s to a class that already has instances requires closing the editor and rebuilding.
- Function-body-only edits are what Live Coding is for.
- A crash on the next PIE or hot reload after a structural change means the process still holds the old layout: full restart.

---

## Packaging and Cooking Failures

### Editor-only module referenced from runtime code

```
Unable to instantiate module 'UnrealEd': Unable to instantiate UnrealEd module for non-editor targets.
(referenced via Target -> MyGame.Build.cs -> UnrealEd.Build.cs)
```

A runtime module depends on `UnrealEd` (or another editor-only module) outside `if (Target.bBuildEditor)`, so building the `Game`, `Client` or `Server` target fails (`Editor/UnrealEd/UnrealEd.Build.cs` throws when `!Target.bCompileAgainstEditor`). The same applies to code that reaches a `Type: Editor`, `EditorNoCommandlet`, `DeveloperTool` or `UncookedOnly` module from a cooked path.

- C++: wrap with `#if WITH_EDITOR` / `#if WITH_EDITORONLY_DATA`
- `Build.cs`: add editor dependencies only inside `if (Target.bBuildEditor)`
- `.uproject`/`.uplugin`: give the module the correct `Type`

### Blueprint references a stripped module

Blueprint graphs that call `UFUNCTION`s from an `Editor`-type module fail to load in cooked builds. Move the functions to a `Runtime` module, or put Blueprint-editor-only nodes (K2Node classes) in an `UncookedOnly` module so the editor compiles them but the cook excludes them.

---

## Quick Reference

| Error type | First thing to check |
|---|---|
| LNK2019 / LNK2001 | Missing dependency or missing `MYGAME_API` |
| C1083 | Missing dependency or path not relative to the module `Public/` root |
| C4996 | `UE_DEPRECATED` replacement named in the message |
| UHT `.generated.h` / `GENERATED_BODY()` | Last include; first statement in the body; `#pragma once` |
| Could not find definition for module | `Build.cs` class name and dependency spelling |
| Circular dependency | Extract interface module or `DynamicallyLoadedModuleNames` |
| Missing precompiled manifest | Installed engine; remove dependency or use a source build |
| Module not found at runtime | `.uproject`/`.uplugin` `Modules`, `ExtraModuleNames`, `LoadingPhase` |
| Cooking failure | Editor-only code or module on a runtime path |
| Live Coding crash | Structural `UObject` change; restart and rebuild |
