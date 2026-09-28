# Material Parameter Reference

Verbatim C++ signatures for the material parameter APIs, plus naming, performance and debugging notes.

---

## UMaterialInstanceDynamic

`Engine/Source/Runtime/Engine/Public/Materials/MaterialInstanceDynamic.h`

### Creation

```cpp
static UMaterialInstanceDynamic* Create(class UMaterialInterface* ParentMaterial, class UObject* InOuter, FName Name = NAME_None);
```

### Setters by name

```cpp
void SetScalarParameterValue(FName ParameterName, float Value);
void SetVectorParameterValue(FName ParameterName, FLinearColor Value);
void SetDoubleVectorParameterValue(FName ParameterName, FVector4 Value);
void SetTextureParameterValue(FName ParameterName, class UTexture* Value);
void SetRuntimeVirtualTextureParameterValue(FName ParameterName, class URuntimeVirtualTexture* Value);
void SetSparseVolumeTextureParameterValue(FName ParameterName, class USparseVolumeTexture* Value);
void SetTextureCollectionParameterValue(FName ParameterName, UTextureCollection* Value);
void SetFontParameterValue(const FMaterialParameterInfo& ParameterInfo, class UFont* FontValue, int32 FontPage);
void ClearParameterValues();
```

### Setters by parameter info

Every setter above except `SetDoubleVectorParameterValue` and `SetSparseVolumeTextureParameterValue` has a `...ByInfo` twin taking `const FMaterialParameterInfo&` in place of the `FName`, which is how you reach parameters inside a material layer or blend:

```cpp
void SetScalarParameterValueByInfo(const FMaterialParameterInfo& ParameterInfo, float Value);
void SetVectorParameterValueByInfo(const FMaterialParameterInfo& ParameterInfo, FLinearColor Value);
void SetTextureParameterValueByInfo(const FMaterialParameterInfo& ParameterInfo, class UTexture* Value);
void SetRuntimeVirtualTextureParameterValueByInfo(const FMaterialParameterInfo& ParameterInfo, class URuntimeVirtualTexture* Value);
void SetTextureCollectionParameterValueByInfo(const FMaterialParameterInfo& ParameterInfo, UTextureCollection* Value);
```

`FMaterialParameterInfo` (`Materials/MaterialParameters.h`) carries `FName Name`, `TEnumAsByte<EMaterialParameterAssociation> Association` and `int32 Index`:

```cpp
FMaterialParameterInfo(FName InName = FName(), EMaterialParameterAssociation InAssociation = EMaterialParameterAssociation::GlobalParameter, int32 InIndex = INDEX_NONE);
```

`EMaterialParameterAssociation` is `LayerParameter`, `BlendParameter` or `GlobalParameter`.

### Index-based fast path

```cpp
bool InitializeScalarParameterAndGetIndex(const FName& ParameterName, float Value, int32& OutParameterIndex);
bool SetScalarParameterByIndex(int32 ParameterIndex, float Value);
bool InitializeVectorParameterAndGetIndex(const FName& ParameterName, const FLinearColor& Value, int32& OutParameterIndex);
bool SetVectorParameterByIndex(int32 ParameterIndex, const FLinearColor& Value);
```

```cpp
// Once, during initialization
int32 AlphaIdx = INDEX_NONE;
int32 ColorIdx = INDEX_NONE;
MyMID->InitializeScalarParameterAndGetIndex(TEXT("Alpha"), 1.0f, AlphaIdx);
MyMID->InitializeVectorParameterAndGetIndex(TEXT("TintColor"), FLinearColor::White, ColorIdx);

// Per frame, no FName hash
MyMID->SetScalarParameterByIndex(AlphaIdx, NewAlpha);
MyMID->SetVectorParameterByIndex(ColorIdx, NewColor);
```

Indices belong to one MID and one parent material. Do not share them between instances, and re-initialize if the parent changes.

### Getters

```cpp
float K2_GetScalarParameterValue(FName ParameterName);
float K2_GetScalarParameterValueByInfo(const FMaterialParameterInfo& ParameterInfo);
FLinearColor K2_GetVectorParameterValue(FName ParameterName);
FLinearColor K2_GetVectorParameterValueByInfo(const FMaterialParameterInfo& ParameterInfo);
class UTexture* K2_GetTextureParameterValue(FName ParameterName);
UTextureCollection* K2_GetTextureCollectionParameterValue(FName ParameterName);
```

The typed forms on `UMaterialInterface` return success and take an info struct:

```cpp
bool GetScalarParameterValue(const FHashedMaterialParameterInfo& ParameterInfo, float& OutValue, bool bOveriddenOnly = false) const;
bool GetVectorParameterValue(const FHashedMaterialParameterInfo& ParameterInfo, FLinearColor& OutValue, bool bOveriddenOnly = false) const;
```

### Copying and interpolating

```cpp
void CopyParameterOverrides(UMaterialInstance* MaterialInstance);
void CopyInterpParameters(UMaterialInstance* Source);
void CopyMaterialUniformParameters(UMaterialInterface* Source);
void K2_CopyMaterialInstanceParameters(UMaterialInterface* Source, bool bQuickParametersOnly = false);
void CopyScalarAndVectorParameters(const UMaterialInterface& SourceMaterialToCopyFrom, EShaderPlatform ShaderPlatform);
void K2_InterpolateMaterialInstanceParams(UMaterialInstance* SourceA, UMaterialInstance* SourceB, float Alpha);
void SetNaniteOverride(UMaterialInterface* InMaterial);
```

| Call | Copies | Cost |
|---|---|---|
| `CopyParameterOverrides` | only parameters explicitly overridden on the source | low |
| `CopyInterpParameters` | the source instance's own scalar, vector, double-vector, texture and font overrides, no hierarchy walk (`MaterialInstanceDynamic.cpp:505`) | low |
| `CopyMaterialUniformParameters` | uniform parameters, skips static parameters | low |
| `CopyScalarAndVectorParameters` | scalar/vector for one `EShaderPlatform` | low |
| `K2_CopyMaterialInstanceParameters` | all non-static parameters, walking the hierarchy (`bQuickParametersOnly` = `CopyMaterialUniformParameters`); static parameters are skipped (`MaterialInstance.cpp:2058`) | high |

`CopyScalarAndVectorParameters` takes an `EShaderPlatform`; derive it with `GetFeatureLevelShaderPlatform(GetWorld()->GetFeatureLevel())` from `RHIGlobals.h`.

---

## UMaterialParameterCollectionInstance

`Materials/MaterialParameterCollectionInstance.h`

```cpp
bool SetScalarParameterValue(FName ParameterName, float ParameterValue);
bool SetVectorParameterValue(FName ParameterName, const FLinearColor& ParameterValue);
bool SetVectorParameterValue(FName ParameterName, const FVector& ParameterValue);
bool SetVectorParameterValue(FName ParameterName, const FVector4& ParameterValue);
bool GetScalarParameterValue(FName ParameterName, float& OutParameterValue) const;
bool GetVectorParameterValue(FName ParameterName, FLinearColor& OutParameterValue) const;
void ForceReturnToDefaultValues();
```

Obtain the per-world instance:

```cpp
UMaterialParameterCollectionInstance* Inst = GetWorld()->GetParameterCollectionInstance(CollectionAsset);
```

`UMaterialParameterCollection` (`Materials/MaterialParameterCollection.h`) holds `TArray<FCollectionScalarParameter> ScalarParameters` and `TArray<FCollectionVectorParameter> VectorParameters`, plus queries `GetScalarParameterNames()`, `GetVectorParameterNames()`, `GetScalarParameterDefaultValue(FName, bool& bParameterFound)`, `GetVectorParameterDefaultValue(FName, bool& bParameterFound)`, `GetScalarParameterByName(FName)` and `GetVectorParameterByName(FName)`. There are no texture parameters in a collection.

The Blueprint-facing wrapper (`Kismet/KismetMaterialLibrary.h`) does the world lookup for you:

```cpp
static void SetScalarParameterValue(UObject* WorldContextObject, UMaterialParameterCollection* Collection, FName ParameterName, float ParameterValue);
static void SetVectorParameterValue(UObject* WorldContextObject, UMaterialParameterCollection* Collection, FName ParameterName, const FLinearColor& ParameterValue);
static float GetScalarParameterValue(UObject* WorldContextObject, UMaterialParameterCollection* Collection, FName ParameterName);
static FLinearColor GetVectorParameterValue(UObject* WorldContextObject, UMaterialParameterCollection* Collection, FName ParameterName);
static class UMaterialInstanceDynamic* CreateDynamicMaterialInstance(UObject* WorldContextObject, class UMaterialInterface* Parent, FName OptionalName = NAME_None, EMIDCreationFlags CreationFlags = EMIDCreationFlags::None);
```

`EMIDCreationFlags` is `None` or `Transient`.

---

## UPrimitiveComponent material slots

`Components/PrimitiveComponent.h`

```cpp
virtual class UMaterialInterface* GetMaterial(int32 ElementIndex) const override;
virtual void SetMaterial(int32 ElementIndex, class UMaterialInterface* Material);
virtual void SetMaterialByName(FName MaterialSlotName, class UMaterialInterface* Material);
virtual int32 GetMaterialIndex(FName MaterialSlotName) const;
virtual TArray<FName> GetMaterialSlotNames() const;
virtual int32 GetNumMaterials() const override;
virtual class UMaterialInstanceDynamic* CreateDynamicMaterialInstance(int32 ElementIndex, class UMaterialInterface* SourceMaterial = NULL, FName OptionalName = NAME_None);
```

`SetMaterial` swaps the static material and does not create a MID. `CreateDynamicMaterialInstance` with `SourceMaterial = nullptr` parents the new MID to whatever is already in the slot.

---

## Material usage flags

`UMaterialInterface` exposes usage through accessors, never through the `bUsedWith*` fields:

```cpp
virtual bool GetUsageByFlag(EMaterialUsage Usage) const;
virtual bool SetMaterialUsage(EMaterialUsage Usage);
bool CheckMaterialUsage(EMaterialUsage Usage);
bool CheckMaterialUsage_Concurrent(EMaterialUsage Usage) const;
bool NeedsSetMaterialUsage_Concurrent(bool& bOutHasUsage, EMaterialUsage Usage) const;
```

`EMaterialUsage` (`Materials/MaterialInterface.h`): `MATUSAGE_SkeletalMesh`, `MATUSAGE_ParticleSprites`, `MATUSAGE_BeamTrails`, `MATUSAGE_MeshParticles`, `MATUSAGE_StaticLighting`, `MATUSAGE_MorphTargets`, `MATUSAGE_SplineMesh`, `MATUSAGE_InstancedStaticMeshes`, `MATUSAGE_GeometryCollections`, `MATUSAGE_Clothing`, `MATUSAGE_NiagaraSprites`, `MATUSAGE_NiagaraRibbons`, `MATUSAGE_NiagaraMeshParticles`, `MATUSAGE_GeometryCache`, `MATUSAGE_Water`, `MATUSAGE_HairStrands`, `MATUSAGE_LidarPointCloud`, `MATUSAGE_VirtualHeightfieldMesh`, `MATUSAGE_Nanite`, `MATUSAGE_Voxels`, `MATUSAGE_VolumetricCloud`, `MATUSAGE_HeterogeneousVolumes`, `MATUSAGE_StaticMesh`, `MATUSAGE_EditorCompositing`, `MATUSAGE_NeuralNetworks`, `MATUSAGE_MeshDeformer`, `MATUSAGE_InstancedSkinnedMesh`, `MATUSAGE_Curves`.

`UMaterial::SetUsageByFlag(EMaterialUsage Usage, bool NewValue)` writes the flag without validating or recompiling — use `SetMaterialUsage` unless you know why you want the raw setter.

---

## Parameter type mapping

| Material parameter node | Setter | C++ type |
|---|---|---|
| Scalar Parameter | `SetScalarParameterValue` | `float` |
| Vector Parameter | `SetVectorParameterValue` | `FLinearColor` |
| Vector Parameter (double precision) | `SetDoubleVectorParameterValue` | `FVector4` |
| Texture Parameter | `SetTextureParameterValue` | `UTexture*` |
| Runtime Virtual Texture | `SetRuntimeVirtualTextureParameterValue` | `URuntimeVirtualTexture*` |
| Sparse Volume Texture | `SetSparseVolumeTextureParameterValue` | `USparseVolumeTexture*` |
| Texture Collection | `SetTextureCollectionParameterValue` | `UTextureCollection*` |
| Font Parameter | `SetFontParameterValue` | `UFont*` plus page index |

`SetVectorParameterValue` also has inline `const FVector&` / `const FVector4&` overloads that convert to `FLinearColor` (`Materials/MaterialInstanceDynamic.h:112-113`); they are not exposed to Blueprint.

Static switch, static bool and static component mask parameters are baked into the shader permutation and cannot change on a MID. Author a `UMaterialInstanceConstant` per permutation and parent the MID to the right one.

---

## MPC or MID

| Scenario | Use |
|---|---|
| Time of day, weather, one global wetness value | Material parameter collection |
| Anything read by 50+ materials at once | Material parameter collection |
| Per-actor damage, tint, health fill | MID |
| Per-instance texture swap | MID |
| Parameters inside a material layer or blend | MID with `...ByInfo` setters |
| A texture that must change globally | MID (collections hold no textures) |

---

## Render target format selection

| Use | Format |
|---|---|
| LDR colour, UI, minimap | `RTF_RGBA8` |
| sRGB-encoded colour | `RTF_RGBA8_SRGB` |
| HDR scene colour | `RTF_RGBA16f` |
| Single-channel data or depth | `RTF_R16f` |
| Full float computation | `RTF_RGBA32f` |
| Packed display output | `RTF_RGB10A2` |

`RTF_RGBA32f` costs four times the memory and bandwidth of `RTF_RGBA16f`; reach for it only when 16-bit float genuinely loses precision.

---

## Scene capture cost control

A `USceneCaptureComponent2D` runs a whole scene render. Cost scales with target resolution, capture source, visible geometry and any post process applied to the capture.

```cpp
// Capture on demand instead of every frame
SceneCapture->bCaptureEveryFrame = false;
SceneCapture->bCaptureOnMovement = false;
SceneCapture->CaptureScene();

// Trim expensive passes out of the capture
SceneCapture->ShowFlags.SetAtmosphere(false);
SceneCapture->ShowFlags.SetFog(false);
SceneCapture->ShowFlags.SetBloom(false);
SceneCapture->ShowFlags.SetMotionBlur(false);
SceneCapture->ShowFlags.SetContactShadows(false);

// Restrict what is drawn
SceneCapture->PrimitiveRenderMode = ESceneCapturePrimitiveRenderMode::PRM_UseShowOnlyList;
SceneCapture->ShowOnlyActors.Add(TargetActor);
SceneCapture->ShowOnlyComponent(TargetComponent);

// Or exclude a few actors from a normal capture
SceneCapture->HiddenActors.Add(PlayerActor);
```

The `ShowFlags.Set*` functions are generated from `Engine/Public/ShowFlagsValues.inl`; `Fog`, `Atmosphere`, `Bloom`, `MotionBlur` and `ContactShadows` are all declared there.

Keep minimap and security-camera targets at 256–512 px. For large mirrors prefer planar reflections over a scene capture.

---

## Parameter naming

Names are `FName`s compared exactly, including case. `"basecolor"`, `"Base Color"` and `"Base_Color"` are three different parameters and none of them matches `"BaseColor"`. A mismatch fails silently — no warning, no log.

Conventional names, none of them enforced by the engine:

| Domain | Typical names |
|---|---|
| PBR surface | `BaseColor`, `Roughness`, `Metallic`, `Emissive`, `EmissiveIntensity`, `Opacity`, `Normal` |
| Damage state | `DamageAmount`, `DamageMask`, `BurnAmount` |
| Environment (collection) | `RainWetness`, `SnowAmount`, `WindStrength`, `TimeOfDay` |
| UI and HUD | `HealthPercent`, `FillAmount`, `TintColor`, `MaskTexture` |

Match whatever the material author used; do not invent names in generated code.

---

## Debugging

**A setter has no visible effect.** Confirm the parameter name in the material editor's `ParameterName` field. If the parameter lives inside a material function or layer, use the `...ByInfo` setter with an `FMaterialParameterInfo` carrying the right `Association` and `Index`. Confirm the MID's parent is the material you think it is.

**The MID exists but the mesh does not change.** Check the cached pointer is a `UPROPERTY` (otherwise it was collected), check the element index against `GetMaterialSlotNames()`, and on skeletal meshes check you targeted the slot the visible LOD actually uses.

**A collection parameter never reaches the shader.** Check the collection asset the component references is the same asset the material samples, and check the parameter name case. Instance writes queue a uniform-buffer update that lands before the next frame renders.

**A parameter reads back as the parent's value.** `GetScalarParameterValue` with `bOveriddenOnly = true` only returns values this instance overrides; pass `false` to fall through to the parent.
