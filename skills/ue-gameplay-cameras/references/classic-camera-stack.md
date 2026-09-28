# Classic Camera Stack Reference (UE 5.8)

Everything here is in the `Engine` module (`Source/Runtime/Engine/Classes/Camera`, `Classes/GameFramework`), the `CinematicCamera` module, or the `EngineCameras` plugin. Build.cs: `"Engine"`, plus `"CinematicCamera"` and `"EngineCameras"` as needed.

## FMinimalViewInfo

`Camera/CameraTypes.h`. This is the struct every camera path produces.

| Field | Type | Notes |
|---|---|---|
| `Location` | `FVector` | world position |
| `Rotation` | `FRotator` | world orientation |
| `FOV` | `float` | horizontal degrees, perspective only |
| `DesiredFOV` | `float` | transient; pre-aspect-adjustment value |
| `FirstPersonFOV` | `float` | applied to primitives tagged first person |
| `FirstPersonScale` | `float` | shrinks first-person primitives toward the camera |
| `OrthoWidth` | `float` | orthographic only |
| `bAutoCalculateOrthoPlanes` | `bool` | with `AutoPlaneShift`, `bUpdateOrthoPlanes`, `bUseCameraHeightAsViewTarget` |
| `OrthoNearClipPlane` / `OrthoFarClipPlane` | `float` | orthographic clip range |
| `PerspectiveNearClipPlane` | `float` | negative means use the global default |
| `AspectRatio` | `float` | width / height |
| `AspectRatioAxisConstraint` | `TOptional<EAspectRatioAxisConstraint>` | override for this view |
| `bConstrainAspectRatio` | `uint32:1` | adds black bars when the viewport differs |
| `bUseFirstPersonParameters` | `uint32:1` | gates `FirstPersonFOV` / `FirstPersonScale` |
| `bUseFieldOfViewForLOD` | `uint32:1` | FOV feeds mesh LOD selection |
| `ProjectionMode` | `TEnumAsByte<ECameraProjectionMode::Type>` | perspective or orthographic |
| `PostProcessBlendWeight` | `float` | 0 disables `PostProcessSettings` |
| `PostProcessSettings` | `FPostProcessSettings` | see below |
| `OffCenterProjectionOffset` | `FVector2D` | proportion of screen dimensions |
| `PreviousViewTransform` | `TOptional<FTransform>` | motion-vector hint |
| `OverscanResolutionFraction` | `float` | resolution scale that follows overscan |
| `CropFraction` | `float` | 1.0 means no crop |

## APlayerCameraManager

`Camera/PlayerCameraManager.h`. `UCLASS(notplaceable, transient, BlueprintType, Blueprintable, Config=Engine, MinimalAPI)`, base `AActor`.

Key state: `PCOwner`, `CameraStyle` (`FName`), `DefaultFOV`, `DefaultOrthoWidth`, `DefaultAspectRatio`, `ViewTarget` and `PendingViewTarget` (`FTViewTarget`), `BlendTimeToGo`, `BlendParams` (`FViewTargetTransitionParams`), `ModifierList`, `DefaultModifiers`, `FreeCamDistance`, `ViewPitchMin`, `ViewPitchMax`, `ViewYawMin`, `ViewYawMax`, `ViewRollMin`, `ViewRollMax`, `FadeColor`, `FadeAmount`, `ColorScale`, `bIsOrthographic`.

Virtuals worth overriding:

```cpp
virtual void UpdateCamera(float DeltaTime);
virtual void DoUpdateCamera(float DeltaTime);
virtual void UpdateViewTarget(FTViewTarget& OutVT, float DeltaTime);
virtual void UpdateViewTargetInternal(FTViewTarget& OutVT, float DeltaTime);
virtual void ApplyCameraModifiers(float DeltaTime, FMinimalViewInfo& InOutPOV);
virtual void InitializeFor(class APlayerController* PC);
virtual void ProcessViewRotation(float DeltaTime, FRotator& OutViewRotation, FRotator& OutDeltaRot);
virtual void SetViewTarget(class AActor* NewViewTarget, FViewTargetTransitionParams TransitionParams = FViewTargetTransitionParams());
virtual void AssignViewTarget(AActor* NewTarget, FTViewTarget& VT, struct FViewTargetTransitionParams TransitionParams = FViewTargetTransitionParams());
virtual FRotator GetCameraRotation() const;
virtual FVector GetCameraLocation() const;
virtual void LimitViewPitch(FRotator& ViewRotation, float InViewPitchMin, float InViewPitchMax);
virtual void LimitViewYaw(FRotator& ViewRotation, float InViewYawMin, float InViewYawMax);
virtual void LimitViewRoll(FRotator& ViewRotation, float InViewRollMin, float InViewRollMax);
virtual void DisplayDebug(class UCanvas* Canvas, const FDebugDisplayInfo& DebugDisplay, float& YL, float& YPos) override;
```

Read-only accessors: `AActor* GetViewTarget() const`, `virtual const FMinimalViewInfo& GetCameraCacheView() const`, `virtual const FMinimalViewInfo& GetLastFrameCameraCacheView() const`, `float GetCameraCacheTime() const`, `float GetLockedFOV() const`. `OnBlendComplete()` returns the `FOnBlendComplete` event fired when `PendingViewTarget` is promoted.

`BlueprintUpdateCamera(AActor* CameraTarget, FVector& NewCameraLocation, FRotator& NewCameraRotation, float& NewCameraFOV)` is the Blueprint hook that short-circuits `UpdateViewTarget` when it returns true.

## FTViewTarget and FViewTargetTransitionParams

```cpp
USTRUCT(BlueprintType)
struct FTViewTarget
{
    TObjectPtr<class AActor> Target;
    struct FMinimalViewInfo POV;
    class APlayerState* GetPlayerState() const;
    void SetNewTarget(AActor* NewTarget);
    class APawn* GetTargetPawn() const;
    bool Equal(const FTViewTarget& OtherTarget) const;
    void CheckViewTarget(APlayerController* OwningController);
};
```

`FViewTargetTransitionParams` holds `BlendTime`, `BlendFunction` (`TEnumAsByte<enum EViewTargetBlendFunction>`), `BlendExp`, `bLockOutgoing`, and `float GetBlendAlpha(const float& TimePct) const`.

`EViewTargetBlendOrder` (`VTBlendOrder_Base`, `VTBlendOrder_Override`) exists alongside `EViewTargetBlendFunction` for systems that stack blends.

## UCameraComponent

`Camera/CameraComponent.h`, base `USceneComponent`, `ClassGroup=Camera`, `meta=(BlueprintSpawnableComponent)`.

Properties: `FieldOfView`, `FirstPersonFieldOfView`, `FirstPersonScale`, `OrthoWidth`, `bAutoCalculateOrthoPlanes`, `AutoPlaneShift`, `OrthoNearClipPlane`, `OrthoFarClipPlane`, `AspectRatio`, `bConstrainAspectRatio`, `bUseFieldOfViewForLOD`, `Overscan`, `bScaleResolutionWithOverscan`, `bCropOverscan`, `ProjectionMode`, `bLockToHmd`, `bUsePawnControlRotation`, `bEnableFirstPersonFieldOfView`, `bEnableFirstPersonScale`, `PostProcessBlendWeight`, `PostProcessSettings`.

Setters exist for nearly all of them (`SetFieldOfView`, `SetOrthoWidth`, `SetAspectRatio`, `SetProjectionMode`, `SetPostProcessBlendWeight`, …). The one virtual you normally override is:

```cpp
virtual void GetCameraView(float DeltaTime, FMinimalViewInfo& DesiredView);
```

Also useful: `void AddOrUpdateBlendable(TScriptInterface<IBlendableInterface> InBlendableObject, float InWeight = 1.0f)`, `void RemoveBlendable(TScriptInterface<IBlendableInterface> InBlendableObject)`, `void AddAdditiveOffset(FTransform const& Transform, float FOV)` / `ClearAdditiveOffset()` / `GetAdditiveOffset(FTransform& OutAdditiveOffset, float& OutAdditiveFOVOffset) const`, `void AddExtraPostProcessBlend(FPostProcessSettings const& PPSettings, float PPBlendWeight)` / `ClearExtraPostProcessBlends()` (stored only — not applied by `GetCameraView`, and only the level-editor viewport reads them, `Camera/CameraComponent.h:323`), and `virtual void NotifyCameraCut()`.

`ACameraActor` wraps one: `TEnumAsByte<EAutoReceiveInput::Type> AutoActivateForPlayer`, `int32 GetAutoActivatePlayerIndex() const`, `class UCameraComponent* GetCameraComponent() const`.

## Post-process on a camera

`UCameraComponent::PostProcessSettings` is an `FPostProcessSettings` (`Engine/Scene.h`) blended at `PostProcessBlendWeight` (0 = off, 1 = full). Every field is gated by a matching `bOverride_` flag — setting `DepthOfFieldFstop` without `bOverride_DepthOfFieldFstop = true` does nothing:

```cpp
// MyCharacter.cpp - include Camera/CameraComponent.h
void AMyCharacter::ApplyCinematicLook(UCameraComponent* Camera)
{
    Camera->PostProcessBlendWeight = 1.f;
    Camera->PostProcessSettings.bOverride_MotionBlurAmount = true;
    Camera->PostProcessSettings.MotionBlurAmount = 0.2f;
    Camera->PostProcessSettings.bOverride_DepthOfFieldFstop = true;
    Camera->PostProcessSettings.DepthOfFieldFstop = 2.8f;
}
```

Camera post-process is blended after all volumes (`LocalPlayer.cpp:970`), so it overrides any `APostProcessVolume` regardless of volume priority. `FWeightedBlendables` inside the settings holds material blendables added by `AddOrUpdateBlendable`.

## UCameraModifier

`Camera/CameraModifier.h`, `UCLASS(BlueprintType, Blueprintable, MinimalAPI)`, base `UObject`.

```cpp
virtual bool ModifyCamera(float DeltaTime, struct FMinimalViewInfo& InOutPOV);
virtual bool ProcessViewRotation(class AActor* ViewTarget, float DeltaTime, FRotator& OutViewRotation, FRotator& OutDeltaRot);
virtual void AddedToCamera(APlayerCameraManager* Camera);
virtual void UpdateAlpha(float DeltaTime);
virtual float GetTargetAlpha();
virtual void EnableModifier();
virtual void DisableModifier(bool bImmediate = false);
virtual void ToggleModifier();
virtual bool IsDisabled() const;
virtual bool IsPendingDisable() const;
virtual AActor* GetViewTarget() const;
virtual void DisplayDebug(class UCanvas* Canvas, const FDebugDisplayInfo& DebugDisplay, float& YL, float& YPos);
```

State: `Priority` (`uint8`, 0 = highest), `bExclusive`, `bDebug`, `CameraOwner`, `AlphaInTime`, `AlphaOutTime`, `Alpha`. Blueprint-only hooks: `BlueprintModifyCamera` and `BlueprintModifyPostProcess`.

There is a second, non-virtual-extension overload `virtual void ModifyCamera(float DeltaTime, FVector ViewLocation, FRotator ViewRotation, float FOV, FVector& NewViewLocation, FRotator& NewViewRotation, float& NewFOV)` used internally to bridge to Blueprint. Override the `FMinimalViewInfo` form.

## Camera shakes

`UCameraShakeBase` (`Camera/CameraShakeBase.h`, `UCLASS(Abstract, Blueprintable, EditInlineNew, MinimalAPI)`) owns a private `RootShakePattern` and exposes `UCameraShakePattern* GetRootShakePattern() const`, `void SetRootShakePattern(UCameraShakePattern* InPattern)` and the template helper `ChangeRootShakePattern<ShakePatternType>()`. Runtime state: `bSingleInstance`, `ShakeScale`, `APlayerCameraManager* GetCameraManager() const`, `ECameraShakePlaySpace GetPlaySpace() const`, `const FMatrix& GetUserPlaySpaceMatrix() const`.

Lifecycle calls the manager makes: `StartShake(APlayerCameraManager* Camera, float Scale, ECameraShakePlaySpace InPlaySpace, FRotator UserPlaySpaceRot = FRotator::ZeroRotator)` or `StartShake(const FCameraShakeBaseStartParams& Params)`, then `UpdateAndApplyCameraShake(float DeltaTime, float Alpha, FMinimalViewInfo& InOutPOV)`, `bool IsFinished() const`, `StopShake(bool bImmediately = true)`, `TeardownShake()`.

### Writing a shake pattern

```cpp
// MyShakePattern.h
#pragma once

#include "CoreMinimal.h"
#include "Shakes/SimpleCameraShakePattern.h"
#include "MyShakePattern.generated.h"

UCLASS()
class MYGAME_API UMyShakePattern : public USimpleCameraShakePattern
{
    GENERATED_BODY()

public:
    UMyShakePattern(const FObjectInitializer& ObjInit);

    UPROPERTY(EditAnywhere, Category = "Shake")
    float PitchAmplitude = 2.f;

    UPROPERTY(EditAnywhere, Category = "Shake")
    float PitchFrequency = 10.f;

protected:
    virtual void UpdateShakePatternImpl(const FCameraShakePatternUpdateParams& Params, FCameraShakePatternUpdateResult& OutResult) override;
};

// MyShakePattern.cpp
#include "MyShakePattern.h"

UMyShakePattern::UMyShakePattern(const FObjectInitializer& ObjInit)
    : Super(ObjInit)
{
    Duration = 0.4f;
    BlendInTime = 0.05f;
    BlendOutTime = 0.2f;
}

void UMyShakePattern::UpdateShakePatternImpl(const FCameraShakePatternUpdateParams& Params, FCameraShakePatternUpdateResult& OutResult)
{
    // Advance the inherited State: it tracks Duration and the blend in/out, and IsFinishedImpl reads it.
    const float BlendWeight = State.Update(Params.DeltaTime);
    OutResult.Rotation.Pitch = PitchAmplitude * FMath::Sin(State.GetElapsedTime() * PitchFrequency * 2.f * UE_PI);
    OutResult.ApplyScale(BlendWeight);
}
```

`USimpleCameraShakePattern` (module `"EngineCameras"`) supplies `Duration`, `BlendInTime`, `BlendOutTime` and implements the info/start/finish/stop/teardown hooks, so a subclass usually only writes `UpdateShakePatternImpl`. That override must call `State.Update(Params.DeltaTime)` and scale its output by the returned weight, as the shipped patterns do (`PerlinNoiseCameraShakePattern.cpp:49-50`); `IsFinishedImpl` returns `!State.IsPlaying()`, so a pattern that never updates `State` never finishes. The full protected set on `UCameraShakePattern`:

```cpp
virtual void GetShakePatternInfoImpl(FCameraShakeInfo& OutInfo) const;
virtual void StartShakePatternImpl(const FCameraShakePatternStartParams& Params);
virtual void UpdateShakePatternImpl(const FCameraShakePatternUpdateParams& Params, FCameraShakePatternUpdateResult& OutResult);
virtual void ScrubShakePatternImpl(const FCameraShakePatternScrubParams& Params, FCameraShakePatternUpdateResult& OutResult);
virtual bool IsFinishedImpl() const;
virtual void StopShakePatternImpl(const FCameraShakePatternStopParams& Params);
virtual void TeardownShakePatternImpl();
```

### Shipped patterns

| Class | Module | Shape |
|---|---|---|
| `UPerlinNoiseCameraShakePattern` | `EngineCameras` | `FPerlinNoiseShaker` (`Amplitude`, `Frequency`) on `X`/`Y`/`Z`, `Pitch`/`Yaw`/`Roll`, `FOV`, plus `LocationAmplitudeMultiplier`, `LocationFrequencyMultiplier`, `RotationAmplitudeMultiplier`, `RotationFrequencyMultiplier` |
| `UWaveOscillatorCameraShakePattern` | `EngineCameras` | `FWaveOscillator` per axis; same multiplier layout |
| `UCompositeCameraShakePattern` | `EngineCameras` | `ChildPatterns`, runs several patterns as one |
| `ULegacyCameraShakePattern` | `EngineCameras` | backs `ULegacyCameraShake`'s oscillator fields |
| `USequenceCameraShakePattern` | `TemplateSequence` | plays a `UCameraAnimationSequence` |

`ULegacyCameraShake` (`UCLASS(MinimalAPI, Blueprintable, HideCategories = (CameraShakePattern))`) keeps the oscillator-style authoring surface (`FFOscillator`, `FROscillator`, `FVOscillator`, `OscillationDuration`, `OscillationBlendInTime`, `OscillationBlendOutTime`) and offers the statics `static ULegacyCameraShake* StartLegacyCameraShake(APlayerCameraManager* PlayerCameraManager, TSubclassOf<ULegacyCameraShake> ShakeClass, float Scale = 1.f, ECameraShakePlaySpace PlaySpace = ECameraShakePlaySpace::CameraLocal, FRotator UserPlaySpaceRot = FRotator::ZeroRotator)` and `static ULegacyCameraShake* StartLegacyCameraShakeFromSource(APlayerCameraManager* PlayerCameraManager, TSubclassOf<ULegacyCameraShake> ShakeClass, UCameraShakeSourceComponent* SourceComponent, float Scale = 1.f, ECameraShakePlaySpace PlaySpace = ECameraShakePlaySpace::CameraLocal, FRotator UserPlaySpaceRot = FRotator::ZeroRotator)`. `UDefaultCameraShakeBase` is a `UCameraShakeBase` whose `RootShakePattern` subobject class is `UPerlinNoiseCameraShakePattern`.

### UCameraModifier_CameraShake

The manager creates one by default; it owns the running shakes.

```cpp
virtual UCameraShakeBase* AddCameraShake(TSubclassOf<UCameraShakeBase> NewShake, const FAddCameraShakeParams& Params);
virtual void GetActiveCameraShakes(TArray<FActiveCameraShakeInfo>& ActiveCameraShakes) const;
virtual void RemoveCameraShake(UCameraShakeBase* ShakeInst, bool bImmediately = true);
virtual void RemoveAllCameraShakesOfClass(TSubclassOf<UCameraShakeBase> ShakeClass, bool bImmediately = true);
virtual void RemoveAllCameraShakesFromSource(const UCameraShakeSourceComponent* SourceComponent, bool bImmediately = true);
virtual void RemoveAllCameraShakesOfClassFromSource(TSubclassOf<UCameraShakeBase> ShakeClass, const UCameraShakeSourceComponent* SourceComponent, bool bImmediately = true);
virtual void RemoveAllCameraShakes(bool bImmediately = true);
```

`FAddCameraShakeParams` carries `Scale`, `PlaySpace`, `UserPlaySpaceRot`, `SourceComponent`, `Initializer` (`FOnInitializeCameraShake`) and `DurationOverride` (`TOptional<float>`). Supplying an `Initializer` opts the shake out of pooling.

### UCameraShakeSourceComponent

`Attenuation` (`ECameraShakeAttenuation`), `InnerAttenuationRadius`, `OuterAttenuationRadius`, `CameraShake`, `bAutoStart`. Methods: `void Start()`, `void StartCameraShake(TSubclassOf<UCameraShakeBase> InCameraShake, float Scale=1.f, ECameraShakePlaySpace PlaySpace = ECameraShakePlaySpace::CameraLocal, FRotator UserPlaySpaceRot = FRotator::ZeroRotator)`, `void StopAllCameraShakesOfType(TSubclassOf<UCameraShakeBase> InCameraShake, bool bImmediately = true)`, `void StopAllCameraShakes(bool bImmediately = true)`, `float GetAttenuationFactor(const FVector& Location) const`. `ACameraShakeSourceActor` is the placeable actor form.

## Camera animation sequences

`UEngineCamerasSubsystem` (a `UWorldSubsystem` in the `EngineCameras` plugin) plays `UCameraAnimationSequence` assets on a player:

```cpp
static UEngineCamerasSubsystem* GetEngineCamerasSubsystem(const UWorld* InWorld);
FCameraAnimationHandle PlayCameraAnimation(APlayerController* PlayerController, UCameraAnimationSequence* Sequence, FCameraAnimationParams Params);
bool IsCameraAnimationActive(APlayerController* PlayerController, const FCameraAnimationHandle& Handle) const;
void StopCameraAnimation(APlayerController* PlayerController, const FCameraAnimationHandle& Handle, bool bImmediate = false);
void StopAllCameraAnimationsOf(APlayerController* PlayerController, UCameraAnimationSequence* Sequence, bool bImmediate = false);
void StopAllCameraAnimations(APlayerController* PlayerController, bool bImmediate = false);
```

`UCameraAnimationCameraModifier` is the modifier that applies them.

## USpringArmComponent

`GameFramework/SpringArmComponent.h`, base `USceneComponent`.

| Property | Effect |
|---|---|
| `TargetArmLength` | boom length before collision |
| `SocketOffset` | offset in the boom's rotated space — moves with the camera |
| `TargetOffset` | offset in world space — stays vertical |
| `bDoCollisionTest` | sweep from the pivot to the camera |
| `ProbeChannel` (`TEnumAsByte<ECollisionChannel>`) | channel for that sweep, normally `ECC_Camera` |
| `ProbeSize` | sphere radius of the sweep |
| `bUsePawnControlRotation` | drive rotation from `APawn::GetViewRotation()` |
| `bInheritPitch` / `bInheritYaw` / `bInheritRoll` | which parent rotation axes pass through |
| `bEnableCameraLag` / `CameraLagSpeed` / `CameraLagMaxDistance` | positional smoothing |
| `bEnableCameraRotationLag` / `CameraRotationLagSpeed` | rotational smoothing |
| `bUseCameraLagSubstepping` / `CameraLagMaxTimeStep` | frame-rate-independent damping |
| `bDrawDebugLagMarkers` | draws the lag target |

Accessors and virtuals: `FRotator GetTargetRotation() const`, `bool IsCollisionFixApplied() const`, `static const FName SocketName`, `virtual FRotator GetDesiredRotation() const`, `virtual void UpdateDesiredArmLocation(bool bDoTrace, bool bDoLocationLag, bool bDoRotationLag, float DeltaTime)`, `virtual FVector BlendLocations(const FVector& DesiredArmLocation, const FVector& TraceHitLocation, bool bHitSomething, float DeltaTime)`.

## Actor and pawn hooks

```cpp
// AActor
virtual void CalcCamera(float DeltaTime, struct FMinimalViewInfo& OutResult);
virtual bool HasActiveCameraComponent(bool bForceFindCamera = false) const;
virtual bool HasActivePawnControlCameraComponent() const;
virtual void GetActorEyesViewPoint(FVector& OutLocation, FRotator& OutRotation) const;
uint8 bFindCameraComponentWhenViewTarget:1;

// APawn
virtual FRotator GetViewRotation() const;
virtual FVector GetPawnViewLocation() const;
virtual void RecalculateBaseEyeHeight();
float BaseEyeHeight;

// APlayerController
virtual void GetPlayerViewPoint(FVector& out_Location, FRotator& out_Rotation) const override;
virtual void CalcCamera(float DeltaTime, struct FMinimalViewInfo& OutResult) override;
virtual void SpawnPlayerCameraManager();
virtual void AutoManageActiveCameraTarget(AActor* SuggestedTarget);
```

`APawn` does not declare `CalcCamera` — the inherited `AActor` version runs, which is why `bFindCameraComponentWhenViewTarget` (default true on every actor, `Actor.cpp:320`) usually routes to the camera component instead. A subclass override of `CalcCamera` always runs for a non-`ACameraActor` view target; only calling `Super` brings the component lookup back.

## Cinematic cameras

`UCineCameraComponent` (module `"CinematicCamera"`, base `UCameraComponent`, hides `SetFieldOfView`/`SetAspectRatio`):

| Member | Type |
|---|---|
| `Filmback` | `FCameraFilmbackSettings` (setter `SetFilmback`) |
| `LensSettings` | `FCameraLensSettings` (setter `SetLensSettings`) |
| `FocusSettings` | `FCameraFocusSettings` (setter `SetFocusSettings`) |
| `CropSettings` | `FPlateCropSettings` (setter `SetCropSettings`) |
| `CurrentFocalLength` | `float` (setter `SetCurrentFocalLength`) |
| `CurrentAperture` | `float` (setter `SetCurrentAperture`) |
| `CurrentFocusDistance` | `float`, read-only |
| `CurrentHorizontalFOV` | `float`, read-only |
| `CustomNearClippingPlane` | `float` with `bOverride_CustomNearClippingPlane` |

Helpers: `float GetHorizontalFieldOfView() const`, `float GetVerticalFieldOfView() const`, `FString GetFilmbackPresetName() const`, `void SetFilmbackPresetByName(const FString& InPresetName)`, `FString GetLensPresetName() const`, `void SetLensPresetByName(const FString& InPresetName)`.

`ACineCameraActor` adds `FCameraLookatTrackingSettings LookatTrackingSettings`, `FVector GetLookatLocation() const`, `bool ShouldTickForTracking() const` and overrides `NotifyCameraCut()`.

Rigs:

| Actor | Animatable members |
|---|---|
| `ACameraRig_Rail` | `CurrentPositionOnRail` (0–1), `bLockOrientationToRail`; spline component via `GetRailSplineComponent` |
| `ACameraRig_Crane` | `CranePitch`, `CraneYaw`, `CraneArmLength`, `bLockMountPitch`, `bLockMountYaw` |

Both override `GetDefaultAttachComponent()` so attaching a camera actor lands on the mount.
