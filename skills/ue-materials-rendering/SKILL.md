---
name: ue-materials-rendering
description: "Use when writing C++ for materials or rendering — dynamic material instances, material parameter collections, render targets, scene capture, post process, decals, Nanite, Lumen, MegaLights, Substrate. Also use when the user mentions 'MID', 'CreateDynamicMaterialInstance', 'SetScalarParameterValue', 'UMaterialInstanceDynamic', 'material parameter collection', 'UTextureRenderTarget2D', 'DrawMaterialToRenderTarget', 'scene capture', 'post process volume', 'FPostProcessSettings', 'bOverride_', 'blendable', 'decal', 'custom depth stencil', 'Nanite', 'Lumen' or 'global shader'. For Niagara materials, see ue-niagara-effects; for UMG materials, see ue-ui-umg-slate."
metadata:
  version: "2.0.0"
  engine: "5.8"
---

# UE Materials and Rendering

Target engine: **UE 5.8**. APIs below are verified against the 5.8 headers; older forms are listed under "Deprecated — do not use".

Runtime material and rendering C++: `UMaterialInstanceDynamic`, `UMaterialParameterCollection`, `UTextureRenderTarget2D`, `USceneCaptureComponent2D`, `FPostProcessSettings`, `UDecalComponent`, plus the 5.8 renderer features (Nanite, Lumen, MegaLights, Substrate, Virtual Shadow Maps, TSR). Material headers live in `Runtime/Engine/Public/Materials`; render targets, scene, post-process volumes in `Runtime/Engine/Classes/Engine`; Blueprint libraries in `Runtime/Engine/Classes/Kismet`. Build.cs: `"Engine"` covers everything above; add `"RenderCore"` and `"RHI"` only for `EShaderPlatform`, RDG and global shaders.

## Context

Read `.agents/ue-project-context.md` if it exists (module names, conventions, enabled plugins, renderer settings). Do not stop if it is missing.

Identify the area from the request and the codebase. Ask only when two plausible readings would produce different code.

| Request is about… | Go to |
|---|---|
| Changing material parameters at runtime on a mesh | [Dynamic Material Instances](#dynamic-material-instances) |
| One value driving many materials (weather, time of day) | [Material Parameter Collections](#material-parameter-collections) |
| Minimap, mirror, security camera, runtime texture painting | [Render Targets](#render-targets) / [Scene Capture](#scene-capture) |
| Bloom, exposure, DOF, color grading, Lumen quality knobs | [Post Process](#post-process) |
| Full-screen effect authored as a material | [Post-Process Materials](#post-process-materials) |
| Bullet holes, blood, projected textures | [Decals](#decals) |
| Nanite mesh/material compatibility, WPO, Nanite override material | [Nanite](#nanite) |
| GI, reflections, many shadowed lights | [Lumen and MegaLights](#lumen-and-megalights) |
| Layered/multi-BSDF material system | [Substrate](#substrate) |
| Outlines, X-ray, stencil masks, shadow/AA settings | [Shadows, Anti-Aliasing and Custom Depth](#shadows-anti-aliasing-and-custom-depth) |
| An API wants `EShaderPlatform` but you have a feature level | [Feature Level and Shader Platform](#feature-level-and-shader-platform) |
| Writing an actual HLSL shader pass from C++ | [Global Shaders and Render Graph](#global-shaders-and-render-graph) |

Full signature tables live in [material-parameter-reference.md](references/material-parameter-reference.md); the `FPostProcessSettings` field tables live in [post-process-settings.md](references/post-process-settings.md); global shader and RDG code lives in [global-shaders-rdg.md](references/global-shaders-rdg.md).

## Dynamic Material Instances

A MID is a per-instance parameter override on top of a parent material. Three ways to make one:

| Need | Call |
|---|---|
| Standalone MID not bound to a slot | `UMaterialInstanceDynamic::Create(Parent, Outer, Name)` |
| MID assigned to a component's material slot | `UPrimitiveComponent::CreateDynamicMaterialInstance(ElementIndex, SourceMaterial, OptionalName)` |
| MID from Blueprint-facing library code | `UKismetMaterialLibrary::CreateDynamicMaterialInstance(WorldContextObject, Parent, OptionalName, CreationFlags)` |

Verbatim signatures:

```cpp
// Materials/MaterialInstanceDynamic.h
static UMaterialInstanceDynamic* Create(class UMaterialInterface* ParentMaterial, class UObject* InOuter, FName Name = NAME_None);

// Components/PrimitiveComponent.h
virtual class UMaterialInstanceDynamic* CreateDynamicMaterialInstance(int32 ElementIndex, class UMaterialInterface* SourceMaterial = NULL, FName OptionalName = NAME_None);

// Kismet/KismetMaterialLibrary.h
static class UMaterialInstanceDynamic* CreateDynamicMaterialInstance(UObject* WorldContextObject, class UMaterialInterface* Parent, FName OptionalName = NAME_None, EMIDCreationFlags CreationFlags = EMIDCreationFlags::None);
```

Create once, cache in a `UPROPERTY`, update in `Tick` or on an event:

```cpp
// MyGlowActor.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/Actor.h"
#include "MyGlowActor.generated.h"

class UMaterialInstanceDynamic;
class UStaticMeshComponent;

UCLASS()
class MYGAME_API AMyGlowActor : public AActor
{
    GENERATED_BODY()

public:
    AMyGlowActor();
    virtual void BeginPlay() override;

protected:
    UPROPERTY(VisibleAnywhere, Category = "Rendering")
    TObjectPtr<UStaticMeshComponent> Mesh;

    UPROPERTY(Transient)
    TObjectPtr<UMaterialInstanceDynamic> GlowMID;
};
```

```cpp
// MyGlowActor.cpp
#include "MyGlowActor.h"

#include "Components/StaticMeshComponent.h"
#include "Materials/MaterialInstanceDynamic.h"

AMyGlowActor::AMyGlowActor()
{
    Mesh = CreateDefaultSubobject<UStaticMeshComponent>(TEXT("Mesh"));
    SetRootComponent(Mesh);
}

void AMyGlowActor::BeginPlay()
{
    Super::BeginPlay();

    GlowMID = Mesh->CreateDynamicMaterialInstance(0);
    if (GlowMID)
    {
        GlowMID->SetVectorParameterValue(TEXT("GlowColor"), FLinearColor(1.f, 0.4f, 0.1f, 1.f));
        GlowMID->SetScalarParameterValue(TEXT("GlowStrength"), 1.f);
    }
}
```

When dozens of parameters change per frame, swap the name lookup for the index API — `InitializeScalarParameterAndGetIndex` once, then `SetScalarParameterByIndex` per frame. Signatures and the vector equivalents are in [material-parameter-reference.md](references/material-parameter-reference.md).

Setters (`MaterialInstanceDynamic.h`) — names are case-sensitive `FName`s that must match the material's parameter exactly:

```cpp
void SetScalarParameterValue(FName ParameterName, float Value);
void SetVectorParameterValue(FName ParameterName, FLinearColor Value);
void SetDoubleVectorParameterValue(FName ParameterName, FVector4 Value);
void SetTextureParameterValue(FName ParameterName, class UTexture* Value);
void SetRuntimeVirtualTextureParameterValue(FName ParameterName, class URuntimeVirtualTexture* Value);
void SetSparseVolumeTextureParameterValue(FName ParameterName, class USparseVolumeTexture* Value);
void SetTextureCollectionParameterValue(FName ParameterName, UTextureCollection* Value);
```

All but the double-vector and sparse-volume setters have a `ByInfo` twin — for example `SetScalarParameterValueByInfo(const FMaterialParameterInfo& ParameterInfo, float Value)` — which is how you reach parameters inside a material layer or function.

Reading values back: `K2_GetScalarParameterValue(FName)`, `K2_GetVectorParameterValue(FName)`, `K2_GetTextureParameterValue(FName)` return the value directly. The typed form on `UMaterialInterface` is `bool GetScalarParameterValue(const FHashedMaterialParameterInfo& ParameterInfo, float& OutValue, bool bOveriddenOnly = false) const` — it takes a parameter *info*, not a bare name.

Copying between instances: `CopyParameterOverrides` (overridden parameters only), `CopyInterpParameters` (the source's own scalar, vector, texture and font overrides, no hierarchy walk), `CopyMaterialUniformParameters` (uniforms, skips static parameters), `K2_CopyMaterialInstanceParameters` (all non-static parameters), `CopyScalarAndVectorParameters(const UMaterialInterface&, EShaderPlatform)` and `K2_InterpolateMaterialInstanceParams(SourceA, SourceB, Alpha)`. `ClearParameterValues()` drops all overrides. Full signatures in [material-parameter-reference.md](references/material-parameter-reference.md).

Static parameters (static switch, static bool, static component mask) cannot change on a MID. Author a `UMaterialInstanceConstant` per permutation in the editor and put the MID on top of it.

## Material Parameter Collections

`UMaterialParameterCollection` is an asset of `ScalarParameters` / `VectorParameters` arrays; every material referencing it reads from one uniform buffer per world. Get the per-world instance from `UWorld`:

```cpp
#include "Materials/MaterialParameterCollection.h"
#include "Materials/MaterialParameterCollectionInstance.h"

void AMyWeatherActor::SetRain(float Intensity)
{
    UMaterialParameterCollectionInstance* Inst = GetWorld()->GetParameterCollectionInstance(RainCollection);
    if (Inst)
    {
        Inst->SetScalarParameterValue(TEXT("RainIntensity"), Intensity);
        Inst->SetVectorParameterValue(TEXT("RainTint"), FLinearColor(0.6f, 0.7f, 0.9f, 1.f));
    }
}
```

Both setters return `false` when the name is not in the collection. Getters are `bool GetScalarParameterValue(FName ParameterName, float& OutParameterValue) const` and the vector equivalent. `ForceReturnToDefaultValues()` resets the instance.

The Blueprint-facing library does the world lookup for you (`Kismet/KismetMaterialLibrary.h`):

```cpp
static void SetScalarParameterValue(UObject* WorldContextObject, UMaterialParameterCollection* Collection, FName ParameterName, float ParameterValue);
static void SetVectorParameterValue(UObject* WorldContextObject, UMaterialParameterCollection* Collection, FName ParameterName, const FLinearColor& ParameterValue);
static float GetScalarParameterValue(UObject* WorldContextObject, UMaterialParameterCollection* Collection, FName ParameterName);
static FLinearColor GetVectorParameterValue(UObject* WorldContextObject, UMaterialParameterCollection* Collection, FName ParameterName);
```

Collections hold no texture parameters. Use a collection for one value read by many materials; use a MID for per-actor state.

## Render Targets

`UKismetRenderingLibrary` (`Kismet/KismetRenderingLibrary.h`) owns the runtime render-target workflow:

```cpp
static UTextureRenderTarget2D* CreateRenderTarget2D(UObject* WorldContextObject, int32 Width = 256, int32 Height = 256, ETextureRenderTargetFormat Format = RTF_RGBA16f, FLinearColor ClearColor = FLinearColor::Black, bool bAutoGenerateMipMaps = false, bool bSupportUAVs = false);
static void ClearRenderTarget2D(UObject* WorldContextObject, UTextureRenderTarget2D* TextureRenderTarget, FLinearColor ClearColor = FLinearColor(0, 0, 0, 1));
static void ResizeRenderTarget2D(UTextureRenderTarget2D* TextureRenderTarget, int32 Width = 256, int32 Height = 256);
static void ReleaseRenderTarget2D(UTextureRenderTarget2D* TextureRenderTarget);
static void DrawMaterialToRenderTarget(UObject* WorldContextObject, UTextureRenderTarget2D* TextureRenderTarget, UMaterialInterface* Material);
static void ExportRenderTarget(UObject* WorldContextObject, UTextureRenderTarget2D* TextureRenderTarget, const FString& FilePath, const FString& FileName);
static FColor ReadRenderTargetPixel(UObject* WorldContextObject, UTextureRenderTarget2D* TextureRenderTarget, int32 X, int32 Y);
```

`ETextureRenderTargetFormat` (`Engine/TextureRenderTarget2D.h`): `RTF_R8`, `RTF_RG8`, `RTF_RGBA8`, `RTF_RGBA8_SRGB`, `RTF_R16f`, `RTF_RG16f`, `RTF_RGBA16f`, `RTF_R32f`, `RTF_RG32f`, `RTF_RGBA32f`, `RTF_RGB10A2`. Pick the smallest that holds the data — `RTF_RGBA8` for LDR/UI, `RTF_RGBA16f` for HDR, `RTF_R16f` for a single channel.

Manual creation when you need an exact pixel format:

```cpp
UTextureRenderTarget2D* RT = NewObject<UTextureRenderTarget2D>(this);
RT->InitCustomFormat(512, 512, PF_FloatRGBA, /*bInForceLinearGamma=*/true);
RT->UpdateResourceImmediate(/*bClearRenderTarget=*/true);
```

Canvas drawing batches many draws into one render-target transition:

```cpp
UCanvas* Canvas = nullptr;
FVector2D CanvasSize = FVector2D::ZeroVector;
FDrawToRenderTargetContext Context;

UKismetRenderingLibrary::BeginDrawCanvasToRenderTarget(this, RT, Canvas, CanvasSize, Context);
Canvas->K2_DrawMaterial(IconMaterial, FVector2D::ZeroVector, CanvasSize, FVector2D::ZeroVector);
UKismetRenderingLibrary::EndDrawCanvasToRenderTarget(this, Context);
```

`UCanvasRenderTarget2D` (`Engine/CanvasRenderTarget2D.h`) wraps that loop: create with `CreateCanvasRenderTarget2D(WorldContextObject, CanvasRenderTarget2DClass, Width, Height)`, bind `OnCanvasRenderTargetUpdate`, and call `UpdateResource()` when the contents must be redrawn.

`ReadRenderTargetPixel` and friends stall the GPU. Use them in editor tooling or one-off captures, never per frame; for a streaming readback use `FRHIGPUTextureReadback`.

## Scene Capture

`USceneCaptureComponent2D` renders the world into a `UTextureRenderTarget2D`. Its own fields: `TextureTarget`, `FOVAngle`, `PostProcessSettings`, `PostProcessBlendWeight`, `bMainViewResolution`, `CaptureScene()`, `CaptureSceneDeferred()`. Inherited from `USceneCaptureComponent`: `CaptureSource`, `bCaptureEveryFrame`, `bCaptureOnMovement`, `PrimitiveRenderMode`, `ShowOnlyActors`, `HiddenActors`, `ShowFlags`, `ShowOnlyComponent()`, `ShowOnlyActorComponents()`.

```cpp
#include "Components/SceneCaptureComponent2D.h"

SceneCapture = CreateDefaultSubobject<USceneCaptureComponent2D>(TEXT("SceneCapture"));
SceneCapture->SetupAttachment(RootComponent);
SceneCapture->FOVAngle = 90.f;
SceneCapture->CaptureSource = ESceneCaptureSource::SCS_FinalColorLDR;
SceneCapture->bCaptureEveryFrame = false;   // drive it manually
SceneCapture->bCaptureOnMovement = false;
SceneCapture->PrimitiveRenderMode = ESceneCapturePrimitiveRenderMode::PRM_UseShowOnlyList;
SceneCapture->ShowFlags.SetFog(false);
SceneCapture->ShowFlags.SetMotionBlur(false);
```

`ESceneCaptureSource` (`Engine/EngineTypes.h`): `SCS_SceneColorHDR`, `SCS_SceneColorHDRNoAlpha`, `SCS_FinalColorLDR`, `SCS_SceneColorSceneDepth`, `SCS_SceneDepth`, `SCS_DeviceDepth`, `SCS_Normal`, `SCS_BaseColor`, `SCS_FinalColorHDR`, `SCS_FinalToneCurveHDR`. `SCS_Normal` and `SCS_BaseColor` are deferred-renderer only.

With `bCaptureEveryFrame = false` and `bCaptureOnMovement = false`, call `SceneCapture->CaptureScene()` when the view actually needs refreshing — a scene capture is a full extra scene render.

## Post Process

`FPostProcessSettings` (`Engine/Scene.h`) is carried by `APostProcessVolume::Settings`, `UPostProcessComponent::Settings`, `UCameraComponent::PostProcessSettings` and `USceneCaptureComponent2D::PostProcessSettings`. Every value field has a paired `bOverride_<Field>` bit; the value is ignored unless the override bit is `true`.

```cpp
#include "Engine/PostProcessVolume.h"
#include "Engine/Scene.h"

void AMyMoodDirector::ApplyDangerMood(APostProcessVolume* Volume)
{
    Volume->bEnabled = true;
    Volume->bUnbound = true;
    Volume->Priority = 10.f;
    Volume->BlendWeight = 1.f;

    FPostProcessSettings& S = Volume->Settings;
    S.bOverride_BloomIntensity = true;
    S.BloomIntensity = 0.35f;
    S.bOverride_VignetteIntensity = true;
    S.VignetteIntensity = 0.8f;
    S.bOverride_ColorSaturation = true;
    S.ColorSaturation = FVector4(0.6f, 0.6f, 0.6f, 1.f);
}
```

`APostProcessVolume` and `UPostProcessComponent` both expose `Settings`, `Priority`, `BlendRadius`, `BlendWeight`, `bEnabled`, `bUnbound` and `AddOrUpdateBlendable(TScriptInterface<IBlendableInterface> InBlendableObject, float InWeight = 1.0f)`. Higher `Priority` wins where volumes overlap; `bUnbound` makes a volume apply everywhere.

Per-camera post process: `UCameraComponent::PostProcessSettings` with `PostProcessBlendWeight`, plus `AddOrUpdateBlendable` / `RemoveBlendable`. (`AddExtraPostProcessBlend` only stores settings — only the editor viewport reads them back, `Camera/CameraComponent.h:323`.) From a camera modifier or camera manager, push a transient blend with `APlayerCameraManager::AddCachedPPBlend(FPostProcessSettings& PPSettings, float BlendWeight, EViewTargetBlendOrder BlendOrder = VTBlendOrder_Base)`; read them back with `GetCachedPostProcessBlends`. The cache is cleared every frame at the start of `ApplyCameraModifiers` (`PlayerCameraManager.cpp:289`), so re-add the blend each frame; `ClearCachedPPBlends` and the `PostProcessBlendCache` array are both protected (`Camera/PlayerCameraManager.h:371,391`).

Field-by-field tables (bloom, exposure, DOF, color grading, film grain, AO, Lumen, SSR, motion blur, lens) are in [post-process-settings.md](references/post-process-settings.md).

## Post-Process Materials

A post-process material is a `UMaterial` whose `MaterialDomain` is `MD_PostProcess` (`MaterialDomain.h`). Relevant `UMaterial` fields: `BlendableLocation`, `BlendablePriority`, `bIsBlendable`, `UserSceneTexture`.

`EBlendableLocation` (`Engine/BlendableInterface.h`): `BL_SceneColorBeforeDOF`, `BL_SceneColorAfterDOF`, `BL_TranslucencyAfterDOF`, `BL_SSRInput`, `BL_SceneColorBeforeBloom`, `BL_ReplacingTonemapper`, `BL_SceneColorAfterTonemapping`.

```cpp
// Weight 0 = invisible, 1 = full. Calling it again with the same object updates the weight.
Volume->AddOrUpdateBlendable(OutlineMaterial, 1.0f);

// A MID works as a blendable too, so the effect can be parameterised at runtime.
UMaterialInstanceDynamic* OutlineMID = UMaterialInstanceDynamic::Create(OutlineMaterial, this);
OutlineMID->SetScalarParameterValue(TEXT("Thickness"), 2.f);
Volume->AddOrUpdateBlendable(OutlineMID, 1.0f);
```

`FPostProcessSettings::AddBlendable(TScriptInterface<IBlendableInterface> InBlendableObject, float InWeight)` and `RemoveBlendable` operate on the `WeightedBlendables` array directly if you hold the settings struct rather than the volume.

## Decals

`UDecalComponent` (`Components/DecalComponent.h`) projects a material onto whatever its box overlaps:

```cpp
void SetDecalMaterial(class UMaterialInterface* NewDecalMaterial);
class UMaterialInterface* GetDecalMaterial() const;
virtual class UMaterialInstanceDynamic* CreateDynamicMaterialInstance();
void SetFadeOut(float StartDelay, float Duration, bool DestroyOwnerAfterFade = true);
void SetFadeIn(float StartDelay, float Duration);
void SetFadeScreenSize(float NewFadeScreenSize);
void SetSortOrder(int32 Value);
void SetDecalColor(const FLinearColor& Color);
```

Fields: `DecalSize` (local-space extent, scales the projection box), `SortOrder`, `FadeScreenSize`, `FadeStartDelay`, `FadeDuration`, `DecalColor`.

Spawn at runtime through `UGameplayStatics` (`Kismet/GameplayStatics.h`):

```cpp
UDecalComponent* Decal = UGameplayStatics::SpawnDecalAtLocation(
    this, BulletHoleMaterial, FVector(16.f, 16.f, 16.f), Hit.ImpactPoint, Hit.ImpactNormal.Rotation(), 10.f);

if (Decal)
{
    UMaterialInstanceDynamic* DecalMID = Decal->CreateDynamicMaterialInstance();
    DecalMID->SetScalarParameterValue(TEXT("Wear"), 0.25f);
}
```

`SpawnDecalAttached(DecalMaterial, DecalSize, AttachToComponent, AttachPointName, Location, Rotation, LocationType, LifeSpan)` follows a moving component. `LifeSpan = 0` means the decal persists. To change it later call `UDecalComponent::SetLifeSpan(float)` (`Components/DecalComponent.h:153`, not a `UFUNCTION`) or drive fading with `SetFadeOut`.

DBuffer decals accumulate into a buffer before the base pass, so they correctly affect lightmapped and sky lighting; they need the restart-required project setting `r.DBuffer` (`URendererSettings::bDBuffer`, `Engine/RendererSettings.h:942`). Without it, regular deferred decals still write the GBuffer but do not affect baked/sky lighting correctly.

## Nanite

`UStaticMesh::HasValidNaniteData()` reports whether a mesh has Nanite render data (`IsNaniteEnabled()` is `WITH_EDITORONLY_DATA`, `Engine/StaticMesh.h:1049` — editor builds only). Settings go through the accessors, not the raw field: `GetNaniteSettings()` / `SetNaniteSettings(const FMeshNaniteSettings&)` / `NotifyNaniteSettingsChanged()`.

```cpp
const UStaticMesh* Mesh = MeshComponent->GetStaticMesh();
const bool bIsNanite = Mesh && Mesh->HasValidNaniteData();
```

Material rules worth knowing before writing code:

| Material property | Nanite |
|---|---|
| Opaque, two-sided | Supported |
| Masked | Supported; gated by `r.Nanite.AllowMaskedMaterials` |
| World Position Offset | Supported; per-component `SetEvaluateWorldPositionOffset(bool)` and `SetWorldPositionOffsetDisableDistance(int32)` on `UStaticMeshComponent` |
| Translucent | Not rendered by Nanite; the mesh falls back |
| Material usage flag | `MATUSAGE_Nanite`; skinned Nanite also needs `MATUSAGE_SkeletalMesh` (`NaniteResources.cpp:2648`) |

A material can carry a Nanite-specific replacement via `FMaterialOverrideNanite` (`Materials/MaterialOverrideNanite.h`, `GetOverrideMaterial()` / `SetOverrideMaterial(UMaterialInterface*, bool)`); on a MID the equivalent is `SetNaniteOverride(UMaterialInterface* InMaterial)`.

Nanite assemblies compose one Nanite mesh from parts: `FNaniteAssemblyData`, `FNaniteAssemblyPart`, `FNaniteAssemblyNode`, `FNaniteAssemblyBoneInfluence` (`Engine/NaniteAssemblyData.h`). Skinned Nanite is queried per component with `USkinnedMeshComponent::HasValidNaniteData()` / `GetNaniteResources()`, and disabled per component with `bDisallowNanite`.

## Lumen and MegaLights

Lumen quality is driven from `FPostProcessSettings`: `LumenSceneDetail`, `LumenSceneLightingQuality`, `LumenSceneLightingUpdateSpeed`, `LumenFinalGatherQuality`, `LumenFinalGatherLightingUpdateSpeed`, `LumenMaxTraceDistance`, `LumenReflectionQuality`, `LumenRayLightingMode` — each with its `bOverride_*` bit. `ELumenRayLightingModeOverride` values are `Default`, `SurfaceCache`, `HitLightingForReflections`, `HitLighting`. The renderer path itself is chosen by `DynamicGlobalIlluminationMethod` and `ReflectionMethod` on the same struct.

Scalability rather than per-volume overrides is the cheaper lever: `sg.GlobalIlluminationQuality` selects a `[GlobalIlluminationQuality@N]` block in `BaseScalability.ini`. Level 1 ("Lumen Lite") switches Lumen to the irradiance-volume final gather with `r.Lumen.FinalGatherMethod=0` and a reduced surface cache; levels 2 and up use screen probe gather. From C++, `Scalability::SetQualityLevels(const FQualityLevels& QualityLevels, bool bForce = false)` in `Scalability.h`.

MegaLights renders many shadow-casting lights with stochastic sampling. Project-wide it is `URendererSettings::bEnableMegaLights` (console variable `r.MegaLights.EnableForProject`); per volume it is `FPostProcessSettings::bMegaLights` with `bOverride_bMegaLights`. Runtime CVars include `r.MegaLights.Allowed` and `r.MegaLights.HardwareRayTracing`. It needs ray-tracing data — hardware ray tracing, or software (Lumen) tracing as the fallback (`MegaLights::HasRequiredTracingData`, `Renderer/Private/MegaLights/MegaLights.cpp:541`) — replaces the other direct-lighting and shadowing paths for the lights it handles, and skips directional lights unless `r.MegaLights.DirectionalLights=1` (default 0, `MegaLights.cpp:239`).

```cpp
FPostProcessSettings& S = Volume->Settings;
S.bOverride_bMegaLights = true;
S.bMegaLights = true;
```

## Substrate

Substrate is the layered material system, enabled per project through `URendererSettings::bEnableSubstrate` (`r.Substrate`), with `r.Substrate.ProjectGBufferFormat` and `r.Substrate.ProjectClosuresPerPixel` controlling memory and closure budget. Substrate material nodes live in `Materials/MaterialExpressionSubstrate.h`.

Custom Substrate expressions build a topology tree; the 5.8 virtual to override (editor-only, inside `#if WITH_EDITOR` in `Materials/MaterialExpression.h:481`) is:

```cpp
virtual FSubstrateOperator* SubstrateGenerateMaterialTopologyTree(struct FSubstrateTranslatorDataInterface& SubstrateTranslatorData, class UMaterialExpression* Parent, int32 OutputIndex) override;
```

The static helper is `SubstrateGenerateMaterialTopologyTreeCommon`, also taking `FSubstrateTranslatorDataInterface&` first. The old overloads that took `FMaterialCompiler*` are deprecated.

## Shadows, Anti-Aliasing and Custom Depth

Virtual Shadow Maps are the default shadowing path; `r.Shadow.Virtual.Enable` gates them. Materials with World Position Offset must have WPO evaluation enabled on the component (`SetEvaluateWorldPositionOffset`) for their shadows to match.

Anti-aliasing is `URendererSettings::DefaultFeatureAntiAliasing` (`EAntiAliasingMethod`, `SceneUtils.h`); `AAM_TSR` is Temporal Super-Resolution. The runtime override is `r.AntiAliasingMethod`.

Custom depth/stencil feeds outline, X-ray and highlight post-process materials. Project setting: `URendererSettings::CustomDepthStencil` (`ECustomDepthStencil`), console variable `r.CustomDepth`. Per component:

```cpp
MeshComponent->SetRenderCustomDepth(true);
MeshComponent->SetCustomDepthStencilValue(1);   // 0-255, read as CustomStencil in a post-process material
```

## Feature Level and Shader Platform

Material query APIs take `EShaderPlatform` (`RHIShaderPlatform.h`), not `ERHIFeatureLevel::Type` (`RHIFeatureLevel.h`). Convert with `GetFeatureLevelShaderPlatform` from `RHIGlobals.h`:

```cpp
#include "RHIGlobals.h"

const ERHIFeatureLevel::Type FeatureLevel = GetWorld()->GetFeatureLevel();
const EShaderPlatform ShaderPlatform = GetFeatureLevelShaderPlatform(FeatureLevel);

if (MyMaterial->IsCompilingOrHadCompileError(ShaderPlatform))   // UMaterial* -- not on UMaterialInterface
{
    // hide the object until its shaders are ready
}
```

Usage flags are read through `GetUsageByFlag(EMaterialUsage Usage)` on `UMaterialInterface`, never by touching a `bUsedWith*` field:

```cpp
if (!MyMaterial->GetUsageByFlag(MATUSAGE_Nanite))
{
    // not authorised for Nanite meshes
}
```

`CheckMaterialUsage(EMaterialUsage)` and `CheckMaterialUsage_Concurrent(EMaterialUsage) const` set the flag and trigger a recompile when missing; `SetMaterialUsage(EMaterialUsage Usage)` is the virtual.

## Global Shaders and Render Graph

For a real HLSL pass, declare an `FGlobalShader` subclass with `DECLARE_GLOBAL_SHADER`, `SHADER_USE_PARAMETER_STRUCT` and a `BEGIN_SHADER_PARAMETER_STRUCT` block, bind it to a `.usf` with `IMPLEMENT_GLOBAL_SHADER`, and add passes to an `FRDGBuilder` with `GraphBuilder.AddPass(...)`, then submit with `GraphBuilder.Execute()`. Build.cs needs `"RenderCore"`, `"RHI"` and `"Projects"`.

Two rules cause most failures. Register the shader virtual path with `AddShaderSourceDirectoryMapping` in `StartupModule`. Give that module `"LoadingPhase": "PostConfigInit"`: with the usual `Default` phase, the global shader type registers too late and hits the "Shader type was loaded too late" `checkf` (`Shader.cpp:315`). Full compute-shader declaration, the `AddPass` form and module startup code: [references/global-shaders-rdg.md](references/global-shaders-rdg.md).

## Deprecated — do not use

| Do not emit | Use in 5.8 | Source |
|---|---|---|
| `Material->bUsedWithNanite` (any `bUsedWith*` field) | `Material->GetUsageByFlag(MATUSAGE_Nanite)` | `UE_DEPRECATED(5.8)` in `Materials/Material.h` (27 flags) |
| `UMaterial::bEnableExecWire` | nothing — the experiment was removed | `UE_DEPRECATED(5.8)` in `Materials/Material.h` |
| `SetMaterialUsage(bool& bNeedsRecompile, EMaterialUsage, UMaterialInterface*)` | `SetMaterialUsage(EMaterialUsage Usage)` | `UE_DEPRECATED(5.8)` in `Materials/Material.h` |
| `IsCompilingOrHadCompileError(ERHIFeatureLevel::Type)` | `IsCompilingOrHadCompileError(EShaderPlatform)` | `UE_DEPRECATED(5.7)` in `Materials/Material.h` |
| `GetMaterialResource(ERHIFeatureLevel::Type, ...)` | `GetMaterialResource(EShaderPlatform, ...)` | `UE_DEPRECATED(5.7)` in `Materials/MaterialInterface.h` |
| `GetUsedTexturesAndIndices(..., ERHIFeatureLevel::Type)` | the `EShaderPlatform` overload | `UE_DEPRECATED(5.7)` in `Materials/MaterialInterface.h` |
| `CopyScalarAndVectorParameters(Source, ERHIFeatureLevel::Type)` | `CopyScalarAndVectorParameters(Source, EShaderPlatform)` | `UE_DEPRECATED(5.7)` in `Materials/MaterialInstanceDynamic.h` |
| `GetUsedMaterialPropertyDesc(ERHIFeatureLevel::Type)` | `GetUsedMaterialPropertyDesc(EShaderPlatform)` | `UE_DEPRECATED(5.7)` in `Components/PrimitiveComponent.h` |
| `UMaterialInstance::Resource` (public render proxy) | `GetRenderProxy()` or `GetInstanceRenderProxy()` | `UE_DEPRECATED(5.8)` in `Materials/MaterialInstance.h` |
| `UMaterialExpressionTextureBase::GetSamplerTypeForTexture` | `MaterialExpressionUtils::GetSamplerTypeForTexture` | `UE_DEPRECATED(5.8)` in `Materials/MaterialExpressionTextureBase.h` |
| `UMaterialExpressionTextureBase::VerifySamplerType` | `MaterialExpressionUtils::VerifySamplerType` | `UE_DEPRECATED(5.8)` in `Materials/MaterialExpressionTextureBase.h` |
| `SubstrateGenerateMaterialTopologyTreeCommon(FMaterialCompiler*, ...)` | the `FSubstrateTranslatorDataInterface&` overload | `UE_DEPRECATED(5.8)` in `Materials/MaterialExpressionSubstrate.h` |
| `FTextureSamplingInfo CalculateTexturesSamplingInfo(UTexture*)` | the const `bool` form with an out-parameter | `UE_DEPRECATED(5.8)` in `Materials/MaterialInterface.h` |
| `IsUsingNewTranslatorPrototype()` | `IsUsingNewHLSLGenerator()` | `UE_DEPRECATED(5.8)` in `Materials/MaterialInterface.h` |
| `CreateMIDForElement` / `CreateMIDForElementFromMaterial` | `CreateDynamicMaterialInstance(ElementIndex, SourceMaterial, OptionalName)` | `DeprecatedFunction` in `Components/PrimitiveComponent.h` |
| `StaticMesh->NaniteSettings` (direct field access) | `GetNaniteSettings()` / `SetNaniteSettings()` | `UE_DEPRECATED(5.7)` in `Engine/StaticMesh.h` |
| `BL_BeforeTranslucency`, `BL_BeforeTonemapping`, `BL_AfterTonemapping` | `BL_SceneColorBeforeDOF`, `BL_SceneColorAfterDOF`, `BL_SceneColorAfterTonemapping` | `UE_DEPRECATED(5.4)` in `Engine/BlendableInterface.h` |

## Common Mistakes

**Creating a MID every frame:** each `CreateDynamicMaterialInstance` allocates a new instance and render proxy. Create once in `BeginPlay`, store it, and only call the setters afterwards.

**Holding a MID in a raw pointer:** a MID is a `UObject` and is collected as soon as nothing references it. Store it in `UPROPERTY() TObjectPtr<UMaterialInstanceDynamic>`, never a bare `UMaterialInstanceDynamic*` member or a static.

**Setting a post-process value without its override bit:** `S.BloomIntensity = 2.f;` alone is a silent no-op. Every assignment needs `S.bOverride_BloomIntensity = true;` next to it.

**Passing a feature level where a shader platform is expected:** the `ERHIFeatureLevel::Type` overloads of `GetMaterialResource`, `IsCompilingOrHadCompileError`, `CopyScalarAndVectorParameters` and `GetUsedMaterialPropertyDesc` are deprecated. Convert once with `GetFeatureLevelShaderPlatform(GetWorld()->GetFeatureLevel())`.

**Reading a `bUsedWith*` field:** those members are deprecated. `GetUsageByFlag(EMaterialUsage)` is the only supported read, `SetMaterialUsage(EMaterialUsage)` the only supported write.

**Expecting double precision from `SetVectorParameterValue`:** the `FVector`/`FVector4` overloads are inline conveniences that convert to `FLinearColor` (float) (`Materials/MaterialInstanceDynamic.h:112-113`). Use `SetDoubleVectorParameterValue(FName, FVector4)` when the material reads a double-precision vector.

**Replicating a MID:** MIDs are client-local objects. Replicate the scalar/vector values and rebuild the MID's state in an `OnRep` on each client.

**Expecting a static switch to change at runtime:** static parameters bake into the shader permutation. Make a `UMaterialInstanceConstant` per permutation and parent the MID to the right one.

**Calling `ReadRenderTargetPixel` in `Tick`:** it flushes rendering commands and stalls the pipeline. Keep readbacks to editor tools, or use `FRHIGPUTextureReadback`.

**Leaving `bCaptureEveryFrame` on:** a `USceneCaptureComponent2D` that captures every frame renders the scene a second time. Turn it off and call `CaptureScene()` when the target actually changes.

## Related Skills

- `ue-niagara-effects` — Niagara systems, emitter parameters and the material usage flags particles require
- `ue-procedural-generation` — PCG and runtime mesh generation that assigns these materials
- `ue-actor-component-architecture` — component setup, attachment and lifetime for the components used here
- `ue-ui-umg-slate` — `MD_UI` materials, widget brushes and rendering UMG into a render target
- `ue-sequencer-cinematics` — animating material parameters and post-process settings from sequences
- `ue-testing-debugging` — profiling, `stat` commands, Insights and GPU capture workflows
- `ue-data-assets-tables` — data assets and tables that drive material and rendering configuration
- `ue-cpp-foundations` — `UObject` lifetime, `UPROPERTY`, `TObjectPtr` and garbage collection
