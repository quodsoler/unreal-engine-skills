---
name: ue-ui-umg-slate
description: "Use when writing or fixing Unreal Engine UI in C++ — UMG user widgets, Slate widgets, Common UI screens, or MVVM view models. Also use when the user mentions 'UUserWidget', 'NativeConstruct', 'BindWidget', 'CreateWidget', 'AddToViewport', 'ESlateVisibility', 'OnClicked', 'UListView', 'widget animation', 'SNew', 'SLATE_BEGIN_ARGS', 'SCompoundWidget', 'UCommonActivatableWidget', 'activatable widget stack', 'UCommonButtonBase', 'gamepad focus', 'FUIInputConfig', 'UMVVMViewModelBase', 'FieldNotify', or 'WidgetComponent'. For input modes and Enhanced Input, see ue-input-system; for Slate in editor tooling, see ue-editor-tools."
metadata:
  version: "2.0.0"
  engine: "5.8"
---

# UE UI: UMG, Slate, and Common UI

Target engine: **UE 5.8**. APIs below are verified against the 5.8 headers; older forms are listed under "Deprecated — do not use".

UMG is the runtime UI framework (`UMG` module, `Engine/Source/Runtime/UMG/Public`) built on Slate (`Slate`, `SlateCore`). Common UI (plugin, production in 5.8) adds cross-platform input routing, activation stacks and platform-aware buttons. Model View Viewmodel (plugin, Beta in 5.8) adds declarative data binding on top of `FieldNotification`.

## Context

Read `.agents/ue-project-context.md` if it exists (module names, conventions, enabled plugins, GAS/networking setup). Do not stop if it is missing.

Identify the area from the request and the codebase. Ask only when two plausible readings would produce different code.

| Request is about… | Go to |
|---|---|
| Widget class, overrides, creation, visibility, GC | [Widget Lifecycle](#widget-lifecycle) |
| Wiring designer widgets into C++ (`meta=(BindWidget)`) | [BindWidget](#bindwidget) |
| Buttons, text, images, bars, lists | [Widget Interaction](#widget-interaction) |
| Playing/stopping UMG animations | [Widget Animation](#widget-animation) |
| Mouse cursor, focus, `SetInputMode` | [Input Mode and Focus](#input-mode-and-focus) |
| Menus, screen stacks, gamepad, back action | [Common UI](#common-ui) |
| Data binding, view models, field notifications | [MVVM](#mvvm) |
| UI rendered in the level | [World-Space UI](#world-space-ui) |
| Iterating widgets, drag/drop, brushes, geometry | [Widget Tree and Helper Libraries](#widget-tree-and-helper-libraries) |
| Custom `SWidget`, `SNew`, `SLATE_BEGIN_ARGS` | [Slate](#slate) |

Per-widget method tables (setters, delegates, enums) live in [references/widget-types.md](references/widget-types.md). Full Common UI plugin setup lives in [references/common-ui-setup.md](references/common-ui-setup.md). A complete custom Slate widget lives in [references/slate-patterns.md](references/slate-patterns.md).

## Widget Lifecycle

```cpp
// MyWidget.h
#pragma once
#include "CoreMinimal.h"
#include "Blueprint/UserWidget.h"
#include "MyWidget.generated.h"

class UButton;

UCLASS()
class MYGAME_API UMyWidget : public UUserWidget
{
    GENERATED_BODY()

protected:
    virtual void NativeOnInitialized() override;
    virtual void NativePreConstruct() override;
    virtual void NativeConstruct() override;
    virtual void NativeDestruct() override;
    virtual void NativeTick(const FGeometry& MyGeometry, float InDeltaTime) override;

    UPROPERTY(meta=(BindWidget))
    TObjectPtr<UButton> PlayButton;

    UFUNCTION()
    void HandlePlayClicked();
};

// MyWidget.cpp — #include "MyWidget.h" and "Components/Button.h"
void UMyWidget::NativeOnInitialized() { Super::NativeOnInitialized(); }
void UMyWidget::NativePreConstruct() { Super::NativePreConstruct(); }
void UMyWidget::NativeTick(const FGeometry& MyGeometry, float InDeltaTime)
{
    Super::NativeTick(MyGeometry, InDeltaTime);
}

void UMyWidget::NativeConstruct()
{
    Super::NativeConstruct();       // Runs on every re-add; pair each bind with an unbind in NativeDestruct
    PlayButton->OnClicked.AddDynamic(this, &UMyWidget::HandlePlayClicked);
}

void UMyWidget::NativeDestruct()
{
    PlayButton->OnClicked.RemoveDynamic(this, &UMyWidget::HandlePlayClicked);
    Super::NativeDestruct();
}
```

| Override (verbatim from `Blueprint/UserWidget.h`) | When it runs |
|---|---|
| `virtual void NativeOnInitialized() override;` | Once per instance, after construction of the widget object |
| `virtual void NativePreConstruct() override;` | After the Slate tree is rebuilt, just before `NativeConstruct` (`UserWidget.cpp:1231`); also runs in the Designer |
| `virtual void NativeConstruct() override;` | Slate widget added to a parent — bind delegates here |
| `virtual void NativeDestruct() override;` | Slate widget removed — unbind here |
| `virtual void NativeTick(const FGeometry& MyGeometry, float InDeltaTime) override;` | Per frame while ticking |
| `virtual int32 NativePaint(const FPaintArgs& Args, const FGeometry& AllottedGeometry, const FSlateRect& MyCullingRect, FSlateWindowElementList& OutDrawElements, int32 LayerId, const FWidgetStyle& InWidgetStyle, bool bParentEnabled) const override;` | Custom drawing; return the last used layer id |
| `virtual FReply NativeOnMouseButtonDown(const FGeometry& InGeometry, const FPointerEvent& InMouseEvent) override;` | Pointer press |
| `virtual FReply NativeOnKeyDown(const FGeometry& InGeometry, const FKeyEvent& InKeyEvent) override;` | Key press while focused |
| `virtual FReply NativeOnFocusReceived(const FGeometry& InGeometry, const FFocusEvent& InFocusEvent) override;` | This widget gained focus |
| `virtual void NativeOnAddedToFocusPath(const FFocusEvent& InFocusEvent) override;` | This widget or a descendant gained focus |
| `virtual bool Initialize() override;` | Builds the widget tree; call `Super::Initialize()` and return its result |
| `virtual TSharedRef<SWidget> RebuildWidget() override;` | Builds the underlying Slate widget |
| `virtual void ReleaseSlateResources(bool bReleaseChildren) override;` | Drop cached `TSharedPtr<SWidget>` members here |

Input handlers return `FReply::Handled()` to consume the event or `FReply::Unhandled()` to let it bubble.

### Creation, viewport, visibility

```cpp
// CreateWidget<T>(Owner, Class, WidgetName = NAME_None). Owner may be a UWidget,
// UWidgetTree, APlayerController, UGameInstance or UWorld.
UMyWidget* Widget = CreateWidget<UMyWidget>(GetOwningPlayer(), MyWidgetClass);
Widget->AddToViewport(0);       // ZOrder: higher draws on top. Roots the widget.
Widget->AddToPlayerScreen(0);   // Split-screen: one player's viewport region; returns bool
Widget->RemoveFromParent();     // Un-roots - collectable unless a UPROPERTY holds it
Widget->SetVisibility(ESlateVisibility::Collapsed);  // Not drawn, takes no space
Widget->SetVisibility(ESlateVisibility::Hidden);     // Not drawn, still takes space
Widget->SetVisibility(ESlateVisibility::Visible);    // Drawn, hit-testable
Widget->SetIsEnabled(false);                         // Greys out, blocks input
```

`HitTestInvisible` draws but passes input through for the widget and its children; `SelfHitTestInvisible` passes input through for the widget only. Hold a `UPROPERTY() TObjectPtr<UMyWidget>` if the widget must survive `RemoveFromParent`. `GetOwningPlayer()` returns the owning `APlayerController`; `GetOwningPlayer<AMyPlayerController>()` casts for you.

## BindWidget

```cpp
UPROPERTY(meta=(BindWidget))                 // Compile error in the Widget BP if missing
TObjectPtr<UTextBlock> ScoreText;
UPROPERTY(meta=(BindWidgetOptional))         // May be null - always null-check
TObjectPtr<UTextBlock> SubtitleText;
UPROPERTY(Transient, meta=(BindWidgetAnim))  // Transient is required for animation bindings
TObjectPtr<UWidgetAnimation> IntroAnim;
```

Rules: the C++ member name must match the Widget Blueprint's widget name exactly (case-sensitive); the declared type must match or be a base of the designer widget's class; bound members are assigned inside `Initialize()` before `NativeOnInitialized` runs (`UserWidget.cpp:153,175`), so they are valid in `NativeOnInitialized`, `NativePreConstruct` and `NativeConstruct` — bind once-per-instance delegates in `NativeOnInitialized`. Declare them `protected` with no `Category`.

## Widget Interaction

Dynamic multicast delegates on UMG components use `AddDynamic`, so the handler must be a `UFUNCTION()`.

```cpp
PlayButton->OnClicked.AddDynamic(this, &UMyWidget::HandlePlayClicked);
PlayButton->SetColorAndOpacity(FLinearColor(1.f, 0.8f, 0.f, 1.f));
ScoreText->SetText(FText::Format(NSLOCTEXT("HUD", "ScoreFmt", "Score: {0}"), FText::AsNumber(Score)));
ScoreText->SetTextOverflowPolicy(ETextOverflowPolicy::Ellipsis);
PlayerAvatar->SetBrushFromSoftTexture(SoftAvatarTexture, /*bMatchSize=*/false);
UMaterialInstanceDynamic* AvatarMID = PlayerAvatar->GetDynamicMaterial();
AvatarMID->SetScalarParameterValue(TEXT("Opacity"), 0.5f);
HealthBar->SetPercent(CurrentHealth / MaxHealth);   // 0.0-1.0
HealthBar->SetFillColorAndOpacity(FLinearColor(0.f, 1.f, 0.f, 1.f));
```

Method and enum tables for `UButton`, `UTextBlock`, `UImage`, `UProgressBar`, `UScrollBox`, `UWidgetSwitcher`, `UCheckBox`, `UEditableTextBox`, `USlider`, `UComboBoxString` and the `UWidget` base are in [references/widget-types.md](references/widget-types.md).

### Lists: UListView, UTileView, UTreeView

All three are virtualised — only visible entry widgets exist, and they are recycled. Items are `UObject*`; the entry widget derives from `UUserWidget` and implements `IUserObjectListEntry`, overriding `virtual void NativeOnListItemObjectSet(UObject* ListItemObject) override;` (call `IUserObjectListEntry::NativeOnListItemObjectSet` first). The full entry-widget example and the `UListView` method table are in [references/widget-types.md](references/widget-types.md).

```cpp
ItemList->ClearListItems();
for (const FMyItemInfo& Info : Items)
{
    UMyItemData* Data = NewObject<UMyItemData>(this);
    Data->DisplayName = Info.DisplayName;
    ItemList->AddItem(Data);
}
UMyItemData* Selected = ItemList->GetSelectedItem<UMyItemData>();
```

`BP_OnItemClicked` is a Blueprint-only private delegate. From C++, bind the native event `ItemList->OnItemClicked().AddUObject(...)` (`FSimpleListItemEvent`, `Components/ListViewBase.h:48`; also `OnItemSelectionChanged()`), or override `virtual void OnItemClickedInternal(UObject* Item) override;` in a `UListView` subclass. Inside an entry widget, read the item with `GetListItem<UMyItemData>()` and query selection with the interface member `IsListItemSelected()` (`Blueprint/IUserListEntry.h:32`).

## Widget Animation

`PlayAnimation` and its variants return an `FWidgetAnimationHandle` (`Animation/WidgetAnimationHandle.h`); keep the handle, not a sequence-player pointer. A valid handle may still resolve to no state once the animation finishes, so check `IsValid()`.

```cpp
// PlayAnimation(UWidgetAnimation*, StartAtTime=0, NumLoopsToPlay=1,
//               EUMGSequencePlayMode::Type=Forward, PlaybackSpeed=1, bRestoreState=false)
FWidgetAnimationHandle IntroHandle = PlayAnimation(IntroAnim);
IntroHandle.SetUserTag(TEXT("Intro"));
PlayAnimationForward(IntroAnim);                        // also PlayAnimationReverse
const float PausedAt = PauseAnimation(IntroAnim);
const bool bPlaying = IsAnimationPlaying(IntroAnim);
StopAnimation(IntroAnim);
```

## Input Mode and Focus

`ue-input-system` owns input modes and Enhanced Input. Minimal correct form:

```cpp
FInputModeUIOnly UIMode;
UIMode.SetWidgetToFocus(Widget->TakeWidget());
UIMode.SetLockMouseToViewportBehavior(EMouseLockMode::DoNotLock);
GetOwningPlayer()->SetInputMode(UIMode);
GetOwningPlayer()->SetShowMouseCursor(true);
// Restore gameplay:
GetOwningPlayer()->SetInputMode(FInputModeGameOnly());
GetOwningPlayer()->SetShowMouseCursor(false);
```

`FInputModeGameAndUI` also has `SetWidgetToFocus` and `SetLockMouseToViewportBehavior`. Set the input mode from the code that added the widget (HUD, PlayerController, GameMode), not from `NativeConstruct`. When Common UI is enabled, do not call `SetInputMode` at all — return an `FUIInputConfig` instead.

## Common UI

Enable the `CommonUI` plugin (it ships both the `CommonUI` and `CommonInput` modules), add both modules to Build.cs, and set the viewport client class to `UCommonGameViewportClient` — it reroutes input to the UI action router before the game (`CommonGameViewportClient.h:21`); gamepad/mouse/touch changes are detected by `UCommonInputSubsystem`'s `FCommonInputPreprocessor` and broadcast from that subsystem. Setup details, layer architecture and settings keys are in [references/common-ui-setup.md](references/common-ui-setup.md).

```cpp
// MyMenuScreen.h
#pragma once
#include "CommonActivatableWidget.h"
#include "MyMenuScreen.generated.h"

class UCommonButtonBase;

UCLASS()
class MYGAME_API UMyMenuScreen : public UCommonActivatableWidget
{
    GENERATED_BODY()

protected:
    virtual void NativeOnActivated() override;
    virtual void NativeOnDeactivated() override;
    virtual UWidget* NativeGetDesiredFocusTarget() const override;
    virtual TOptional<FUIInputConfig> GetDesiredInputConfig() const override;
    virtual bool NativeOnHandleBackAction() override;   // Return true when handled

    UPROPERTY(meta=(BindWidget))
    TObjectPtr<UCommonButtonBase> CloseButton;

    void HandleCloseClicked();
};

// MyMenuScreen.cpp — #include "MyMenuScreen.h" and "CommonButtonBase.h"
void UMyMenuScreen::NativeOnActivated()
{
    Super::NativeOnActivated();     // Broadcasts OnActivated(), which activates this widget's tree node
    CloseButton->OnClicked().AddUObject(this, &UMyMenuScreen::HandleCloseClicked);
}

void UMyMenuScreen::NativeOnDeactivated()
{
    CloseButton->OnClicked().RemoveAll(this);
    Super::NativeOnDeactivated();   // Broadcasts OnDeactivated(); the router moves focus to the parent
}

UWidget* UMyMenuScreen::NativeGetDesiredFocusTarget() const { return CloseButton; }

TOptional<FUIInputConfig> UMyMenuScreen::GetDesiredInputConfig() const
{
    return FUIInputConfig(ECommonInputMode::Menu, EMouseCaptureMode::NoCapture);
}

bool UMyMenuScreen::NativeOnHandleBackAction() { DeactivateWidget(); return true; }
void UMyMenuScreen::HandleCloseClicked() { DeactivateWidget(); }
```

Public API: `ActivateWidget()`, `DeactivateWidget()`, `IsActivated()`, `GetDesiredFocusTarget()`, `ClearFocusRestorationTarget()`, `RequestRefreshFocus()`, plus the non-dynamic `OnActivated()` / `OnDeactivated()` delegates. `BP_GetDesiredFocusTarget` is the `BlueprintImplementableEvent` that `NativeGetDesiredFocusTarget` routes to — override the native one in C++. Editable defaults: `bAutoActivate`, `bIsBackHandler`, `bIsBackActionDisplayedInActionBar`, `bIsModal`, `bAutoRestoreFocus`, `bSetVisibilityOnActivated`/`ActivatedVisibility`, `bSetVisibilityOnDeactivated`/`DeactivatedVisibility`. `ECommonInputMode` is `Menu` (UI only), `Game` (game only) or `All`.

### Containers

`UCommonActivatableWidgetContainerBase` (`Widgets/CommonActivatableWidgetContainer.h`) is the base; `UCommonActivatableWidgetStack` shows the topmost widget and re-activates the one beneath when it deactivates; `UCommonActivatableWidgetQueue` shows one at a time and advances when the active one deactivates. `UCommonActivatableWidgetSwitcher` is an index-based switcher, not a stack — do not use it here.

```cpp
UPROPERTY(meta=(BindWidget))
TObjectPtr<UCommonActivatableWidgetStack> MenuLayer;
// AddWidget<T> creates (or pools) the instance, adds it, and the container activates it.
UMyMenuScreen* Screen = MenuLayer->AddWidget<UMyMenuScreen>(UMyMenuScreen::StaticClass());
// Overload with an init lambda that runs after creation, before the widget is added:
MenuLayer->AddWidget<UMyMenuScreen>(UMyMenuScreen::StaticClass(),
    [](UMyMenuScreen& Instance) { Instance.SetIsEnabled(true); });

if (UCommonActivatableWidget* Active = MenuLayer->GetActiveWidget())
{
    Active->DeactivateWidget();         // Pops the stack
}
// RemoveWidget(UCommonActivatableWidget&) deactivates the active widget, or yanks an inactive one
MenuLayer->RemoveWidget(*Screen);
MenuLayer->ClearWidgets();              // GetNumWidgets() reports the current depth
```

### Buttons, input type, action routing

```cpp
// UCommonButtonBase::OnClicked() returns FCommonButtonEvent (DECLARE_EVENT, no params).
MyButton->OnClicked().AddUObject(this, &UMyMenuScreen::HandleCloseClicked);
MyButton->OnClicked().RemoveAll(this);
MyButton->SetIsSelected(true);
MyButton->SetIsInteractionEnabled(true);
UCommonInputSubsystem* InputSubsystem = UCommonInputSubsystem::Get(GetOwningLocalPlayer());
const ECommonInputType InputType = InputSubsystem->GetCurrentInputType();  // MouseAndKeyboard, Gamepad, Touch
InputSubsystem->OnInputMethodChangedNative.AddUObject(this, &UMyMenuScreen::HandleInputMethodChanged);

// UCommonUIActionRouterBase is the local-player subsystem that routes input through the activation tree.
if (UCommonUIActionRouterBase* Router = UCommonUIActionRouterBase::Get(*CloseButton))
{
    const ECommonInputMode Mode = Router->GetActiveInputMode();
}

// On a UCommonUserWidget subclass, with UPROPERTY(EditDefaultsOnly) TObjectPtr<UInputAction> BackInputAction:
FUIActionBindingHandle BackHandle = RegisterUIActionBinding(
    FBindUIActionArgs(BackInputAction, /*bShouldDisplayInActionBar=*/true,
        FSimpleDelegate::CreateUObject(this, &UMyMenuScreen::HandleCloseClicked)));
```

`OnInputMethodChangedNative` is a `DECLARE_EVENT_OneParam`, so `HandleInputMethodChanged(ECommonInputType)` needs no `UFUNCTION()`. Enhanced Input and Common Input are unified in 5.8: with `bEnableEnhancedInputSupport` on (`UCommonInputSettings::IsEnhancedInputSupportEnabled()`), `FBindUIActionArgs` takes a `const UInputAction*` and `UCommonActivatableWidget::InputMapping` / `InputMappingPriority` push a mapping context on activation.

## MVVM

ModelViewViewModel is Beta in 5.8. A view model is a `UMVVMViewModelBase`; `FieldNotify` properties get a generated field descriptor, and `UE_MVVM_SET_PROPERTY_VALUE` compares, assigns and broadcasts in one call.

```cpp
// MyGameViewModel.h
#pragma once
#include "MVVMViewModelBase.h"
#include "MyGameViewModel.generated.h"

UCLASS()
class MYGAME_API UMyGameViewModel : public UMVVMViewModelBase
{
    GENERATED_BODY()

public:
    void SetScore(int32 NewScore) { UE_MVVM_SET_PROPERTY_VALUE(Score, NewScore); }
    int32 GetScore() const { return Score; }

private:
    UPROPERTY(BlueprintReadOnly, FieldNotify, Getter, meta=(AllowPrivateAccess))
    int32 Score = 0;
};

// Binding at runtime - the UMVVMView extension on the user widget owns the generated bindings.
UMyGameViewModel* ViewModel = NewObject<UMyGameViewModel>(this);
if (UMVVMView* View = UMVVMSubsystem::GetViewFromUserWidget(MyWidget))
{
    View->SetViewModel(TEXT("ScoreViewModel"), ViewModel);   // Name from the Bindings panel
}
```

`UE_MVVM_SET_PROPERTY_VALUE(MemberName, NewValue)` expands to `SetPropertyValue(MemberName, NewValue, ThisClass::FFieldNotificationClassDescriptor::MemberName)`. Use `UE_MVVM_SET_PROPERTY_VALUE_INLINE` for values that cannot be passed as function arguments (bitfields), and `UE_MVVM_BROADCAST_FIELD_VALUE_CHANGED(MemberName)` when you assigned the member yourself; both end in `BroadcastFieldValueChanged(UE::FieldNotification::FFieldId)` from `INotifyFieldValueChanged`.

UHT generates the descriptor for every `FieldNotify` property and function. Declare fields by hand only under the `UCLASS(CustomFieldNotify)` class specifier (not a meta key; see `Components/Widget.h:215`), with the `UE_FIELD_NOTIFICATION_DECLARE_CLASS_DESCRIPTOR_BEGIN` / `UE_FIELD_NOTIFICATION_DECLARE_FIELD(Name, API_STRING)` / `UE_FIELD_NOTIFICATION_DECLARE_ENUM_FIELD(Name)` / `..._END` family in `FieldNotificationDeclaration.h` (`FieldNotification` module; the old `FieldNotification/` path is deprecated since 5.3), paired with `UE_FIELD_NOTIFICATION_IMPLEMENTATION_BEGIN` and `UE_FIELD_NOTIFICATION_IMPLEMENTATION_END` in the .cpp. `UMVVMView` also exposes `GetViewModel(FName)`, `SetViewModelByClass(...)` and `ExecuteViewModelBindings(FName)`; `UMVVMSubsystem` is a `UEngineSubsystem` and `GetViewFromUserWidget` is static.

## World-Space UI

`UWidgetComponent` (`Components/WidgetComponent.h`, `UMG` module) renders a `UUserWidget` onto a plane or cylinder in the level.

```cpp
// In the constructor of AMyCharacter (NameplateComponent is a UPROPERTY member):
NameplateComponent = CreateDefaultSubobject<UWidgetComponent>(TEXT("Nameplate"));
NameplateComponent->SetupAttachment(RootComponent);
NameplateComponent->SetWidgetSpace(EWidgetSpace::World);        // or EWidgetSpace::Screen
NameplateComponent->SetDrawSize(FVector2D(256.f, 64.f));
NameplateComponent->SetGeometryMode(EWidgetGeometryMode::Plane);// or Cylinder
NameplateComponent->SetTwoSided(false);
// At runtime:
NameplateComponent->SetWidget(NameplateWidget);
UUserWidget* Current = NameplateComponent->GetUserWidgetObject();
NameplateComponent->SetTickMode(ETickMode::Disabled);   // Then drive updates manually
NameplateComponent->RequestRedraw();
```

Enable `bReceiveHardwareInput` only for a widget that must take real mouse input; otherwise drive interaction with a `UWidgetInteractionComponent`.

## Widget Tree and Helper Libraries

```cpp
WidgetTree->ForEachWidget([](UWidget* Widget) { Widget->SetRenderOpacity(1.f); });
WidgetTree->ForEachWidgetAndDescendants([](UWidget* W) { W->SetIsEnabled(true); });
UButton* Found = WidgetTree->FindWidget<UButton>(TEXT("PlayButton"));
UMyEntryWidget* Built = WidgetTree->ConstructWidget<UMyEntryWidget>(EntryWidgetClass);
```

`UWidgetBlueprintLibrary` (`Blueprint/WidgetBlueprintLibrary.h`) holds the brush, drag-and-drop, `FEventReply` and `FPaintContext` drawing helpers; `USlateBlueprintLibrary` (`Blueprint/SlateBlueprintLibrary.h`) converts between local, absolute and viewport space. Both are enumerated in [references/widget-types.md](references/widget-types.md).

## Slate

Use Slate for editor extensions and for runtime widgets UMG does not expose. `SWidget` is the base; `SCompoundWidget` is the usual parent for a custom widget.

A custom widget declares its arguments between `SLATE_BEGIN_ARGS` / `SLATE_END_ARGS` — `SLATE_ATTRIBUTE` for a `TAttribute<T>` that can be bound to a lambda, `SLATE_ARGUMENT` for a plain value, `SLATE_EVENT` for a delegate — and builds its children into `ChildSlot` inside `Construct(const FArguments& InArgs)`. Bind Slate delegates with `CreateSP` so a destroyed widget never gets called. A complete `SCompoundWidget` with all three argument kinds: [references/slate-patterns.md](references/slate-patterns.md).

`SNew(WidgetType)` returns a `TSharedRef<WidgetType>`; `SAssignNew(Var, WidgetType)` does the same and stores it in a `TSharedPtr`. `FOnClicked` is `DECLARE_DELEGATE_RetVal(FReply, FOnClicked)`, so a click handler must return `FReply`. `UWidget::GetCachedWidget()` gives the `TSharedPtr<SWidget>` behind a UMG widget; `UUserWidget::TakeWidget()` builds and returns the `TSharedRef<SWidget>`.

**Public `TAttribute` members on stock Slate widgets are being privatised.** Do not read or write them directly; use the accessors named in the deprecation message — `Set<Name>` to assign, `Get<Name>Attribute()` to read the `TSlateAttributeRef`. In 5.8 this hits `SCheckBox::PaddingOverride` (`SetPaddingOverride` / `GetPaddingOverrideAttribute`), `SCheckBox::ForegroundColorOverride` (`SetForegroundColorOverride` / `GetForegroundColorOverrideAttribute`), `SEditableText::Font` (`SetFont` / `GetFontAttribute`), `SEditableText::ColorAndOpacity` (`SetColorAndOpacity` / `GetColorAndOpacityAttribute`) and `SMenuAnchor::Placement` (`SetMenuPlacement` / `GetPlacementAttribute`).

**Slate vs UMG:** Slate for editor tools and maximum control; UMG for game runtime UI, Blueprint extensibility, animations and `BindWidget`.

## Build.cs

```csharp
PublicDependencyModuleNames.AddRange(new string[] { "UMG", "Slate", "SlateCore" });
PrivateDependencyModuleNames.AddRange(new string[] {
    "CommonUI", "GameplayTags", // Activatable widgets, buttons, action router, FUIInputConfig; FBindUIActionArgs' FUIActionTag links GameplayTags
    "CommonInput", "EnhancedInput", // UCommonInputSubsystem, ECommonInputType/Mode; UInputAction UPROPERTYs
    "ModelViewViewModel",  // MVVM (Beta in 5.8)
    "FieldNotification" }); // Field-notification declaration macros
```

## Deprecated — do not use

| Do not emit | Use in 5.8 | Source |
|---|---|---|
| `UUserWidget::RemoveFromViewport()` | `RemoveFromParent()` | `UE_DEPRECATED(5.1)` in `Blueprint/UserWidget.h:356` |
| `UUMGSequencePlayer*` returned from `PlayAnimation` | `FWidgetAnimationHandle` / `FWidgetAnimationState` | `UE_DEPRECATED(5.6)` in `Animation/UMGSequencePlayer.h:29` |
| `UUserWidget::ActiveSequencePlayers` / `StoppedSequencePlayers` | `ActiveAnimations` | `UE_DEPRECATED(5.6)` in `Blueprint/UserWidget.h:1491,1499` |
| `UUserWidget::InitializeInputComponent()` | `bAutomaticallyRegisterInputOnConstruction` | `UE_DEPRECATED(5.7)` in `Blueprint/UserWidget.h:1713` |
| `FWidgetStateBitfield` enum-state accessors | Binary and named states only | `UE_DEPRECATED(5.5)` in `Binding/States/WidgetStateBitfield.h:35` |
| `UListView::InitHorizontalEntrySpacing` / `InitVerticalEntrySpacing` | `SetHorizontalEntrySpacing` / `SetVerticalEntrySpacing` | `UE_DEPRECATED(5.6)` in `Components/ListView.h:343,346` |
| `USlateBlueprintLibrary::TransformScalarAbsoluteToLocal` / `TransformVectorAbsoluteToLocal` | `Scalar_LocalToAbsolute` / `Vector_LocalToAbsolute` | `UE_DEPRECATED(5.6)` in `Blueprint/SlateBlueprintLibrary.h:77,87` |
| `UGameViewportSubsystem::Get(UWorld*)` | `UGameViewportSubsystem::Get()` | `UE_DEPRECATED(5.8)` in `Blueprint/GameViewportSubsystem.h:64` |
| `UButton::WidgetStyle` / `ColorAndOpacity` / `ClickMethod` direct field access | `SetStyle` / `SetColorAndOpacity` / `SetClickMethod` | `UE_DEPRECATED(5.2)` in `Components/Button.h:37,42,52` |
| `UCommonUIActionRouter` | `UCommonUIActionRouterBase` | Class name in `Input/CommonUIActionRouterBase.h:66` |
| `RegisterLinkedPreprocessor(Widget, Processor, int32 DesiredIndex)` | overload taking `FInputPreprocessorRegistrationKey` | `UE_DEPRECATED(5.5)` in `Input/CommonUIActionRouterBase.h:87` |
| `FUIActionBinding::TryCreate(Widget, Args)` | `TryCreate(Widget, Args, UserIndex)` | `UE_DEPRECATED(5.6)` in `Input/UIActionBinding.h:35` |
| `FOnRerouteTouchInputDelegate`, `UCommonGameViewportClient::OnRerouteTouchInput()` | `FTouchRerouteDelegate`, `GetRerouteTouchRegistration()` / `GetRerouteTouchDelegate()` | `UE_DEPRECATED(5.8)` in `CommonGameViewportClient.h:15,47` |
| `#include "Input/CommonInputMode.h"` (CommonUI) | `#include "CommonInputModeTypes.h"` (CommonInput) | `UE_DEPRECATED_HEADER(5.1)` in `Input/CommonInputMode.h` |
| `UMVVMViewModelBase::K2_BroadcastFieldValueChanged` | the generated property setter | `UE_DEPRECATED(5.3)` in `MVVMViewModelBase.h:71` |

## Common Mistakes

**Binding in `NativeConstruct` without unbinding:** `NativeConstruct` runs again every time the widget is re-added, so an unpaired `AddDynamic` there binds twice. Bind once in `NativeOnInitialized` (bound widgets are already valid there), or pair `NativeConstruct` with `NativeDestruct` — never bind in `NativeTick`.

**Using `SetVisibility(Hidden)` to close a menu:** the widget still occupies layout and stays in the viewport (rooted). Call `RemoveFromParent()`, or `SetVisibility(ESlateVisibility::Collapsed)` if it must stay in the tree.

**`CreateWidget(GetWorld(), ...)` for player UI:** pass the owning `APlayerController` so the widget has a local player, split-screen works, and `GetOwningPlayer()` is valid.

**`AddDynamic` on `UCommonButtonBase::OnClicked()`:** it returns `FCommonButtonEvent` (a `DECLARE_EVENT`, not a dynamic delegate). Use `AddUObject` in `NativeOnActivated` and `RemoveAll(this)` in `NativeOnDeactivated`.

**Skipping `Super::NativeOnActivated` / `Super::NativeOnDeactivated`:** the Super calls broadcast `OnActivated()` / `OnDeactivated()`, which the widget's activation-tree node listens to (`UIActionRouterTypes.cpp:1372`), and push/pop `InputMapping`, so focus, back handling and input config stop working without them.

**No `UPROPERTY` after `RemoveFromParent`:** the widget is unrooted and collectable. Hold a `UPROPERTY() TObjectPtr<>` if you intend to re-add it.

**Unspecified Z-order:** widgets added at the same Z-order have undefined draw order. Pass an explicit value to `AddToViewport(ZOrder)` and reserve ranges (for example HUD 0-9, menus 10-19, popups 20+). Slate ticks a widget from its paint pass (`SWidget.cpp:1511`), so `NativeTick` stops while it is `Hidden` or `Collapsed` — do not rely on `NativeTick` to un-hide it.

**Calling `SetInputMode` under Common UI:** it bypasses the activation tree and fights the router. Return an `FUIInputConfig` from `GetDesiredInputConfig()` instead.

## Related Skills

- `ue-input-system` — Enhanced Input, input mapping contexts, `FInputModeUIOnly` / `FInputModeGameAndUI` / `FInputModeGameOnly`, input priority
- `ue-gameplay-framework` — where HUD/PlayerController create and own widgets, split-screen ownership
- `ue-cpp-foundations` — `UPROPERTY`/`UFUNCTION` specifiers, `TObjectPtr`, GC rules, subsystems
- `ue-editor-tools` — Slate in detail customisations, editor utility widgets, `UToolMenus`
- `ue-materials-rendering` — UI materials, `UMaterialInstanceDynamic` parameters driven from widgets
- `ue-data-assets-tables` — data assets and data tables that back list items and icon sets
- `ue-gameplay-abilities` — attribute change delegates that drive health/resource bars
- `ue-blueprint-cpp-interop` — exposing C++ to Blueprint: UFUNCTION/UPROPERTY meta keys, latent actions and async nodes
- `ue-gameplay-cameras` — spring arms, view targets, camera modifiers, shakes and the Gameplay Camera System
