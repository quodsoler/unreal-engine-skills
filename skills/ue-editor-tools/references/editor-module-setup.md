# Editor Module Setup

Boilerplate and registration patterns for a UE 5.8 editor module. Everything that touches `UnrealEd`, `PropertyEditor`, `Blutility`, `ToolMenus`, `AssetTools` or `AssetDefinition` must live in a module whose descriptor says `"Type": "Editor"`, so it is stripped from packaged builds.

---

## Why an editor module is separate

Editor modules are built only for editor targets. `UnrealEd` and its dependants assert this in their own `Build.cs` (`UnrealEd.Build.cs:12-14` and `EditorStyle.Build.cs:13-15` throw a `BuildException` when `Target.bCompileAgainstEditor` is false). A Runtime module that unconditionally lists `UnrealEd` in `PublicDependencyModuleNames` or `PrivateDependencyModuleNames` therefore fails to build for Game, Client and Server targets, and the project will not package.

The rule is directional:

- Editor module → Runtime module: allowed and normal (`"MyGame"` in `PrivateDependencyModuleNames`).
- Runtime module → Editor module: never unconditionally. `#if WITH_EDITOR` guards code, not the Build.cs dependency; engine runtime modules that need editor code add it under `if (Target.bBuildEditor) { PrivateDependencyModuleNames.Add("UnrealEd"); }` (`AIModule.Build.cs:31-34`) and wrap the includes in `#if WITH_EDITOR`. Prefer a separate editor module.

Runtime classes expose editor behaviour through virtuals that already exist on `UObject`, guarded by `WITH_EDITOR`.

---

## Project configuration

```json
{
  "Modules": [
    { "Name": "MyGame",       "Type": "Runtime", "LoadingPhase": "Default" },
    { "Name": "MyGameEditor", "Type": "Editor",  "LoadingPhase": "PostEngineInit" }
  ]
}
```

The same two keys apply in a `.uplugin`.

| LoadingPhase | Use when |
|---|---|
| `Default` | Detail customizations, `UToolMenus::RegisterStartupCallback` menus, commands (engine `CommonUIEditor` registers its detail customizations at `Default`) |
| `PostEngineInit` | `StartupModule` needs `GEditor`, editor subsystems or other engine-initialised state directly |
| `PreDefault` | The module provides services other modules need before their own startup |

---

## Directory layout

```
MyProject/
  Source/
    MyGame/                          <- Runtime module
      MyGame.Build.cs
      Public/
      Private/
    MyGameEditor/                    <- Editor module
      MyGameEditor.Build.cs
      Public/
        MyGameEditor.h
        Customizations/
          MyDataAssetCustomization.h
          MyRangeCustomization.h
        Modes/
          MyEditorMode.h
        AssetTypes/
          MyAssetDefinition.h
          MyDataAssetFactory.h
      Private/
        MyGameEditorModule.cpp
        Customizations/
          MyDataAssetCustomization.cpp
          MyRangeCustomization.cpp
        Modes/
          MyEditorMode.cpp
        AssetTypes/
          MyAssetDefinition.cpp
          MyDataAssetFactory.cpp
```

---

## Build.cs

```csharp
// MyGameEditor.Build.cs
using UnrealBuildTool;

public class MyGameEditor : ModuleRules
{
    public MyGameEditor(ReadOnlyTargetRules Target) : base(Target)
    {
        PCHUsage = PCHUsageMode.UseExplicitOrSharedPCHs;

        PublicDependencyModuleNames.AddRange(new string[]
        {
            "Core",
            "CoreUObject",
            "Engine",
        });

        PrivateDependencyModuleNames.AddRange(new string[]
        {
            "UnrealEd",          // GEditor, UFactory, UEdMode, FScopedTransaction, toolkits
            "EditorFramework",   // FEditorModeInfo, IToolkit
            "EditorSubsystem",   // UEditorSubsystem
            "Slate",
            "SlateCore",         // FAppStyle, FSlateIcon
            "InputCore",         // FKey, FInputChord
            "PropertyEditor",    // IDetailCustomization, IPropertyTypeCustomization
            "Blutility",         // UEditorUtilityWidget, UAssetActionUtility
            "UMG", "UMGEditor",  // EditorUtilityWidget(Blueprint).h include UMG headers; Blutility links them privately
            "ToolMenus",         // UToolMenus
            "AssetTools",        // IAssetTools
            "AssetDefinition",   // UAssetDefinition
            "ContentBrowser",    // IContentBrowserSingleton (only if you drive the browser)
            "LevelEditor",       // ULevelEditorSubsystem, FLevelEditorModule
            "MyGame",            // your runtime module
        });
    }
}
```

`FAppStyle` lives in `SlateCore` (`Styling/AppStyle.h`), not in the `EditorStyle` module. Add `EditorInteractiveToolsFramework` and `InteractiveToolsFramework` only when the module builds `UInteractiveTool` classes for a `UEdMode`.

---

## Module header and implementation

```cpp
// MyGameEditor.h
#pragma once

#include "Modules/ModuleManager.h"

class FMyGameEditorModule : public IModuleInterface
{
public:
    //~ IModuleInterface
    virtual void StartupModule() override;
    virtual void ShutdownModule() override;

private:
    void RegisterDetailCustomizations();
    void UnregisterDetailCustomizations();
    void RegisterThumbnailRenderers();
    void UnregisterThumbnailRenderers();
    void RegisterMenus();

    static void OpenMyPanel();
};
```

```cpp
// MyGameEditorModule.cpp
#include "MyGameEditor.h"

#include "Customizations/MyDataAssetCustomization.h"
#include "Customizations/MyRangeCustomization.h"
#include "MyDataAsset.h"
#include "MyRange.h"
#include "MyThumbnailRenderer.h"

#include "PropertyEditorModule.h"
#include "ToolMenus.h"
#include "ToolMenuSection.h"
#include "ToolMenuEntry.h"
#include "ThumbnailRendering/ThumbnailManager.h"
#include "Styling/AppStyle.h"
#include "Editor.h"
#include "EditorUtilitySubsystem.h"
#include "EditorUtilityWidgetBlueprint.h"

void FMyGameEditorModule::StartupModule()
{
    RegisterDetailCustomizations();
    RegisterThumbnailRenderers();

    // Menus are not available until the editor has built them.
    UToolMenus::RegisterStartupCallback(
        FSimpleMulticastDelegate::FDelegate::CreateRaw(this, &FMyGameEditorModule::RegisterMenus));
}

void FMyGameEditorModule::ShutdownModule()
{
    UToolMenus::UnRegisterStartupCallback(this);
    UToolMenus::UnregisterOwner(this);

    UnregisterThumbnailRenderers();
    UnregisterDetailCustomizations();
}

void FMyGameEditorModule::RegisterDetailCustomizations()
{
    FPropertyEditorModule& PropertyModule =
        FModuleManager::LoadModuleChecked<FPropertyEditorModule>("PropertyEditor");

    PropertyModule.RegisterCustomClassLayout(
        UMyDataAsset::StaticClass()->GetFName(),
        FOnGetDetailCustomizationInstance::CreateStatic(&FMyDataAssetCustomization::MakeInstance),
        FRegisterCustomClassLayoutParams());

    PropertyModule.RegisterCustomPropertyTypeLayout(
        FMyRange::StaticStruct()->GetFName(),
        FOnGetPropertyTypeCustomizationInstance::CreateStatic(&FMyRangeCustomization::MakeInstance));

    PropertyModule.NotifyCustomizationModuleChanged();
}

void FMyGameEditorModule::UnregisterDetailCustomizations()
{
    if (!FModuleManager::Get().IsModuleLoaded("PropertyEditor"))
    {
        return;
    }

    FPropertyEditorModule& PropertyModule =
        FModuleManager::GetModuleChecked<FPropertyEditorModule>("PropertyEditor");
    PropertyModule.UnregisterCustomClassLayout(UMyDataAsset::StaticClass()->GetFName());
    PropertyModule.UnregisterCustomPropertyTypeLayout(FMyRange::StaticStruct()->GetFName());
    PropertyModule.NotifyCustomizationModuleChanged();
}

void FMyGameEditorModule::RegisterThumbnailRenderers()
{
    UThumbnailManager::Get().RegisterCustomRenderer(
        UMyDataAsset::StaticClass(), UMyThumbnailRenderer::StaticClass());
}

void FMyGameEditorModule::UnregisterThumbnailRenderers()
{
    if (UObjectInitialized())
    {
        UThumbnailManager::Get().UnregisterCustomRenderer(UMyDataAsset::StaticClass());
    }
}

void FMyGameEditorModule::OpenMyPanel()
{
    if (UEditorUtilityWidgetBlueprint* WidgetBP = LoadObject<UEditorUtilityWidgetBlueprint>(
            nullptr, TEXT("/Game/EditorWidgets/BP_MyTool.BP_MyTool")))
    {
        GEditor->GetEditorSubsystem<UEditorUtilitySubsystem>()->SpawnAndRegisterTab(WidgetBP);
    }
}

void FMyGameEditorModule::RegisterMenus()
{
    FToolMenuOwnerScoped OwnerScoped(this);

    UToolMenu* WindowMenu = UToolMenus::Get()->ExtendMenu("LevelEditor.MainMenu.Window");
    FToolMenuSection& Section = WindowMenu->FindOrAddSection("MyGame");
    Section.Label = FText::FromString("My Game");
    Section.AddMenuEntry(
        "OpenMyPanel",
        FText::FromString("My Tool Panel"),
        FText::FromString("Open the My Game tool panel"),
        FSlateIcon(FAppStyle::GetAppStyleSetName(), "Icons.Settings"),
        FUIAction(FExecuteAction::CreateStatic(&FMyGameEditorModule::OpenMyPanel)));
}

IMPLEMENT_MODULE(FMyGameEditorModule, MyGameEditor)
```

`IMPLEMENT_MODULE(FMyGameEditorModule, MyGameEditor)` goes last: the macro instantiates the class, so the declaration and both function definitions must already be visible. The second argument is the module name exactly as it appears in the descriptor and in `MyGameEditor.Build.cs`.

---

## Commands with TCommands

`TCommands` (`Framework/Commands/Commands.h`, module `Slate`) gives menu entries a shared `FUICommandInfo` with a keyboard chord and a tooltip.

```cpp
// MyCommands.h
#pragma once
#include "Framework/Commands/Commands.h"
#include "Styling/AppStyle.h"

class FMyCommands : public TCommands<FMyCommands>
{
public:
    FMyCommands()
        : TCommands<FMyCommands>("MyGameEditor", INVTEXT("My Game Editor"),
            NAME_None, FAppStyle::GetAppStyleSetName())
    {
    }

    virtual void RegisterCommands() override;

    TSharedPtr<FUICommandInfo> OpenPanel;
};
```

```cpp
// MyCommands.cpp
#include "MyCommands.h"

#define LOCTEXT_NAMESPACE "FMyCommands"

void FMyCommands::RegisterCommands()
{
    UI_COMMAND(OpenPanel, "My Panel", "Opens the My Game panel",
        EUserInterfaceActionType::Button, FInputChord());
}

#undef LOCTEXT_NAMESPACE
```

Call `FMyCommands::Register()` in `StartupModule` and `FMyCommands::Unregister()` in `ShutdownModule`. Bind with `CommandList->MapAction(FMyCommands::Get().OpenPanel, FExecuteAction::CreateStatic(&FMyGameEditorModule::OpenMyPanel))` and add the entry with the command overload:

```cpp
Section.AddMenuEntry(FMyCommands::Get().OpenPanel);
```

---

## WITH_EDITOR in a runtime class

```cpp
// MyRuntimeSettings.h
#pragma once
#include "UObject/Object.h"
#include "MyRuntimeSettings.generated.h"

UCLASS()
class MYGAME_API UMyRuntimeSettings : public UObject
{
    GENERATED_BODY()

public:
    UPROPERTY(EditAnywhere, Category = "Tuning")
    float Speed = 600.f;

    UPROPERTY(EditAnywhere, Category = "Tuning")
    float MaxSpeed = 1200.f;

    UPROPERTY(EditAnywhere, Category = "Tuning")
    bool bEnableAdvanced = false;

    UPROPERTY(EditAnywhere, Category = "Tuning", meta = (EditCondition = "bEnableAdvanced"))
    float AdvancedSetting = 1.f;

#if WITH_EDITOR
    virtual void PostEditChangeProperty(FPropertyChangedEvent& PropertyChangedEvent) override;
    virtual bool CanEditChange(const FProperty* InProperty) const override;
#endif
};
```

```cpp
// MyRuntimeSettings.cpp
#include "MyRuntimeSettings.h"

#if WITH_EDITOR
void UMyRuntimeSettings::PostEditChangeProperty(FPropertyChangedEvent& PropertyChangedEvent)
{
    Super::PostEditChangeProperty(PropertyChangedEvent);

    const FName PropertyName = PropertyChangedEvent.GetPropertyName();
    if (PropertyName == GET_MEMBER_NAME_CHECKED(UMyRuntimeSettings, Speed))
    {
        Speed = FMath::Clamp(Speed, 0.f, MaxSpeed);
    }
}

bool UMyRuntimeSettings::CanEditChange(const FProperty* InProperty) const
{
    if (!Super::CanEditChange(InProperty))
    {
        return false;
    }

    if (InProperty->GetFName() == GET_MEMBER_NAME_CHECKED(UMyRuntimeSettings, AdvancedSetting))
    {
        return bEnableAdvanced;
    }
    return true;
}
#endif
```

Signatures from `CoreUObject/Public/UObject/Object.h`:
`virtual void PostEditChangeProperty(struct FPropertyChangedEvent& PropertyChangedEvent)` and
`virtual bool CanEditChange(const FProperty* InProperty) const`. `FPropertyChangedEvent::GetPropertyName()` is in `UObject/UnrealType.h`.

For project settings pages, derive from `UDeveloperSettings` (`Engine/DeveloperSettings.h`, module `DeveloperSettings`) and override `GetContainerName()`, `GetCategoryName()` and `GetSectionName()` — that class is runtime-safe and needs no editor module.

---

## Asset definition, factory and thumbnail

```cpp
// MyAssetDefinition.h
#pragma once
#include "AssetDefinitionDefault.h"
#include "MyAssetDefinition.generated.h"

UCLASS()
class MYGAMEEDITOR_API UMyAssetDefinition : public UAssetDefinitionDefault
{
    GENERATED_BODY()

public:
    virtual FText GetAssetDisplayName() const override;
    virtual FLinearColor GetAssetColor() const override;
    virtual TSoftClassPtr<UObject> GetAssetClass() const override;
    virtual TConstArrayView<FAssetCategoryPath> GetAssetCategories() const override;
};
```

```cpp
// MyAssetDefinition.cpp
#include "MyAssetDefinition.h"
#include "MyDataAsset.h"

FText UMyAssetDefinition::GetAssetDisplayName() const
{
    return NSLOCTEXT("MyGameEditor", "MyDataAssetName", "My Data Asset");
}

FLinearColor UMyAssetDefinition::GetAssetColor() const
{
    return FLinearColor(FColor(200, 100, 50));
}

TSoftClassPtr<UObject> UMyAssetDefinition::GetAssetClass() const
{
    return UMyDataAsset::StaticClass();
}

TConstArrayView<FAssetCategoryPath> UMyAssetDefinition::GetAssetCategories() const
{
    static const auto Categories = { EAssetCategoryPaths::Gameplay };
    return Categories;
}
```

The definition is found from its CDO, so there is no registration call and nothing to unregister. `UAssetDefinitionDefault` (`UnrealEd/Public/AssetDefinitionDefault.h`) already implements
`virtual EAssetCommandResult OpenAssets(const FAssetOpenArgs& OpenArgs) const` and
`virtual EAssetCommandResult PerformAssetDiff(const FAssetDiffArgs& DiffArgs) const`; override `OpenAssets` only to open your own toolkit instead of the default one.

Context-menu entries for the type are `UToolMenus` extensions of `ContentBrowser.AssetContextMenu` — a dynamic section on the menu, not a virtual on the definition.

```cpp
// MyDataAssetFactory.h
#pragma once
#include "Factories/Factory.h"
#include "MyDataAssetFactory.generated.h"

UCLASS()
class MYGAMEEDITOR_API UMyDataAssetFactory : public UFactory
{
    GENERATED_BODY()

public:
    UMyDataAssetFactory();

    virtual UObject* FactoryCreateNew(UClass* InClass, UObject* InParent, FName InName,
        EObjectFlags Flags, UObject* Context, FFeedbackContext* Warn) override;
    virtual bool ShouldShowInNewMenu() const override;
};
```

```cpp
// MyDataAssetFactory.cpp
#include "MyDataAssetFactory.h"
#include "MyDataAsset.h"

UMyDataAssetFactory::UMyDataAssetFactory()
{
    SupportedClass = UMyDataAsset::StaticClass();
    bCreateNew = true;
    bEditAfterNew = true;
}

UObject* UMyDataAssetFactory::FactoryCreateNew(UClass* InClass, UObject* InParent, FName InName,
    EObjectFlags Flags, UObject* Context, FFeedbackContext* Warn)
{
    return NewObject<UMyDataAsset>(InParent, InClass, InName, Flags);
}

bool UMyDataAssetFactory::ShouldShowInNewMenu() const
{
    return true;
}
```

Factories are auto-discovered too. `UFactory` also offers `virtual bool ConfigureProperties()` for a pre-creation dialog and `virtual UClass* ResolveSupportedClass()` when one factory makes several types.

```cpp
// MyThumbnailRenderer.h
#pragma once
#include "ThumbnailRendering/ThumbnailRenderer.h"
#include "MyThumbnailRenderer.generated.h"

UCLASS()
class MYGAMEEDITOR_API UMyThumbnailRenderer : public UThumbnailRenderer
{
    GENERATED_BODY()

public:
    virtual void Draw(UObject* Object, int32 X, int32 Y, uint32 Width, uint32 Height,
        FRenderTarget* Viewport, FCanvas* Canvas, bool bAdditionalViewFamily) override;
};
```

Register it with `UThumbnailManager::Get().RegisterCustomRenderer(UMyDataAsset::StaticClass(), UMyThumbnailRenderer::StaticClass())` and pair it with `UnregisterCustomRenderer(UMyDataAsset::StaticClass())`, as in the module implementation above.

---

## Editor mode with a toolkit

`UEdMode` subclasses are registered automatically: `UAssetEditorSubsystem::RegisterEditorModes` iterates every `UEdMode` CDO once `FCoreDelegates::OnAllModuleLoadingPhasesComplete` fires. Fill `Info` in the constructor; never call `FEditorModeRegistry` for a `UEdMode`.

```cpp
// MyEditorMode.h
#pragma once
#include "Tools/UEdMode.h"
#include "MyEditorMode.generated.h"

UCLASS()
class MYGAMEEDITOR_API UMyEditorMode : public UEdMode
{
    GENERATED_BODY()

public:
    static const FEditorModeID EM_MyEditorMode;

    UMyEditorMode();

    virtual void Enter() override;
    virtual void Exit() override;
    virtual void ModeTick(float DeltaTime) override;
    virtual void CreateToolkit() override;
    virtual bool UsesToolkits() const override;
};
```

```cpp
// MyEditorMode.cpp
#include "MyEditorMode.h"

#include "Toolkits/BaseToolkit.h"
#include "Styling/AppStyle.h"

const FEditorModeID UMyEditorMode::EM_MyEditorMode = TEXT("EM_MyEditorMode");

UMyEditorMode::UMyEditorMode()
{
    Info = FEditorModeInfo(
        EM_MyEditorMode,
        FText::FromString("My Editor Mode"),
        FSlateIcon(FAppStyle::GetAppStyleSetName(), "LevelEditor.SelectMode"),
        /*InVisibility=*/true);
}

void UMyEditorMode::Enter()
{
    Super::Enter();
    // RegisterTool(FMyCommands::Get().MyTool, TEXT("MyTool"), NewObject<UMyToolBuilder>(this));
}

void UMyEditorMode::Exit()
{
    Super::Exit();
}

void UMyEditorMode::ModeTick(float DeltaTime)
{
}

void UMyEditorMode::CreateToolkit()
{
    Toolkit = MakeShared<FModeToolkit>();
}

bool UMyEditorMode::UsesToolkits() const
{
    return true;
}
```

`FEditorModeInfo` (`EditorFramework/Public/Tools/Modes.h`):

```cpp
FEditorModeInfo(FEditorModeID InID, FText InName = FText(), FSlateIcon InIconBrush = FSlateIcon(),
    TAttribute<bool> InVisibility = false, int32 InPriorityOrder = MAX_int32);
```

`UEdMode` members you will use (`UnrealEd/Public/Tools/UEdMode.h`):

```cpp
virtual void Initialize();
virtual void Enter();
virtual void Exit();
virtual void ModeTick(float DeltaTime);
virtual void RegisterTool(TSharedPtr<FUICommandInfo> UICommand, FString ToolIdentifier,
    UInteractiveToolBuilder* Builder, EToolsContextScope ToolScope = EToolsContextScope::Default);
virtual bool ShouldToolStartBeAllowed(const FString& ToolIdentifier) const;
virtual bool UsesToolkits() const;
virtual void CreateToolkit();
UEditorInteractiveToolsContext* GetInteractiveToolsContext(
    EToolsContextScope ToolScope = EToolsContextScope::Default) const;
FEditorModeTools* GetModeManager() const;
const FEditorModeInfo& GetModeInfo() const;
```

`EToolsContextScope` is `Editor`, `EdMode` or `Default`. The mode's own context object is a `UEdModeInteractiveToolsContext` held in the protected `ModeToolsContext` member.

Custom toolkit panel:

```cpp
class FMyEditorModeToolkit : public FModeToolkit
{
public:
    virtual void Init(const TSharedPtr<IToolkitHost>& InitToolkitHost,
        TWeakObjectPtr<UEdMode> InOwningMode) override;
    virtual FName GetToolkitFName() const override;
    virtual FText GetBaseToolkitName() const override;
    virtual TSharedPtr<SWidget> GetInlineContent() const override;
    virtual void GetToolPaletteNames(TArray<FName>& PaletteNames) const override;
};
```

Viewport input: implement `ILegacyEdModeViewportInterface` (`UnrealEd/Public/Tools/LegacyEdModeInterfaces.h`) on the mode for `MouseMove(FEditorViewportClient*, FViewport*, int32 x, int32 y)`, `InputKey(FEditorViewportClient*, FViewport*, FKey, EInputEvent)`, `HandleClick(FEditorViewportClient*, HHitProxy*, const FViewportClick&)`, `InputDelta(FEditorViewportClient*, FViewport*, FVector&, FRotator&, FVector&)` and `Tick(FEditorViewportClient*, float)`. `Render(const FSceneView*, FViewport*, FPrimitiveDrawInterface*)` and `DrawHUD(FEditorViewportClient*, FViewport*, const FSceneView*, FCanvas*)` are on `ILegacyEdModeWidgetInterface`; `UBaseLegacyWidgetEdMode` (`Tools/LegacyEdModeWidgetHelpers.h`) implements both interfaces and forwards them to the `FLegacyEdModeWidgetHelper` your `CreateWidgetHelper()` returns.

Activate and query through `FEditorModeTools` (`UnrealEd/Public/EditorModeManager.h`), reached with `GLevelEditorModeTools()`:

```cpp
GLevelEditorModeTools().ActivateMode(UMyEditorMode::EM_MyEditorMode, /*bToggle=*/false);
const bool bActive = GLevelEditorModeTools().IsModeActive(UMyEditorMode::EM_MyEditorMode);
UEdMode* Mode = GLevelEditorModeTools().GetActiveScriptableMode(UMyEditorMode::EM_MyEditorMode);
GLevelEditorModeTools().DeactivateMode(UMyEditorMode::EM_MyEditorMode);
```

`FEdMode` (`UnrealEd/Public/EdMode.h`) is the legacy shared-pointer mode. It derives from `FLegacyEdModeWidgetHelper` and is the only mode kind that `FEditorModeRegistry::Get().RegisterMode<T>(FEditorModeID, FText Name, FSlateIcon IconBrush, bool bVisible, int32 PriorityOrder)` / `UnregisterMode(FEditorModeID)` handles. Write new modes as `UEdMode`.

---

## Asset editor toolkit

```cpp
// MyAssetEditorToolkit.h
#pragma once
#include "Toolkits/AssetEditorToolkit.h"

class UMyDataAsset;

class FMyAssetEditorToolkit : public FAssetEditorToolkit
{
public:
    void InitMyEditor(const EToolkitMode::Type Mode,
        const TSharedPtr<IToolkitHost>& InitToolkitHost, UMyDataAsset* Asset);

    //~ FAssetEditorToolkit
    virtual FName GetToolkitFName() const override;
    virtual FText GetBaseToolkitName() const override;
    virtual FString GetWorldCentricTabPrefix() const override;
    virtual void RegisterTabSpawners(const TSharedRef<FTabManager>& InTabManager) override;
    virtual void UnregisterTabSpawners(const TSharedRef<FTabManager>& InTabManager) override;
};
```

```cpp
// MyAssetEditorToolkit.cpp
#include "MyAssetEditorToolkit.h"
#include "MyDataAsset.h"
#include "Framework/Docking/TabManager.h"

FName FMyAssetEditorToolkit::GetToolkitFName() const
{
    return FName("MyAssetEditor");
}

FText FMyAssetEditorToolkit::GetBaseToolkitName() const
{
    return INVTEXT("My Asset Editor");
}

FString FMyAssetEditorToolkit::GetWorldCentricTabPrefix() const
{
    return TEXT("MyAsset ");
}

void FMyAssetEditorToolkit::InitMyEditor(const EToolkitMode::Type Mode,
    const TSharedPtr<IToolkitHost>& InitToolkitHost, UMyDataAsset* Asset)
{
    const TSharedRef<FTabManager::FLayout> Layout =
        FTabManager::NewLayout("Standalone_MyAssetEditor_Layout_v1")
        ->AddArea
        (
            FTabManager::NewPrimaryArea()
            ->SetOrientation(Orient_Vertical)
            ->Split
            (
                FTabManager::NewStack()
                ->SetSizeCoefficient(1.f)
                ->AddTab("MyAssetDetailsTab", ETabState::OpenedTab)
            )
        );

    InitAssetEditor(Mode, InitToolkitHost, FName("MyAssetEditorApp"), Layout,
        /*bCreateDefaultStandaloneMenu=*/true, /*bCreateDefaultToolbar=*/true, Asset);
}
```

`EToolkitMode::Type` is `Standalone` or `WorldCentric` (`EditorFramework/Public/Toolkits/IToolkit.h`). `RegisterTabSpawners` / `UnregisterTabSpawners` must call `FAssetEditorToolkit::RegisterTabSpawners(InTabManager)` / `UnregisterTabSpawners(InTabManager)` before adding or removing your own spawners.

---

## Checklist: new editor module

- [ ] `"Type": "Editor"` and a `LoadingPhase` (`Default`, or `PostEngineInit` if startup touches `GEditor`) in the `.uproject` or `.uplugin`
- [ ] `Build.cs` lists `UnrealEd`, `Slate`, `SlateCore`, `PropertyEditor`, `ToolMenus`, `EditorSubsystem`, `Blutility`, `AssetTools`, `AssetDefinition` as needed
- [ ] Runtime module has no dependency on any editor module
- [ ] `FMyGameEditorModule : public IModuleInterface` declared, `StartupModule` and `ShutdownModule` defined, `IMPLEMENT_MODULE(FMyGameEditorModule, MyGameEditor)` last in the `.cpp`
- [ ] `RegisterCustomClassLayout` / `RegisterCustomPropertyTypeLayout` paired with their `Unregister*` calls, guarded by `IsModuleLoaded("PropertyEditor")`
- [ ] `NotifyCustomizationModuleChanged()` after registering and after unregistering
- [ ] Menus built inside `UToolMenus::RegisterStartupCallback`, under a `FToolMenuOwnerScoped`
- [ ] `UToolMenus::UnRegisterStartupCallback(this)` and `UToolMenus::UnregisterOwner(this)` in `ShutdownModule`
- [ ] `RegisterCustomRenderer` paired with `UnregisterCustomRenderer`
- [ ] `UEdMode`, `UAssetDefinition`, `UFactory` and `UEditorValidatorBase` subclasses left unregistered — they are discovered from their CDOs
- [ ] No editor header included from a runtime module header
