---
name: ue-editor-tools
description: "Use when extending the Unreal Editor from C++ with detail customizations, editor utility widgets, menus, editor modes, asset definitions or editor subsystems. Also use when the user mentions 'IDetailCustomization', 'IPropertyTypeCustomization', 'CustomizeDetails', 'RegisterCustomClassLayout', 'IDetailLayoutBuilder', 'AddCustomRow', 'IPropertyHandle', 'UToolMenus', 'ExtendMenu', 'FToolMenuEntry', 'UEditorUtilityWidget', 'Blutility', 'UAssetActionUtility', 'UEditorSubsystem', 'UEdMode', 'UAssetDefinition', 'UFactory', 'FScopedTransaction', or 'editor module'. For Slate fundamentals, see ue-ui-umg-slate; for Build.cs wiring, see ue-module-build-system."
metadata:
  version: "2.0.0"
  engine: "5.8"
---

# UE Editor Tools

Target engine: **UE 5.8**. APIs below are verified against the 5.8 headers; older forms are listed under "Deprecated — do not use".

Editor extensions live in a module with `"Type": "Editor"` and depend on `UnrealEd`, `Slate`, `SlateCore`, `PropertyEditor` (detail panels), `ToolMenus` (menus and toolbars), `EditorSubsystem` (`UEditorSubsystem`), `Blutility` (editor utility widgets and scripted actions), `AssetTools` and `AssetDefinition` (Content Browser asset types). None of these may be referenced unconditionally from a Runtime module (engine runtime modules add them only under `if (Target.bBuildEditor)` plus `#if WITH_EDITOR`, e.g. `AIModule.Build.cs:31-34`).

## Context

Read `.agents/ue-project-context.md` if it exists (module names, conventions, enabled plugins, GAS/networking setup). Do not stop if it is missing.

Identify the area from the request and the codebase. Ask only when two plausible readings would produce different code.

| Request is about… | Go to |
|---|---|
| Creating the editor module, Build.cs, registration lifecycle | [Editor Module Setup](#editor-module-setup) |
| Custom property panel for a class or struct | [Detail Customizations](#detail-customizations) |
| UMG panel run inside the editor, right-click asset/actor actions | [Editor Utility Widgets and Scripted Actions](#editor-utility-widgets-and-scripted-actions) |
| Main menu, toolbar, context-menu entries, commands | [Menus and Toolbars](#menus-and-toolbars) |
| Editor-lifetime singleton, `GEditor->GetEditorSubsystem<T>()` | [Editor Subsystems](#editor-subsystems) |
| Undo/redo, marking objects dirty, reacting to editor events | [Transactions and Editor Delegates](#transactions-and-editor-delegates) |
| Viewport mode with custom interaction or tools | [Editor Modes](#editor-modes) |
| Content Browser type colour, category, factory, thumbnail | [Asset Types Factories and Thumbnails](#asset-types-factories-and-thumbnails) |
| Tabbed asset editor window | [Asset Editor Toolkits](#asset-editor-toolkits) |
| Validating assets on save or in CI | [Data Validation](#data-validation) |

## Editor Module Setup

`.uproject` (or `.uplugin`) module entry — `Default` is enough for detail customizations and `UToolMenus::RegisterStartupCallback` menus (`CommonUIEditor` registers its detail customizations at `Default`); use `PostEngineInit` when `StartupModule` itself needs `GEditor` or other engine-initialised state:

```json
{ "Name": "MyGameEditor", "Type": "Editor", "LoadingPhase": "PostEngineInit" }
```

```csharp
// MyGameEditor.Build.cs
PrivateDependencyModuleNames.AddRange(new string[] {
    "Core", "CoreUObject", "Engine", "UnrealEd",
    "Slate", "SlateCore", "InputCore",
    "PropertyEditor",   // IDetailCustomization, IPropertyTypeCustomization
    "EditorSubsystem",  // UEditorSubsystem
    "Blutility",        // UEditorUtilityWidget, UAssetActionUtility
    "UMG", "UMGEditor",  // EditorUtilityWidget(Blueprint).h include UMG headers; Blutility links them privately
    "ToolMenus",        // UToolMenus
    "AssetTools",       // IAssetTools
    "AssetDefinition",  // UAssetDefinition
    "MyGame"            // your runtime module
});
```

Declare the module class, define both lifecycle functions, then place `IMPLEMENT_MODULE` last:

```cpp
// MyGameEditor.h
#pragma once
#include "Modules/ModuleManager.h"

class FMyGameEditorModule : public IModuleInterface
{
public:
    virtual void StartupModule() override;
    virtual void ShutdownModule() override;

private:
    void RegisterMenus();
    static void OpenMyPanel();
};

// MyGameEditorModule.cpp — includes MyDataAsset.h, MyRange.h, the two customization
// headers, PropertyEditorModule.h and ToolMenus.h
void FMyGameEditorModule::StartupModule()
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

    UToolMenus::RegisterStartupCallback(
        FSimpleMulticastDelegate::FDelegate::CreateRaw(this, &FMyGameEditorModule::RegisterMenus));
}

void FMyGameEditorModule::ShutdownModule()
{
    UToolMenus::UnRegisterStartupCallback(this);
    UToolMenus::UnregisterOwner(this);

    if (FModuleManager::Get().IsModuleLoaded("PropertyEditor"))
    {
        FPropertyEditorModule& PropertyModule =
            FModuleManager::GetModuleChecked<FPropertyEditorModule>("PropertyEditor");
        PropertyModule.UnregisterCustomClassLayout(UMyDataAsset::StaticClass()->GetFName());
        PropertyModule.UnregisterCustomPropertyTypeLayout(FMyRange::StaticStruct()->GetFName());
        PropertyModule.NotifyCustomizationModuleChanged();
    }
}

IMPLEMENT_MODULE(FMyGameEditorModule, MyGameEditor)
```

`RegisterMenus` and `OpenMyPanel` are defined under [Menus and Toolbars](#menus-and-toolbars). `FRegisterCustomClassLayoutParams` carries `TOptional<int32> OptionalOrder`; set it when two modules customize the same class and the order matters, otherwise pass a default-constructed struct.

Directory layout, `WITH_EDITOR` guards, factories, thumbnails, toolkits and the full mode boilerplate: [editor-module-setup.md](references/editor-module-setup.md).

## Detail Customizations

`IDetailCustomization` replaces the whole details panel layout for a `UClass`. `IPropertyTypeCustomization` replaces the row for a `USTRUCT` or instanced object property. Both are registered from `StartupModule` as shown above.

```cpp
// MyDataAssetCustomization.h
#pragma once
#include "IDetailCustomization.h"
#include "Templates/SharedPointer.h"

class FMyDataAssetCustomization : public IDetailCustomization
{
public:
    static TSharedRef<IDetailCustomization> MakeInstance();

    virtual void CustomizeDetails(IDetailLayoutBuilder& DetailBuilder) override;
    virtual void CustomizeDetails(const TSharedPtr<IDetailLayoutBuilder>& DetailBuilder) override;

private:
    TWeakPtr<IDetailLayoutBuilder> WeakBuilder;
};

// MyDataAssetCustomization.cpp
void FMyDataAssetCustomization::CustomizeDetails(const TSharedPtr<IDetailLayoutBuilder>& DetailBuilder)
{
    WeakBuilder = DetailBuilder;   // store it before calling ForceRefreshDetails() later
    CustomizeDetails(*DetailBuilder);
}
```

The single-argument override does the layout work: `EditCategory`, `GetProperty`, `AddProperty`, `AddCustomRow`, `HideProperty`, `GetObjectsBeingCustomized` — full bodies in the reference below.

The `USTRUCT` counterpart, `FMyRangeCustomization : public IPropertyTypeCustomization`, overrides `CustomizeHeader(TSharedRef<IPropertyHandle> PropertyHandle, FDetailWidgetRow& HeaderRow, IPropertyTypeCustomizationUtils& CustomizationUtils)` and `CustomizeChildren(TSharedRef<IPropertyHandle> PropertyHandle, IDetailChildrenBuilder& ChildBuilder, IPropertyTypeCustomizationUtils& CustomizationUtils)`.

Key interfaces (all in `Engine/Source/Editor/PropertyEditor/Public`):

| Type | Header | What you call |
|---|---|---|
| `IDetailLayoutBuilder` | `DetailLayoutBuilder.h` | `EditCategory`, `HideCategory`, `GetProperty`, `HideProperty`, `AddPropertyToCategory`, `GetObjectsBeingCustomized`, `GetSelectedObjects`, `ForceRefreshDetails`, `RegisterInstancedCustomPropertyTypeLayout` |
| `IDetailCategoryBuilder` | `DetailCategoryBuilder.h` | `AddProperty`, `AddCustomRow`, `AddCustomBuilder`, `AddGroup`, `AddExternalObjects`, `SetSortOrder`, `InitiallyCollapsed` (`IDetailChildrenBuilder` in `IDetailChildrenBuilder.h` offers the same five adders, plus `AddExternalStructure`, inside `CustomizeChildren`) |
| `FDetailWidgetRow` | `DetailWidgetRow.h` | `NameContent()`, `ValueContent()`, `WholeRowContent()`, `ExtensionContent()`, `FilterString`, `Visibility`, `IsEnabled` |
| `IPropertyHandle` | `PropertyHandle.h` | `GetValue`/`SetValue` overloads, `GetNumChildren`, `GetChildHandle`, `SetOnPropertyValueChanged`, `SetOnChildPropertyValueChanged`, `CreatePropertyNameWidget`, `CreatePropertyValueWidget`, `EnumerateRawData`, `NotifyPreChange`, `NotifyPostChange` |
| `IDetailPropertyRow` | `IDetailPropertyRow.h` | `DisplayName`, `ToolTip`, `EditCondition`, `IsEnabled`, `Visibility`, `GetDefaultWidgets`, `CustomWidget` |
| `PropertyCustomizationHelpers` | `PropertyCustomizationHelpers.h` | `MakeAddButton`, `MakeRemoveButton`, `MakeClearButton`, `MakeBrowseButton`, `MakeUseSelectedButton`, `MakePropertyComboBox`, `SObjectPropertyEntryBox`, `SClassPropertyEntryBox` |

`GetValue`/`SetValue` return `FPropertyAccess::Result` (`Success`, `Fail`, `MultipleValues`). `NotifyPostChange` takes `EPropertyChangeType::Type`, e.g. `EPropertyChangeType::ValueSet`. `ECategoryPriority::Type` sorts top-down: `Variable`, `Transform`, `Important`, `TypeSpecific`, `Default`, `Uncommon`.

Full row patterns, `CustomizeHeader`/`CustomizeChildren` bodies, `SObjectPropertyEntryBox`, per-object value enumeration and fonts: [detail-customization-patterns.md](references/detail-customization-patterns.md).

## Editor Utility Widgets and Scripted Actions

`UEditorUtilityWidget` (`EditorUtilityWidget.h`, module `Blutility`) is a `UUserWidget` that runs in the editor. Subclass in C++ to expose helper functions, then build the visuals in a Blueprint child.

```cpp
// MyEditorUtilityWidget.h
#pragma once
#include "EditorUtilityWidget.h"
#include "MyEditorUtilityWidget.generated.h"

UCLASS()
class MYGAMEEDITOR_API UMyEditorUtilityWidget : public UEditorUtilityWidget
{
    GENERATED_BODY()

public:
    UFUNCTION(BlueprintCallable, Category = "My Tools")
    void PrefixSelectedAssets(const FString& Prefix);
};

// MyEditorUtilityWidget.cpp
void UMyEditorUtilityWidget::PrefixSelectedAssets(const FString& Prefix)
{
    UEditorAssetSubsystem* AssetSubsystem = GEditor->GetEditorSubsystem<UEditorAssetSubsystem>();
    for (UObject* Asset : UEditorUtilityLibrary::GetSelectedAssets())
    {
        const FString Source = Asset->GetOutermost()->GetName();
        AssetSubsystem->RenameAsset(
            Source, FPaths::GetPath(Source) / (Prefix + TEXT("_") + Asset->GetName()));
    }
}
```

`UEditorUtilityLibrary` (`EditorUtilityLibrary.h`, `Blutility`) also provides `GetSelectedAssetData()`, `GetSelectedAssetsOfClass(UClass*)`, `GetSelectionSet()` (level actors) and `SyncBrowserToFolders(const TArray<FString>&)`.

Open a widget Blueprint from code through `UEditorUtilitySubsystem` (`EditorUtilitySubsystem.h`):

| `UEditorUtilitySubsystem` call | Effect |
|---|---|
| `SpawnAndRegisterTab(UEditorUtilityWidgetBlueprint*)` | Registers the tab and opens it; returns the `UEditorUtilityWidget*` |
| `SpawnAndRegisterTabAndGetID(UEditorUtilityWidgetBlueprint*, FName& OutTabID)` / `SpawnAndRegisterTabWithId(UEditorUtilityWidgetBlueprint*, FName InTabID)` | Same, writing back or supplying the tab id |
| `RegisterTabAndGetID(UEditorUtilityWidgetBlueprint*, FName& OutTabID)` | Registers the spawner without opening the tab |
| `TryRun(UObject* Asset)` | Runs a `UEditorUtilityObject`-derived asset (invokes its `Run`) |

Scripted actions: any `UFUNCTION(CallInEditor)` on a `UAssetActionUtility` subclass appears in the Content Browser context menu; `UActorActionUtility` does the same for level actors. Both derive from `UEditorUtilityObject`. Filter with the `SupportedClasses` array (`TArray<TSoftClassPtr<UObject>>`, editable in Class Defaults); read it back with `GetSupportedClasses()`.

```cpp
// MyAssetActionUtility.h
#pragma once
#include "AssetActionUtility.h"
#include "MyAssetActionUtility.generated.h"

UCLASS()
class MYGAMEEDITOR_API UMyAssetActionUtility : public UAssetActionUtility
{
    GENERATED_BODY()

public:
    UFUNCTION(CallInEditor, Category = "My Tools")
    void SetTextureCompressionToUI();
};

// MyAssetActionUtility.cpp
void UMyAssetActionUtility::SetTextureCompressionToUI()
{
    for (UObject* Asset : UEditorUtilityLibrary::GetSelectedAssets())
    {
        if (UTexture2D* Texture = Cast<UTexture2D>(Asset))
        {
            Texture->CompressionSettings = TC_EditorIcon;
            Texture->MarkPackageDirty();
            Texture->PostEditChange();
        }
    }
}
```

The editor builds these menus without loading the utility asset, from asset-registry tags wrapped by `FAssetActionUtilityPrototype` (`EditorUtilityAssetPrototype.h`). Keep the filter in `SupportedClasses` on the CDO rather than computing it at run time.

## Menus and Toolbars

`UToolMenus` (module `ToolMenus`) owns every editor menu and toolbar. Register from the startup callback so Slate and the menus you extend exist.

```cpp
// MyGameEditorModule.cpp (continued) — ToolMenuSection.h, ToolMenuEntry.h,
// Styling/AppStyle.h, EditorUtilitySubsystem.h, EditorUtilityWidgetBlueprint.h
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
        FText::FromString("My Panel"),
        FText::FromString("Open the My Game tool panel"),
        FSlateIcon(FAppStyle::GetAppStyleSetName(), "Icons.Settings"),
        FUIAction(FExecuteAction::CreateStatic(&FMyGameEditorModule::OpenMyPanel)));

    UToolMenu* PlayToolBar =
        UToolMenus::Get()->ExtendMenu("LevelEditor.LevelEditorToolBar.PlayToolBar");
    PlayToolBar->FindOrAddSection("MyTools").AddEntry(
        FToolMenuEntry::InitToolBarButton(
            "MyToolButton",
            FUIAction(FExecuteAction::CreateStatic(&FMyGameEditorModule::OpenMyPanel)),
            FText::FromString("My Tool"),
            FText::FromString("Run the My Game tool"),
            FSlateIcon(FAppStyle::GetAppStyleSetName(), "Icons.Toolbar.Settings")));
}
```

| Call | Use |
|---|---|
| `UToolMenus::Get()` / `UToolMenus::TryGet()` | Singleton; `TryGet` does not force the module to load |
| `RegisterMenu(FName Name, FName Parent = NAME_None, EMultiBoxType Type = EMultiBoxType::Menu, bool bWarnIfAlreadyRegistered = true)` | Declare a menu you own |
| `ExtendMenu(FName)` | Add to a menu someone else owns; queues an extension if it is not registered yet |
| `FindMenu(FName)` | Look up a registered menu; `nullptr` if absent |
| `UToolMenu::FindOrAddSection(FName)` / `AddSection(FName, Label, FToolMenuInsert)` | Get or create a section |
| `FToolMenuSection::AddMenuEntry(FName, Label, ToolTip, FSlateIcon, FToolUIActionChoice, EUserInterfaceActionType, FName TutorialHighlight, TOptional<FText> InputBindingOverride)` | Menu item |
| `FToolMenuSection::AddSubMenu` / `AddSeparator` / `AddEntry` | Sub-menus, separators, prebuilt entries |
| `FToolMenuEntry::InitToolBarButton` / `InitComboButton` / `InitWidget` / `InitMenuEntry` | Build entries to pass to `AddEntry` |
| `UToolMenus::RegisterStartupCallback(const FSimpleMulticastDelegate::FDelegate&)` | Defer registration until menus exist |
| `UToolMenus::UnRegisterStartupCallback(this)` + `UToolMenus::UnregisterOwner(this)` | Mandatory in `ShutdownModule` |

`FToolMenuOwnerScoped` tags everything created in its scope with the owner so `UnregisterOwner` removes it all. Useful menu names: `LevelEditor.MainMenu.Window`, `LevelEditor.MainMenu.Tools`, `LevelEditor.LevelEditorToolBar.PlayToolBar`, `LevelEditor.LevelEditorToolBar.User`, `LevelEditor.ActorContextMenu`, `ContentBrowser.AssetContextMenu`. Icons are `FSlateIcon(FAppStyle::GetAppStyleSetName(), "StyleName")`.

Keyboard chords and reusable commands use `TCommands` with `UI_COMMAND`; see [editor-module-setup.md](references/editor-module-setup.md).

## Editor Subsystems

`UEditorSubsystem` (module `EditorSubsystem`, header `EditorSubsystem.h`) is auto-instantiated once the owning module is loaded — there is no registration call. Reach it with `GEditor->GetEditorSubsystem<T>()`.

```cpp
// MyEditorSubsystem.h
#pragma once
#include "EditorSubsystem.h"
#include "MyEditorSubsystem.generated.h"

UCLASS()
class MYGAMEEDITOR_API UMyEditorSubsystem : public UEditorSubsystem
{
    GENERATED_BODY()

public:
    virtual void Initialize(FSubsystemCollectionBase& Collection) override;
    virtual void Deinitialize() override;

private:
    void OnMapOpened(const FString& Filename, bool bAsTemplate);
};

// MyEditorSubsystem.cpp
void UMyEditorSubsystem::Initialize(FSubsystemCollectionBase& Collection)
{
    Super::Initialize(Collection);
    FEditorDelegates::OnMapOpened.AddUObject(this, &UMyEditorSubsystem::OnMapOpened);
}

void UMyEditorSubsystem::Deinitialize()
{
    FEditorDelegates::OnMapOpened.RemoveAll(this);
    Super::Deinitialize();
}

void UMyEditorSubsystem::OnMapOpened(const FString& Filename, bool bAsTemplate)
{
    UE_LOG(LogMyGame, Display, TEXT("Opened map %s"), *Filename);
}
```

`UEditorSubsystem` derives from `UDynamicSubsystem`; `Initialize(FSubsystemCollectionBase&)`, `Deinitialize()` and `ShouldCreateSubsystem(UObject* Outer) const` come from `USubsystem`, so always chain to `Super::`. Engine editor subsystems worth calling before writing your own:

| Subsystem | Module | Use |
|---|---|---|
| `UEditorActorSubsystem` | `UnrealEd` | `GetAllLevelActors`, `GetSelectedLevelActors`, `SetSelectedLevelActors`, `SpawnActorFromClass`, `DestroyActor` |
| `UEditorAssetSubsystem` | `UnrealEd` | `LoadAsset`, `RenameAsset`, `DuplicateAsset`, `DeleteAsset`, `SaveLoadedAsset`, `ListAssets` |
| `UAssetEditorSubsystem` | `UnrealEd` | `OpenEditorForAsset`, `FindEditorForAsset`, `CloseAllEditorsForAsset`, `GetEditorModeInfoOrderedByPriority` |
| `UEditorUtilitySubsystem` | `Blutility` | `SpawnAndRegisterTab`, `RegisterTabAndGetID`, `TryRun` |
| `ULevelEditorSubsystem` | `LevelEditor` | Level open/save/new operations |

The canonical subsystem type table (game instance, world, local player, engine, editor) lives in `ue-cpp-foundations`.

## Transactions and Editor Delegates

Every editor mutation of a `UObject` must be wrapped in a transaction and preceded by `Modify()`, or undo restores stale data:

```cpp
#include "ScopedTransaction.h"
#include "Subsystems/EditorActorSubsystem.h"

void UMyEditorSubsystem::OffsetSelection(const FVector& Delta)
{
    const FScopedTransaction Transaction(
        NSLOCTEXT("MyGameEditor", "OffsetSelection", "Offset Selection"));
    UEditorActorSubsystem* ActorSubsystem = GEditor->GetEditorSubsystem<UEditorActorSubsystem>();
    for (AActor* Actor : ActorSubsystem->GetSelectedLevelActors())
    {
        Actor->Modify();
        Actor->SetActorLocation(Actor->GetActorLocation() + Delta);
    }
}
```

`FScopedTransaction` (`ScopedTransaction.h`, `UnrealEd`) also offers `Cancel()` and `IsOutstanding()`. After a direct property write on an asset call `MarkPackageDirty()` and `PostEditChange()`.

| Delegate | Signature |
|---|---|
| `FEditorDelegates::PostUndoRedo` | `FSimpleMulticastDelegate` |
| `FEditorDelegates::OnMapOpened` | `(const FString& Filename, bool bAsTemplate)` |
| `FEditorDelegates::PreBeginPIE` / `PostPIEStarted` / `EndPIE` | `(const bool bIsSimulating)` |
| `FEditorDelegates::OnAssetPostImport` | `(UFactory*, UObject*)` |
| `FEditorDelegates::OnNewActorsPlaced` | `(UObject*, const TArray<AActor*>&)` |
| `FEditorDelegates::OnPackageDeleted` / `OnAssetsPreDelete` | `(UPackage*)` / `(const TArray<UObject*>&)` |
| `FCoreUObjectDelegates::OnObjectPropertyChanged` / `OnPreObjectPropertyChanged` | `(UObject*, FPropertyChangedEvent&)` / `(UObject*, const FEditPropertyChain&)` |
| `FCoreUObjectDelegates::OnObjectModified` | `(UObject*)` |

## Editor Modes

`UEdMode` (`Tools/UEdMode.h`, `UnrealEd`) is the mode base class. Every `UEdMode` subclass is discovered automatically by `UAssetEditorSubsystem::RegisterEditorModes` once all module loading phases complete — fill `Info` in the constructor and do not call a registry.

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

- Set `Info = FEditorModeInfo(EM_MyEditorMode, DisplayName, FSlateIcon(...), /*InVisibility=*/true)` in the constructor; `Info` is a protected `UEdMode` member. `Enter()` and `Exit()` must call `Super::`.
- Interactive tools: from `Enter()`, call `RegisterTool(TSharedPtr<FUICommandInfo> UICommand, FString ToolIdentifier, UInteractiveToolBuilder* Builder, EToolsContextScope ToolScope = EToolsContextScope::Default)`. `GetInteractiveToolsContext(EToolsContextScope)` returns the `UEditorInteractiveToolsContext`; the mode's own context object is a `UEdModeInteractiveToolsContext`.
- Viewport input (`MouseMove`, `InputKey`, `HandleClick`, `InputDelta`, `Tick`) comes from implementing `ILegacyEdModeViewportInterface` (`Tools/LegacyEdModeInterfaces.h`). `Render` and `DrawHUD` come from `ILegacyEdModeWidgetInterface`; `UBaseLegacyWidgetEdMode` implements both and forwards to an `FLegacyEdModeWidgetHelper` returned by your `CreateWidgetHelper()`.
- Activate with `GLevelEditorModeTools().ActivateMode(UMyEditorMode::EM_MyEditorMode)`; query with `IsModeActive` / `GetActiveScriptableMode`; leave with `DeactivateMode`.
- `FEdMode` (`EdMode.h`) is the legacy shared-pointer mode; it derives from `FLegacyEdModeWidgetHelper` and is registered through `FEditorModeRegistry::Get().RegisterMode<T>(...)` / `UnregisterMode(...)`. Write new modes as `UEdMode`.

Full mode implementation with a toolkit: [editor-module-setup.md](references/editor-module-setup.md).

## Asset Types Factories and Thumbnails

`UAssetDefinition` (module `AssetDefinition`, header `AssetDefinition.h`) is the Content Browser integration point in 5.8. Derive from `UAssetDefinitionDefault` (`AssetDefinitionDefault.h`, `UnrealEd`) so `OpenAssets` already opens a generic `FSimpleAssetEditor` (`AssetDefinitionDefault.cpp:16-25`). Definitions are discovered from their CDOs — no registration call.

Override `FText GetAssetDisplayName() const`, `FLinearColor GetAssetColor() const`, `TSoftClassPtr<UObject> GetAssetClass() const` and `TConstArrayView<FAssetCategoryPath> GetAssetCategories() const`; override `EAssetCommandResult OpenAssets(const FAssetOpenArgs& OpenArgs) const` only when the default toolkit routing is not what you want.

`EAssetCategoryPaths` supplies `Basic`, `Animation`, `Audio`, `Blueprint`, `Cinematics`, `Data`, `Foliage`, `FX`, `Gameplay`, `AI` and more; nest with `FAssetCategoryPath(EAssetCategoryPaths::Gameplay, INVTEXT("My Game"))`. Context-menu entries for the type are added through `UToolMenus` extensions of `ContentBrowser.AssetContextMenu`, not through the definition class.

`FAssetTypeActions_Base` (`AssetTypeActions_Base.h`, module `AssetTools`) with `IAssetTools::RegisterAssetTypeActions` / `UnregisterAssetTypeActions` still compiles, but `IAssetTypeActions` members are being deprecated in favour of the definition system. Use `UAssetDefinition` for new types.

- **Factory**: a `UFactory` subclass in the editor module sets `SupportedClass`, `bCreateNew`, `bEditAfterNew` in its constructor and overrides `FactoryCreateNew(UClass*, UObject*, FName, EObjectFlags, UObject*, FFeedbackContext*)`. It is auto-discovered; no registration.
- **Thumbnail**: subclass `UThumbnailRenderer`, override `Draw(UObject*, int32 X, int32 Y, uint32 Width, uint32 Height, FRenderTarget*, FCanvas*, bool bAdditionalViewFamily)`, then `UThumbnailManager::Get().RegisterCustomRenderer(UMyDataAsset::StaticClass(), UMyThumbnailRenderer::StaticClass())` in `StartupModule` and `UnregisterCustomRenderer(...)` in `ShutdownModule`.
- **Queries**: build an `FARFilter` (`AssetRegistry/ARFilter.h`) with `ClassPaths`, `PackagePaths`, `bRecursiveClasses`, `bRecursivePaths` and pass it to `IAssetRegistry::GetAssets(const FARFilter&, TArray<FAssetData>&, bool bSkipARFilteredAssets = true)`.

Full definition, factory and thumbnail implementations: [editor-module-setup.md](references/editor-module-setup.md).

## Asset Editor Toolkits

`FAssetEditorToolkit` (`Toolkits/AssetEditorToolkit.h`) hosts a tabbed editor window. `GetToolkitFName`, `GetBaseToolkitName` and `GetWorldCentricTabPrefix` are pure virtual; your init function builds a layout with `FTabManager::NewLayout` and calls `InitAssetEditor(Mode, InitToolkitHost, AppIdentifier, StandaloneDefaultLayout, bCreateDefaultStandaloneMenu, bCreateDefaultToolbar, ObjectToEdit)`.

Find or focus an open editor with `IAssetEditorInstance* Instance = GEditor->GetEditorSubsystem<UAssetEditorSubsystem>()->FindEditorForAsset(Asset, /*bFocusIfOpen=*/false);` then `Instance->FocusWindow(Asset)` or `Instance->CloseWindow(EAssetEditorCloseReason::AssetEditorHostClosed)`.

## Data Validation

`UEditorValidatorBase` ships in the DataValidation plugin (enabled by default; module `DataValidation`, `LoadingPhase: PreDefault`). Validators are discovered from their CDOs; override `CanValidateAsset_Implementation(const FAssetData& InAssetData, UObject* InObject, FDataValidationContext& InContext) const` and `ValidateLoadedAsset_Implementation(const FAssetData& InAssetData, UObject* InAsset, FDataValidationContext& Context)`. Run headless with `UnrealEditor-Cmd <Project>.uproject -run=DataValidation`.

## Deprecated — do not use

| Do not emit | Use in 5.8 | Source |
|---|---|---|
| `FEditorStyle::Get()`, `FEditorStyle::GetBrush/GetFontStyle` | `FAppStyle::Get()`, `FAppStyle::GetBrush/GetFontStyle` | `UE_DEPRECATED(5.1)` in `Editor/EditorStyle/Public/EditorStyleSet.h` |
| `IPropertyHandle::CreatePropertyNameWidget(Name, ToolTip, bDisplayResetToDefault, ...)` | `CreatePropertyNameWidget(NameOverride, ToolTipOverride)` | `UE_DEPRECATED(5.0)` in `PropertyEditor/Public/PropertyHandle.h:641` |
| `UAssetActionUtility::GetSupportedClass()` / `UActorActionUtility::GetSupportedClass()` | `SupportedClasses` array + `GetSupportedClasses()` | `UE_DEPRECATED(5.2)` in `Blutility/Classes/AssetActionUtility.h:69`, `ActorActionUtility.h:24` |
| `IAssetEditorInstance::CloseWindow()` (no arguments) | `CloseWindow(EAssetEditorCloseReason)` | `UE_DEPRECATED(5.3)` in `UnrealEd/Public/Subsystems/AssetEditorSubsystem.h:62` |
| `FAssetEditorToolkit::OnRequestClose()` (no arguments) | `OnRequestClose(EAssetEditorCloseReason)` | `UE_DEPRECATED(5.3)` in `UnrealEd/Public/Toolkits/AssetEditorToolkit.h:435` |
| `UEditorValidatorBase::CanValidateAsset_Implementation(UObject*)`, `ValidateLoadedAsset_Implementation(UObject*, TArray<FText>&)` | overloads taking `(const FAssetData&, UObject*, FDataValidationContext&)` | `UE_DEPRECATED("5.4")` in `DataValidation/Public/EditorValidatorBase.h:125,130` |
| `IDetailLayoutBuilder::GetDetailsView()` | `GetDetailsViewSharedPtr()` | `UE_DEPRECATED(5.5)` in `PropertyEditor/Public/DetailLayoutBuilder.h:63,74` |
| `IDetailTreeNode::GetNodeDetailsView()` | `GetNodeDetailsViewSharedPtr()` | `UE_DEPRECATED(5.5)` in `PropertyEditor/Public/IDetailTreeNode.h:128` |
| `IAssetTypeActions::GetThumbnailOverlay` | `UAssetDefinition` + `GetThumbnailActionOverlay` | `UE_DEPRECATED(5.5)` in `Developer/AssetTools/Public/IAssetTypeActions.h:122` |
| `FEditorModeRegistry::GetModeInfo` iteration helpers, `OnRegisteredModesChanged` | `UAssetEditorSubsystem::GetEditorModeInfoOrderedByPriority`, `FindEditorModeInfo`, `OnEditorModesChanged` | `UE_DEPRECATED(4.26)` in `UnrealEd/Public/EditorModeRegistry.h:98,104,141` |
| `UEdModeInteractiveToolsContext::UsesDragTools()` / `OnRegisterViewportInteractions()` | `UsesViewportInteractions()` / `OnBuildViewportInteractions()` (experimental) | `UE_DEPRECATED(5.8)` in `UnrealEd/Public/Tools/EdModeInteractiveToolsContext.h:427,441` |
| `FEditorModeTools::GetOverrideCursorVisibility(bool&, bool&, bool)` (by-value `bSoftwareCursorVisible`) | `GetCursorVisibilityOverride(bool&, bool&, bool&)` | `UE_DEPRECATED(5.8)` in `UnrealEd/Public/EditorModeManager.h:321` |

## Common Mistakes

**Editor headers in a Runtime module:** `UnrealEd`, `PropertyEditor`, `Blutility` and `ToolMenus` are editor-only and are not linked into packaged builds, so a Runtime module that depends on them unconditionally fails to package (UBT throws `BuildException`, `UnrealEd.Build.cs:12-14`). Put the code in `MyGameEditor`; use `#if WITH_EDITOR` only for runtime-class hooks such as `PostEditChangeProperty`.

**`IMPLEMENT_MODULE` before the class exists:** the macro must follow the `FMyGameEditorModule` declaration and both function definitions — put it at the end of the `.cpp`.

**Register without unregister:** every `RegisterCustomClassLayout`, `RegisterCustomPropertyTypeLayout`, `RegisterCustomRenderer` and `RegisterStartupCallback` needs its pair in `ShutdownModule`, guarded by `FModuleManager::Get().IsModuleLoaded(...)`. Missing pairs crash on Live Coding reload.

**Editing a `UObject` without a transaction:** always `FScopedTransaction` plus `Object->Modify()` before the write.

**Calling `FEditorModeRegistry::RegisterMode` for a `UEdMode`:** `UEdMode` subclasses are auto-registered from their CDOs; the registry is only for `FEdMode`.

**Raw `this` captured in a Slate lambda:** capture `TWeakPtr`/`TWeakObjectPtr` and pin before use — detail-panel widgets outlive the customization object.

**`ForceRefreshDetails()` on every value change:** it destroys and rebuilds the layout. Call it only when the row structure changes; drive values with `TAttribute` and `_Lambda` bindings.

**Touching `GEditor` in `StartupModule` at `LoadingPhase: Default`:** project modules at `Default` load before the editor engine exists; defer through `UToolMenus::RegisterStartupCallback` / `FCoreDelegates::GetOnPostEngineInit()` (the `OnPostEngineInit` member is `UE_DEPRECATED(5.8)`, `CoreDelegates.h:240`), or load at `PostEngineInit`.

## Related Skills

- `ue-module-build-system` — `.Build.cs` syntax, module types, public vs private dependencies, plugin descriptors
- `ue-cpp-foundations` — `UCLASS`/`UPROPERTY`/`UFUNCTION` specifiers, canonical subsystem type table, `UObject` lifecycle
- `ue-ui-umg-slate` — Slate widget fundamentals (`SNew`, `TAttribute`, `FReply`, compound widgets) used by every custom row
- `ue-data-assets-tables` — `UDataAsset`, `UPrimaryDataAsset` and `UDataTable` types that these editors and definitions wrap
- `ue-testing-debugging` — `UE_LOG` categories and verbosity, automation tests for editor code, Insights profiling
- `ue-actor-component-architecture` — actor and component classes whose details panels you customize
- `ue-project-context` — writing and consuming `.agents/ue-project-context.md`
- `ue-blueprint-cpp-interop` — exposing C++ to Blueprint: UFUNCTION/UPROPERTY meta keys, latent actions and async nodes
