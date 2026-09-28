# UMG Widget Type Reference

Target engine: **UE 5.8**. Verified against `Engine/Source/Runtime/UMG/Public/Components/` and `Engine/Source/Runtime/UMG/Public/Blueprint/`.

---

## UButton

**Header:** `Components/Button.h`
**Base class:** `UContentWidget` (single child slot)
**Slate backing:** `SButton`

### Key Delegates (BlueprintAssignable)

| Delegate | Type | Description |
|---|---|---|
| `OnClicked` | `FOnButtonClickedEvent` | Mouse release after press (default DownAndUp) |
| `OnPressed` | `FOnButtonPressedEvent` | Mouse/key press begins |
| `OnReleased` | `FOnButtonReleasedEvent` | Mouse/key released |
| `OnHovered` | `FOnButtonHoverEvent` | Cursor enters bounds |
| `OnUnhovered` | `FOnButtonHoverEvent` | Cursor leaves bounds |

### Key Methods

```cpp
void SetStyle(const FButtonStyle& InStyle);
const FButtonStyle& GetStyle() const;
void SetColorAndOpacity(FLinearColor InColorAndOpacity);
FLinearColor GetColorAndOpacity() const;
void SetBackgroundColor(FLinearColor InBackgroundColor);
FLinearColor GetBackgroundColor() const;
bool IsPressed() const;
void SetClickMethod(EButtonClickMethod::Type InClickMethod);
void SetTouchMethod(EButtonTouchMethod::Type InTouchMethod);
void SetPressMethod(EButtonPressMethod::Type InPressMethod);
void SetAllowDragDrop(bool bInAllowDragDrop);
```

### Click Methods

| `EButtonClickMethod` | Behavior |
|---|---|
| `DownAndUp` | Default. Click fires on mouse up after mouse down. |
| `MouseDown` | Fires immediately on mouse down. |
| `MouseUp` | Fires on mouse up regardless of where mouse down was. |
| `PreciseClick` | Inside a list: fires only on a precise tap, so dragging scrolls the list instead. |

### Deprecated field access

Direct access to `WidgetStyle`, `ColorAndOpacity`, `BackgroundColor`, `ClickMethod`, `TouchMethod`, `PressMethod` and `IsFocusable` is deprecated (`UE_DEPRECATED(5.2)`, `Components/Button.h:37-67`). Use the getters and setters above; focusability is read with `GetIsFocusable()`.

### BindWidget Example

```cpp
UPROPERTY(meta=(BindWidget))
TObjectPtr<UButton> ConfirmButton;

// In NativeConstruct (HandleConfirm is a UFUNCTION() on UMyWidget):
ConfirmButton->OnClicked.AddDynamic(this, &UMyWidget::HandleConfirm);
ConfirmButton->SetColorAndOpacity(FLinearColor(0.2f, 0.8f, 0.2f, 1.f));
```

---

## UTextBlock

**Header:** `Components/TextBlock.h`
**Base class:** `UTextLayoutWidget`
**Slate backing:** `STextBlock`

### Key Properties (always use the setters)

| Property | Type | Description |
|---|---|---|
| `Text` | `FText` | Displayed text |
| `Font` | `FSlateFontInfo` | Font, size, typeface |
| `ColorAndOpacity` | `FSlateColor` | Text color |
| `ShadowOffset` | `FVector2D` | Drop shadow offset |
| `ShadowColorAndOpacity` | `FLinearColor` | Drop shadow color |
| `MinDesiredWidth` | `float` | Minimum widget width |
| `TextTransformPolicy` | `ETextTransformPolicy` | None, ToLower, ToUpper |
| `TextOverflowPolicy` | `ETextOverflowPolicy` | Clip, Ellipsis, MultilineEllipsis |
| `bSimpleTextMode` | `bool` | Fast path for ASCII/numeric text |

### Key Methods

```cpp
FText GetText() const;
void SetText(FText InText);                                   // Wipes Blueprint binding!
void SetColorAndOpacity(FSlateColor InColorAndOpacity);
void SetOpacity(float InOpacity);
void SetShadowColorAndOpacity(FLinearColor InShadowColorAndOpacity);
void SetShadowOffset(FVector2D InShadowOffset);
void SetFont(FSlateFontInfo InFontInfo);
void SetStrikeBrush(FSlateBrush InStrikeBrush);
void SetMinDesiredWidth(float InMinDesiredWidth);
void SetAutoWrapText(bool InAutoTextWrap);
void SetTextTransformPolicy(ETextTransformPolicy InTransformPolicy);
void SetTextOverflowPolicy(ETextOverflowPolicy InOverflowPolicy);
void SetFontMaterial(UMaterialInterface* InMaterial);
void SetFontOutlineMaterial(UMaterialInterface* InMaterial);
UMaterialInstanceDynamic* GetDynamicFontMaterial();
UMaterialInstanceDynamic* GetDynamicOutlineMaterial();
```

### Common Patterns

```cpp
// Simple string (avoid — prefer FText for localization)
ScoreLabel->SetText(FText::FromString(TEXT("Score: 100")));

// Localized with format argument
ScoreLabel->SetText(FText::Format(
    NSLOCTEXT("HUD", "ScoreFmt", "Score: {0}"),
    FText::AsNumber(Score)));

// Number with grouping (e.g., "1,234,567")
ScoreLabel->SetText(FText::AsNumber(Score));

// Uppercase transform
ScoreLabel->SetTextTransformPolicy(ETextTransformPolicy::ToUpper);

// Ellipsis for overflow
ScoreLabel->SetTextOverflowPolicy(ETextOverflowPolicy::Ellipsis);

// Resize the current font (FCoreStyle::GetDefaultFont() returns a FCompositeFont ref, not an FSlateFontInfo)
FSlateFontInfo FontInfo = ScoreLabel->GetFont();
FontInfo.Size = 32;
ScoreLabel->SetFont(FontInfo);
```

---

## UImage

**Header:** `Components/Image.h`
**Base class:** `UWidget` (no children)
**Slate backing:** `SImage`

### Key Properties

| Property | Type | Description |
|---|---|---|
| `Brush` | `FSlateBrush` | The image resource and drawing settings |
| `ColorAndOpacity` | `FLinearColor` | Tint applied to the image |
| `bFlipForRightToLeftFlowDirection` | `bool` | RTL localization support |

### Key Methods

```cpp
void SetColorAndOpacity(FLinearColor InColorAndOpacity);
void SetOpacity(float InOpacity);
void SetBrush(const FSlateBrush& InBrush);
const FSlateBrush& GetBrush() const;
void SetBrushFromAsset(USlateBrushAsset* Asset);
void SetBrushFromTexture(UTexture2D* Texture, bool bMatchSize = false);
void SetBrushFromTextureDynamic(UTexture2DDynamic* Texture, bool bMatchSize = false);
void SetBrushFromMaterial(UMaterialInterface* Material);
void SetBrushFromSoftTexture(TSoftObjectPtr<UTexture2D> SoftTexture, bool bMatchSize = false);
void SetBrushFromSoftMaterial(TSoftObjectPtr<UMaterialInterface> SoftMaterial);
void SetBrushFromAtlasInterface(TScriptInterface<ISlateTextureAtlasInterface> AtlasRegion, bool bMatchSize = false);
void SetDesiredSizeOverride(FVector2D DesiredSize);
void SetBrushTintColor(FSlateColor TintColor);
void SetBrushResourceObject(UObject* ResourceObject);
UMaterialInstanceDynamic* GetDynamicMaterial();
void SetFlipForRightToLeftFlowDirection(bool bFlip);
```

### Common Patterns

```cpp
// Static texture
AvatarImage->SetBrushFromTexture(LoadObject<UTexture2D>(nullptr, TEXT("/Game/UI/T_Avatar")));

// Material instance dynamic for animation or parameters
AvatarImage->SetBrushFromMaterial(AvatarMaterial);
UMaterialInstanceDynamic* MID = AvatarImage->GetDynamicMaterial();
MID->SetScalarParameterValue(TEXT("GlowIntensity"), 1.5f);
MID->SetVectorParameterValue(TEXT("Tint"), FLinearColor::Yellow);

// Async stream from soft pointer (no blocking load)
AvatarImage->SetBrushFromSoftTexture(SoftAvatarTexture, /*bMatchSize=*/true);

// Size override
AvatarImage->SetDesiredSizeOverride(FVector2D(64.f, 64.f));

// Tint without touching the brush
AvatarImage->SetColorAndOpacity(FLinearColor(1.f, 0.5f, 0.5f, 1.f));
```

---

## UProgressBar

**Header:** `Components/ProgressBar.h`
**Base class:** `UWidget` (no children)
**Slate backing:** `SProgressBar`

### Key Properties

| Property | Type | Description |
|---|---|---|
| `Percent` | `float` | Fill value 0.0–1.0 |
| `BarFillType` | `EProgressBarFillType` | LeftToRight, RightToLeft, FillFromCenter, etc. |
| `BarFillStyle` | `EProgressBarFillStyle` | Scale or Mask |
| `bIsMarquee` | `bool` | Indeterminate animation |
| `FillColorAndOpacity` | `FLinearColor` | Fill color |

### Key Methods

```cpp
float GetPercent() const;
void SetPercent(float InPercent);                  // Primary runtime update
void SetFillColorAndOpacity(FLinearColor InColor);
FLinearColor GetFillColorAndOpacity() const;
void SetIsMarquee(bool InbIsMarquee);
bool UseMarquee() const;
void SetBarFillType(EProgressBarFillType::Type InBarFillType);
void SetBarFillStyle(EProgressBarFillStyle::Type InBarFillStyle);
void SetBorderPadding(FVector2D InBorderPadding);
```

### Fill Types

| `EProgressBarFillType` | Description |
|---|---|
| `LeftToRight` | Default, fills left to right |
| `RightToLeft` | Fills right to left |
| `FillFromCenter` | Scales outward from the midpoint on both axes |
| `FillFromCenterHorizontal` | Fills outward from center (horizontal) |
| `FillFromCenterVertical` | Fills up/down from vertical center |
| `TopToBottom` | Fills top to bottom |
| `BottomToTop` | Fills bottom to top |

### Common Patterns

```cpp
// Health bar
HealthBar->SetPercent(Health / MaxHealth);
HealthBar->SetFillColorAndOpacity(Health > MaxHealth * 0.3f
    ? FLinearColor(0.f, 0.8f, 0.f, 1.f)  // Green
    : FLinearColor(0.9f, 0.1f, 0.f, 1.f));// Red

// Loading spinner (marquee)
LoadingBar->SetIsMarquee(true);

// Experience bar filling right-to-left
XPBar->SetBarFillType(EProgressBarFillType::RightToLeft);
XPBar->SetPercent(XP / XPToNextLevel);
```

---

## UListView

**Header:** `Components/ListView.h`
**Base class:** `UListViewBase`, `ITypedUMGListView<UObject*>`
**Slate backing:** `SListView<UObject*>`

The list is virtualized: only visible entry widgets exist. Items are data objects; entry widgets are reused as the list scrolls.

### Key Delegates

| Delegate | Signature | Description |
|---|---|---|
| `BP_OnItemClicked` | `(UObject* Item)` | Item clicked |
| `BP_OnItemDoubleClicked` | `(UObject* Item)` | Item double-clicked |
| `BP_OnItemSelectionChanged` | `(UObject* Item, bool bIsSelected)` | Selection changed |
| `BP_OnEntryInitialized` | `(UObject* Item, UUserWidget* Widget)` | Entry widget assigned |
| `BP_OnItemScrolledIntoView` | `(UObject* Item, UUserWidget* Widget)` | Item scrolled visible |

### Key Methods

```cpp
// Add / remove items
void AddItem(UObject* Item);
void RemoveItem(UObject* Item);
void ClearListItems();
void SetListItems(const TArray<ItemObjectT, AllocatorType>& InListItems); // Template

// Query
const TArray<UObject*>& GetListItems() const;
UObject* GetItemAt(int32 Index) const;
int32 GetNumItems() const;
int32 GetIndexForItem(const UObject* Item) const;
bool IsRefreshPending() const;

// Selection (single-selection recommended for GetSelectedItem)
void SetSelectedItem(const UObject* Item);
void SetSelectedIndex(int32 Index);
UObject* GetSelectedItem() const;               // Template: GetSelectedItem<UMyItemData>()
bool BP_GetSelectedItems(TArray<UObject*>& Items) const;
int32 BP_GetNumItemsSelected() const;
void BP_ClearSelection();
void SetSelectionMode(TEnumAsByte<ESelectionMode::Type> SelectionMode);

// Navigation and scrolling
void ScrollIndexIntoView(int32 Index);
void NavigateToIndex(int32 Index);
void BP_ScrollItemIntoView(UObject* Item);
void BP_NavigateToItem(UObject* Item);
bool BP_IsItemVisible(UObject* Item) const;

// Entry widget retrieval (only valid when the entry is visible)
RowWidgetT* GetEntryWidgetFromItem(const UObject* Item) const; // Template
```

### IUserObjectListEntry (entry widget interface)

```cpp
// MyItemEntry.h
#pragma once

#include "Blueprint/IUserObjectListEntry.h"
#include "Blueprint/UserWidget.h"
#include "MyItemEntry.generated.h"

class UTextBlock;

UCLASS()
class MYGAME_API UMyItemEntry : public UUserWidget, public IUserObjectListEntry
{
    GENERATED_BODY()

protected:
    // Called each time the entry is assigned a new item object
    virtual void NativeOnListItemObjectSet(UObject* ListItemObject) override;

    UPROPERTY(meta=(BindWidget))
    TObjectPtr<UTextBlock> NameText;
};

// MyItemEntry.cpp
#include "MyItemEntry.h"
#include "Components/TextBlock.h"

void UMyItemEntry::NativeOnListItemObjectSet(UObject* ListItemObject)
{
    IUserObjectListEntry::NativeOnListItemObjectSet(ListItemObject);   // Routes the event to BP
    if (UMyItemData* Data = Cast<UMyItemData>(ListItemObject))
    {
        NameText->SetText(Data->DisplayName);
    }
}
```

Other overridables from `IUserListEntry` (`Blueprint/IUserListEntry.h`), each expecting a Super call:

```cpp
virtual void NativeOnItemSelectionChanged(bool bIsSelected);
virtual void NativeOnItemExpansionChanged(bool bIsExpanded);   // TreeView only
virtual void NativeOnEntryReleased();
virtual bool IsListItemSelectable() const;                     // Native-only; return false for separators
```

Inside the entry widget, read the item with the template accessor `GetListItem<UMyItemData>()` and query state with the interface members `IsListItemSelected()`, `IsListItemExpanded()` and `GetOwningListView()` (`Blueprint/IUserListEntry.h:32-38`); the static libraries below are the Blueprint-facing equivalents:

```cpp
// UUserListEntryLibrary (Blueprint/IUserListEntry.h)
bool bSelected = UUserListEntryLibrary::IsListItemSelected(this);
bool bExpanded = UUserListEntryLibrary::IsListItemExpanded(this);
// UUserObjectListEntryLibrary (Blueprint/IUserObjectListEntry.h)
UObject* Item  = UUserObjectListEntryLibrary::GetListItemObject(this);
int32 Index    = UUserObjectListEntryLibrary::GetListItemIndex(this);
bool bFirst    = UUserObjectListEntryLibrary::IsFirstWidget(this);
bool bLast     = UUserObjectListEntryLibrary::IsLastWidget(this);
```

### UTileView and UTreeView

`UTileView` works identically to `UListView` but arranges entries in a 2D grid; tile width and height are set on the `UTileView`. `UTreeView` adds expansion state and uses `IUserObjectListEntry` too. Both share the `UListViewBase` / `ITypedUMGListView<UObject*>` API above.

### C++ selection and click handling

`BP_OnItemClicked`, `BP_OnItemDoubleClicked` and `BP_OnItemSelectionChanged` are Blueprint-only private delegates. From C++, subclass `UListView` and override the `ListViewBase` hooks:

```cpp
virtual void OnItemClickedInternal(UObject* Item) override;
virtual void OnItemDoubleClickedInternal(UObject* Item) override;
virtual void OnSelectionChangedInternal(UObject* FirstSelectedItem) override;
```

`InitHorizontalEntrySpacing` and `InitVerticalEntrySpacing` are deprecated (`UE_DEPRECATED(5.6)`, `Components/ListView.h:343,346`) — call `SetHorizontalEntrySpacing` / `SetVerticalEntrySpacing`.

---

## UScrollBox

**Header:** `Components/ScrollBox.h`
**Base class:** `UPanelWidget` (multiple children)

```cpp
// Programmatic scroll
MyScrollBox->ScrollToStart();
MyScrollBox->ScrollToEnd();
MyScrollBox->ScrollWidgetIntoView(ChildWidget, /*AnimateScroll=*/true);
MyScrollBox->SetScrollOffset(200.f);
float Offset = MyScrollBox->GetScrollOffset();
float EndOffset = MyScrollBox->GetScrollOffsetOfEnd();
MyScrollBox->EndInertialScrolling();

// Layout and behaviour
MyScrollBox->SetOrientation(Orient_Vertical);            // EOrientation: Orient_Horizontal, Orient_Vertical
MyScrollBox->SetScrollBarVisibility(ESlateVisibility::Collapsed);
MyScrollBox->SetAlwaysShowScrollbar(false);
MyScrollBox->SetAllowOverscroll(true);
MyScrollBox->SetAnimateWheelScrolling(true);
MyScrollBox->SetWheelScrollMultiplier(1.5f);
MyScrollBox->SetConsumeMouseWheel(EConsumeMouseWheel::WhenScrollingPossible);
```

Full signature: `ScrollWidgetIntoView(UWidget* WidgetToFind, bool AnimateScroll = true, EDescendantScrollDestination ScrollDestination = EDescendantScrollDestination::IntoView, float Padding = 0)`.

---

## UWidgetSwitcher

**Header:** `Components/WidgetSwitcher.h`

```cpp
// Only one child is visible at a time (zero-based index)
MySwitcher->SetActiveWidgetIndex(1);
MySwitcher->SetActiveWidget(MyChildWidget);
int32 Index = MySwitcher->GetActiveWidgetIndex();
UWidget* Active = MySwitcher->GetActiveWidget();
int32 Count = MySwitcher->GetNumWidgets();
```

---

## UCheckBox

```cpp
// State
bool bChecked = MyCheckBox->IsChecked();
MyCheckBox->SetIsChecked(true);
ECheckBoxState State = MyCheckBox->GetCheckedState();
// ECheckBoxState: Unchecked, Checked, Undetermined

// Delegate
MyCheckBox->OnCheckStateChanged.AddDynamic(this, &UMyWidget::HandleCheckChanged);
// Signature: void HandleCheckChanged(bool bIsChecked)
```

---

## UEditableTextBox

```cpp
FText Text = MyInput->GetText();
MyInput->SetText(FText::FromString(TEXT("Default")));
MyInput->SetHintText(NSLOCTEXT("UI", "SearchHint", "Search..."));
MyInput->SetIsReadOnly(true);
MyInput->SetIsPassword(true); // Mask characters

// Delegates
MyInput->OnTextChanged.AddDynamic(this, &UMyWidget::HandleTextChanged);
MyInput->OnTextCommitted.AddDynamic(this, &UMyWidget::HandleTextCommitted);
// Committed signature: void(const FText& Text, ETextCommit::Type CommitMethod)
// ETextCommit: OnEnter, OnUserMovedFocus, OnCleared, Default
```

---

## USlider

```cpp
float Value = MySlider->GetValue();       // In [MinValue, MaxValue] (default 0 – 1)
MySlider->SetValue(0.5f);
MySlider->SetMinValue(0.f);
MySlider->SetMaxValue(100.f);
MySlider->SetStepSize(1.f);
MySlider->SetOrientation(EOrientation::Orient_Horizontal);

MySlider->OnValueChanged.AddDynamic(this, &UMyWidget::HandleSliderChanged);
// Signature: void(float Value)
```

---

## UComboBoxString

```cpp
MyCombo->AddOption(TEXT("Option A"));
MyCombo->AddOption(TEXT("Option B"));
MyCombo->SetSelectedOption(TEXT("Option A"));
FString Selected = MyCombo->GetSelectedOption();
int32 Count = MyCombo->GetOptionCount();
MyCombo->ClearOptions();
MyCombo->RefreshOptions();

MyCombo->OnSelectionChanged.AddDynamic(this, &UMyWidget::HandleComboChanged);
// Signature: void(FString SelectedItem, ESelectInfo::Type SelectionType)
```

---

## Visibility Reference

| `ESlateVisibility` | Rendered | Takes Space | Receives Input |
|---|---|---|---|
| `Visible` | Yes | Yes | Yes |
| `Collapsed` | No | No | No |
| `Hidden` | No | Yes | No |
| `HitTestInvisible` | Yes | Yes | No (passes through) |
| `SelfHitTestInvisible` | Yes | Yes | Children only |

```cpp
Widget->SetVisibility(ESlateVisibility::Collapsed);
ESlateVisibility V = Widget->GetVisibility();
bool bVisible = V == ESlateVisibility::Visible;
```

---

## UWidget Base (All Widgets)

All UMG widgets inherit from `UWidget`:

```cpp
// Enable/disable (grays out and blocks input)
Widget->SetIsEnabled(false);
bool bEnabled = Widget->GetIsEnabled();

// Render opacity (does not affect layout or hit testing)
Widget->SetRenderOpacity(0.5f);
float Opacity = Widget->GetRenderOpacity();

// Transform (applied as render transform, not layout)
Widget->SetRenderTransformAngle(45.f);
Widget->SetRenderTranslation(FVector2D(10.f, 0.f));
Widget->SetRenderScale(FVector2D(1.2f, 1.2f));

// Tooltip
Widget->SetToolTipText(NSLOCTEXT("UI", "Tip", "Click to confirm"));

// Cursor
Widget->SetCursor(EMouseCursor::Hand);

// Focus
Widget->SetKeyboardFocus();
bool bHasFocus = Widget->HasKeyboardFocus();

// Slate widget access (for Slate-only APIs)
TSharedPtr<SWidget> SlateWidget = Widget->GetCachedWidget();
```

---

## UWidgetTree

`UUserWidget::WidgetTree` owns every widget built from the Widget Blueprint.

```cpp
// Iteration (Blueprint/WidgetTree.h)
WidgetTree->ForEachWidget([](UWidget* Widget) { Widget->SetRenderOpacity(1.f); });
WidgetTree->ForEachWidgetAndDescendants([](UWidget* Widget) { Widget->SetIsEnabled(true); });
// ForEachWidgetUntil exists but is not UMG_API-exported (WidgetTree.h:96): LNK2019 outside UMG
UWidgetTree::ForWidgetAndChildren(RootWidget, [](UWidget* Widget) { Widget->SetCursor(EMouseCursor::Default); });

// Lookup
UButton* Found = WidgetTree->FindWidget<UButton>(TEXT("PlayButton"));
UWidget* ByName = WidgetTree->FindWidget(FName(TEXT("PlayButton")));

TArray<UWidget*> AllWidgets;
WidgetTree->GetAllWidgets(AllWidgets);

TArray<UWidget*> Children;
UWidgetTree::GetChildWidgets(ParentWidget, Children);

int32 ChildIndex = INDEX_NONE;
UPanelWidget* Parent = UWidgetTree::FindWidgetParent(Found, ChildIndex);

// Construction and removal (ConstructWidget forwards UUserWidget subclasses to CreateWidget(this, ...), WidgetTree.h:110)
UMyItemEntry* Entry = WidgetTree->ConstructWidget<UMyItemEntry>(EntryWidgetClass);
WidgetTree->RemoveWidget(Entry);
```

---

## UWidgetBlueprintLibrary

**Header:** `Blueprint/WidgetBlueprintLibrary.h` — static helpers, each a `UFUNCTION(BlueprintCallable)` or `UFUNCTION(BlueprintPure)`.

```cpp
// Brushes
FSlateBrush TextureBrush  = UWidgetBlueprintLibrary::MakeBrushFromTexture(MyTexture, 64, 64);
FSlateBrush MaterialBrush = UWidgetBlueprintLibrary::MakeBrushFromMaterial(MyMaterial, 32, 32);
FSlateBrush EmptyBrush    = UWidgetBlueprintLibrary::NoResourceBrush();
UMaterialInstanceDynamic* BrushMID = UWidgetBlueprintLibrary::GetDynamicMaterial(MaterialBrush);
UTexture2D* BrushTexture = UWidgetBlueprintLibrary::GetBrushResourceAsTexture2D(TextureBrush);

// Event replies (FEventReply, the Blueprint-facing wrapper around FReply)
FEventReply Reply = UWidgetBlueprintLibrary::Handled();
Reply = UWidgetBlueprintLibrary::CaptureMouse(Reply, CapturingWidget);
Reply = UWidgetBlueprintLibrary::SetUserFocus(Reply, FocusWidget, /*bInAllUsers=*/false);
Reply = UWidgetBlueprintLibrary::DetectDrag(Reply, WidgetDetectingDrag, EKeys::LeftMouseButton);
Reply = UWidgetBlueprintLibrary::EndDragDrop(Reply);
FEventReply Unhandled = UWidgetBlueprintLibrary::Unhandled();

// Drag and drop state
bool bDragging = UWidgetBlueprintLibrary::IsDragDropping();
UDragDropOperation* Payload = UWidgetBlueprintLibrary::GetDragDroppingContent();
UWidgetBlueprintLibrary::CancelDragDrop();
UWidgetBlueprintLibrary::DismissAllMenus();

// Custom painting - only valid inside NativePaint / OnPaint, where Context is the FPaintContext
UWidgetBlueprintLibrary::DrawLine(Context, FVector2D(0.f, 0.f), FVector2D(100.f, 0.f),
    FLinearColor::White, /*bAntiAlias=*/true, /*Thickness=*/1.f);
// DrawText (FString) is meta=DeprecatedFunction; use DrawTextFormatted (WidgetBlueprintLibrary.h:117,127)
UWidgetBlueprintLibrary::DrawTextFormatted(Context, NSLOCTEXT("HUD", "Tag", "HUD"), FVector2D(8.f, 8.f), MyFont, 16.f);
```

Input-mode helpers also live here: `SetInputMode_UIOnlyEx`, `SetInputMode_GameAndUIEx`, `SetInputMode_GameOnly`, `SetFocusToGameViewport` — prefer the `FInputModeUIOnly` / `FInputModeGameAndUI` / `FInputModeGameOnly` structs on `APlayerController` from C++ (see `ue-input-system`).

---

## USlateBlueprintLibrary

**Header:** `Blueprint/SlateBlueprintLibrary.h` — geometry space conversions.

```cpp
// MyGeometry is the FGeometry handed to NativePaint / NativeTick / an input handler.
FVector2D Absolute = USlateBlueprintLibrary::LocalToAbsolute(MyGeometry, LocalPoint);
FVector2D Local    = USlateBlueprintLibrary::AbsoluteToLocal(MyGeometry, AbsolutePoint);
FVector2D PixelPos, ViewportPos;
USlateBlueprintLibrary::LocalToViewport(this, MyGeometry, LocalPoint, PixelPos, ViewportPos);
USlateBlueprintLibrary::ScreenToWidgetLocal(this, MyGeometry, ScreenPoint, Local);
FVector2D VectorOut = USlateBlueprintLibrary::Vector_LocalToAbsolute(MyGeometry, LocalVector);
float ScalarOut     = USlateBlueprintLibrary::Scalar_AbsoluteToLocal(MyGeometry, AbsoluteScalar);
```

`TransformScalarAbsoluteToLocal`, `TransformScalarLocalToAbsolute`, `TransformVectorAbsoluteToLocal` and `TransformVectorLocalToAbsolute` are deprecated (`UE_DEPRECATED(5.6)`, `Blueprint/SlateBlueprintLibrary.h:77-92`) — they returned inverted results, so the replacements are the opposite-direction `Scalar_*` / `Vector_*` functions.
