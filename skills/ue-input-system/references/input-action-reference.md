# Enhanced Input: Trigger and Modifier Reference

Companion reference for the `ue-input-system` skill. Covers the built-in `UInputTrigger` and `UInputModifier`
classes with their exact properties and defaults, the mapping-context and subsystem APIs, input-mode filtering,
and the `UEnhancedInputUserSettings` rebinding path. Everything lives in the `EnhancedInput` plugin module.

Source headers: `InputAction.h`, `InputActionValue.h`, `InputTriggers.h`, `InputModifiers.h`,
`InputMappingContext.h`, `EnhancedActionKeyMapping.h`, `EnhancedInputSubsystemInterface.h`,
`EnhancedInputSubsystems.h`, `EnhancedInputDeveloperSettings.h`, `EnhancedInputPlatformSettings.h`,
`PlayerMappableKeySettings.h`, `EnhancedInputLibrary.h`, `UserSettings/EnhancedInputUserSettings.h`.

---

## UInputAction Properties

```cpp
class UInputAction : public UDataAsset
```

| Property | Type | Default | Description |
|---|---|---|---|
| `ValueType` | `EInputActionValueType` | `Boolean` | Shape of the value: `Boolean`, `Axis1D` (float), `Axis2D` (FVector2D), `Axis3D` (FVector) |
| `AccumulationBehavior` | `EInputActionAccumulationBehavior` | `TakeHighestAbsoluteValue` | How multiple mappings to the same action are combined |
| `bConsumeInput` | `bool` | `true` | If true, lower-priority Enhanced Input mappings to the same keys are blocked |
| `bConsumesActionAndAxisMappings` | `bool` | `false` | If true, legacy Action/Axis mappings to the same key are also blocked |
| `bReserveAllMappings` | `bool` | `false` | Mappings are not automatically overridden by higher-priority contexts |
| `bTriggerWhenPaused` | `bool` | `false` | Action fires even while game is paused |
| `Triggers` | `TArray<UInputTrigger*>` | empty | Action-level triggers; applied after per-mapping triggers |
| `Modifiers` | `TArray<UInputModifier*>` | empty | Action-level modifiers; applied after per-mapping modifiers |

### EInputActionAccumulationBehavior

| Value | Behavior |
|---|---|
| `TakeHighestAbsoluteValue` | The mapping with the highest absolute value wins. Pressing W (-0.3) and D (0.5) gives 0.5. |
| `Cumulative` | All mapping values are added. Pressing W (1.0) and S (-1.0) cancels to 0.0. Useful for WASD pairs. |

---

## FInputActionInstance (Runtime)

Available inside `FInputActionInstance` callbacks:

| Method | Returns | Description |
|---|---|---|
| `GetValue()` | `FInputActionValue` | Current value in every phase (`EnhancedInput.bAlwaysGetRealValueFromActionInstanceData` defaults to 1, `InputAction.cpp:18`; set it to 0 for the legacy zero-unless-`Triggered` behaviour the header comment still describes) |
| `GetTriggerEvent()` | `ETriggerEvent` | Current event state |
| `GetElapsedTime()` | `float` | Seconds since action began evaluating (Started + Ongoing + Triggered) |
| `GetTriggeredTime()` | `float` | Seconds the action has been in `Triggered` state only |
| `GetLastTriggeredWorldTime()` | `float` | World time of last trigger |
| `GetSourceAction()` | `const UInputAction*` | The originating action asset |

---

## ETriggerEvent

Bitmask enum. Represents state transitions observed in a single tick.

| Value | Bit | State Transition | Fires when |
|---|---|---|---|
| `None` | 0x00 | — | No significant transition, no active inputs |
| `Triggered` | 0x01 | None->Triggered, Ongoing->Triggered, Triggered->Triggered | Action is actively firing; also fires on the first triggered frame |
| `Started` | 0x02 | None->Ongoing, None->Triggered | First frame any input begins evaluation |
| `Ongoing` | 0x04 | Ongoing->Ongoing | Input is held and processing but trigger condition not yet met |
| `Canceled` | 0x08 | Ongoing->None | Evaluation began but was abandoned before triggering (e.g., held key released early) |
| `Completed` | 0x10 | Triggered->None | Trigger was active and has now ended |

`Completed` will not fire if any trigger on the same action reports `Ongoing` that frame.

---

## Trigger Classes

### ETriggerState (Internal)

Triggers return one of three states:

| State | Meaning |
|---|---|
| `None` | No conditions met |
| `Ongoing` | Conditions partially met; continue evaluating |
| `Triggered` | All conditions met; fire the action |

### ETriggerType (Multi-Trigger Evaluation)

| Type | Rule |
|---|---|
| `Explicit` | At least one Explicit trigger must be in Triggered state |
| `Implicit` | All Implicit triggers must be in Triggered state |
| `Blocker` | If blocking, prevents all other triggers from firing |

---

### UInputTriggerDown

```
DisplayName: "Down"
Class: UInputTriggerDown : public UInputTrigger
TriggerType: Explicit
SupportedEvents: Instant
```

**Behavior:** Fires `Triggered` every frame input magnitude exceeds `ActuationThreshold`.
This is the implicit default when an action has no triggers assigned.

**Properties:**

| Property | Type | Default | Description |
|---|---|---|---|
| `ActuationThreshold` | `float` | `0.5` | Minimum magnitude to consider the input actuated |

**Use for:** Continuous actions where you want firing as long as a key is held (before adding a proper trigger).

---

### UInputTriggerPressed

```
DisplayName: "Pressed"
Class: UInputTriggerPressed : public UInputTrigger
TriggerType: Explicit
SupportedEvents: Instant
```

**Behavior:** Fires `Triggered` exactly once on the first frame input exceeds `ActuationThreshold`.
Holding the input does not fire again. Re-press is required for another trigger.

**Properties:**

| Property | Type | Default | Description |
|---|---|---|---|
| `ActuationThreshold` | `float` | `0.5` | Minimum magnitude |

**Use for:** Discrete press-once actions: jump initiation, weapon fire on semi-auto, menu confirm.

---

### UInputTriggerReleased

```
DisplayName: "Released"
Class: UInputTriggerReleased : public UInputTrigger
TriggerType: Explicit
SupportedEvents: Instant
```

**Behavior:** Returns `Ongoing` while input exceeds `ActuationThreshold`. Fires `Triggered`
once when input drops back below the threshold after having been actuated.

**Properties:**

| Property | Type | Default | Description |
|---|---|---|---|
| `ActuationThreshold` | `float` | `0.5` | Threshold |

**Use for:** Release-on-let-go: throwing after wind-up on a dedicated action, "lift to release" grenades.

---

### UInputTriggerHold

```
DisplayName: "Hold"
Class: UInputTriggerHold : public UInputTriggerTimedBase
TriggerType: Explicit
SupportedEvents: Ongoing (Started, Ongoing, Triggered, Canceled)
```

**Behavior:** Returns `Ongoing` while input is held. After `HoldTimeThreshold` seconds,
fires `Triggered`. If `bIsOneShot=false`, continues firing `Triggered` every frame.
If input is released before the threshold, fires `Canceled`.

**Properties:**

| Property | Type | Default | Description |
|---|---|---|---|
| `HoldTimeThreshold` | `float` | `1.0` | Seconds of continuous hold required |
| `bIsOneShot` | `bool` | `false` | If true, fires `Triggered` only once then stops; if false, fires every frame after threshold |
| `bAffectedByTimeDilation` | `bool` | `false` | Use actor time dilation when accumulating hold duration |
| `ActuationThreshold` | `float` | `0.5` | Minimum magnitude |

**Use for:** Hold-to-interact prompts, charged attacks where you need the `Ongoing` callback
to show a progress bar. Combine with `GetElapsedTime()` in the `Ongoing` handler.

**Example — charge bar:**
```cpp
void AMyCharacter::UpdateChargeBar(const FInputActionInstance& Instance)
{
    // Instance.GetElapsedTime() grows while key is held
    const float Progress = FMath::Clamp(
        Instance.GetElapsedTime() / HoldTrigger->HoldTimeThreshold, 0.f, 1.f);
    ChargeBarWidget->SetPercent(Progress);
}
```

---

### UInputTriggerHoldAndRelease

```
DisplayName: "Hold And Release"
Class: UInputTriggerHoldAndRelease : public UInputTriggerTimedBase
TriggerType: Explicit
SupportedEvents: Ongoing
```

**Behavior:** Returns `Ongoing` while input is held. Fires `Triggered` only when input
is released after having been held for at least `HoldTimeThreshold` seconds.
If released before the threshold, does not trigger.

**Properties:**

| Property | Type | Default | Description |
|---|---|---|---|
| `HoldTimeThreshold` | `float` | `0.5` | Minimum hold duration before release triggers |
| `bAffectedByTimeDilation` | `bool` | `false` | Use actor time dilation |
| `ActuationThreshold` | `float` | `0.5` | Minimum magnitude |

**Use for:** "Pull back and release" bows, charged throws where the shot fires on release.
Different from `UInputTriggerHold` in that the payload fires on release, not during hold.

---

### UInputTriggerTap

```
DisplayName: "Tap"
Class: UInputTriggerTap : public UInputTriggerTimedBase
TriggerType: Explicit
SupportedEvents: Instant
```

**Behavior:** Fires `Triggered` if input is actuated then released within
`TapReleaseTimeThreshold` seconds. Returns `Ongoing` while held within the window.
If held past the threshold without release, the trigger does not fire.

**Properties:**

| Property | Type | Default | Description |
|---|---|---|---|
| `TapReleaseTimeThreshold` | `float` | `0.2` | Max seconds between press and release to count as a tap |
| `bAffectedByTimeDilation` | `bool` | `false` | Use actor time dilation |
| `ActuationThreshold` | `float` | `0.5` | Minimum magnitude |

**Use for:** Quick-dash on tap (vs hold for a sustained dash). Combine with a separate
`UInputTriggerHold` on a second binding of the same key to distinguish tap vs hold.

---

### UInputTriggerRepeatedTap

```
DisplayName: "Repeated Tap"
Class: UInputTriggerRepeatedTap : public UInputTriggerTimedBase
TriggerType: Explicit
SupportedEvents: Ongoing (inherited from UInputTriggerTimedBase)
```

**Behavior:** Fires `Triggered` when the input is tapped `NumberOfTapsWhichTriggerRepeat`
times in succession, with each individual tap completing within `TapReleaseTimeThreshold`
and each gap between taps within `RepeatDelay` seconds.

**Properties:**

| Property | Type | Default | Description |
|---|---|---|---|
| `NumberOfTapsWhichTriggerRepeat` | `int32` | `2` | Number of rapid taps required to trigger |
| `RepeatDelay` | `double` | `0.5` | Max seconds allowed between consecutive taps |
| `TapReleaseTimeThreshold` | `float` | `0.2` | Max hold duration per tap |
| `bAffectedByTimeDilation` | `bool` | `false` | Use actor time dilation |

**Use for:** Double-tap dodge roll (`NumberOfTapsWhichTriggerRepeat=2`), triple-tap finisher.

---

### UInputTriggerPulse

```
DisplayName: "Pulse"
Class: UInputTriggerPulse : public UInputTriggerTimedBase
TriggerType: Explicit
SupportedEvents: Ongoing
```

**Behavior:** While input is held, fires `Triggered` at regular `Interval` second intervals.
If `bTriggerOnStart=true`, also fires immediately on the first actuation frame.
Stops after `TriggerLimit` fires if `TriggerLimit > 0`.

**Properties:**

| Property | Type | Default | Description |
|---|---|---|---|
| `bTriggerOnStart` | `bool` | `true` | Fire on the first frame of actuation before the first interval elapses |
| `Interval` | `float` | `1.0` | Seconds between successive triggers |
| `TriggerLimit` | `int32` | `0` | Maximum number of fires; 0 = unlimited |
| `bAffectedByTimeDilation` | `bool` | `false` | Use actor time dilation |
| `ActuationThreshold` | `float` | `0.5` | Minimum magnitude |

**Use for:** Automatic weapon fire rate, repeated ability pulses while holding a key,
tick-based resource drain.

---

### UInputTriggerChordAction

```
DisplayName: "Chorded Action"
Class: UInputTriggerChordAction : public UInputTrigger
TriggerType: Implicit
SupportedEvents: Instant
NotInputConfigurable: true (not exposed in per-mapping settings)
```

**Behavior:** This action only fires when `ChordAction` is simultaneously in `Triggered`
state. When mappings rebuild, the subsystem injects a `UInputTriggerChordBlocker` into every
lower-priority mapping that shares the chorded mapping's key, so that key's solo actions are
blocked while the chord is active (`EnhancedInputSubsystemInterface.cpp:587-609`).

**Properties:**

| Property | Type | Default | Description |
|---|---|---|---|
| `ChordAction` | `const UInputAction*` | `nullptr` | The action that must be active at the same time |
| `ActuationThreshold` | `float` | `0.5` | Minimum magnitude |

**Setup pattern — Shift+E for special interact:**

1. Create `IA_Sprint` (Bool) bound to Shift in the IMC
2. Create `IA_SpecialInteract` (Bool) bound to E, and `IA_Interact` (Bool) also bound to E in a later (lower-priority) mapping
3. On `IA_SpecialInteract`, add a `UInputTriggerChordAction` with `ChordAction = IA_Sprint`
4. Result: E fires `IA_Interact`; Shift+E fires `IA_SpecialInteract`; E while Shift is held does NOT fire `IA_Interact`

---

## Modifier Classes

### UInputModifierDeadZone

```
DisplayName: "Dead Zone"
Class: UInputModifierDeadZone : public UInputModifier
```

**Behavior:** Input values with magnitude below `LowerThreshold` are zeroed out.
Values are remapped from `[LowerThreshold, UpperThreshold]` to `[0, 1]`.
Values above `UpperThreshold` are clamped to 1.

**Properties:**

| Property | Type | Default | Description |
|---|---|---|---|
| `LowerThreshold` | `float` | `0.2` | Below this, input is zero |
| `UpperThreshold` | `float` | `1.0` | Above this, input is clamped to 1 |
| `Type` | `EDeadZoneType` | `Radial` | `Axial`: per-axis (chamfered corners); `Radial`: circular smooth; `UnscaledRadial`: circular, no smoothing |

**EDeadZoneType:**

| Value | Description |
|---|---|
| `Axial` | Dead zone applied independently per axis. Results in chamfered corners on 2D sticks; on a 1D axis it behaves exactly like `Radial`. |
| `Radial` | Display name "Smoothed Radial". Dead zone applied to the combined magnitude, with smoothing. Recommended for most sticks. |
| `UnscaledRadial` | Radial dead zone without the value smoothing. Can feel "jumpy" at the threshold boundary. |

**Use for:** All gamepad analog stick mappings. Add per-stick-mapping in the IMC, not on the action asset.

---

### UInputModifierScalar

```
DisplayName: "Scalar"
Class: UInputModifierScalar : public UInputModifier
```

**Behavior:** Multiplies each axis component of the input by the corresponding component
of `Scalar`. Has no effect on Boolean value types.

**Properties:**

| Property | Type | Default | Description |
|---|---|---|---|
| `Scalar` | `FVector` | `(1, 1, 1)` | Per-axis multiplier |

**Use for:** Mouse sensitivity scaling (`Scalar=(0.4, 0.4, 1.0)`), axis inversion without
using `Negate` (set component to -1.0), adjusting controller stick sensitivity.

---

### UInputModifierScaleByDeltaTime

```
DisplayName: "Scale By Delta Time"
Class: UInputModifierScaleByDeltaTime : public UInputModifier
```

**Behavior:** Multiplies the input value by the current frame's DeltaTime.
No configurable properties.

**Use for:** Making input frame-rate independent when feeding raw values into physics
or math that expects a per-second rate. Not typically needed for movement (AddMovementInput
already accounts for DeltaTime internally).

---

### UInputModifierNegate

```
DisplayName: "Negate"
Class: UInputModifierNegate : public UInputModifier
```

**Behavior:** Inverts selected axes by multiplying by -1.

**Properties:**

| Property | Type | Default | Description |
|---|---|---|---|
| `bX` | `bool` | `true` | Invert X axis |
| `bY` | `bool` | `true` | Invert Y axis |
| `bZ` | `bool` | `true` | Invert Z axis |

**Use for:** Y-axis inversion for "inverted look" options, negating the S and A keys in
WASD-to-2D composition.

**Common setup — invert Y only:**
Set `bX=false`, `bY=true`, `bZ=false`.

---

### UInputModifierSwizzleAxis

```
DisplayName: "Swizzle Input Axis Values"
Class: UInputModifierSwizzleAxis : public UInputModifier
```

**Behavior:** Reorders the X, Y, Z components of the input value according to `Order`.

**Properties:**

| Property | Type | Default | Description |
|---|---|---|---|
| `Order` | `EInputAxisSwizzle` | `YXZ` | How to reorder axes |

**EInputAxisSwizzle:**

| Value | Result | Description |
|---|---|---|
| `YXZ` | Output=(Y,X,Z) | Swap X and Y. Maps a 1D W/S key into the Y component of an Axis2D action. |
| `ZYX` | Output=(Z,Y,X) | Swap X and Z |
| `XZY` | Output=(X,Z,Y) | Swap Y and Z |
| `YZX` | Output=(Y,Z,X) | Rotate all axes Y-first |
| `ZXY` | Output=(Z,X,Y) | Rotate all axes Z-first |

**Use for:** The canonical WASD-to-Axis2D mapping. W produces a 1D value of 1.0 on X.
Adding `SwizzleAxis(YXZ)` converts it to Y=1.0, which is forward in the Axis2D Move action.

---

### UInputModifierSmooth

```
DisplayName: "Smooth"
Class: UInputModifierSmooth : public UInputModifier
```

**Behavior:** Averages the input value over recent samples to reduce per-frame jitter.
Uses a rolling average with `TotalSampleTime` as the accumulation window
(`SMOOTH_TOTAL_SAMPLE_TIME_DEFAULT` = `0.0083f`). Resets when input returns to zero.

No configurable properties: `TotalSampleTime` is a protected member with no `UPROPERTY`,
so it is not editable on the asset and not reachable from Blueprint.

**Use for:** Smoothing raw mouse delta input for look controls. Add alongside `Scalar`
on mouse look mappings.

---

### UInputModifierSmoothDelta

```
DisplayName: "Smooth Delta"
Class: UInputModifierSmoothDelta : public UInputModifier
```

**Behavior:** Produces a smoothed normalized delta between the current and previous frame's
input value, using one of many configurable interpolation methods.

**Properties:**

| Property | Type | Default | Description |
|---|---|---|---|
| `SmoothingMethod` | `ENormalizeInputSmoothingType` | `Lerp` | Interpolation algorithm |
| `Speed` | `float` | `0.5` | Speed or alpha for the interpolation. If 0, jumps immediately to target. |
| `EasingExponent` | `float` | `2.0` | Degree of the ease curve; only for `Interp_Ease_*` methods |

**ENormalizeInputSmoothingType options:**
`Lerp`, `Interp_To`, `Interp_Constant_To`, `Interp_Circular_In/Out/In_Out`,
`Interp_Ease_In/Out/In_Out`, `Interp_Expo_In/Out/In_Out`, `Interp_Sin_In/Out/In_Out`
(`None` also exists, `UMETA(Hidden)`, `InputModifiers.h:83`)

**Use for:** Smooth acceleration/deceleration on stick or mouse input where you want
to author the feel curve precisely.

---

### UInputModifierResponseCurveExponential

```
DisplayName: "Response Curve - Exponential"
Class: UInputModifierResponseCurveExponential : public UInputModifier
```

**Behavior:** Applies `sign(x) * |x|^CurveExponent` per axis, creating a non-linear
response. Values below 1 create a "softer" near-center feel; values above 1 create a
"harder" aggressive feel.

**Properties:**

| Property | Type | Default | Description |
|---|---|---|---|
| `CurveExponent` | `FVector` | `(1, 1, 1)` | Exponent per axis. 1.0 = linear. 2.0 = squared. |

**Use for:** Analog stick look sensitivity curves — a squared or cubed response gives
fine control near center with fast movement at the edges.

---

### UInputModifierResponseCurveUser

```
DisplayName: "Response Curve - User Defined"
Class: UInputModifierResponseCurveUser : public UInputModifier
```

**Behavior:** Applies separate `UCurveFloat` assets per axis, evaluating each curve with
the current axis value as input. Gives full artist control over the response shape.

**Properties:**

| Property | Type | Default | Description |
|---|---|---|---|
| `ResponseX` | `UCurveFloat*` | `nullptr` | Curve for the X axis |
| `ResponseY` | `UCurveFloat*` | `nullptr` | Curve for the Y axis |
| `ResponseZ` | `UCurveFloat*` | `nullptr` | Curve for the Z axis |

**Use for:** Precise feel tuning with a visible curve asset. Preferred over
`ResponseCurveExponential` when the exact shape matters and needs iteration in the
Curve Editor.

---

### UInputModifierFOVScaling

```
DisplayName: "FOV Scaling"
Class: UInputModifierFOVScaling : public UInputModifier
```

**Behavior:** Scales the mouse look input value by the current camera FOV, maintaining
a consistent angular displacement per mouse unit regardless of zoom level.

**Properties:**

| Property | Type | Default | Description |
|---|---|---|---|
| `FOVScale` | `float` | `1.0` | Additional scalar applied on top of the FOV calculation |
| `FOVScalingType` | `EFOVScalingType` | `Standard` | `Standard` for new work; `UE4_BackCompat` reproduces the older incorrect `UPlayerInput::MassageAxisInput` calculation for back-compat only |

**Use for:** Mouse look on any character with a zoom/aim-down-sights mechanic.
Without this, zoomed-in aim feels faster than unzoomed aim in terms of screen pixels.

---

### UInputModifierToWorldSpace

```
DisplayName: "To World Space"
Class: UInputModifierToWorldSpace : public UInputModifier
```

**Behavior:** Converts a 2D axis input into world space. The up/down axis maps to world
forward (X), and left/right maps to world right (Y). Allows the value to be passed
directly to `AddMovementInput` or similar world-space functions without manual conversion.

No configurable properties.

**Use for:** Simplifying movement handler code when you want to receive a world-space
direction directly from the callback rather than rotating by the actor's facing.

---

## Modifier Execution Order

Modifiers are applied in this sequence:

1. Per-mapping modifiers (defined on each `FEnhancedActionKeyMapping` in the IMC)
2. Action-level modifiers (defined on the `UInputAction` asset `Modifiers` array)

Within each group, modifiers execute in array order. The output of modifier N is the
input to modifier N+1.

**Typical keyboard+mouse stack for a look action** (per mapping, in order):
```
Scalar (0.4, 0.4, 1.0)  ->  Smooth  ->  FOV Scaling
```

**Typical gamepad stick move action** (per mapping, in order):
```
Y stick axis mapping: Dead Zone (Radial, 0.2, 1.0)  ->  Swizzle Input Axis Values (YXZ)
X stick axis mapping: Dead Zone (Radial, 0.2, 1.0)
```
No action-level modifiers are needed in either case.

---

## Custom Trigger Interface

Override in a `UInputTrigger` subclass:

| Virtual Method | Signature | Description |
|---|---|---|
| `UpdateState_Implementation` | `ETriggerState(const UEnhancedPlayerInput*, FInputActionValue, float DeltaTime)` | Core per-tick evaluation; return None/Ongoing/Triggered |
| `GetTriggerType_Implementation` | `ETriggerType()` | Return Explicit, Implicit, or Blocker |
| `GetSupportedTriggerEvents` | `ETriggerEventsSupported() const` | Declare which `ETriggerEvent` types this trigger can produce |
| `IsBlocking` | `bool(const ETriggerState State) const` | Return true to block all other triggers (used by `UInputTriggerChordBlocker`) |
| `GetDebugState` | `FString() const` | Text shown in the `ShowDebug EnhancedInput` HUD |
| `ReceiveTriggerReinstanced_Implementation` | `void(const UInputTrigger* OldTrigger)` | Transfer runtime state when a mapping rebuild replaces this trigger instance; call `Super` |

`UInputTrigger::IsActuated(const FInputActionValue&)` helper:
Returns `true` if `Value.GetMagnitudeSq() >= ActuationThreshold * ActuationThreshold`.

`UInputTriggerTimedBase` base class provides:
- `HeldDuration` — accumulated time while actuated
- `CalculateHeldDuration(PlayerInput, DeltaTime)` — increments with optional time dilation
- Automatically transitions to `Ongoing` on first actuation

---

## Custom Modifier Interface

Override in a `UInputModifier` subclass:

| Virtual Method | Signature | Description |
|---|---|---|
| `ModifyRaw_Implementation` | `FInputActionValue(const UEnhancedPlayerInput* PlayerInput, FInputActionValue CurrentValue, float DeltaTime)` | Transform and return the modified value |
| `GetVisualizationColor_Implementation` | `FLinearColor(FInputActionValue SampleValue, FInputActionValue FinalValue) const` | Color used in debug visualization overlays |

The returned `FInputActionValue` will be cast back to the action's `ValueType` before
further processing, so you can return any type internally.

**Worked example:**

```cpp
// MyClampMagnitudeModifier.h
#pragma once

#include "InputModifiers.h"
#include "MyClampMagnitudeModifier.generated.h"

UCLASS(EditInlineNew, meta = (DisplayName = "Clamp Magnitude"))
class MYGAME_API UMyClampMagnitudeModifier : public UInputModifier
{
	GENERATED_BODY()

public:
	UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Settings")
	float MaxMagnitude = 1.0f;

protected:
	virtual FInputActionValue ModifyRaw_Implementation(
		const UEnhancedPlayerInput* PlayerInput, FInputActionValue CurrentValue, float DeltaTime) override
	{
		FVector Direction = CurrentValue.Get<FVector>();
		if (Direction.SizeSquared() > MaxMagnitude * MaxMagnitude)
		{
			Direction = Direction.GetSafeNormal() * MaxMagnitude;
		}
		return FInputActionValue(CurrentValue.GetValueType(), Direction);
	}
};
```

---

## UInputMappingContext Properties

```cpp
class UInputMappingContext : public UDataAsset
```

| Property / Method | Type | Description |
|---|---|---|
| `DefaultKeyMappings` | `FInputMappingContextMappingData` | Default key-to-action bindings; read the array with `GetMappings()` |
| `MappingProfileOverrides` | `TMap<FString, FInputMappingContextMappingData>` | Per key-profile overrides, keyed by profile id string |
| `RegistrationTrackingMode` | `EMappingContextRegistrationTrackingMode` | `Untracked` or `CountRegistrations` |
| `InputModeFilterOptions` | `EMappingContextInputModeFilterOptions` | `UseProjectDefaultQuery`, `UseCustomQuery`, `DoNotFilter` |
| `InputModeQueryOverride` | `FGameplayTagQuery` | Used when `InputModeFilterOptions` is `UseCustomQuery` |
| `ContextDescription` | `FText` | Localized description shown in the editor |
| `GetMappings()` | `const TArray<FEnhancedActionKeyMapping>&` | Accessor for the default mappings |
| `GetMappingsForProfile(ProfileId)` | `const TArray<FEnhancedActionKeyMapping>&` | Override mappings for a profile, falling back to the defaults |
| `ForEachKeyMapping(Func)` | `void` | Visit every default and override mapping |
| `HasMappingsForProfile(ProfileId)` / `GetProfilesWithOverridenMappings()` | `bool` / `TArray<FString>` | Profile override queries |
| `ShouldFilterMappingByInputMode()` / `GetInputModeQuery()` | `bool` / `FGameplayTagQuery` | Resolved input-mode filtering for this IMC |
| `GetRegistrationTrackingMode()` / `GetInputModeFilterOptions()` | enum | Accessors for the two enums above |
| `MapKey(Action, Key)` | `FEnhancedActionKeyMapping&` | Editor/config-screen only: add a key binding to the asset |
| `UnmapKey(Action, Key)` / `UnmapAllKeysFromAction(Action)` | `void` | Editor/config-screen only: remove bindings from the asset |
| `HasMappingForInputAction(Action)` | `bool` | True if the action appears in the default or override mappings |

After mutating an IMC, call `UEnhancedInputLibrary::RequestRebuildControlMappingsUsingContext(Context, bForceImmediately)`.

### FEnhancedActionKeyMapping

| Member | Type | Description |
|---|---|---|
| `Action` | `TObjectPtr<const UInputAction>` | The action this key feeds |
| `Key` | `FKey` | The physical key |
| `Triggers` / `Modifiers` | `TArray<TObjectPtr<UInputTrigger>>` / `TArray<TObjectPtr<UInputModifier>>` | Per-mapping, evaluated before the action-level arrays |
| `SettingBehavior` | `EPlayerMappableKeySettingBehaviors` | `InheritSettingsFromAction`, `OverrideSettings`, `IgnoreSettings` |
| `PlayerMappableKeySettings` | `TObjectPtr<UPlayerMappableKeySettings>` | Used when `SettingBehavior` is `OverrideSettings` |
| `GetPlayerMappableKeySettings()` | `UPlayerMappableKeySettings*` | Resolves the settings per `SettingBehavior`; template form casts to a subclass |
| `GetMappingName()` | `FName` | The player-mapping row name used by the user settings |

### UPlayerMappableKeySettings

| Property | Type | Description |
|---|---|---|
| `Name` | `FName` | The mapping name used as the user-settings row key |
| `DisplayName` | `FText` | Label for a rebinding UI |
| `DisplayCategory` | `FText` | Grouping for a rebinding UI |
| `Metadata` | `TObjectPtr<UObject>` | Free-form payload (icons, ability assets, and so on) |
| `SupportedKeyProfileIds` | `TArray<FString>` | Restricts the mapping to specific key profiles |

`UPlayerMappableKeySettings::GetKnownMappingNames()` returns every registered mapping name and is the
`GetOptions` source for `UPROPERTY(meta=(GetOptions="EnhancedInput.PlayerMappableKeySettings.GetKnownMappingNames"))`.

### EMappingContextRegistrationTrackingMode

| Value | Behavior |
|---|---|
| `Untracked` | First `RemoveMappingContext` call removes the IMC regardless of how many times it was added |
| `CountRegistrations` | IMC stays active until `RemoveMappingContext` is called the same number of times as `AddMappingContext`. Useful when multiple subsystems share an IMC. |

---

## UEnhancedInputLocalPlayerSubsystem API

```cpp
// Add a context (BeginPlay or on mode change)
Subsystem->AddMappingContext(IMC, Priority, Options);

// Remove a context (on mode change, unpossess)
Subsystem->RemoveMappingContext(IMC, Options);

// Remove all contexts
Subsystem->ClearAllMappings();

// Check if a context is currently active
Subsystem->HasMappingContext(IMC);

// Query current value of an action without a binding
Subsystem->GetPlayerInput()->GetActionValue(InputAction);
```

**FModifyContextOptions** — every field already holds the default below, so pass the struct only to change one:

| Field | Type | Default | Description |
|---|---|---|---|
| `bIgnoreAllPressedKeysUntilRelease` | `bool` | `true` | Keys already down when mappings rebuild are ignored until released, which is what stops "stuck key" ghost inputs across a context switch. Set `false` to let a held key carry into the new context. |
| `bForceImmediately` | `bool` | `false` | Rebuild mapping caches immediately rather than deferring to the end of the frame |
| `bNotifyUserSettings` | `bool` | `false` | Register the context's mappings with `UEnhancedInputUserSettings` (needed for a rebinding UI) |

Other members of `IEnhancedInputSubsystemInterface`, shared by the local-player and world subsystems:

| Member | Signature | Purpose |
|---|---|---|
| `GetPlayerInput()` | `UEnhancedPlayerInput*` | The player input object; `GetActionValue(Action)` reads a current value |
| `GetUserSettings()` | `UEnhancedInputUserSettings*` | Player key-rebinding settings |
| `HasMappingContext(IMC)` / `HasMappingContext(IMC, int32& OutFoundPriority)` | `bool` | Is the context applied, and at what priority |
| `QueryKeysMappedToAction(Action)` | `TArray<FKey>` | Keys currently feeding an action |
| `GetAllPlayerMappableActionKeyMappings()` | `TArray<FEnhancedActionKeyMapping>` | Every applied mapping that is player-mappable |
| `InjectInputForAction(Action, RawValue, Modifiers, Triggers)` | `void` | Simulate input as if it came from a device |
| `InjectInputVectorForAction(Action, Value, Modifiers, Triggers)` | `void` | Same, with an explicit `FVector` |
| `ShowDebugInfo(Canvas)` | `void(UCanvas*)` | Backing call for `ShowDebug EnhancedInput` |

---

## Input Mode Filtering

Enable `bEnableInputModeFiltering` on `UEnhancedInputDeveloperSettings` first. The subsystem keeps a
`FGameplayTagContainer` of the current mode; contexts whose query fails stay registered but stop producing input,
so priorities and registration counts are untouched.

| Member | Signature | Purpose |
|---|---|---|
| `GetInputMode()` | `FGameplayTagContainer() const` | The current mode |
| `SetInputMode(NewMode, Options)` | `void(const FGameplayTagContainer&, const FModifyContextOptions&)` | Replace the mode |
| `AppendTagsToInputMode(TagsToAdd, Options)` | `void(const FGameplayTagContainer&, const FModifyContextOptions&)` | Add several tags |
| `AddTagToInputMode(TagToAdd, Options)` | `void(const FGameplayTag&, const FModifyContextOptions&)` | Add one tag |
| `RemoveTagsFromInputMode(TagsToRemove, Options)` | `void(const FGameplayTagContainer&, const FModifyContextOptions&)` | Remove several tags |
| `RemoveTagFromInputMode(TagToRemove, Options)` | `void(const FGameplayTag&, const FModifyContextOptions&)` | Remove one tag |

### EMappingContextInputModeFilterOptions

| Value | Behavior |
|---|---|
| `UseProjectDefaultQuery` | Match against `UEnhancedInputDeveloperSettings::DefaultMappingContextInputModeQuery` (the default) |
| `UseCustomQuery` | Match against this IMC's own `InputModeQueryOverride` |
| `DoNotFilter` | Always active, whatever the current mode |

`UEnhancedInputDeveloperSettings::DefaultInputMode` is the tag container every new `UEnhancedPlayerInput` starts with.

---

## UEnhancedInputUserSettings

Header: `UserSettings/EnhancedInputUserSettings.h`. A `USaveGame` created per subsystem when
`bEnableUserSettings` is set; the class comes from `UserSettingsClass` and the save slot from
`InputSettingsSaveSlotName`. This is the supported runtime key-rebinding path.

| Member | Signature | Purpose |
|---|---|---|
| `LoadOrCreateSettings(LP)` | `static UEnhancedInputUserSettings*(ULocalPlayer*)` | Load or create for a local player |
| `RegisterInputMappingContext(IMC)` | `bool(const UInputMappingContext*)` | Make an IMC's mappings visible to the settings |
| `RegisterInputMappingContexts(MappingContexts)` | `bool(const TSet<UInputMappingContext*>&)` | Batch form |
| `UnregisterInputMappingContext(IMC)` / `UnregisterInputMappingContexts(MappingContexts)` | `bool` | Inverse of the above |
| `MapPlayerKey(InArgs, FailureReason)` | `void(const FMapPlayerKeyArgs&, FGameplayTagContainer&)` | Set one player mapping |
| `UnMapPlayerKey(InArgs, FailureReason)` | `void(const FMapPlayerKeyArgs&, FGameplayTagContainer&)` | Clear one player mapping |
| `ResetAllPlayerKeysInRow(InArgs, FailureReason)` | `void(const FMapPlayerKeyArgs&, FGameplayTagContainer&)` | Reset every slot of one mapping name |
| `ResetKeyProfileIdToDefault(ProfileId, FailureReason)` | `void(const FString&, FGameplayTagContainer&)` | Reset a whole profile |
| `FindKeyMapping(InArgs)` | `FPlayerKeyMapping*(const FMapPlayerKeyArgs&) const` | Look up a single mapping |
| `FindMappingsInRow(MappingName)` | `const TSet<FPlayerKeyMapping>&(const FName) const` | Every mapping for one name |
| `FindCurrentMappingForSlot(MappingName, InSlot)` | `const FPlayerKeyMapping*(const FName, const EPlayerMappableKeySlot) const` | One slot's mapping |
| `FindInputActionForMapping(MappingName)` | `const UInputAction*(const FName) const` | The action behind a mapping name |
| `SaveSettings()` / `AsyncSaveSettings()` / `ApplySettings()` | `void` | Persist and apply |
| `GetActiveKeyProfile()` / `GetActiveKeyProfileAs<T>()` | `UEnhancedPlayerMappableKeyProfile*` | Current profile |
| `GetActiveKeyProfileId()` | `const FString&` | Current profile id |
| `SetActiveKeyProfile(InProfileId)` / `SetKeyProfileToDefault()` | `bool` | Switch profile |
| `GetAllAvailableKeyProfiles()` | `const TMap<FString, TObjectPtr<UEnhancedPlayerMappableKeyProfile>>&` | Every saved profile |
| `IsKeyProfileAvailable(ProfileId)` / `GetKeyProfileWithId(ProfileId)` / `GetKeyProfileWithIdAs<T>(ProfileId)` | — | Profile lookup |
| `GetDefaultKeyProfile()` | `UEnhancedPlayerMappableKeyProfile*` | The engine-created default profile |
| `CreateNewKeyProfile(InArgs)` | `UEnhancedPlayerMappableKeyProfile*(const FPlayerMappableKeyProfileCreationArgs&)` | Add a profile |
| `OnSettingsApplied` / `OnKeyProfileChanged` | dynamic multicast delegates | Refresh a settings UI |

Profile identifiers are `FString`. `MapPlayerKey` and its siblings report problems through the
`FGameplayTagContainer& FailureReason` out-parameter rather than a return value: an empty container means success.

### FMapPlayerKeyArgs

| Field | Type | Description |
|---|---|---|
| `MappingName` | `FName` | The player-mapping row, from `UPlayerMappableKeySettings::Name` or a per-mapping override |
| `Slot` | `EPlayerMappableKeySlot` | `First` through `Seventh`, plus `Unspecified` |
| `NewKey` | `FKey` | The key to bind |
| `HardwareDeviceId` | `FName` | Optional hardware-device qualifier |
| `ProfileIdString` | `FString` | Profile to write to; empty means the active profile |
| `bCreateMatchingSlotIfNeeded` | `uint8 : 1` | Create the slot when no mapping matches the slot plus device |
| `bDeferOnSettingsChangedBroadcast` | `uint8 : 1` | Defer the changed delegate to the next frame |

### UEnhancedPlayerMappableKeyProfile

| Member | Signature | Purpose |
|---|---|---|
| `GetProfileIdString()` | `const FString&` | Profile identifier |
| `GetProfileDisplayName()` / `SetDisplayName(NewDisplayName)` | `const FText&` / `void(const FText&)` | Display name |
| `GetPlayerMappingRows()` | `const TMap<FName, FKeyMappingRow>&` | Every mapping row in the profile |
| `FindKeyMappingRow(InMappingName)` / `FindKeyMappingRowMutable(InMappingName)` | `const FKeyMappingRow*` / `FKeyMappingRow*` | One row |
| `ResetMappingToDefault(InMappingName)` | `void(const FName)` | Reset one row |
| `DumpProfileToLog()` | `void` | Debug dump |

`FKeyMappingRow` holds a `TSet<FPlayerKeyMapping> Mappings` and answers `HasAnyMappings()`.

### FPlayerKeyMapping

| Member | Signature | Purpose |
|---|---|---|
| `IsCustomized()` | `bool() const` | True when the player changed it from the default |
| `IsValid()` | `bool() const` | True when the mapping is usable |
| `IsDirty()` | `const bool() const` | True when it differs from what was last saved |
| `SetCurrentKey(NewKey)` | `void(const FKey&)` | Assign a key |
| `SetHardwareDeviceId(InDeviceId)` | `void(const FHardwareDeviceIdentifier&)` | Assign a device |
| `ResetToDefault()` | `void` | Restore the mapping's default key |
| `ToString()` | `FString() const` | Debug text |

---

## UEnhancedInputWorldSubsystem

A `UWorldSubsystem` implementing `IEnhancedInputSubsystemInterface`, DisplayName "Enhanced Input World Subsystem
(Experimental)". Use it to bind input on actors that will never have an owning `APlayerController`. Enable it with
`bEnableWorldSubsystem` in `UEnhancedInputDeveloperSettings`; `DefaultWorldInputClass` chooses its
`UEnhancedPlayerInput` class.

| Member | Signature | Purpose |
|---|---|---|
| `AddActorInputComponent(Actor)` | `void(AActor*)` | Push the actor's `InputComponent` onto the subsystem's stack |
| `RemoveActorInputComponent(Actor)` | `bool(AActor*)` | Pop it off again |
| `GetPlayerInput()` | `UEnhancedPlayerInput*` | The subsystem's own player input |
| `ShowDebugInfo(Canvas)` | `void(UCanvas*)` | Backing call for `ShowDebug WorldSubsystemInput` |

The actor needs an `InputComponent`. `AActor::EnableInput` only creates one when given a `PlayerController` (`Actor.cpp:4895-4900`), so with no controller create it yourself: `NewObject<UEnhancedInputComponent>(this)`, `RegisterComponent()`, assign it to `InputComponent`, then add it (as in the SKILL.md example).
`AddMappingContext` and the rest of the interface behave exactly as on the local-player subsystem, but there is no
`UEnhancedInputUserSettings` here.

---

## EnhancedInputPlatformSettings

Header: `EnhancedInputPlatformSettings.h`. Build.cs: add `DeveloperSettings` when you call `Get()` — it is inline and
calls `UPlatformSettingsManager` (`DeveloperSettings/Public/Engine/PlatformSettingsManager.h`), so the link fails without it.

```cpp
UEnhancedInputPlatformSettings* PlatformSettings = UEnhancedInputPlatformSettings::Get();
```

| Member | Signature | Purpose |
|---|---|---|
| `Get()` | `static UEnhancedInputPlatformSettings*` | Resolves through `UPlatformSettingsManager::Get().GetSettingsForPlatform` |
| `GetInputData()` | `const TArray<TSoftClassPtr<UEnhancedInputPlatformData>>&` | Configured platform data classes |
| `ForEachInputData(Predicate)` | `void(TFunctionRef<void(const UEnhancedInputPlatformData&)>)` | Visit each loaded platform data |
| `GetAllMappingContextRedirects(OutRedirects)` | `void(TMap<TObjectPtr<const UInputMappingContext>, TObjectPtr<const UInputMappingContext>>&)` | Collect every redirect |

`UEnhancedInputPlatformData` is an abstract, Blueprintable `UObject`. Subclass it per platform and fill
`MappingContextRedirects` to swap one `UInputMappingContext` for another; `GetContextRedirect(InContext)` returns
the replacement, or `InContext` itself when there is none.

---

## UEnhancedInputLibrary

A `UBlueprintFunctionLibrary` in `EnhancedInputLibrary.h`.

| Function | Signature | Purpose |
|---|---|---|
| `RequestRebuildControlMappingsUsingContext` | `static void(const UInputMappingContext* Context, bool bForceImmediately = false)` | Rebuild mappings after editing an IMC |
| `ForEachSubsystem` | `static void(TFunctionRef<void(IEnhancedInputSubsystemInterface*)>)` | Visit every Enhanced Input subsystem |
| `GetPlayerMappableKeySettings` | `static UPlayerMappableKeySettings*(const FEnhancedActionKeyMapping&)` | Resolved settings for a mapping |
| `GetMappingName` | `static FName(const FEnhancedActionKeyMapping&)` | Player-mapping row name |
| `IsActionKeyMappingPlayerMappable` | `static bool(const FEnhancedActionKeyMapping&)` | Whether a settings UI should list it |
| `MakeInputActionValueOfType` / `BreakInputActionValue` | `static` | Construct or decompose an `FInputActionValue` |
| `GetBoundActionValue` | `static FInputActionValue(AActor*, const UInputAction*)` | Current value of an action bound on an actor |

---

## FInputActionValue

| Member | Signature | Notes |
|---|---|---|
| `Get<T>()` | `T() const` | `T` is `bool`, `FInputActionValue::Axis1D` (`float`), `Axis2D` (`FVector2D`) or `Axis3D` (`FVector`) |
| `GetValueType()` | `EInputActionValueType() const` | `Boolean`, `Axis1D`, `Axis2D`, `Axis3D` |
| `GetMagnitude()` / `GetMagnitudeSq()` | `float() const` | Shape-aware magnitude; `Boolean` and `Axis1D` use X only |
| `IsNonZero(Tolerance = KINDA_SMALL_NUMBER)` | `bool(float) const` | Squared-length test |
| `ConvertToType(Type)` / `ConvertToType(Other)` | `FInputActionValue&` | Reshape in place |
| `GetValueTypeFromKey(Key)` | `static EInputActionValueType(FKey)` | Derives the shape from an `FKey` |
| `FInputActionValue(EInputActionValueType InValueType, Axis3D InValue)` | constructor | Builds an arbitrary shape from a `FVector` — the form custom modifiers return |
