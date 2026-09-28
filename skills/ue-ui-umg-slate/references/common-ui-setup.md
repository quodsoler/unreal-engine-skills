# Common UI Plugin — Setup and Patterns

Target engine: **UE 5.8**. Verified against `Engine/Plugins/Runtime/CommonUI/Source/CommonUI/Public/` and `.../CommonInput/Public/`. CommonUI is production (not Experimental, not Beta) in 5.8.

Common UI is Epic's cross-platform UI framework built on top of UMG. It handles input method detection (gamepad vs. mouse vs. touch), input routing through an activation tree, and provides base classes for platform-agnostic buttons and screens. The `CommonUI` module holds the widgets, the action router and `FUIInputConfig` (`Input/UIActionBindingHandle.h`); `CommonInput` holds `UCommonInputSubsystem`, `ECommonInputType` and `ECommonInputMode`.

---

## Plugin Setup

### 1. Enable the Plugin

`CommonUI` is one plugin that ships three modules — `CommonUI`, `CommonInput` and `CommonUIEditor`. There is no separate CommonInput plugin, so enable `CommonUI` only. It is not enabled by default (`"EnabledByDefault": false`) and it pulls in `EnhancedInput`, `GameplayTagsEditor` and `EngineAssetDefinitions` as plugin dependencies.

In `<ProjectName>.uproject`:

```json
{
  "Plugins": [
    { "Name": "CommonUI", "Enabled": true }
  ]
}
```

### 2. Build.cs Dependencies

```csharp
// <ProjectName>.Build.cs — both modules come from the one plugin
PrivateDependencyModuleNames.AddRange(new string[]
{
    "CommonUI",     // Widgets, containers, UCommonUIActionRouterBase, FUIInputConfig
    "GameplayTags", // FBindUIActionArgs / FUIActionTag (link errors without it)
    "CommonInput",  // UCommonInputSubsystem, ECommonInputMode/Type
    "EnhancedInput", // UInputAction for FBindUIActionArgs / InputMapping UPROPERTYs
});
```

### 3. Configure the Game Viewport Client

Common UI requires a custom viewport client so the UI action router gets input before the game.

**Project Settings → Maps & Modes → Game Viewport Client Class** → set to `CommonGameViewportClient`.

Or in `DefaultEngine.ini`:

```ini
[/Script/Engine.Engine]
GameViewportClientClassName=/Script/CommonUI.CommonGameViewportClient
```

`UCommonGameViewportClient` reroutes key, axis and touch input to the `UCommonUIActionRouterBase` before `Super::InputKey` (`CommonGameViewportClient.h:21`, `CommonGameViewportClient.cpp:44-61`). Input-method detection (gamepad vs mouse vs touch) is done separately by `UCommonInputSubsystem`'s `FCommonInputPreprocessor`, a Slate input preprocessor, and broadcast from that subsystem.

### 4. Configure Common Input Settings

`UCommonInputSettings` is `UCLASS(config = Game, defaultconfig)`, so it lives in **Project Settings → Game → Common Input Settings** and serialises to `DefaultGame.ini`:

```ini
[/Script/CommonInput.CommonInputSettings]
InputData=/Game/UI/BP_CommonInputData.BP_CommonInputData_C
bEnableEnhancedInputSupport=True
bEnableDefaultInputConfig=True
bEnableAutomaticGamepadTypeDetection=True
```

| Config property | Effect |
|---|---|
| `InputData` | `TSoftClassPtr<UCommonUIInputData>` — your derived class naming the default Click and Back actions |
| `PlatformInput` | `FPerPlatformSettings` — which input types each target platform supports |
| `bEnableEnhancedInputSupport` | Turns on the Enhanced Input path (requires an editor restart; read it back with `UCommonInputSettings::IsEnhancedInputSupportEnabled()`) |
| `bEnableDefaultInputConfig` | Applies a default input config when no active widget returns one from `GetDesiredInputConfig()` |
| `bEnableAutomaticGamepadTypeDetection` | Off means you call `UCommonInputSubsystem::SetGamepadInputType()` yourself |
| `bEnableInputMethodThrashingProtection`, `InputMethodThrashingLimit`, `InputMethodThrashingWindowInSeconds`, `InputMethodThrashingCooldownInSeconds` | Debounce rapid input-method flapping |
| `ActionDomainTable` | `TSoftObjectPtr<UCommonInputActionDomainTable>` — ordered action domains |

---

## Core Classes

### UCommonActivatableWidget

The foundation of Common UI's screen management. A widget that can be activated (brought to focus) and deactivated without being created or destroyed.

**Header:** `CommonActivatableWidget.h`
**Base class:** `UCommonUserWidget` → `UUserWidget`

```cpp
// Activation state
bool IsActivated() const;
void ActivateWidget();
void DeactivateWidget();

// Delegates (non-dynamic multicast)
FSimpleMulticastDelegate& OnActivated() const;
FSimpleMulticastDelegate& OnDeactivated() const;

// Focus
UWidget* GetDesiredFocusTarget() const;
void ClearFocusRestorationTarget();
void RequestRefreshFocus();

// Visibility binding to another widget's activation
void SetBindVisibilities(ESlateVisibility OnActivatedVisibility, ESlateVisibility OnDeactivatedVisibility, bool bInAllActive);
void BindVisibilityToActivation(UCommonActivatableWidget* ActivatableWidget);
```

Overridables — `GetActivationMetadata` and `GetDesiredInputConfig` are public (`CommonActivatableWidget.h:102,108`), the rest protected (`:140-183`); all expect a `Super` call except the pure getters:

```cpp
virtual void NativeOnActivated();
virtual void NativeOnDeactivated();
virtual UWidget* NativeGetDesiredFocusTarget() const;
virtual TOptional<FUIInputConfig> GetDesiredInputConfig() const;
virtual TOptional<FActivationMetadata> GetActivationMetadata() const;
virtual bool NativeOnHandleBackAction();
virtual void ActivateMappingContext();
virtual void DeactivateMappingContext();
virtual TSharedRef<SWidget> RebuildWidget() override;
virtual void OnWidgetRebuilt() override;
virtual void ReleaseSlateResources(bool bReleaseChildren) override;
virtual void NativeConstruct() override;
virtual void NativeDestruct() override;
```

`BP_GetDesiredFocusTarget` and `BP_GetDesiredInputConfig` are the `BlueprintImplementableEvent` counterparts that the native versions route to — override the native ones from C++.

**Key UPROPERTY settings (configure in Blueprint defaults or C++ constructor):**

```cpp
// Auto-activate when constructed (false by default)
UPROPERTY(EditAnywhere, Category = Activation)
bool bAutoActivate = false;

// Acts as a modal — blocks input routing to all ancestors
UPROPERTY(EditAnywhere, Category = Activation, meta = (EditCondition = bSupportsActivationFocus))
bool bIsModal = false;

// Receives "Back" action; override NativeOnHandleBackAction to act on it
UPROPERTY(EditAnywhere, Category = Back)
bool bIsBackHandler = false;

// Show the Back action in a bound action bar
UPROPERTY(EditAnywhere, Category = Back)
bool bIsBackActionDisplayedInActionBar = false;

// Restore focus to the previously-focused widget when re-activating
UPROPERTY(EditAnywhere, Category = Activation, meta = (EditCondition = bSupportsActivationFocus))
bool bAutoRestoreFocus = false;

// Set visibility automatically on activation/deactivation
UPROPERTY(EditAnywhere, Category = Activation)
bool bSetVisibilityOnActivated = false;
UPROPERTY(EditAnywhere, Category = Activation)
ESlateVisibility ActivatedVisibility = ESlateVisibility::SelfHitTestInvisible;
UPROPERTY(EditAnywhere, Category = Activation)
bool bSetVisibilityOnDeactivated = false;
UPROPERTY(EditAnywhere, Category = Activation)
ESlateVisibility DeactivatedVisibility = ESlateVisibility::Collapsed;
```

**Subclassing pattern:**

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
    virtual bool NativeOnHandleBackAction() override;

    void HandleConfirm();

    UPROPERTY(meta=(BindWidget)) TObjectPtr<UCommonButtonBase> CloseButton;
    UPROPERTY(meta=(BindWidget)) TObjectPtr<UCommonButtonBase> ConfirmButton;
};
```

```cpp
// MyMenuScreen.cpp
#include "MyMenuScreen.h"
#include "CommonButtonBase.h"

void UMyMenuScreen::NativeOnActivated()
{
    Super::NativeOnActivated();
    CloseButton->OnClicked().AddUObject(this, &UMyMenuScreen::DeactivateWidget);
    ConfirmButton->OnClicked().AddUObject(this, &UMyMenuScreen::HandleConfirm);
}

void UMyMenuScreen::NativeOnDeactivated()
{
    CloseButton->OnClicked().RemoveAll(this);
    ConfirmButton->OnClicked().RemoveAll(this);
    Super::NativeOnDeactivated();
}

UWidget* UMyMenuScreen::NativeGetDesiredFocusTarget() const
{
    // The first focusable button receives focus when this screen activates
    return ConfirmButton;
}

TOptional<FUIInputConfig> UMyMenuScreen::GetDesiredInputConfig() const
{
    // ECommonInputMode::Menu — enables gamepad menu navigation, hides cursor on gamepad
    return FUIInputConfig(ECommonInputMode::Menu, EMouseCaptureMode::NoCapture);
}

bool UMyMenuScreen::NativeOnHandleBackAction()
{
    DeactivateWidget();
    return true;   // Return false to let the action bubble to an ancestor
}

void UMyMenuScreen::HandleConfirm()
{
    DeactivateWidget();
}
```

---

### UCommonActivatableWidgetContainerBase and its subclasses

**Header:** `Widgets/CommonActivatableWidgetContainer.h`

`UCommonActivatableWidgetContainerBase` is an abstract `UWidget` that manages N activatable widgets and displays one at a time. Two concrete subclasses ship with the plugin:

- **`UCommonActivatableWidgetStack`** — only the top widget is displayed and activated; when it deactivates it is removed and the preceding entry is activated. An optional `RootContentWidgetClass` generates a permanent bottom element that can never be removed. Use this for game, menu and modal layers.
- **`UCommonActivatableWidgetQueue`** — one widget active at a time; when it deactivates it is removed, released to the pool, and the next queued widget is shown. Use this for notification toasts.

`UCommonActivatableWidgetSwitcher` is **not** a container in this sense — it inherits from `UCommonAnimatedSwitcher`/`UWidgetSwitcher`, is index-based, and has no `AddWidget()`.

Container API:

```cpp
// Templated add - returns nullptr if the class is not a child of ActivatableWidgetT
template <typename ActivatableWidgetT = UCommonActivatableWidget>
ActivatableWidgetT* AddWidget(TSubclassOf<UCommonActivatableWidget> ActivatableWidgetClass);

// Same, with an init callback that runs after creation and before the widget is added
template <typename ActivatableWidgetT = UCommonActivatableWidget>
ActivatableWidgetT* AddWidget(TSubclassOf<UCommonActivatableWidget> ActivatableWidgetClass,
                              TFunctionRef<void(ActivatableWidgetT&)> InstanceInitFunc);

void AddWidgetInstance(UCommonActivatableWidget& ActivatableWidget);  // Legacy; you own the instance
void RemoveWidget(UCommonActivatableWidget& WidgetToRemove);          // Reference, not pointer
UCommonActivatableWidget* GetActiveWidget() const;
const TArray<UCommonActivatableWidget*>& GetWidgetList() const;
int32 GetNumWidgets() const;
void ClearWidgets();
void SetTransitionDuration(float Duration);
float GetTransitionDuration() const;

// Events
FOnDisplayedWidgetChanged& OnDisplayedWidgetChanged() const;   // DECLARE_EVENT_OneParam(..., UCommonActivatableWidget*)
FTransitioningChanged OnTransitioningChanged;                  // Public member, DECLARE_EVENT_TwoParams(container, bIsTransitioning)

// UCommonActivatableWidgetStack only
UCommonActivatableWidget* GetRootContent() const;
```

```cpp
// In a root HUD widget:
UPROPERTY(meta=(BindWidget))
TObjectPtr<UCommonActivatableWidgetStack> MenuLayer;

void UMyHudWidget::ShowOptionsScreen()
{
    // AddWidget creates (or pools) the instance, adds it, and the stack activates it.
    UMyOptionsScreen* Screen = MenuLayer->AddWidget<UMyOptionsScreen>(UMyOptionsScreen::StaticClass());
    if (Screen)
    {
        Screen->SetIsEnabled(true);
    }
}

void UMyHudWidget::GoBack()
{
    if (UCommonActivatableWidget* Active = MenuLayer->GetActiveWidget())
    {
        Active->DeactivateWidget();   // The stack re-activates the widget beneath
    }
}
```

---

### UCommonButtonBase

**Header:** `CommonButtonBase.h`
**Base class:** `UCommonUserWidget`

A replacement for `UButton` that is aware of the current input method (mouse, gamepad, touch) and can display platform-appropriate icons (e.g., "A button" on Xbox, "Cross" on PlayStation).

```cpp
// OnClicked returns FCommonButtonEvent (DECLARE_EVENT-based) — NOT a DYNAMIC delegate
// Use AddUObject or AddWeakLambda instead of AddDynamic
FCommonButtonEvent& OnClicked() const;

// Example binding in NativeOnActivated (remove in NativeOnDeactivated)
MyButton->OnClicked().AddUObject(this, &UMyScreen::HandleButtonClicked);
MyButton->OnClicked().RemoveAll(this);

// Enable/disable
MyButton->SetIsEnabled(false);

// Selected state (toggle-style buttons)
MyButton->SetIsSelected(true);
bool bSelected = MyButton->GetSelected();

// Interaction gating (distinct from SetIsEnabled - keeps focus but ignores input)
MyButton->SetIsInteractionEnabled(true);
```

`FCommonButtonEvent` is `DECLARE_EVENT(UCommonButtonBase, FCommonButtonEvent)` — zero parameters, so handlers take no arguments. The full event set is `OnClicked()`, `OnDoubleClicked()`, `OnPressed()`, `OnReleased()`, `OnHovered()`, `OnUnhovered()`, `OnFocusReceived()`, `OnFocusLost()`, `OnLockClicked()`, `OnLockDoubleClicked()`. For Blueprint use there are the dynamic counterparts `FCommonButtonBaseClicked` and `FCommonSelectedStateChangedBase`.

Native overridables on `UCommonButtonBase` (each expecting a `Super` call): `NativeOnClicked`, `NativeOnDoubleClicked`, `NativeOnPressed`, `NativeOnReleased`, `NativeOnHovered`, `NativeOnUnhovered`, `NativeOnSelected(bool bBroadcast)`, `NativeOnDeselected(bool bBroadcast)`, `NativeOnEnabled`, `NativeOnDisabled`.

**Styling:** `UCommonButtonBase` uses a `UCommonButtonStyle` Blueprint class (`TSubclassOf<UCommonButtonStyle> Style`; the class CDO is read) instead of `FButtonStyle`. Configure this in the widget Blueprint.

---

### UCommonUIActionRouterBase

**Header:** `Input/CommonUIActionRouterBase.h`
**Base class:** `ULocalPlayerSubsystem`

The class is `UCommonUIActionRouterBase`; no shorter spelling of that name exists in 5.8. It decides which activatable widget receives input actions (confirm, back, etc.) from the current activation tree: each activatable widget gets a node when its Slate widget is rebuilt (`CommonActivatableWidget.cpp:258`, `CommonUIActionRouterBase.cpp:2046`), the node starts receiving input when the widget activates, and actions route to the leaf-most active node first.

```cpp
#include "Input/CommonUIActionRouterBase.h"

// Get() takes a const UWidget& — call it with any widget you already have in scope.
if (UCommonUIActionRouterBase* Router = UCommonUIActionRouterBase::Get(*ContextWidget))
{
    ECommonInputMode Mode = Router->GetActiveInputMode();                    // Default: All
    EMouseCaptureMode Capture = Router->GetActiveMouseCaptureMode();         // Default: NoCapture
    UCommonActivatableWidget* Leaf = Router->GetLeafmostActivatableWidget();
    UCommonInputSubsystem& Input = Router->GetInputSubsystem();
    TArray<FUIActionBindingHandle> Bindings = Router->GatherActiveBindings();
    Router->SetIsActivatableTreeEnabled(true);
    Router->FlushInput();

    Router->OnActiveInputModeChanged().AddUObject(this, &UMyScreen::HandleInputModeChanged);
    Router->OnBoundActionsUpdated().AddUObject(this, &UMyScreen::HandleBoundActionsUpdated);
}

// Static helpers that walk up the Slate tree to the nearest activatable widget:
UCommonActivatableWidget* Owner = UCommonUIActionRouterBase::FindOwningActivatable(SlateWidget, LocalPlayer);
UCommonActivatableWidget* Self  = UCommonUIActionRouterBase::FindActivatable(SlateWidget, LocalPlayer);
```

`RegisterLinkedPreprocessor(const UWidget&, const TSharedRef<IInputProcessor>&, int32 DesiredIndex)` is deprecated (`UE_DEPRECATED(5.5)`, `Input/CommonUIActionRouterBase.h:87`); call the two-argument overload or the one taking a `FInputPreprocessorRegistrationKey`.

You usually configure behaviour per widget rather than touching the router:
- `bIsBackHandler` / `NativeOnHandleBackAction()` on `UCommonActivatableWidget`
- `GetDesiredInputConfig()` override
- `InputMapping` / `InputMappingPriority` (Enhanced Input context) on `UCommonActivatableWidget`
- `UCommonUserWidget::RegisterUIActionBinding(const FBindUIActionArgs&)`

---

### UCommonInputSubsystem

Provides runtime information about the current input method and allows observing changes.

```cpp
// Get the subsystem
UCommonInputSubsystem* InputSubsystem = UCommonInputSubsystem::Get(GetOwningLocalPlayer());

// Query current input type. ECommonInputType (CommonInputTypeEnum.h):
// MouseAndKeyboard, Gamepad, Touch, Count — there is no None value.
ECommonInputType CurrentType = InputSubsystem->GetCurrentInputType();
ECommonInputType PlatformDefault = InputSubsystem->GetDefaultInputType();
bool bGamepad = InputSubsystem->IsInputMethodActive(ECommonInputType::Gamepad);
bool bPointer = InputSubsystem->IsUsingPointerInput();
FName GamepadName = InputSubsystem->GetCurrentGamepadName();

// Observe input type changes. OnInputMethodChangedNative is a DECLARE_EVENT_OneParam,
// so the handler is a plain member function - no UFUNCTION() needed.
InputSubsystem->OnInputMethodChangedNative.AddUObject(this, &UMyScreen::HandleInputMethodChanged);

void UMyScreen::HandleInputMethodChanged(ECommonInputType NewInputType)
{
    const bool bShowMouse = NewInputType == ECommonInputType::MouseAndKeyboard
                         || NewInputType == ECommonInputType::Touch;
    GetOwningPlayer()->SetShowMouseCursor(bShowMouse);
}
```

---

## Input Configuration

### FUIInputConfig

**Header:** `Input/UIActionBindingHandle.h` (CommonUI module). Returned from `UCommonActivatableWidget::GetDesiredInputConfig()` to define how input is handled while this widget is the active leaf.

```cpp
FUIInputConfig();
FUIInputConfig(ECommonInputMode InInputMode, EMouseCaptureMode InMouseCaptureMode,
               bool bInHideCursorDuringViewportCapture = true);
FUIInputConfig(ECommonInputMode InInputMode, EMouseCaptureMode InMouseCaptureMode,
               EMouseLockMode InMouseLockMode, bool bInHideCursorDuringViewportCapture = true);
```

```cpp
// Menu mode: gamepad navigates widgets, no mouse capture
return FUIInputConfig(ECommonInputMode::Menu, EMouseCaptureMode::NoCapture);

// Game mode: game receives all input
return FUIInputConfig(ECommonInputMode::Game, EMouseCaptureMode::CapturePermanently);

// All mode: both game and UI receive input (e.g., for HUD with tooltips)
return FUIInputConfig(ECommonInputMode::All, EMouseCaptureMode::NoCapture, EMouseLockMode::DoNotLock);
```

Read the config back with `GetInputMode()`, `GetMouseCaptureMode()`, `GetMouseLockMode()` and `HideCursorDuringViewportCapture()`; the public fields `bIgnoreMoveInput` and `bIgnoreLookInput` gate pawn movement and look while the config is active. `ECommonInputMode` comes from `CommonInputModeTypes.h` (CommonInput module); `EMouseCaptureMode` and `EMouseLockMode` come from `Engine/EngineBaseTypes.h`.

| `ECommonInputMode` | Description |
|---|---|
| `Menu` | Game input suspended. UI navigation active. |
| `Game` | Game receives input. UI does not route actions. |
| `All` | Both game and UI receive input simultaneously. |

### Enhanced Input Integration

Enhanced Input and Common Input are unified in 5.8. Set `bEnableEnhancedInputSupport=True` in `[/Script/CommonInput.CommonInputSettings]` (editor restart required) — the `InputMapping` properties below stay hidden in the details panel until it is on, gated by an `EditCondition` on `CommonInput.CommonInputSettings.IsEnhancedInputSupportEnabled`.

`UCommonActivatableWidget` then exposes:

```cpp
// CommonActivatableWidget.h:229-235 - set in the widget's Blueprint defaults or C++ constructor
UPROPERTY(EditAnywhere, Category="Input")
TObjectPtr<UInputMappingContext> InputMapping;

UPROPERTY(EditAnywhere, Category="Input")
int32 InputMappingPriority = 0;
```

The context is pushed by `ActivateMappingContext()` on activation and popped by `DeactivateMappingContext()` on deactivation; both are virtual, so you can override them.

`FBindUIActionArgs` (`Input/CommonUIInputTypes.h`) takes an Enhanced Input action directly:

```cpp
FBindUIActionArgs(FUIActionTag InActionTag, const FSimpleDelegate& InOnExecuteAction);
FBindUIActionArgs(FUIActionTag InActionTag, bool bShouldDisplayInActionBar, const FSimpleDelegate& InOnExecuteAction);
FBindUIActionArgs(const UInputAction* InInputAction, const FSimpleDelegate& InOnExecuteAction);
FBindUIActionArgs(const UInputAction* InInputAction, bool bShouldDisplayInActionBar, const FSimpleDelegate& InOnExecuteAction);
FBindUIActionArgs(const FDataTableRowHandle& InLegacyActionTableRow, const FSimpleDelegate& InOnExecuteAction);
```

```cpp
// UCommonUserWidget::RegisterUIActionBinding returns an FUIActionBindingHandle you keep and unregister.
FUIActionBindingHandle BackHandle = RegisterUIActionBinding(
    FBindUIActionArgs(BackInputAction, /*bShouldDisplayInActionBar=*/true,
        FSimpleDelegate::CreateUObject(this, &UMyMenuScreen::HandleBackPressed)));
```

`FUIActionBinding::TryCreate(const UWidget&, const FBindUIActionArgs&)` without a user index is deprecated (`UE_DEPRECATED(5.6)`, `Input/UIActionBinding.h:35`) — use the overload that takes `int32 UserIndex`.

---

## UI Layer Architecture Pattern

A typical Common UI hierarchy for a game with main menu, pause, and HUD:

```
AHUD or PlayerController
    └── UMyRootUIWidget (UUserWidget, AddToViewport ZOrder=0)
            ├── UCommonActivatableWidgetStack "GameLayer"
            │       └── UMyHUDWidget (always active during gameplay)
            ├── UCommonActivatableWidgetStack "MenuLayer"
            │       ├── UMyMainMenuWidget
            │       ├── UMyOptionsScreen
            │       └── UMyPauseMenuWidget
            └── UCommonActivatableWidgetStack "ModalLayer"
                    └── UMyConfirmationDialog (modal, bIsModal=true)
```

**Implementation:**

```cpp
// MyRootUIWidget.h
#pragma once

#include "Blueprint/UserWidget.h"
#include "Widgets/CommonActivatableWidgetContainer.h"
#include "MyRootUIWidget.generated.h"

UCLASS()
class MYGAME_API UMyRootUIWidget : public UUserWidget
{
    GENERATED_BODY()

public:
    // AddWidget<T> creates (or pools) the instance, pushes it, and the stack activates it.
    template <typename T>
    T* PushMenuWidget(TSubclassOf<T> WidgetClass)
    {
        return MenuLayer->AddWidget<T>(WidgetClass);
    }

    void PopMenuWidget()
    {
        if (UCommonActivatableWidget* Active = MenuLayer->GetActiveWidget())
        {
            Active->DeactivateWidget();
        }
    }

protected:
    UPROPERTY(meta=(BindWidget))
    TObjectPtr<UCommonActivatableWidgetStack> GameLayer;

    UPROPERTY(meta=(BindWidget))
    TObjectPtr<UCommonActivatableWidgetStack> MenuLayer;

    UPROPERTY(meta=(BindWidget))
    TObjectPtr<UCommonActivatableWidgetStack> ModalLayer;
};
```

---

## Common UI Gamepad Navigation

Common UI provides automatic focus management for gamepad navigation. When a `UCommonActivatableWidget` activates:

1. `GetDesiredFocusTarget()` is called.
2. The returned widget receives Slate keyboard focus.
3. Directional pad / left stick navigates between focusable widgets automatically.
4. "Back" / B-button / Escape triggers the back action if `bIsBackHandler = true`.

`NativeGetDesiredFocusTarget()` returns the widget that gets focus when the screen activates — normally the first interactive widget in tab order (see `UMyMenuScreen::NativeGetDesiredFocusTarget` above, which returns `ConfirmButton`). Call `RequestRefreshFocus()` when the target appears late (animated-in buttons, deeply nested switchers); it only re-focuses if this node is still the leaf-most active one. `ClearFocusRestorationTarget()` drops the cached target kept for `bAutoRestoreFocus`.

`UCommonButtonBase` is focusable by default. For a plain `UButton`, set it before the Slate widget is constructed — `InitIsFocusable` is `protected`, so call it from a `UButton` subclass constructor — and read it back with the getter — the `IsFocusable` field itself is deprecated (`UE_DEPRECATED(5.2)`, `Components/Button.h:67`):

```cpp
UMyFocusableButton::UMyFocusableButton() { InitIsFocusable(true); }   // protected, Components/Button.h:206
const bool bFocusable = MyPlainButton->GetIsFocusable();
```

---

## Deprecated — do not use

| Do not emit | Use in 5.8 | Source |
|---|---|---|
| `UCommonUIActionRouter` | `UCommonUIActionRouterBase` | Class name in `Input/CommonUIActionRouterBase.h:66` |
| `RegisterLinkedPreprocessor(Widget, Processor, int32 DesiredIndex)` | two-argument overload, or the `FInputPreprocessorRegistrationKey` overload | `UE_DEPRECATED(5.5)` in `Input/CommonUIActionRouterBase.h:87` |
| `FPreprocessorRegistration` | `FInputPreprocessorRegistration` | `UE_DEPRECATED(5.5)` in `Input/CommonUIActionRouterBase.h:219` |
| `FUIActionBinding::TryCreate(Widget, Args)` | `TryCreate(Widget, Args, UserIndex)` | `UE_DEPRECATED(5.6)` in `Input/UIActionBinding.h:35` |
| `FOnRerouteTouchInputDelegate` | `FTouchRerouteDelegate` (takes `FTouchId`) | `UE_DEPRECATED(5.8)` in `CommonGameViewportClient.h:15` |
| `UCommonGameViewportClient::OnRerouteTouchInput()` | `GetRerouteTouchRegistration()` | `UE_DEPRECATED(5.8)` in `CommonGameViewportClient.h:47` |
| `HandleRerouteTouch(FInputDeviceId, uint32 TouchId, ...)` | `HandleRerouteTouch(FTouchId, ETouchType::Type, const FVector2D&, FReply&)` | `UE_DEPRECATED(5.8)` in `CommonGameViewportClient.h:68` |
| `RerouteTouchInput` member | `GetRerouteTouchDelegate()` | `UE_DEPRECATED(5.8)` in `CommonGameViewportClient.h:87` |
| `#include "Input/CommonInputMode.h"` | `#include "CommonInputModeTypes.h"` (CommonInput module) | `UE_DEPRECATED_HEADER(5.1)` in `Input/CommonInputMode.h` |
| `ECommonInputType::None` | no such value — the enum is `MouseAndKeyboard, Gamepad, Touch, Count` | `CommonInputTypeEnum.h:9` |

`UCommonGameViewportClient` also overrides `InputTouch(FViewport*, const FTouchId TouchId, const ETouchType::Type, const FVector2D&, const float Force, const uint64 Timestamp)` — touch identity is an `FTouchId` handle, no longer a raw `uint32`.

---

## Common Mistakes

**Not removing the delegate binding when deactivated**
```cpp
// BAD: leaks bindings, fires after the screen is gone
void UMyScreen::NativeOnActivated() {
    MyButton->OnClicked().AddUObject(this, &UMyScreen::HandleClick);
    // Never removed!
}

// GOOD
void UMyScreen::NativeOnActivated() {
    MyButton->OnClicked().AddUObject(this, &UMyScreen::HandleClick);
}
void UMyScreen::NativeOnDeactivated() {
    MyButton->OnClicked().RemoveAll(this);
}
```

**Using AddDynamic with UCommonButtonBase::OnClicked()**
```cpp
// WRONG: OnClicked() returns FCommonButtonEvent, not a DYNAMIC delegate
MyButton->OnClicked().AddDynamic(this, &UMyScreen::HandleClick); // Compile error

// CORRECT
MyButton->OnClicked().AddUObject(this, &UMyScreen::HandleClick);
```

**Skipping the viewport client configuration**

Common UI's input routing requires `UCommonGameViewportClient`. Without it the router logs "CommonUI Input routing will not function correctly" (`CommonUIActionRouterBase.cpp:333-336`) and viewport input never reaches bound UI actions or back handlers first.

**Mixing Common UI and raw SetInputMode calls**

Once Common UI is active, avoid calling `GetOwningPlayer()->SetInputMode(FInputModeUIOnly())`. Common UI manages input mode internally through `FUIInputConfig`. Direct calls bypass the activation tree and cause conflicts.

```cpp
// BAD with Common UI
GetOwningPlayer()->SetInputMode(FInputModeUIOnly());

// GOOD: return the config from the activatable widget
TOptional<FUIInputConfig> UMyScreen::GetDesiredInputConfig() const {
    return FUIInputConfig(ECommonInputMode::Menu, EMouseCaptureMode::NoCapture);
}
```

**Not calling Super in activation overrides**
```cpp
// REQUIRED — Super sets up internal state, delegates, and focus routing
void UMyScreen::NativeOnActivated() {
    Super::NativeOnActivated(); // Must be called
    // Your code
}
void UMyScreen::NativeOnDeactivated() {
    // Your cleanup
    Super::NativeOnDeactivated(); // Must be called
}
```

---

## Checklist: Adding a New Screen

1. Create a `UCommonActivatableWidget` subclass (C++ or BP).
2. In C++: override `NativeOnActivated`, `NativeOnDeactivated`, `NativeGetDesiredFocusTarget`, `GetDesiredInputConfig`.
3. In the UMG Blueprint: design the layout using `UCommonButtonBase` for all interactive buttons.
4. Set `bIsBackHandler = true` if this screen should close on Back/Escape/B-button.
5. Set `bAutoActivate = false` (default) unless you need activation on construction.
6. Push via the appropriate layer stack (game, menu, or modal).
7. Verify focus flows correctly on gamepad by testing `NativeGetDesiredFocusTarget`.
8. Verify `GetDesiredInputConfig` returns the correct `ECommonInputMode`.
