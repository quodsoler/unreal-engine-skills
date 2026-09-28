# Detail Customization Patterns

Reference for `IDetailCustomization` and `IPropertyTypeCustomization` in UE 5.8. Every signature below is copied from `Engine/Source/Editor/PropertyEditor/Public`. Module: `PropertyEditor`.

Header names to include: `IDetailCustomization.h`, `IPropertyTypeCustomization.h`, `DetailLayoutBuilder.h`, `DetailCategoryBuilder.h`, `IDetailChildrenBuilder.h`, `IDetailPropertyRow.h`, `DetailWidgetRow.h`, `PropertyHandle.h`, `PropertyCustomizationHelpers.h`, `PropertyEditorModule.h`.

---

## IDetailLayoutBuilder — verbatim signatures

From `DetailLayoutBuilder.h`:

```cpp
static FSlateFontInfo GetDetailFont();
static FSlateFontInfo GetDetailFontBold();
static FSlateFontInfo GetDetailFontItalic();

virtual const TArray<TWeakObjectPtr<UObject>>& GetSelectedObjects() const = 0;
virtual void GetObjectsBeingCustomized(TArray<TWeakObjectPtr<UObject>>& OutObjects) const = 0;

virtual IDetailCategoryBuilder& EditCategory(FName CategoryName,
    const FText& NewLocalizedDisplayName = FText::GetEmpty(),
    ECategoryPriority::Type CategoryType = ECategoryPriority::Default) = 0;
virtual IDetailCategoryBuilder& EditCategoryAllowNone(FName CategoryName,
    const FText& NewLocalizedDisplayName = FText::GetEmpty(),
    ECategoryPriority::Type CategoryType = ECategoryPriority::Default) = 0;
virtual void HideCategory(FName CategoryName) = 0;

virtual IDetailPropertyRow& AddPropertyToCategory(TSharedPtr<IPropertyHandle> InPropertyHandle) = 0;

virtual TSharedRef<IPropertyHandle> GetProperty(const FName PropertyPath,
    const UStruct* ClassOutermost = NULL, FName InstanceName = NAME_None) const = 0;
virtual void HideProperty(const TSharedPtr<IPropertyHandle> PropertyHandle) = 0;
virtual void HideProperty(FName PropertyPath, const UStruct* ClassOutermost = NULL,
    FName InstanceName = NAME_None) = 0;

virtual void ForceRefreshDetails() = 0;

virtual void RegisterInstancedCustomPropertyTypeLayout(FName PropertyTypeName,
    FOnGetPropertyTypeCustomizationInstance PropertyTypeLayoutDelegate,
    TSharedPtr<IPropertyTypeIdentifier> Identifier = nullptr) = 0;

virtual TSharedPtr<IDetailsView> GetDetailsViewSharedPtr() = 0;
```

`ECategoryPriority::Type` (same header), highest first: `Variable`, `Transform`, `Important`, `TypeSpecific`, `Default`, `Uncommon`.

```cpp
IDetailCategoryBuilder& CoreCategory =
    DetailBuilder.EditCategory("Core", FText::GetEmpty(), ECategoryPriority::Important);
IDetailCategoryBuilder& AdvancedCategory =
    DetailBuilder.EditCategory("Advanced", FText::GetEmpty(), ECategoryPriority::Uncommon);
```

---

## Full class customization

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
```

```cpp
// MyDataAssetCustomization.cpp
#include "MyDataAssetCustomization.h"
#include "MyDataAsset.h"
#include "DetailLayoutBuilder.h"
#include "DetailCategoryBuilder.h"
#include "DetailWidgetRow.h"
#include "IDetailPropertyRow.h"
#include "PropertyHandle.h"
#include "Widgets/Input/SButton.h"
#include "Widgets/Text/STextBlock.h"
#include "Widgets/SBoxPanel.h"

TSharedRef<IDetailCustomization> FMyDataAssetCustomization::MakeInstance()
{
    return MakeShared<FMyDataAssetCustomization>();
}

void FMyDataAssetCustomization::CustomizeDetails(const TSharedPtr<IDetailLayoutBuilder>& DetailBuilder)
{
    WeakBuilder = DetailBuilder;
    CustomizeDetails(*DetailBuilder);
}

void FMyDataAssetCustomization::CustomizeDetails(IDetailLayoutBuilder& DetailBuilder)
{
    TArray<TWeakObjectPtr<UObject>> Objects;
    DetailBuilder.GetObjectsBeingCustomized(Objects);
    if (Objects.Num() != 1)
    {
        return;
    }

    IDetailCategoryBuilder& Category =
        DetailBuilder.EditCategory("Tuning", FText::GetEmpty(), ECategoryPriority::Important);

    TSharedRef<IPropertyHandle> SpeedHandle =
        DetailBuilder.GetProperty(GET_MEMBER_NAME_CHECKED(UMyDataAsset, Speed));
    IDetailPropertyRow& SpeedRow = Category.AddProperty(SpeedHandle);
    SpeedRow.DisplayName(FText::FromString("Speed"))
            .ToolTip(FText::FromString("Movement speed in cm/s"));

    DetailBuilder.HideProperty(
        DetailBuilder.GetProperty(GET_MEMBER_NAME_CHECKED(UMyDataAsset, InternalCache)));

    // Rebuild the layout when the value changes in a way that changes which rows exist.
    TWeakPtr<IDetailLayoutBuilder> LocalBuilder = WeakBuilder;
    SpeedHandle->SetOnPropertyValueChanged(FSimpleDelegate::CreateLambda([LocalBuilder]()
    {
        if (TSharedPtr<IDetailLayoutBuilder> Pinned = LocalBuilder.Pin())
        {
            Pinned->ForceRefreshDetails();
        }
    }));

    Category.AddCustomRow(FText::FromString("Reset Speed"))
    .NameContent()
    [
        SNew(STextBlock)
        .Text(FText::FromString("Reset"))
        .Font(IDetailLayoutBuilder::GetDetailFont())
    ]
    .ValueContent()
    .MinDesiredWidth(125.f)
    .MaxDesiredWidth(400.f)
    [
        SNew(SButton)
        .Text(FText::FromString("Reset Speed"))
        .OnClicked_Lambda([SpeedHandle]()
        {
            SpeedHandle->SetValue(600.f);
            return FReply::Handled();
        })
    ];
}
```

---

## IDetailCategoryBuilder and IDetailChildrenBuilder

`DetailCategoryBuilder.h`:

```cpp
virtual IDetailCategoryBuilder& InitiallyCollapsed(bool bShouldBeInitiallyCollapsed) = 0;
virtual IDetailCategoryBuilder& HeaderContent(TSharedRef<SWidget> InHeaderContent,
    bool bWholeRowContent = false) = 0;
virtual void SetSortOrder(int32 InSortOrder) = 0;

virtual IDetailPropertyRow& AddProperty(FName PropertyPath, UClass* ClassOutermost = nullptr,
    FName InstanceName = NAME_None,
    EPropertyLocation::Type Location = EPropertyLocation::Default) = 0;
virtual IDetailPropertyRow& AddProperty(TSharedPtr<IPropertyHandle> PropertyHandle,
    EPropertyLocation::Type Location = EPropertyLocation::Default) = 0;

virtual IDetailPropertyRow* AddExternalObjects(const TArray<UObject*>& Objects,
    EPropertyLocation::Type Location = EPropertyLocation::Default,
    const FAddPropertyParams& Params = FAddPropertyParams()) = 0;
virtual IDetailPropertyRow* AddExternalStructure(TSharedPtr<FStructOnScope> StructData,
    EPropertyLocation::Type Location = EPropertyLocation::Default) = 0;

virtual FDetailWidgetRow& AddCustomRow(const FText& FilterString, bool bForAdvanced = false) = 0;
virtual void AddCustomBuilder(TSharedRef<IDetailCustomNodeBuilder> InCustomBuilder,
    bool bForAdvanced = false) = 0;
virtual IDetailGroup& AddGroup(FName GroupName, const FText& LocalizedDisplayName,
    bool bForAdvanced = false, bool bStartExpanded = false) = 0;
```

`IDetailChildrenBuilder.h` (what `CustomizeChildren` receives):

```cpp
virtual IDetailChildrenBuilder& AddCustomBuilder(TSharedRef<IDetailCustomNodeBuilder> InCustomBuilder) = 0;
virtual IDetailGroup& AddGroup(FName GroupName, const FText& LocalizedDisplayName,
    const bool bStartExpanded = false) = 0;
virtual FDetailWidgetRow& AddCustomRow(const FText& SearchString) = 0;
virtual IDetailPropertyRow& AddProperty(TSharedRef<IPropertyHandle> PropertyHandle) = 0;
```

Note the argument difference: `IDetailCategoryBuilder::AddProperty` takes a `TSharedPtr`, `IDetailChildrenBuilder::AddProperty` takes a `TSharedRef`.

---

## IDetailPropertyRow — fluent modifiers

From `IDetailPropertyRow.h`; each returns `IDetailPropertyRow&`:

```cpp
virtual IDetailPropertyRow& DisplayName(const FText& InDisplayName) = 0;
virtual IDetailPropertyRow& ToolTip(const FText& InToolTip) = 0;
virtual IDetailPropertyRow& ShowPropertyButtons(bool bShowPropertyButtons) = 0;
virtual IDetailPropertyRow& EditCondition(TAttribute<bool> EditConditionValue,
    FOnBooleanValueChanged OnEditConditionValueChanged,
    ECustomEditConditionMode EditConditionMode = ECustomEditConditionMode::Override) = 0;
virtual IDetailPropertyRow& EditConditionHides(bool bEditConditionHidesValue) = 0;
virtual IDetailPropertyRow& IsEnabled(TAttribute<bool> InIsEnabled) = 0;
virtual IDetailPropertyRow& ShouldAutoExpand(bool bForceExpansion = true) = 0;
virtual IDetailPropertyRow& Visibility(TAttribute<EVisibility> Visibility) = 0;
virtual IDetailPropertyRow& OverrideResetToDefault(const FResetToDefaultOverride& ResetToDefault) = 0;

virtual void GetDefaultWidgets(TSharedPtr<SWidget>& OutNameWidget,
    TSharedPtr<SWidget>& OutValueWidget, bool bAddWidgetDecoration = false) = 0;
virtual FDetailWidgetRow& CustomWidget(bool bShowChildren = false) = 0;
```

Wrap the engine's own widgets instead of rebuilding them:

```cpp
IDetailPropertyRow& Row = Category.AddProperty(Handle);
TSharedPtr<SWidget> DefaultNameWidget;
TSharedPtr<SWidget> DefaultValueWidget;
Row.GetDefaultWidgets(DefaultNameWidget, DefaultValueWidget);

Row.CustomWidget(/*bShowChildren=*/true)
.NameContent()
[
    DefaultNameWidget.ToSharedRef()
]
.ValueContent()
[
    SNew(SHorizontalBox)
    + SHorizontalBox::Slot().FillWidth(1.f)
    [
        DefaultValueWidget.ToSharedRef()
    ]
    + SHorizontalBox::Slot().AutoWidth().Padding(4.f, 0.f)
    [
        SNew(SButton)
        .Text(FText::FromString("Pick"))
        .OnClicked_Lambda([]() { return FReply::Handled(); })
    ]
];
```

---

## FDetailWidgetRow

From `DetailWidgetRow.h`. The content accessors return `FDetailWidgetDecl&`; the modifiers return `FDetailWidgetRow&`:

```cpp
FDetailWidgetDecl& NameContent();
FDetailWidgetDecl& ValueContent();
FDetailWidgetDecl& WholeRowContent();
FDetailWidgetDecl& ExtensionContent();
FDetailWidgetDecl& ResetToDefaultContent();

FDetailWidgetRow& FilterString(const FText& InFilterString);
FDetailWidgetRow& Visibility(const TAttribute<EVisibility>& InVisibility);   // sets VisibilityAttr
FDetailWidgetRow& IsEnabled(const TAttribute<bool>& InIsEnabled);
FDetailWidgetRow& IsValueEnabled(const TAttribute<bool>& InIsEnabled);
FDetailWidgetRow& operator[](TSharedRef<SWidget> InWidget);                  // whole row
```

`FDetailWidgetDecl` supports `HAlign(EHorizontalAlignment)`, `VAlign(EVerticalAlignment)`, `MinDesiredWidth(TOptional<float>)`, `MaxDesiredWidth(TOptional<float>)` and `operator[]` for the widget itself. The row's own visibility is stored in the public `TAttribute<EVisibility> VisibilityAttr` member that `Visibility()` assigns.

Whole-row layout, used for banners and full-width tools:

```cpp
Category.AddCustomRow(FText::FromString("Bake"))
.WholeRowContent()
[
    SNew(SHorizontalBox)
    + SHorizontalBox::Slot().FillWidth(1.f).VAlign(VAlign_Center)
    [
        SNew(STextBlock)
        .Text(FText::FromString("Baked data is out of date"))
        .Font(IDetailLayoutBuilder::GetDetailFontBold())
    ]
    + SHorizontalBox::Slot().AutoWidth().Padding(FMargin(4.f, 0.f))
    [
        SNew(SButton)
        .Text(FText::FromString("Bake Now"))
        .OnClicked_Lambda([]() { return FReply::Handled(); })
    ]
];
```

---

## IPropertyTypeCustomization

`IPropertyTypeCustomization.h` — both are pure virtual, copy them verbatim:

```cpp
virtual void CustomizeHeader(TSharedRef<IPropertyHandle> PropertyHandle,
    FDetailWidgetRow& HeaderRow, IPropertyTypeCustomizationUtils& CustomizationUtils) = 0;
virtual void CustomizeChildren(TSharedRef<IPropertyHandle> PropertyHandle,
    IDetailChildrenBuilder& ChildBuilder, IPropertyTypeCustomizationUtils& CustomizationUtils) = 0;
virtual bool ShouldInlineKey() const;   // optional
```

`CustomizeHeader` draws the collapsed row; `CustomizeChildren` draws the expanded rows. If `CustomizeHeader` adds nothing, the children are inlined where the header would have been.

```cpp
// MyRangeCustomization.cpp
#include "MyRangeCustomization.h"
#include "MyRange.h"
#include "DetailWidgetRow.h"
#include "IDetailChildrenBuilder.h"
#include "PropertyHandle.h"
#include "Widgets/Text/STextBlock.h"

TSharedRef<IPropertyTypeCustomization> FMyRangeCustomization::MakeInstance()
{
    return MakeShared<FMyRangeCustomization>();
}

void FMyRangeCustomization::CustomizeHeader(TSharedRef<IPropertyHandle> PropertyHandle,
    FDetailWidgetRow& HeaderRow, IPropertyTypeCustomizationUtils& CustomizationUtils)
{
    TSharedPtr<IPropertyHandle> MinHandle =
        PropertyHandle->GetChildHandle(GET_MEMBER_NAME_CHECKED(FMyRange, Min));
    TSharedPtr<IPropertyHandle> MaxHandle =
        PropertyHandle->GetChildHandle(GET_MEMBER_NAME_CHECKED(FMyRange, Max));

    HeaderRow
    .NameContent()
    [
        PropertyHandle->CreatePropertyNameWidget()
    ]
    .ValueContent()
    .MinDesiredWidth(200.f)
    [
        SNew(STextBlock)
        .Text_Lambda([MinHandle, MaxHandle]()
        {
            float MinValue = 0.f;
            float MaxValue = 0.f;
            MinHandle->GetValue(MinValue);
            MaxHandle->GetValue(MaxValue);
            return FText::Format(FText::FromString("[{0} .. {1}]"),
                FText::AsNumber(MinValue), FText::AsNumber(MaxValue));
        })
        .Font(IPropertyTypeCustomizationUtils::GetRegularFont())
    ];
}

void FMyRangeCustomization::CustomizeChildren(TSharedRef<IPropertyHandle> PropertyHandle,
    IDetailChildrenBuilder& ChildBuilder, IPropertyTypeCustomizationUtils& CustomizationUtils)
{
    uint32 NumChildren = 0;
    PropertyHandle->GetNumChildren(NumChildren);
    for (uint32 Index = 0; Index < NumChildren; ++Index)
    {
        ChildBuilder.AddProperty(PropertyHandle->GetChildHandle(Index).ToSharedRef());
    }
}
```

Leave `CustomizeChildren` empty to suppress the expanded rows entirely (compact inline struct display).

---

## IPropertyHandle

From `PropertyHandle.h`:

```cpp
// Typed access. Overloads exist for float, double, bool, int8/16/32/64, uint8/16/32/64,
// FString, FText, FName, FVector, FVector2D, FVector4, FQuat, FRotator, UObject*,
// const UObject*, FAssetData, FProperty*, const FProperty*.
virtual FPropertyAccess::Result GetValue(float& OutValue) const = 0;
virtual FPropertyAccess::Result SetValue(const float& InValue,
    EPropertyValueSetFlags::Type Flags = EPropertyValueSetFlags::DefaultFlags) = 0;

virtual FPropertyAccess::Result GetValueAsDisplayString(FString& OutValue,
    EPropertyPortFlags PortFlags = PPF_PropertyWindow) const = 0;
virtual FPropertyAccess::Result GetValueAsFormattedText(FText& OutValue) const = 0;
virtual FPropertyAccess::Result SetValueFromFormattedString(const FString& InValue,
    EPropertyValueSetFlags::Type Flags = EPropertyValueSetFlags::DefaultFlags) = 0;

virtual void SetOnPropertyValueChanged(const FSimpleDelegate& InOnPropertyValueChanged) = 0;
virtual void SetOnPropertyValueChangedWithData(
    const TDelegate<void(const FPropertyChangedEvent&)>& InOnPropertyValueChanged) = 0;
virtual void SetOnChildPropertyValueChanged(const FSimpleDelegate& InOnChildPropertyValueChanged) = 0;

virtual void NotifyPreChange() = 0;
virtual void NotifyPostChange(EPropertyChangeType::Type ChangeType) = 0;

virtual TSharedPtr<IPropertyHandle> GetChildHandle(FName ChildName, bool bRecurse = true) const = 0;
virtual TSharedPtr<IPropertyHandle> GetChildHandle(uint32 Index) const = 0;
virtual FPropertyAccess::Result GetNumChildren(uint32& OutNumChildren) const = 0;

typedef TFunctionRef<bool(void* /*RawData*/, const int32 /*DataIndex*/, const int32 /*NumDatas*/)>
    EnumerateRawDataFuncRef;
virtual void EnumerateRawData(const EnumerateRawDataFuncRef& InRawDataCallback) = 0;

virtual TSharedRef<SWidget> CreatePropertyNameWidget(const FText& NameOverride = FText::GetEmpty(),
    const FText& ToolTipOverride = FText::GetEmpty()) const = 0;
virtual TSharedRef<SWidget> CreatePropertyValueWidget(bool bDisplayDefaultPropertyButtons = true) const = 0;
```

`FPropertyAccess::Result` (declared in `PropertyEditorModule.h`) is `MultipleValues`, `Fail` or `Success`. Always branch on `MultipleValues` when multiple objects can be selected:

```cpp
float SpeedValue = 0.f;
if (Handle->GetValue(SpeedValue) == FPropertyAccess::MultipleValues)
{
    // several selected objects disagree — show a blank or "Multiple Values" widget
}
```

`SetValue` handles the transaction and `PostEditChange` for you. `EnumerateRawData` writes each selected object's value directly, so bracket it yourself:

```cpp
Handle->NotifyPreChange();
Handle->EnumerateRawData([](void* RawData, const int32 DataIndex, const int32 NumDatas) -> bool
{
    *static_cast<float*>(RawData) = 100.f;
    return true;   // continue enumeration
});
Handle->NotifyPostChange(EPropertyChangeType::ValueSet);
```

Common `EPropertyChangeType::Type` values (`UObject/UnrealType.h`): `Unspecified`, `ArrayAdd`, `ArrayRemove`, `ArrayClear`, `ValueSet`, `Duplicate`, `Interactive`, `ArrayMove`, `ToggleEditable`, `ResetToDefault`. `Interactive` is sent while dragging a slider and is followed by a `ValueSet`.

---

## PropertyCustomizationHelpers

From `PropertyCustomizationHelpers.h`, namespace `PropertyCustomizationHelpers`:

```cpp
TSharedRef<SWidget> MakeAddButton(FSimpleDelegate OnAddClicked,
    TAttribute<FText> OptionalToolTipText = FText(), TAttribute<bool> IsEnabled = true);
TSharedRef<SWidget> MakeRemoveButton(FSimpleDelegate OnRemoveClicked,
    TAttribute<FText> OptionalToolTipText = FText(), TAttribute<bool> IsEnabled = true);
TSharedRef<SWidget> MakeClearButton(FSimpleDelegate OnClearClicked,
    TAttribute<FText> OptionalToolTipText = FText(), TAttribute<bool> IsEnabled = true);
TSharedRef<SWidget> MakeInsertDeleteDuplicateButton(FExecuteAction OnInsertClicked,
    FExecuteAction OnDeleteClicked, FExecuteAction OnDuplicateClicked);
TSharedRef<SWidget> MakeUseSelectedButton(FSimpleDelegate OnUseSelectedClicked,
    TAttribute<FText> OptionalToolTipText = FText(), TAttribute<bool> IsEnabled = true,
    const bool IsActor = false);
TSharedRef<SWidget> MakeEditConfigHierarchyButton(FSimpleDelegate OnEditConfigClicked,
    TAttribute<FText> OptionalToolTipText = FText(), TAttribute<bool> IsEnabled = true);
TSharedRef<SWidget> MakePropertyComboBox(const FPropertyComboBoxArgs& InArgs);
```

Widget classes in the same header: `SObjectPropertyEntryBox`, `SClassPropertyEntryBox`, `SStructPropertyEntryBox`, `SMaterialSlotWidget`.

```cpp
Category.AddCustomRow(FText::FromString("Mesh"))
.NameContent()
[
    MeshHandle->CreatePropertyNameWidget()
]
.ValueContent()
.MinDesiredWidth(250.f)
[
    SNew(SObjectPropertyEntryBox)
    .PropertyHandle(MeshHandle)
    .AllowedClass(UStaticMesh::StaticClass())
    .AllowClear(true)
    .DisplayThumbnail(true)
    .ThumbnailPool(CustomizationUtils.GetThumbnailPool())   // in CustomizeDetails: DetailBuilder.GetThumbnailPool() (DetailLayoutBuilder.h:288)
];
```

`SObjectPropertyEntryBox` arguments include `ObjectPath`, `PropertyHandle`, `ThumbnailPool`, `AllowedClass`, `NewAssetFactories`, `AllowClear`, `AllowCreate`, `DisplayUseSelected`, `DisplayBrowse`, `EnableContentPicker`, `DisplayCompactSize`, `DisplayThumbnail`, and the events `OnShouldSetAsset`, `OnObjectChanged`, `OnShouldFilterAsset`, `OnIsEnabled`, `OnShouldFilterActor`.

---

## Fonts

```cpp
IDetailLayoutBuilder::GetDetailFont()         // FAppStyle "PropertyWindow.NormalFont"
IDetailLayoutBuilder::GetDetailFontBold()     // FAppStyle "PropertyWindow.BoldFont"
IDetailLayoutBuilder::GetDetailFontItalic()   // FAppStyle "PropertyWindow.ItalicFont"

IPropertyTypeCustomizationUtils::GetRegularFont()   // FAppStyle "PropertyWindow.NormalFont"
IPropertyTypeCustomizationUtils::GetBoldFont()      // FAppStyle "PropertyWindow.BoldFont"
```

Use these statics rather than hand-written `FAppStyle::GetFontStyle` calls so rows stay consistent with the rest of the details panel.

---

## Registration reference

From `PropertyEditorModule.h`:

```cpp
virtual void RegisterCustomClassLayout(FName ClassName,
    FOnGetDetailCustomizationInstance DetailLayoutDelegate,
    FRegisterCustomClassLayoutParams Params = FRegisterCustomClassLayoutParams());
virtual void UnregisterCustomClassLayout(FName ClassName);

virtual void RegisterCustomPropertyTypeLayout(FName PropertyTypeName,
    FOnGetPropertyTypeCustomizationInstance PropertyTypeLayoutDelegate,
    TSharedPtr<IPropertyTypeIdentifier> Identifier = nullptr);
virtual void UnregisterCustomPropertyTypeLayout(FName PropertyTypeName,
    TSharedPtr<IPropertyTypeIdentifier> InIdentifier = nullptr);

virtual void NotifyCustomizationModuleChanged();
```

```cpp
struct FRegisterCustomClassLayoutParams
{
    /* Optional order to register this class layout with. Registration order is used when not
       specified. Lower values are added first */
    TOptional<int32> OptionalOrder;
};
```

Delegate types (`PropertyEditorDelegates.h`):

```cpp
DECLARE_DELEGATE_RetVal(TSharedRef<IDetailCustomization>, FOnGetDetailCustomizationInstance);
DECLARE_DELEGATE_RetVal(TSharedRef<IPropertyTypeCustomization>, FOnGetPropertyTypeCustomizationInstance);
```

Registration and the matching unregistration, with the ordering parameter set:

```cpp
// StartupModule
FPropertyEditorModule& PropertyModule =
    FModuleManager::LoadModuleChecked<FPropertyEditorModule>("PropertyEditor");

FRegisterCustomClassLayoutParams LayoutParams;
LayoutParams.OptionalOrder = 0;   // runs before layouts registered without an order

PropertyModule.RegisterCustomClassLayout(
    UMyDataAsset::StaticClass()->GetFName(),
    FOnGetDetailCustomizationInstance::CreateStatic(&FMyDataAssetCustomization::MakeInstance),
    LayoutParams);

PropertyModule.RegisterCustomPropertyTypeLayout(
    FMyRange::StaticStruct()->GetFName(),
    FOnGetPropertyTypeCustomizationInstance::CreateStatic(&FMyRangeCustomization::MakeInstance));

PropertyModule.NotifyCustomizationModuleChanged();

// ShutdownModule
if (FModuleManager::Get().IsModuleLoaded("PropertyEditor"))
{
    FPropertyEditorModule& ShutdownModuleRef =
        FModuleManager::GetModuleChecked<FPropertyEditorModule>("PropertyEditor");
    ShutdownModuleRef.UnregisterCustomClassLayout(UMyDataAsset::StaticClass()->GetFName());
    ShutdownModuleRef.UnregisterCustomPropertyTypeLayout(FMyRange::StaticStruct()->GetFName());
    ShutdownModuleRef.NotifyCustomizationModuleChanged();
}
```

Register a type customization for one details panel only, from inside `CustomizeDetails`:

```cpp
void FMyDataAssetCustomization::CustomizeDetails(IDetailLayoutBuilder& DetailBuilder)
{
    DetailBuilder.RegisterInstancedCustomPropertyTypeLayout(
        FMyRange::StaticStruct()->GetFName(),
        FOnGetPropertyTypeCustomizationInstance::CreateStatic(
            &FMyRangeInContextCustomization::MakeInstance));
}
```

---

## Checklist

- Use `GET_MEMBER_NAME_CHECKED(ClassName, MemberName)` so a renamed property is a compile error.
- Override both `CustomizeDetails` overloads and store a `TWeakPtr<IDetailLayoutBuilder>` if you ever call `ForceRefreshDetails()`.
- Capture handles and weak pointers in Slate lambdas, never `this` or a raw `IDetailLayoutBuilder&`.
- `AddProperty` for standard rows, `AddCustomRow` for fully custom rows, `AddCustomBuilder` for dynamic lists driven by an `IDetailCustomNodeBuilder`.
- `CustomWidget(/*bShowChildren=*/true)` keeps child rows while replacing the parent row's widgets.
- Call `NotifyCustomizationModuleChanged()` after registering or unregistering so open panels rebuild.
- Bracket direct writes through `EnumerateRawData` with `NotifyPreChange()` / `NotifyPostChange(EPropertyChangeType::ValueSet)`; `SetValue` already does this.
