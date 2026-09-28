# Slate Widget Patterns

A complete custom Slate widget: a button with a bindable label, an initial enabled state and a press event. It shows the three `SLATE_BEGIN_ARGS` argument kinds, `SNew` / `SAssignNew`, slot syntax and a `CreateSP` click binding. `SMyLabelButton` has no module API macro, so it is private to its module; add `MYGAME_API` if another module constructs it.

```cpp
// SMyLabelButton.h
#pragma once
#include "CoreMinimal.h"
#include "Widgets/SCompoundWidget.h"
#include "Widgets/DeclarativeSyntaxSupport.h"

class SButton;

class SMyLabelButton : public SCompoundWidget
{
public:
    SLATE_BEGIN_ARGS(SMyLabelButton)
        : _LabelText(FText::GetEmpty())
        , _bStartEnabled(true)
    {}
        SLATE_ATTRIBUTE(FText, LabelText)       // TAttribute<FText>, supports _Lambda binding
        SLATE_ARGUMENT(bool, bStartEnabled)     // Plain by-value argument
        SLATE_EVENT(FSimpleDelegate, OnPressed) // Delegate argument
    SLATE_END_ARGS()

    void Construct(const FArguments& InArgs);

private:
    FReply HandleClicked();

    TAttribute<FText> LabelText;
    FSimpleDelegate OnPressed;
    TSharedPtr<SButton> ButtonWidget;
};

// SMyLabelButton.cpp — #include "SMyLabelButton.h", "Widgets/SBoxPanel.h",
// "Widgets/Input/SButton.h", "Widgets/Text/STextBlock.h"
void SMyLabelButton::Construct(const FArguments& InArgs)
{
    LabelText = InArgs._LabelText;
    OnPressed = InArgs._OnPressed;
    ChildSlot
    [
        SNew(SVerticalBox)
        + SVerticalBox::Slot()
        .AutoHeight()
        .Padding(4.f)
        [
            SAssignNew(ButtonWidget, SButton)
            .IsEnabled(InArgs._bStartEnabled)
            .OnClicked(FOnClicked::CreateSP(this, &SMyLabelButton::HandleClicked))
            [
                SNew(STextBlock).Text(LabelText)
            ]
        ]
    ];
}

FReply SMyLabelButton::HandleClicked() { OnPressed.ExecuteIfBound(); return FReply::Handled(); }
```

Using it from another widget, with the label bound to a lambda so it re-evaluates every paint:

```cpp
SNew(SMyLabelButton)
    .LabelText_Lambda([this]() { return FText::AsNumber(Score); })
    .bStartEnabled(false)
    .OnPressed(FSimpleDelegate::CreateSP(this, &SMyPanel::HandlePressed))
```
