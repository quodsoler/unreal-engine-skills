# Post-Process Settings Reference

`FPostProcessSettings` fields as declared in `Engine/Source/Runtime/Engine/Classes/Engine/Scene.h`. Every value field has a paired `bOverride_<Field>` bit; the value is ignored unless that bit is `true`.

Carriers of the struct: `APostProcessVolume::Settings`, `UPostProcessComponent::Settings`, `UCameraComponent::PostProcessSettings`, `USceneCaptureComponent2D::PostProcessSettings`.

---

## Usage Pattern

```cpp
FPostProcessSettings& S = Volume->Settings;

S.bOverride_BloomIntensity = true;
S.BloomIntensity = 1.5f;

// Without the override bit this assignment does nothing:
S.BloomIntensity = 1.5f;
```

---

## Bloom

| Field | Type | Meaning |
|-------|------|---------|
| `BloomMethod` | `TEnumAsByte<EBloomMethod>` | `BM_SOG` (sum of Gaussians) or `BM_FFT` (convolution) |
| `BloomIntensity` | float | Overall bloom brightness multiplier; 0 disables |
| `BloomThreshold` | float | Luminance threshold; -1 lets every pixel contribute |
| `BloomSizeScale` | float | Scales all Gaussian bloom kernel sizes |
| `BloomDirtMask` | `TObjectPtr<UTexture>` | Lens dirt texture applied over bloom |
| `BloomDirtMaskIntensity` | float | Strength of the dirt mask |

Raising `BloomThreshold` toward 1.0 keeps bloom on genuinely bright pixels and is closer to physical behaviour than the default catch-all.

---

## Exposure (Eye Adaptation)

| Field | Type | Meaning |
|-------|------|---------|
| `AutoExposureMethod` | `TEnumAsByte<EAutoExposureMethod>` | `AEM_Histogram`, `AEM_Basic`, `AEM_Manual` |
| `AutoExposureMinBrightness` | float | Lower clamp on the adapted luminance |
| `AutoExposureMaxBrightness` | float | Upper clamp on the adapted luminance |
| `AutoExposureBias` | float | Exposure compensation in EV; +1 doubles perceived brightness |
| `AutoExposureSpeedUp` | float | Adaptation rate when the scene gets brighter |
| `AutoExposureSpeedDown` | float | Adaptation rate when the scene gets darker |
| `AutoExposureLowPercent` | float | Histogram low percentile used for metering |
| `AutoExposureHighPercent` | float | Histogram high percentile used for metering |

Locking exposure:

```cpp
S.bOverride_AutoExposureMethod = true;
S.AutoExposureMethod = AEM_Manual;
S.bOverride_AutoExposureBias = true;
S.AutoExposureBias = 0.0f;
```

Setting `AutoExposureMinBrightness` equal to `AutoExposureMaxBrightness` also pins adaptation while keeping the histogram path.

---

## Depth of Field

| Field | Type | Meaning |
|-------|------|---------|
| `DepthOfFieldFstop` | float | Aperture in f-stops; lower means shallower focus |
| `DepthOfFieldMinFstop` | float | Minimum aperture clamp |
| `DepthOfFieldBladeCount` | int32 | Aperture blade count, shapes the bokeh |
| `DepthOfFieldFocalDistance` | float | Distance from camera to the focal plane, in cm |
| `DepthOfFieldSensorWidth` | float | Sensor width in mm |
| `DepthOfFieldDepthBlurRadius` | float | Depth blur radius at 1 m |
| `DepthOfFieldDepthBlurAmount` | float | Multiplier for the depth blur radius |
| `DepthOfFieldNearTransitionRegion` | float | Blur transition distance in front of the focal plane, cm |
| `DepthOfFieldFarTransitionRegion` | float | Blur transition distance behind the focal plane, cm |

`DepthOfFieldFstop = 1.4` with `DepthOfFieldFocalDistance = 200` gives a portrait-style shallow focus; `Fstop = 22` is near-infinite focus.

---

## Color Grading

All grading fields are `FVector4` — RGB plus a master component in W.

| Field | Scope |
|-------|-------|
| `ColorSaturation`, `ColorContrast`, `ColorGamma`, `ColorGain`, `ColorOffset` | Whole image |
| `ColorSaturationShadows`, `ColorContrastShadows`, `ColorGammaShadows`, `ColorGainShadows`, `ColorOffsetShadows` | Shadow range |
| `ColorSaturationMidtones`, `ColorContrastMidtones`, `ColorGammaMidtones`, `ColorGainMidtones`, `ColorOffsetMidtones` | Midtone range |
| `ColorSaturationHighlights`, `ColorContrastHighlights`, `ColorGammaHighlights`, `ColorGainHighlights`, `ColorOffsetHighlights` | Highlight range |

```cpp
// Desaturated shadows, warm highlights
S.bOverride_ColorSaturationShadows = true;
S.ColorSaturationShadows = FVector4(0.5f, 0.5f, 0.5f, 1.0f);
S.bOverride_ColorGainHighlights = true;
S.ColorGainHighlights = FVector4(1.05f, 0.98f, 0.9f, 1.0f);
```

`ColorSaturation = FVector4(0, 0, 0, 1)` is full greyscale.

Filmic tonemapper controls live alongside them: `FilmSlope`, `FilmToe`, `FilmShoulder`, `FilmBlackClip`, `FilmWhiteClip`.

---

## Vignette and Film Grain

| Field | Type | Meaning |
|-------|------|---------|
| `VignetteIntensity` | float | 0 is off, 1 is heavy edge darkening |
| `FilmGrainIntensity` | float | Overall grain strength |
| `FilmGrainIntensityShadows` | float | Grain weighting in shadows |
| `FilmGrainIntensityMidtones` | float | Grain weighting in midtones |
| `FilmGrainIntensityHighlights` | float | Grain weighting in highlights |
| `FilmGrainShadowsMax` | float | Luminance below which the shadow weighting applies |
| `FilmGrainHighlightsMin` | float | Luminance above which the highlight weighting applies |

---

## Ambient Occlusion

| Field | Type | Meaning |
|-------|------|---------|
| `AmbientOcclusionIntensity` | float | 0 is off, 1 is full occlusion |
| `AmbientOcclusionRadius` | float | Sample radius |
| `AmbientOcclusionRadiusInWS` | bitfield | When set, `AmbientOcclusionRadius` is world space rather than view space |
| `AmbientOcclusionBias` | float | Bias that suppresses self-occlusion |
| `AmbientOcclusionPower` | float | Exponent applied to the AO term |
| `AmbientOcclusionQuality` | float | Sample count knob |

---

## Global Illumination and Reflections

| Field | Type | Meaning |
|-------|------|---------|
| `DynamicGlobalIlluminationMethod` | `TEnumAsByte<EDynamicGlobalIlluminationMethod::Type>` | `None`, `Lumen`, `ScreenSpace` (deprecated, `Engine/EngineTypes.h:463`), `Plugin` |
| `ReflectionMethod` | `TEnumAsByte<EReflectionMethod::Type>` | `None`, `Lumen`, `ScreenSpace` |
| `LumenSceneDetail` | float | Surface cache detail multiplier |
| `LumenSceneLightingQuality` | float | Lumen scene lighting quality |
| `LumenSceneLightingUpdateSpeed` | float | How fast scene lighting changes propagate |
| `LumenFinalGatherQuality` | float | Final gather quality for diffuse GI |
| `LumenFinalGatherLightingUpdateSpeed` | float | Final gather update rate |
| `LumenMaxTraceDistance` | float | Maximum trace distance in cm |
| `LumenReflectionQuality` | float | Reflection trace quality |
| `LumenRayLightingMode` | `ELumenRayLightingModeOverride` | `Default`, `SurfaceCache`, `HitLightingForReflections`, `HitLighting` |

`EDynamicGlobalIlluminationMethod::Type` and `EReflectionMethod::Type` are declared in `Engine/EngineTypes.h`; `ELumenRayLightingModeOverride` in `Engine/Scene.h`.

Raising `LumenSceneDetail` helps interiors where surface-cache misses produce splotchy GI. `HitLightingForReflections` trades GPU time for accurate mirror-like reflections. Reducing `LumenMaxTraceDistance` is a cheap win in enclosed levels.

---

## MegaLights

| Field | Type | Meaning |
|-------|------|---------|
| `bMegaLights` | bitfield | Enables MegaLights for views affected by this volume |

```cpp
S.bOverride_bMegaLights = true;
S.bMegaLights = true;
```

The project default is `URendererSettings::bEnableMegaLights` (`r.MegaLights.EnableForProject`). MegaLights needs hardware ray tracing or, as fallback, software (Lumen) tracing data (`MegaLights::HasRequiredTracingData`, `Renderer/Private/MegaLights/MegaLights.cpp:541`), replaces the other direct-lighting and shadowing paths for the lights it handles, and skips directional lights unless `r.MegaLights.DirectionalLights=1` (default 0, `MegaLights.cpp:239`). Runtime CVars include `r.MegaLights.Allowed`, `r.MegaLights.HardwareRayTracing` and `r.MegaLights.Denoiser`.

---

## Screen Space Reflections

Used when `ReflectionMethod` is `ScreenSpace`.

| Field | Type | Meaning |
|-------|------|---------|
| `ScreenSpaceReflectionIntensity` | float | SSR strength |
| `ScreenSpaceReflectionQuality` | float | SSR sample count |
| `ScreenSpaceReflectionMaxRoughness` | float | Roughness cutoff above which SSR stops contributing |

---

## Motion Blur

| Field | Type | Meaning |
|-------|------|---------|
| `MotionBlurAmount` | float | Blur strength; 0 disables |
| `MotionBlurMax` | float | Maximum blur length as a percentage of screen width |
| `MotionBlurTargetFPS` | int32 | Frame rate the blur length is normalised to |
| `MotionBlurPerObjectSize` | float | Screen-size threshold for per-object blur |

---

## Lens Artefacts

| Field | Type | Meaning |
|-------|------|---------|
| `SceneFringeIntensity` | float | Chromatic aberration strength |
| `ChromaticAberrationStartOffset` | float | Radial start of the aberration, 0 at centre |
| `LensFlareIntensity` | float | Flare strength around bright sources |
| `LensFlareBokehSize` | float | Bokeh flare size |
| `LensFlareThreshold` | float | Luminance threshold for flares |

---

## Blendables (Post-Process Materials)

`FPostProcessSettings` holds a `FWeightedBlendables WeightedBlendables` array. Do not edit it directly; use:

```cpp
void AddBlendable(TScriptInterface<IBlendableInterface> InBlendableObject, float InWeight);
void RemoveBlendable(TScriptInterface<IBlendableInterface> InBlendableObject);
```

`APostProcessVolume`, `UPostProcessComponent` and `UCameraComponent` each forward to `AddBlendable` through `AddOrUpdateBlendable(TScriptInterface<IBlendableInterface> InBlendableObject, float InWeight = 1.0f)`.

```cpp
// A material or a MID both satisfy IBlendableInterface.
Volume->AddOrUpdateBlendable(OutlineMaterial, 1.0f);

UMaterialInstanceDynamic* OutlineMID = UMaterialInstanceDynamic::Create(OutlineMaterial, this);
OutlineMID->SetScalarParameterValue(TEXT("Thickness"), 2.f);
Volume->AddOrUpdateBlendable(OutlineMID, 1.0f);
```

The material's `MaterialDomain` must be `MD_PostProcess` (`MaterialDomain.h`). Its `BlendableLocation` field selects where the pass runs; `EBlendableLocation` (`Engine/BlendableInterface.h`):

| Value | Runs |
|-------|------|
| `BL_SceneColorBeforeDOF` | Between translucency distortion and depth of field |
| `BL_SceneColorAfterDOF` | Between DOF and after-DOF translucency |
| `BL_TranslucencyAfterDOF` | On the after-DOF translucency pass |
| `BL_SSRInput` | Supplies the screen-space reflection input |
| `BL_SceneColorBeforeBloom` | After translucency, before bloom |
| `BL_ReplacingTonemapper` | Replaces the tonemapper entirely |
| `BL_SceneColorAfterTonemapping` | After tonemapping, in display space |

Ordering within one location is `UMaterial::BlendablePriority`. `UMaterial::bIsBlendable` controls whether several weighted instances of the material at one location are merged into one pass with interpolated parameters (`false` gives each its own pass; a material with a `UserSceneTexture` output is never merged, `Material.cpp:2097`), and `UMaterial::UserSceneTexture` names a user scene texture the pass writes.

---

## Volume Priority and Blending

| Field | Effect |
|-------|--------|
| `Priority` | Higher wins where volumes overlap; order is undefined on ties |
| `BlendWeight` | 0 contributes nothing, 1 contributes fully |
| `BlendRadius` | World-space distance outside the volume over which the contribution ramps in |
| `bUnbound` | Applies everywhere, ignoring the volume bounds |
| `bEnabled` | Turns the volume off without changing its settings |

```cpp
BackgroundVolume->Priority = 1.f;
DangerVolume->Priority = 10.f;   // wins wherever both apply
```

Only properties whose `bOverride_*` bit is set participate; everything else falls through to the next volume down.

---

## Presets

### Dark and oppressive

```cpp
FPostProcessSettings& S = Volume->Settings;

S.bOverride_BloomIntensity = true;            S.BloomIntensity = 0.2f;
S.bOverride_VignetteIntensity = true;         S.VignetteIntensity = 1.0f;
S.bOverride_ColorSaturation = true;           S.ColorSaturation = FVector4(0.6f, 0.6f, 0.6f, 1.0f);
S.bOverride_AutoExposureMinBrightness = true; S.AutoExposureMinBrightness = 0.01f;
S.bOverride_AutoExposureMaxBrightness = true; S.AutoExposureMaxBrightness = 1.0f;
S.bOverride_FilmGrainIntensity = true;        S.FilmGrainIntensity = 0.8f;
```

### High-contrast cinematic

```cpp
FPostProcessSettings& S = Volume->Settings;

S.bOverride_BloomIntensity = true;            S.BloomIntensity = 0.5f;
S.bOverride_ColorContrast = true;             S.ColorContrast = FVector4(1.3f, 1.3f, 1.3f, 1.0f);
S.bOverride_ColorSaturation = true;           S.ColorSaturation = FVector4(1.2f, 1.2f, 1.2f, 1.0f);
S.bOverride_DepthOfFieldFstop = true;         S.DepthOfFieldFstop = 2.0f;
S.bOverride_DepthOfFieldFocalDistance = true; S.DepthOfFieldFocalDistance = 500.0f;
S.bOverride_MotionBlurAmount = true;          S.MotionBlurAmount = 0.4f;
```

### Tech overlay

```cpp
FPostProcessSettings& S = Volume->Settings;

S.bOverride_BloomIntensity = true;             S.BloomIntensity = 2.0f;
S.bOverride_SceneFringeIntensity = true;       S.SceneFringeIntensity = 2.0f;
S.bOverride_ColorSaturationHighlights = true;  S.ColorSaturationHighlights = FVector4(0.7f, 1.2f, 1.4f, 1.0f);
S.bOverride_VignetteIntensity = true;          S.VignetteIntensity = 0.6f;
```

### Suppress a volume

```cpp
Volume->BlendWeight = 0.0f;   // keeps the settings, contributes nothing
Volume->bEnabled = false;     // removes it from the blend entirely
```
