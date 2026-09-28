---
name: ue-input-system
description: "Use when wiring player input in Unreal C++ with Enhanced Input — input actions, mapping contexts, triggers, modifiers, binding handlers and runtime key rebinding. Also use when the user mentions 'Enhanced Input', 'UInputAction', 'UInputMappingContext', 'AddMappingContext', 'BindAction', 'SetupPlayerInputComponent', 'ETriggerEvent', 'input trigger', 'input modifier', 'SwizzleAxis', 'dead zone', 'WASD', 'gamepad', 'key rebinding', 'UEnhancedInputUserSettings' or 'input mapping priority'. For UI input modes and CommonUI routing, see ue-ui-umg-slate; for PlayerController/Pawn possession, see ue-gameplay-framework."
metadata:
  version: "2.0.0"
  engine: "5.8"
---

# UE Input System

Target engine: **UE 5.8**. APIs below are verified against the 5.8 headers; older forms are listed under "Deprecated — do not use".

Enhanced Input is the engine's only supported input path. It lives in the `EnhancedInput` plugin (`Engine/Plugins/EnhancedInput`, enabled by default in 5.8); add `"EnhancedInput"` to `PublicDependencyModuleNames` in your `Build.cs`, plus `"InputCore"` if you touch `FKey`/`EKeys` directly and `"GameplayTags"` if you use input-mode filtering or the user-settings failure-reason containers. The runtime pieces are `UEnhancedPlayerInput`, `UEnhancedInputComponent`, `UEnhancedInputLocalPlayerSubsystem`, the data assets `UInputAction` / `UInputMappingContext`, and `UEnhancedInputUserSettings` for player rebinding.

## Context

Read `.agents/ue-project-context.md` if it exists (module names, conventions, enabled plugins, GAS/networking setup). Do not stop if it is missing.

Identify the area from the request and the codebase. Ask only when two plausible readings would produce different code.

| Request is about… | Go to |
|---|---|
| Plugin, module, ini setup | [Setup](#setup) |
| Creating actions and mapping contexts | [Input Assets](#input-assets) |
| Binding handlers in C++ | [Binding Actions in C++](#binding-actions-in-c) |
| Which event to bind (`Started` vs `Triggered` vs `Completed`) | [Trigger Events](#trigger-events) |
| Hold, tap, double-tap, pulse, chord | [Built-in Triggers](#built-in-triggers) |
| Dead zones, sensitivity, WASD→2D, invert Y | [Built-in Modifiers](#built-in-modifiers) |
| Adding/removing contexts, priority, split-screen | [Mapping Contexts and Priority](#mapping-contexts-and-priority) |
| Enabling/disabling contexts by gameplay state | [Input Mode Filtering](#input-mode-filtering) |
| Player key rebinding, settings screens, saving | [Runtime Key Rebinding](#runtime-key-rebinding) |
| Input on actors with no PlayerController | [Input Without a PlayerController](#input-without-a-playercontroller) |
| Per-platform mapping context swaps | [Per-Platform Input Data](#per-platform-input-data) |
| Writing a new trigger or modifier | [Custom Triggers](#custom-triggers), [Custom Modifiers](#custom-modifiers) |
| Menus, cursors, CommonUI | [UI Input Mode](#ui-input-mode) |
| Debug overlays | [Debugging](#debugging) |

Full per-class parameter tables live in [references/input-action-reference.md](references/input-action-reference.md).

## Setup

```csharp
// Build.cs
PublicDependencyModuleNames.AddRange(new string[] { "Core", "CoreUObject", "Engine", "InputCore", "EnhancedInput" });
```

`DefaultInput.ini` — point the engine's default classes at Enhanced Input:
```ini
[/Script/Engine.InputSettings]
DefaultPlayerInputClass=/Script/EnhancedInput.EnhancedPlayerInput
DefaultInputComponentClass=/Script/EnhancedInput.EnhancedInputComponent
```

Enhanced Input project settings live on `UEnhancedInputDeveloperSettings` (Project Settings → Engine → Enhanced Input), written to `DefaultInput.ini` under `[/Script/EnhancedInput.EnhancedInputDeveloperSettings]`. Keys that change C++ behaviour: `bEnableUserSettings`, `UserSettingsClass`, `DefaultPlayerMappableKeyProfileClass`, `InputSettingsSaveSlotName`, `bEnableWorldSubsystem`, `DefaultWorldInputClass`, `bEnableInputModeFiltering`, `DefaultMappingContextInputModeQuery`, `DefaultInputMode`, `bEnableDefaultMappingContexts`, `bSendTriggeredEventsWhenInputIsFlushed`.

## Input Assets

`UInputAction : UDataAsset` — one per logical player action. Properties that change C++ behaviour:

| Property | Default | Meaning |
|---|---|---|
| `ValueType` | `EInputActionValueType::Boolean` | `Boolean`, `Axis1D` (float), `Axis2D` (`FVector2D`), `Axis3D` (`FVector`) |
| `AccumulationBehavior` | `TakeHighestAbsoluteValue` | Or `Cumulative` — all mappings sum, so W + S cancel |
| `bConsumeInput` | `true` | Blocks lower-priority Enhanced Input mappings on the same keys |
| `bConsumesActionAndAxisMappings` | `false` | Also blocks legacy Action/Axis mappings on those keys |
| `bReserveAllMappings` | `false` | Mappings are not overridden by higher-priority contexts |
| `bTriggerWhenPaused` | `false` | Action fires while the game is paused |
| `Triggers` / `Modifiers` | empty | Action-level, applied **after** the per-mapping ones |
| `PlayerMappableKeySettings` | `nullptr` | `UPlayerMappableKeySettings` — makes the action rebindable |

`UInputMappingContext : UDataAsset` — key-to-action bindings, added and removed as a set.

| Member | Notes |
|---|---|
| `DefaultKeyMappings` | `FInputMappingContextMappingData`; read the array with `GetMappings()` |
| `MappingProfileOverrides` | `TMap<FString, FInputMappingContextMappingData>` — per key-profile overrides |
| `RegistrationTrackingMode` | `Untracked` (default) or `CountRegistrations` |
| `InputModeFilterOptions` | `UseProjectDefaultQuery` (default), `UseCustomQuery`, `DoNotFilter` |
| `MapKey(Action, Key)` / `UnmapKey(Action, Key)` | Editor/config-screen helpers, **not** a runtime rebinding API |
| `HasMappingForInputAction(Action)` | Search default and profile-override mappings |

Each `FEnhancedActionKeyMapping` carries `Action`, `Key`, per-mapping `Triggers`, `Modifiers`, a `SettingBehavior` (`EPlayerMappableKeySettingBehaviors`: `InheritSettingsFromAction`, `OverrideSettings`, `IgnoreSettings`) and an optional `PlayerMappableKeySettings` override. Reach them with `GetPlayerMappableKeySettings()` and `GetMappingName()`.

## Binding Actions in C++

```cpp
// MyCharacter.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/Character.h"
#include "MyCharacter.generated.h"

class UInputAction;
class UInputMappingContext;
struct FInputActionValue;

UCLASS()
class MYGAME_API AMyCharacter : public ACharacter
{
	GENERATED_BODY()

public:
	virtual void BeginPlay() override;
	virtual void SetupPlayerInputComponent(UInputComponent* PlayerInputComponent) override;

protected:
	UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Input")
	TObjectPtr<UInputMappingContext> DefaultMappingContext;
	UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Input")
	TObjectPtr<UInputAction> MoveAction;
	UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Input")
	TObjectPtr<UInputAction> JumpAction;

	void Move(const FInputActionValue& Value);
	void StartJump();
	void StopJump();
};
```

```cpp
// MyCharacter.cpp
#include "MyCharacter.h"

#include "EnhancedInputComponent.h"
#include "EnhancedInputSubsystems.h"
#include "InputActionValue.h"
#include "Engine/LocalPlayer.h"
#include "GameFramework/PlayerController.h"

void AMyCharacter::SetupPlayerInputComponent(UInputComponent* PlayerInputComponent)
{
	Super::SetupPlayerInputComponent(PlayerInputComponent);

	UEnhancedInputComponent* EIC = Cast<UEnhancedInputComponent>(PlayerInputComponent);
	if (!EIC)
	{
		return;
	}

	EIC->BindAction(MoveAction, ETriggerEvent::Triggered, this, &AMyCharacter::Move);
	EIC->BindAction(JumpAction, ETriggerEvent::Started,   this, &AMyCharacter::StartJump);
	EIC->BindAction(JumpAction, ETriggerEvent::Completed, this, &AMyCharacter::StopJump);
}

void AMyCharacter::BeginPlay()
{
	Super::BeginPlay();

	APlayerController* PC = Cast<APlayerController>(GetController());
	if (UEnhancedInputLocalPlayerSubsystem* Subsystem =
			ULocalPlayer::GetSubsystemFromController<UEnhancedInputLocalPlayerSubsystem>(PC))
	{
		Subsystem->AddMappingContext(DefaultMappingContext, 0);
	}
}

void AMyCharacter::Move(const FInputActionValue& Value)
{
	const FVector2D Axis = Value.Get<FVector2D>();
	AddMovementInput(GetActorForwardVector(), Axis.Y);
	AddMovementInput(GetActorRightVector(), Axis.X);
}

void AMyCharacter::StartJump() { Jump(); }
void AMyCharacter::StopJump() { StopJumping(); }
```

### Handler signatures

`UEnhancedInputComponent::BindAction` returns `FEnhancedInputActionEventBinding&` and accepts a member function matching one of three native delegate signatures, or a `UFUNCTION` by name:

| Signature | Handler |
|---|---|
| `FEnhancedInputActionHandlerSignature` | `void Handler()` |
| `FEnhancedInputActionHandlerValueSignature` | `void Handler(const FInputActionValue& Value)` |
| `FEnhancedInputActionHandlerInstanceSignature` | `void Handler(const FInputActionInstance& Instance)` |
| `FEnhancedInputActionHandlerDynamicSignature` | `BindAction(Action, Event, Object, FName("OnFirePressed"))` |

`FInputActionInstance` exposes `GetValue()`, `GetTriggerEvent()`, `GetElapsedTime()` (Started + Ongoing + Triggered), `GetTriggeredTime()` (Triggered only), `GetLastTriggeredWorldTime()`, `GetTriggers()`, `GetModifiers()` and `GetSourceAction()`.

Lambdas and pollable values:
```cpp
EIC->BindActionValueLambda(InteractAction, ETriggerEvent::Triggered,
	[this](const FInputActionValue& Value) { TryInteract(Value.Get<bool>()); });

EIC->BindActionInstanceLambda(ChargeAction, ETriggerEvent::Ongoing,
	[this](const FInputActionInstance& Instance) { SetChargeProgress(Instance.GetElapsedTime()); });

// No delegate — just reflects the current value, poll it from Tick
FEnhancedInputActionValueBinding& LookBinding = EIC->BindActionValue(LookAction);
const FVector2D LookAxis = LookBinding.GetValue().Get<FVector2D>();
```

Removing bindings:
```cpp
FEnhancedInputActionEventBinding& Binding =
	EIC->BindAction(JumpAction, ETriggerEvent::Started, this, &AMyCharacter::StartJump);
const uint32 BindingHandle = Binding.GetHandle(); // uint32, EnhancedInputComponent.h:165

EIC->RemoveBindingByHandle(BindingHandle);   // one binding, later
EIC->ClearBindingsForObject(this);           // every binding owned by this object; ClearActionBindings() drops all
```

## Trigger Events

`ETriggerEvent` is a bitmask (`ENUM_CLASS_FLAGS`) describing the trigger-state transitions seen this tick.

| Event | Bit | State transition | Bind it for |
|---|---|---|---|
| `Triggered` | `1 << 0` | None→Triggered, Ongoing→Triggered, Triggered→Triggered | Every active frame; continuous movement |
| `Started` | `1 << 1` | None→Ongoing, None→Triggered | First frame of evaluation; press-once actions |
| `Ongoing` | `1 << 2` | Ongoing→Ongoing | Held but not yet triggered (charge build-up) |
| `Canceled` | `1 << 3` | Ongoing→None | Released before the trigger condition was met |
| `Completed` | `1 << 4` | Triggered→None | Release after triggering; stop continuous actions |

`Started` always fires before `Triggered` when both occur on the same tick. `Completed` does **not** fire on a tick where any trigger on the action reports `Ongoing` — split press and release into two actions if you need both reliably.

## Built-in Triggers

Per-class properties and defaults: [references/input-action-reference.md](references/input-action-reference.md).

| Class | Display name | Behavior |
|---|---|---|
| `UInputTriggerDown` | Down | Fires every frame the value is actuated (implicit default when an action has no triggers) |
| `UInputTriggerPressed` | Pressed | Once on first actuation; holding does not repeat |
| `UInputTriggerReleased` | Released | Once when the value drops below `ActuationThreshold` after actuation |
| `UInputTriggerHold` | Hold | After `HoldTimeThreshold` seconds; `bIsOneShot = false` keeps firing |
| `UInputTriggerHoldAndRelease` | Hold And Release | On release, if held at least `HoldTimeThreshold` |
| `UInputTriggerTap` | Tap | Released within `TapReleaseTimeThreshold` |
| `UInputTriggerRepeatedTap` | Repeated Tap | `NumberOfTapsWhichTriggerRepeat` taps, each gap under `RepeatDelay` |
| `UInputTriggerPulse` | Pulse | Every `Interval` seconds while held; `TriggerLimit` caps the count |
| `UInputTriggerChordAction` | Chorded Action | Only while `ChordAction` is triggered; injects a `UInputTriggerChordBlocker` into lower-priority mappings of the same key |

`ETriggerType` decides how triggers combine on one action: `Explicit` (at least one must fire), `Implicit` (all must fire), `Blocker` (blocks everything while `IsBlocking` returns true). `ETriggerEventsSupported` (`None`, `Instant`, `Uninterruptible`, `Ongoing`, `All`) declares which `ETriggerEvent`s a trigger can ever produce.

## Built-in Modifiers

Per-mapping modifiers run first, then the action-level `Modifiers` array; each group runs in array order.

| Class | Display name | Effect |
|---|---|---|
| `UInputModifierDeadZone` | Dead Zone | Zero below `LowerThreshold`, remap to 1 at `UpperThreshold`; `Type` = `Axial`, `Radial`, `UnscaledRadial` |
| `UInputModifierScalar` | Scalar | Multiply per axis by `FVector Scalar` |
| `UInputModifierScaleByDeltaTime` | Scale By Delta Time | Multiply by frame DeltaTime |
| `UInputModifierNegate` | Negate | Invert the axes selected by `bX` / `bY` / `bZ` |
| `UInputModifierSwizzleAxis` | Swizzle Input Axis Values | Reorder axes; `Order` defaults to `EInputAxisSwizzle::YXZ` |
| `UInputModifierSmooth` | Smooth | Rolling average over recent samples |
| `UInputModifierSmoothDelta` | Smooth Delta | Smoothed normalized delta; `SmoothingMethod`, `Speed`, `EasingExponent` |
| `UInputModifierResponseCurveExponential` | Response Curve - Exponential | `sign(x) * pow(abs(x), CurveExponent)` per axis |
| `UInputModifierResponseCurveUser` | Response Curve - User Defined | One `UCurveFloat` per axis |
| `UInputModifierFOVScaling` | FOV Scaling | Scale by camera FOV for constant angular speed across zoom |
| `UInputModifierToWorldSpace` | To World Space | 2D axis → world space |

**WASD → Axis2D** (`AccumulationBehavior = Cumulative` on the action):
- `W`: `Swizzle Input Axis Values (YXZ)` → Y = +1
- `S`: `Swizzle Input Axis Values (YXZ)` + `Negate (bY)` → Y = −1
- `D`: none → X = +1
- `A`: `Negate (bX)` → X = −1

**Gamepad stick**: `Dead Zone (Radial, LowerThreshold = 0.2)` on each stick mapping.
**Mouse look**: `Scalar(0.4, 0.4, 1.0)`, then `Smooth`, then `FOV Scaling` on the mapping.

## Mapping Contexts and Priority

```cpp
Subsystem->AddMappingContext(GameplayContext, 0);   // higher integer = higher priority
Subsystem->AddMappingContext(VehicleContext, 1);

FModifyContextOptions Options;
Options.bForceImmediately = true;     // rebuild now instead of at end of frame
Options.bNotifyUserSettings = true;   // register these mappings with UEnhancedInputUserSettings
Subsystem->AddMappingContext(MenuContext, 2, Options);

Subsystem->RemoveMappingContext(VehicleContext);
Subsystem->ClearAllMappings();

int32 FoundPriority = 0;
const bool bActive = Subsystem->HasMappingContext(GameplayContext, FoundPriority);
const FInputActionValue Current = Subsystem->GetPlayerInput()->GetActionValue(MoveAction);
```

`FModifyContextOptions` defaults: `bIgnoreAllPressedKeysUntilRelease = true`, `bForceImmediately = false`, `bNotifyUserSettings = false`. The first one already suppresses ghost inputs from keys held across a context switch — do not set it explicitly thinking it is opt-in; set it to `false` only when you deliberately want a held key to carry into the new context.

`bConsumeInput = true` on a `UInputAction` (the default) means a higher-priority context mapping the same physical key blocks every lower-priority binding of that key. Use priority plus `bConsumeInput` to keep modes apart — a vehicle context consuming Spacebar stops the character's Jump action firing while driving.

`RegistrationTrackingMode` on the IMC: `Untracked` (the first `RemoveMappingContext` wins, whatever the add count) or `CountRegistrations` (stays applied until removed as many times as it was added — use it when several systems share one IMC).

Each local player owns a separate `UEnhancedInputLocalPlayerSubsystem`, so split-screen contexts never leak between players. Fetch a specific player's subsystem with `ULocalPlayer::GetSubsystem<UEnhancedInputLocalPlayerSubsystem>(LocalPlayer)` or `ULocalPlayer::GetSubsystemFromController<UEnhancedInputLocalPlayerSubsystem>(PlayerController)`. Simulate input without a device with `InjectInputForAction(Action, RawValue, Modifiers, Triggers)` or `InjectInputVectorForAction(Action, Value, Modifiers, Triggers)`.

## Input Mode Filtering

Gate whole mapping contexts on a gameplay-tag state instead of adding and removing them. Requires `bEnableInputModeFiltering` on `UEnhancedInputDeveloperSettings`.

- The subsystem holds the current mode: `GetInputMode()`, `SetInputMode(Tags, Options)`, `AppendTagsToInputMode`, `AddTagToInputMode`, `RemoveTagsFromInputMode`, `RemoveTagFromInputMode` — all take an optional `FModifyContextOptions`.
- Each `UInputMappingContext` chooses how it is matched via `InputModeFilterOptions`: `UseProjectDefaultQuery` (the project's `DefaultMappingContextInputModeQuery`), `UseCustomQuery` (the IMC's own `InputModeQueryOverride`), or `DoNotFilter`.
- `UEnhancedInputDeveloperSettings::DefaultInputMode` seeds the mode on every new `UEnhancedPlayerInput`.
- Contexts whose query fails stay registered but stop producing input, so priorities and registration counts are untouched.

```cpp
Subsystem->SetInputMode(FGameplayTagContainer(MyGameplayTags::TAG_InputMode_Menu));
```

## Runtime Key Rebinding

`UEnhancedInputUserSettings` (`UserSettings/EnhancedInputUserSettings.h`) is the sanctioned rebinding path. It is a `USaveGame`, created per subsystem when `bEnableUserSettings` is on, and saved to the slot named by `InputSettingsSaveSlotName`.

```cpp
#include "EnhancedInputSubsystems.h"
#include "UserSettings/EnhancedInputUserSettings.h"

void UMyRebindWidget::RebindKey(FName MappingName, const FKey& NewKey)
{
	UEnhancedInputLocalPlayerSubsystem* Subsystem =
		ULocalPlayer::GetSubsystemFromController<UEnhancedInputLocalPlayerSubsystem>(GetOwningPlayer());
	UEnhancedInputUserSettings* Settings = Subsystem ? Subsystem->GetUserSettings() : nullptr;
	if (!Settings)
	{
		return;
	}

	FMapPlayerKeyArgs Args;
	Args.MappingName = MappingName;
	Args.Slot = EPlayerMappableKeySlot::First;
	Args.NewKey = NewKey;

	FGameplayTagContainer FailureReason;
	Settings->MapPlayerKey(Args, FailureReason);
	if (FailureReason.IsEmpty())
	{
		Settings->SaveSettings();
	}
}
```

Register any IMC the settings UI must list — even one never added to the subsystem — with `RegisterInputMappingContext(IMC)`; adding a context with `bNotifyUserSettings = true` does it for you. Read mappings back with `FindMappingsInRow(MappingName)` and `FindCurrentMappingForSlot(MappingName, Slot)`, reset with `ResetAllPlayerKeysInRow` or `ResetKeyProfileIdToDefault`, and persist with `SaveSettings()` / `AsyncSaveSettings()`. Profile identifiers are `FString`, not `FGameplayTag` — `ProfileIdString`, `GetProfileIdString()`, `GetActiveKeyProfileId()`, `GetKeyProfileWithId()`, `SetActiveKeyProfile()`.

An action or mapping is only rebindable if it carries a `UPlayerMappableKeySettings` (`Name`, `DisplayName`, `DisplayCategory`, `Metadata`, `SupportedKeyProfileIds`) — either on the `UInputAction` or on the `FEnhancedActionKeyMapping` with `SettingBehavior = OverrideSettings`. Full API and struct tables: [references/input-action-reference.md](references/input-action-reference.md).

## Input Without a PlayerController

`UEnhancedInputWorldSubsystem` (Experimental in 5.8) lets actors that will never be possessed receive input delegates. Enable it with `bEnableWorldSubsystem` in `UEnhancedInputDeveloperSettings`.

```cpp
#include "EnhancedInputComponent.h"
#include "EnhancedInputSubsystems.h"
#include "Engine/World.h"

void AMyDoor::BeginPlay()
{
	Super::BeginPlay();

	// AddActorInputComponent reads AActor::InputComponent, created only for possessed actors.
	if (!InputComponent)
	{
		InputComponent = NewObject<UEnhancedInputComponent>(this, TEXT("DoorInputComponent"));
		InputComponent->RegisterComponent();
	}

	UEnhancedInputComponent* EIC = CastChecked<UEnhancedInputComponent>(InputComponent);
	EIC->BindAction(OpenAction, ETriggerEvent::Started, this, &AMyDoor::Open);

	if (UEnhancedInputWorldSubsystem* WorldInput =
			GetWorld()->GetSubsystem<UEnhancedInputWorldSubsystem>())
	{
		WorldInput->AddActorInputComponent(this);
		WorldInput->AddMappingContext(DoorContext, 0);
	}
}
```

Pair every `AddActorInputComponent` with `RemoveActorInputComponent` on `EndPlay`. The subsystem implements the same `IEnhancedInputSubsystemInterface` as the local-player one, so `AddMappingContext`, `InjectInputForAction` and friends behave identically — but it has no user settings and no local player.

## Per-Platform Input Data

`UEnhancedInputPlatformSettings` (`UPlatformSettings`, `UEnhancedInputPlatformSettings::Get()`) holds an `InputData` array of `UEnhancedInputPlatformData` subclasses. Each one defines `MappingContextRedirects`, a `TMap` swapping one `UInputMappingContext` for another on that platform; `GetContextRedirect(Context)` returns the replacement or the original. Use it to substitute a touch or console context without branching in gameplay code. Calling `Get()` needs `DeveloperSettings` in Build.cs (it inlines a `UPlatformSettingsManager` call).

## Custom Triggers

Subclass `UInputTrigger` and override `UpdateState_Implementation`, returning `ETriggerState::None`, `Ongoing` or `Triggered`.

```cpp
// MyDoubleClickTrigger.h
#pragma once

#include "EnhancedPlayerInput.h"
#include "Engine/World.h"
#include "InputTriggers.h"
#include "MyDoubleClickTrigger.generated.h"

UCLASS(EditInlineNew, meta = (DisplayName = "Double Click"))
class MYGAME_API UMyDoubleClickTrigger : public UInputTrigger
{
	GENERATED_BODY()

public:
	UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Trigger Settings")
	float DoubleClickThreshold = 0.3f;

protected:
	virtual ETriggerType GetTriggerType_Implementation() const override { return ETriggerType::Explicit; }

	virtual ETriggerState UpdateState_Implementation(
		const UEnhancedPlayerInput* PlayerInput, FInputActionValue ModifiedValue, float DeltaTime) override
	{
		const bool bActuated = IsActuated(ModifiedValue);
		const float Now = PlayerInput->GetWorld()->GetTimeSeconds();

		ETriggerState Result = ETriggerState::None;
		if (bActuated && !bWasActuated)
		{
			if ((Now - LastPressTime) <= DoubleClickThreshold)
			{
				LastPressTime = 0.f;
				Result = ETriggerState::Triggered;
			}
			else
			{
				LastPressTime = Now;
			}
		}

		bWasActuated = bActuated;
		return Result;
	}

private:
	float LastPressTime = 0.f;
	bool bWasActuated = false;
};
```

`IsActuated(Value)` is `Value.GetMagnitudeSq() >= ActuationThreshold * ActuationThreshold`. For time-based triggers derive from `UInputTriggerTimedBase`, which transitions to `Ongoing` on actuation and gives you `HeldDuration` plus `CalculateHeldDuration(PlayerInput, DeltaTime)` and `bAffectedByTimeDilation`. Override `IsBlocking(const ETriggerState State) const` for blockers and `GetDebugState() const` for the debug HUD. If your trigger keeps state across mapping rebuilds, override `ReceiveTriggerReinstanced_Implementation(const UInputTrigger* OldTrigger)` and call `Super`.

## Custom Modifiers

Subclass `UInputModifier` and override:
```cpp
virtual FInputActionValue ModifyRaw_Implementation(
	const UEnhancedPlayerInput* PlayerInput, FInputActionValue CurrentValue, float DeltaTime) override;
```

The returned value is converted back to the action's `ValueType` before further processing, so you may return any shape internally — `FInputActionValue(CurrentValue.GetValueType(), Vector)` preserves it. Override `GetVisualizationColor_Implementation(FInputActionValue SampleValue, FInputActionValue FinalValue) const` to control the debug overlay colour. Worked example: [references/input-action-reference.md](references/input-action-reference.md).

## UI Input Mode

Without CommonUI, switch modes on the PlayerController with `SetInputMode(FInputModeUIOnly())`, `SetInputMode(FInputModeGameAndUI())` or `SetInputMode(FInputModeGameOnly())`.

CommonUI routes input through `UCommonActivatableWidget` stacks and removes most manual `SetInputMode` calls. Enhanced Input and Common Input are unified: turn on `UCommonInputSettings::bEnableEnhancedInputSupport` (Project Settings → Game → Common Input; requires a restart) and CommonUI's click and back actions become `UInputAction` assets — `UCommonUIInputData::EnhancedInputClickAction` and `EnhancedInputBackAction` — instead of data-table rows. Widget stacks, activatable widgets and input configs belong to `ue-ui-umg-slate`.

## Debugging

`ShowDebug EnhancedInput` prints the applied mapping contexts, each action's value and trigger state, and each trigger's `GetDebugState()`. `ShowDebug WorldSubsystemInput` does the same for `UEnhancedInputWorldSubsystem`; `ShowDebug InputSettings` dumps the developer settings; `LogWorldSubsystemInput` at `VeryVerbose` logs every key the world subsystem processes.

## Deprecated — do not use

| Do not emit | Use in 5.8 | Source |
|---|---|---|
| `UInputTriggerCombo` | No replacement — build the sequence from `UInputTriggerChordAction`, timers, or a state machine | `UE_DEPRECATED(5.8)` in `InputTriggers.h:571` |
| `FInputComboStepData`, `FInputCancelAction` | No replacement | `UE_DEPRECATED(5.8)` in `InputTriggers.h:530,551` |
| `FEnhancedInputKeys::ComboKey` | No replacement | `UE_DEPRECATED(5.8)` in `EnhancedInputModule.h:20` |
| `UInputMappingContext::Mappings` | `DefaultKeyMappings` / `GetMappings()` | `UE_DEPRECATED(5.7)` in `InputMappingContext.h:93` |
| `UEnhancedInputPlatformSettings::LoadInputDataClasses()`, `InputDataClasses` | `GetInputData()` / `ForEachInputData()` | `UE_DEPRECATED(5.7)` in `EnhancedInputPlatformSettings.h:77,88` |
| `UEnhancedPlayerInput::GetAppliedInputContexts()` | `GetAppliedInputContextData()` | `UE_DEPRECATED(5.6)` in `EnhancedPlayerInput.h:175` |
| `FMapPlayerKeyArgs::ProfileId` (`FGameplayTag`), `SetKeyProfile(FGameplayTag)`, `GetCurrentKeyProfile()`, `GetCurrentKeyProfileIdentifier()`, `GetAllSavedKeyProfiles()`, `GetKeyProfileWithIdentifier()`, `ResetKeyProfileToDefault(FGameplayTag)`, `GetProfileIdentifer()` | `ProfileIdString` (`FString`), `SetActiveKeyProfile(FString)`, `GetActiveKeyProfile()`, `GetActiveKeyProfileId()`, `GetAllAvailableKeyProfiles()`, `GetKeyProfileWithId()`, `ResetKeyProfileIdToDefault(FString)`, `GetProfileIdString()` | `UE_DEPRECATED(5.6)` in `UserSettings/EnhancedInputUserSettings.h:86,706,730,722,754,776,673,327` |
| `UPlayerMappableKeySettings::SupportedKeyProfiles` | `SupportedKeyProfileIds` (`TArray<FString>`) | `UE_DEPRECATED(5.6)` in `PlayerMappableKeySettings.h:62` |
| `UEnhancedInputDeveloperSettings::bShouldLogAllWorldSubsystemInputs` | `LogWorldSubsystemInput` at `VeryVerbose` | `UE_DEPRECATED(5.6)` in `EnhancedInputDeveloperSettings.h:150` |
| `UPlayerMappableInputConfig` | `UEnhancedInputUserSettings` | `UE_DEPRECATED(5.3)` in `PlayerMappableInputConfig.h:23` |
| `UInputComponent::BindAxis`, `BindAction(FName, EInputEvent, Object, Func)`, `BindAxisKey`, `BindVectorAxis`, `BindKey`, `BindTouch`, `BindGesture` | `UEnhancedInputComponent::BindAction(UInputAction*, ETriggerEvent, Object, Func)` | `= delete` on `UEnhancedInputComponent`, `EnhancedInputComponent.h:613-634` |

## Common Mistakes

**Adding the mapping context in `SetupPlayerInputComponent`:** that call can run before the controller and local player are resolved. Add contexts in `BeginPlay`, `OnPossess`/`PossessedBy`, or a `PlayerControllerChanged` handler.

**Setting `bIgnoreAllPressedKeysUntilRelease = true` "to be safe":** it is already the default on `FModifyContextOptions`. Only touch it to set `false`.

**Binding `Triggered` for a button press:** `Triggered` fires every active frame. Use `Started` for press-once and `Completed` for release.

**Expecting `Completed` alongside `Ongoing`:** `Completed` is suppressed on any tick where a trigger reports `Ongoing`. Split into two actions with their own triggers.

**No dead zone on sticks:** analog sticks rest at non-zero. Add `Dead Zone (Radial)` to every stick mapping.

**Missing swizzle for WASD:** a keyboard key produces a 1D value on X. Without `Swizzle Input Axis Values (YXZ)` on W and S, forward/back never reaches the Y component of an `Axis2D` action.

**Calling `MapKey`/`UnmapKey` for player rebinding:** those mutate the IMC asset and are editor/config helpers. Use `UEnhancedInputUserSettings::MapPlayerKey` instead, and pass `bNotifyUserSettings = true` when adding contexts you want listed in a rebinding UI.

**Replicating trigger events:** input is client-local. Replicate the result — movement input, ability activation — never the `ETriggerEvent`.

**Legacy binding on a `UEnhancedInputComponent`:** `BindAxis` and the `FName` `BindAction` overloads are `= delete` and fail to compile. Migrate them rather than defining `ENHANCED_INPUT_ALLOW_LEGACY_BINDING=1`.

## Related Skills

- `ue-gameplay-framework` — `APlayerController`, possession, `SetupPlayerInputComponent` lifecycle
- `ue-ui-umg-slate` — CommonUI activatable-widget stacks, input configs, cursor and focus
- `ue-character-movement` — consuming `FInputActionValue` through `AddMovementInput`
- `ue-gameplay-abilities` — activating abilities from input, `UAbilitySystemComponent` input IDs
- `ue-actor-component-architecture` — `UActorComponent`/`UInputComponent` ownership, `EnableInput`, component-owned input
- Also relevant: `ue-cpp-foundations`, `ue-game-features`, `ue-gameplay-cameras`, `ue-mover`
