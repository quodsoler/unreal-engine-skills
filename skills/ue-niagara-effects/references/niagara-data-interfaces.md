# Niagara Data Interfaces — Built-In Reference

Data interfaces (DIs) extend Niagara scripts with external data sources that cannot be expressed as
scalar or vector parameters. They derive from `UNiagaraDataInterface`
(`Classes/NiagaraDataInterface.h`), which itself derives from `UNiagaraDataInterfaceBase` in the
`NiagaraCore` module.

DIs surface in the Niagara editor as `User.*` parameters of DI type and are bound at runtime with
`UNiagaraFunctionLibrary` helpers or `SetVariableObject`. All paths below are relative to
`Engine/Plugins/FX/Niagara/Source/Niagara/`.

**Include reachability**: `Public/` and `Classes/` headers are includable from game modules.
`Internal/` and `Private/` headers are not — you can still bind those DIs through
`UNiagaraFunctionLibrary` and `SetVariableObject`, but you cannot include the class or cast to it
from a game module.

---

## Mesh DIs

### UNiagaraDataInterfaceSkeletalMesh
**Header**: `Classes/NiagaraDataInterfaceSkeletalMesh.h`
**Sim target**: CPU + GPU (partial)

Samples positions, normals, UVs and bone transforms from a live `USkeletalMeshComponent`, skinned at
the moment of sampling.

```cpp
#include "NiagaraFunctionLibrary.h"

// Bind by component. The name argument is const FString&.
UNiagaraFunctionLibrary::OverrideSystemUserVariableSkeletalMeshComponent(
    NiagaraComp, TEXT("User.SourceMesh"), SkeletalMeshComp);

// Destructive filters — they modify the DI instance data.
UNiagaraFunctionLibrary::SetSkeletalMeshDataInterfaceFilteredBones(
    NiagaraComp, TEXT("User.SourceMesh"), { FName("spine_01"), FName("head") });

UNiagaraFunctionLibrary::SetSkeletalMeshDataInterfaceSamplingRegions(
    NiagaraComp, TEXT("User.SourceMesh"), { FName("UpperBody") });

UNiagaraFunctionLibrary::SetSkeletalMeshDataInterfaceFilteredSockets(
    NiagaraComp, TEXT("User.SourceMesh"), { FName("foot_l_socket") });

// Typed access, or the dedicated getter.
UNiagaraDataInterfaceSkeletalMesh* SkelDI =
    UNiagaraFunctionLibrary::GetSkeletalMeshDataInterface(NiagaraComp, TEXT("User.SourceMesh"));
```

**Key properties**:
- `SourceMode` (`ENDISkeletalMesh_SourceMode`): `Default`, `Source`, `AttachParent`, `DefaultMeshOnly`, `ChildrenOnly`
- `SoftSourceActor` — `TSoftObjectPtr<AActor>`; the DI resolves its skeletal mesh component
- `FilteredBones`, `FilteredSockets`, `SamplingRegions` — `TArray<FName>` sampling restrictions
- `bRequireCurrentFrameData` — wait for this frame's skinning before sampling (default true)

**Notes**:
- CPU sampling needs `bAllowCPUAccess` on the skeletal mesh asset.
- Pre-skinned vertex sampling is driven by `bUsesPreSkinnedVerts` on the DI's usage flags; it forces
  the bone matrices to be gathered.
- Not every function has a GPU implementation; the editor reports the unsupported ones.

---

### UNiagaraDataInterfaceStaticMesh
**Header**: `Internal/DataInterface/NiagaraDataInterfaceStaticMesh.h` (module-internal)
**Sim target**: CPU + GPU (partial)

Samples vertices, triangles, UVs and sockets from a static mesh, with section filtering and
instanced-static-mesh instance selection.

```cpp
UNiagaraFunctionLibrary::OverrideSystemUserVariableStaticMeshComponent(
    NiagaraComp, TEXT("User.ScatterMesh"), StaticMeshComp);

UNiagaraFunctionLibrary::OverrideSystemUserVariableStaticMesh(
    NiagaraComp, TEXT("User.ScatterMesh"), ScatterMesh);
```

`UNiagaraDataInterfaceStaticMesh::SetNiagaraStaticMeshDIInstanceIndex(UNiagaraComponent*, const FName UserParameterName, int32 NewInstanceIndex)`
selects which ISM instance to read from. It is `NIAGARA_API`, but its header is under `Internal/`,
so from a game module it is only reachable through Blueprint.

**Key properties**:
- `SourceMode` (`ENDIStaticMesh_SourceMode`): `Default`, `Source`, `AttachParent`, `DefaultMeshOnly`, `MeshParameterBinding`
- `DefaultMesh` — fallback `UStaticMesh*`
- `SectionFilter.AllowedMaterialSlots` — `TArray<int32>` restricting triangle sampling
- `LODIndex` (default 0) and `LODIndexUserParameter`
- `bCaptureTransformsPerFrame` (default true), `bAllowSamplingFromStreamingLODs` (default false)

CPU sampling needs `bAllowCPUAccess` on the static mesh asset.

---

## Curve DIs

All curve DIs derive from `UNiagaraDataInterfaceCurveBase` and bake a LUT; with `bUseLUT` (default
true, `Private/NiagaraDataInterfaceCurveBase.cpp:154`) both CPU and GPU sample the LUT, so runtime key
edits have no effect in cooked builds unless `bUseLUT` is off.

### UNiagaraDataInterfaceCurve
**Header**: `Classes/NiagaraDataInterfaceCurve.h` — output `float`

```cpp
UNiagaraDataInterfaceCurve* CurveDI =
    UNiagaraFunctionLibrary::GetDataInterface<UNiagaraDataInterfaceCurve>(
        NiagaraComp, FName("User.SpeedCurve"));

if (CurveDI)
{
    CurveDI->Curve.Reset();
    CurveDI->Curve.AddKey(0.0f, 0.0f);
    CurveDI->Curve.AddKey(0.5f, 1.0f);
    CurveDI->Curve.AddKey(1.0f, 0.0f);
    // UpdateLUT() rebuilds the look-up table and is editor-only data.
    // MarkRenderStateDirty() does not exist here: UNiagaraDataInterface is a UObject.
#if WITH_EDITORONLY_DATA
    CurveDI->UpdateLUT();
#endif
}
```

- `UNiagaraDataInterfaceVectorCurve` — `Classes/NiagaraDataInterfaceVectorCurve.h`, `XCurve`/`YCurve`/`ZCurve`
- `UNiagaraDataInterfaceColorCurve` — `Classes/NiagaraDataInterfaceColorCurve.h`, RGBA rich curves
- `UNiagaraDataInterfaceVector2DCurve` — `Classes/NiagaraDataInterfaceVector2DCurve.h`
- `UNiagaraDataInterfaceVector4Curve` — `Classes/NiagaraDataInterfaceVector4Curve.h`

---

## Array DIs

Array DIs hold a typed `TArray` that scripts index into — the primary channel for per-frame C++ data.
Base class `UNiagaraDataInterfaceArray` (`Classes/NiagaraDataInterfaceArray.h`).

| DI Class | Element Type | Header |
|---|---|---|
| `UNiagaraDataInterfaceArrayFloat` | `float` | `Classes/NiagaraDataInterfaceArrayFloat.h` |
| `UNiagaraDataInterfaceArrayFloat2` | `FVector2f` internal / `FVector2D` API | same |
| `UNiagaraDataInterfaceArrayFloat3` | `FVector3f` internal / `FVector` API | same |
| `UNiagaraDataInterfaceArrayFloat4` | `FVector4f` internal / `FVector4` API | same |
| `UNiagaraDataInterfaceArrayPosition` | `FNiagaraPosition` | same |
| `UNiagaraDataInterfaceArrayColor` | `FLinearColor` | same |
| `UNiagaraDataInterfaceArrayQuat` | `FQuat4f` internal / `FQuat` API | same |
| `UNiagaraDataInterfaceArrayMatrix` | `FMatrix44f` internal / `FMatrix` API | same |
| `UNiagaraDataInterfaceArrayInt32` | `int32` | `Classes/NiagaraDataInterfaceArrayInt.h` |

```cpp
#include "NiagaraDataInterfaceArrayFunctionLibrary.h"

TArray<FVector> PositionArray = BuildPositions();
UNiagaraDataInterfaceArrayFunctionLibrary::SetNiagaraArrayVector(
    NiagaraComp, FName("User.SpawnPositions"), PositionArray);
```

Do not write the DI's `UPROPERTY` array (`FloatData` etc.) directly at runtime: scripts read the
DI proxy, which only `SetArrayData` updates — that is what the library does
(`Private/NiagaraDataInterfaceArrayFunctionLibrary.cpp:38`). For large arrays use the
non-UFUNCTION `TConstArrayView` overloads, which skip the `TArray` copy and LWC conversion:

```cpp
TArray<FVector3f> Positions = BuildPositionsFloat();
UNiagaraDataInterfaceArrayFunctionLibrary::SetNiagaraArrayVector(
    NiagaraComp, FName("User.SpawnPositions"), TConstArrayView<FVector3f>(Positions));
```

The LWC variants keep a public LWC array plus a float mirror: Float2/3/4 use `FloatData` /
`InternalFloatData`, Quat uses `QuatData` / `InternalQuatData`, Matrix uses `MatrixData` /
`InternalMatrixData` (`Classes/NiagaraDataInterfaceArrayFloat.h`).

---

## Texture and Render Target DIs

| DI Class | Header | Sim target |
|---|---|---|
| `UNiagaraDataInterfaceTexture` | `Classes/NiagaraDataInterfaceTexture.h` | GPU |
| `UNiagaraDataInterface2DArrayTexture` | `Classes/NiagaraDataInterface2DArrayTexture.h` | GPU |
| `UNiagaraDataInterfaceVolumeTexture` | `Classes/NiagaraDataInterfaceVolumeTexture.h` | GPU |
| `UNiagaraDataInterfaceCubeTexture` | `Classes/NiagaraDataInterfaceCubeTexture.h` | GPU |
| `UNiagaraDataInterfaceRenderTarget2D` | `Classes/NiagaraDataInterfaceRenderTarget2D.h` | GPU read/write |
| `UNiagaraDataInterfaceRenderTarget2DArray` | `Classes/NiagaraDataInterfaceRenderTarget2DArray.h` | GPU read/write |

```cpp
UNiagaraFunctionLibrary::SetTextureObject(NiagaraComp, TEXT("User.FlowTexture"), FlowTexture);
UNiagaraFunctionLibrary::SetTexture2DArrayObject(NiagaraComp, TEXT("User.TexArray"), TexArray);
UNiagaraFunctionLibrary::SetVolumeTextureObject(NiagaraComp, TEXT("User.DensityVol"), VolumeTexture);

// Or through the component, for a Texture-typed User parameter.
NiagaraComp->SetVariableTexture(FName("User.FlowTexture"), FlowTexture);
```

The scene-capture DI lives in `Private/`, so configure it through the public helper:
`UNiagaraFunctionLibrary::SetSceneCapture2DDataInterfaceManagedMode(NiagaraComp, DIName, ManagedCaptureSource, ManagedTextureSize, ManagedTextureFormat, ManagedProjectionType, ManagedFOVAngle, ManagedOrthoWidth, bManagedCaptureEveryFrame, bManagedCaptureOnMovement, ShowOnlyActors, ManagedLODDistanceFactor)`.
It is destructive: it modifies the DI instance.

---

## Noise, Grid and Simulation DIs

| DI Class | Header | Sim target |
|---|---|---|
| `UNiagaraDataInterfaceCurlNoise` | `Classes/NiagaraDataInterfaceCurlNoise.h` | CPU + GPU |
| `UNiagaraDataInterfaceGrid2DCollection` | `Classes/NiagaraDataInterfaceGrid2DCollection.h` | GPU |
| `UNiagaraDataInterfaceGrid2DCollectionReader` | `Classes/NiagaraDataInterfaceGrid2DCollectionReader.h` | GPU |
| `UNiagaraDataInterfaceGrid3DCollection` | `Classes/NiagaraDataInterfaceGrid3DCollection.h` | GPU |
| `UNiagaraDataInterfaceNeighborGrid3D` | `Classes/NiagaraDataInterfaceNeighborGrid3D.h` | GPU |

Grid DIs are configured in the Niagara editor, not from C++. `Grid2DCollectionReader` lets one
emitter read another emitter's grid.

---

## Collision and Physics DIs

### UNiagaraDataInterfaceCollisionQuery
**Header**: `Classes/NiagaraDataInterfaceCollisionQuery.h`
**Sim target**: CPU (synchronous traces) + GPU (depth buffer and global distance field)

Hardware-ray-traced GPU collision goes through `UNiagaraDataInterfaceAsyncGpuTrace` (below); these
library helpers manage its HWRT collision groups:

```cpp
// Acquire a collision group, tag primitives into it, release when done.
const int32 GroupIdx = UNiagaraFunctionLibrary::AcquireNiagaraGPURayTracedCollisionGroup(this);

UNiagaraFunctionLibrary::SetComponentNiagaraGPURayTracedCollisionGroup(this, TargetPrimitive, GroupIdx);
UNiagaraFunctionLibrary::SetActorNiagaraGPURayTracedCollisionGroup(this, TargetActor, GroupIdx);

UNiagaraFunctionLibrary::ReleaseNiagaraGPURayTracedCollisionGroup(this, GroupIdx);
```

| DI Class | Header | Notes |
|---|---|---|
| `UNiagaraDataInterfaceAsyncGpuTrace` | `Classes/NiagaraDataInterfaceAsyncGpuTrace.h` | GPU ray traces; results land the following frame |
| `UNiagaraDataInterfacePhysicsAsset` | `Public/NiagaraDataInterfacePhysicsAsset.h` | Physics asset body transforms |
| `UNiagaraDataInterfaceRigidMeshCollisionQuery` | `Public/NiagaraDataInterfaceRigidMeshCollisionQuery.h` | SDF collision against rigid meshes |

---

## Audio DIs

There is no abstract audio DI base class in 5.8. The three concrete classes are:

| DI Class | Header | Purpose |
|---|---|---|
| `UNiagaraDataInterfaceAudioOscilloscope` | `Classes/NiagaraDataInterfaceAudioOscilloscope.h` | Time-domain waveform from a `USoundSubmix* Submix` |
| `UNiagaraDataInterfaceAudioSpectrum` | `Classes/NiagaraDataInterfaceAudioSpectrum.h` | FFT spectrum from a `USoundSubmix* Submix` |
| `UNiagaraDataInterfaceAudioPlayer` | `Classes/NiagaraDataInterfaceAudioPlayer.h` | Plays sounds from inside a simulation; settings on `UNiagaraDataInterfaceAudioPlayerSettings` |

---

## Scene and Actor DIs

| DI Class | Header | Notes |
|---|---|---|
| `UNiagaraDataInterfaceCamera` | `Classes/NiagaraDataInterfaceCamera.h` | Camera transform, FOV, depth buffer access |
| `UNiagaraDataInterfaceSpline` | `Classes/NiagaraDataInterfaceSpline.h` | Positions, tangents and up vectors along a `USplineComponent` |
| `UNiagaraDataInterfaceLandscape` | `Classes/NiagaraDataInterfaceLandscape.h` | Height, normal and layer weights from a landscape |
| `UNiagaraDataInterfaceOcclusion` | `Classes/NiagaraDataInterfaceOcclusion.h` | Renderer occlusion results |
| `UNiagaraDataInterfaceParticleRead` | `Classes/NiagaraDataInterfaceParticleRead.h` | Read another emitter's particle attributes |
| `UNiagaraDataInterfaceActorComponent` | `Internal/DataInterface/NiagaraDataInterfaceActorComponent.h` | An actor component's transform and velocity |
| `UNiagaraDataInterfaceMaterialInstanceDynamic` | `Private/NiagaraDataInterfaceMaterialInstanceDynamic.h` | Read scalars and vectors from a `UMaterialInstanceDynamic` |
| `UNiagaraDataInterfaceMaterialParameterCollection` | `Private/NiagaraDataInterfaceMaterialParameterCollection.h` | Read a `UMaterialParameterCollection` |

---

## Data Channel DIs

**Headers**: `Internal/DataInterface/NiagaraDataInterfaceDataChannelRead.h` and
`Internal/DataInterface/NiagaraDataInterfaceDataChannelWrite.h`

These are the Niagara-script side of Niagara Data Channels: an emitter reads elements published to a
`UNiagaraDataChannelAsset`, or writes elements other systems and game code consume. The C++/Blueprint
side is `UNiagaraDataChannelLibrary` (`Public/NiagaraDataChannelFunctionLibrary.h`) plus
`UNiagaraDataChannelWriter` / `UNiagaraDataChannelReader` (`Public/NiagaraDataChannelAccessor.h`) —
see the Niagara Data Channels section of the main skill.

---

## Export DI

### UNiagaraDataInterfaceExport
**Header**: `Classes/NiagaraDataInterfaceExport.h`
**Sim target**: CPU + GPU

Pushes particle data back out to a `UObject` each tick. The receiving object is named by the DI's
`CallbackHandlerParameter` (`FNiagaraUserParameterBinding`) and must implement
`INiagaraParticleCallbackHandler`.

`ReceiveParticleData` is declared `UFUNCTION(BlueprintCallable, BlueprintNativeEvent)`, so the C++
override is `ReceiveParticleData_Implementation`. Overriding `ReceiveParticleData` itself does not
satisfy the interface and the callback never fires.

```cpp
// MyParticleReceiver.h
#pragma once

#include "CoreMinimal.h"
#include "NiagaraDataInterfaceExport.h"
#include "UObject/Object.h"
#include "MyParticleReceiver.generated.h"

class UNiagaraSystem;

UCLASS(BlueprintType)
class MYGAME_API UMyParticleReceiver : public UObject, public INiagaraParticleCallbackHandler
{
    GENERATED_BODY()

public:
    virtual void ReceiveParticleData_Implementation(
        const TArray<FBasicParticleData>& Data,
        UNiagaraSystem* NiagaraSystem,
        const FVector& SimulationPositionOffset) override;
};
```

```cpp
// MyParticleReceiver.cpp
#include "MyParticleReceiver.h"

#include "NiagaraSystem.h"

void UMyParticleReceiver::ReceiveParticleData_Implementation(
    const TArray<FBasicParticleData>& Data,
    UNiagaraSystem* NiagaraSystem,
    const FVector& SimulationPositionOffset)
{
    for (const FBasicParticleData& Particle : Data)
    {
        const FVector WorldPosition = Particle.Position + SimulationPositionOffset;
        ProcessImpact(WorldPosition, Particle.Velocity, Particle.Size);
    }
}
```

Bind the receiver at runtime with `NiagaraComp->SetVariableObject(FName("User.ExportTarget"), Receiver);`
using whatever `User.` parameter the DI's `CallbackHandlerParameter` points at.

`FBasicParticleData` carries `Position`, `Size` and `Velocity`. For GPU simulations the DI reserves a
buffer sized by `ENDIExport_GPUAllocationMode` — `FixedSize` uses `GPUAllocationFixedSize`,
`PerParticle` multiplies the emitter particle count by `GPUAllocationPerParticleSize`.

---

## Writing a Custom Data Interface

Subclass `UNiagaraDataInterface`. The function list is editor-only data, so the override is
`GetFunctionsInternal(TArray<FNiagaraFunctionSignature>&) const` guarded by `WITH_EDITORONLY_DATA` —
matching the base declaration exactly.

```cpp
// MyTerrainDataInterface.h
#pragma once

#include "CoreMinimal.h"
#include "NiagaraDataInterface.h"
#include "MyTerrainDataInterface.generated.h"

class FNiagaraSystemInstance;

UCLASS(EditInlineNew, Category = "Terrain", meta = (DisplayName = "My Terrain Query"))
class MYGAME_API UMyTerrainDataInterface : public UNiagaraDataInterface
{
    GENERATED_BODY()

public:
    virtual void GetVMExternalFunction(const FVMExternalFunctionBindingInfo& BindingInfo, void* InstanceData, FVMExternalFunction& OutFunc) override;

    virtual int32 PerInstanceDataSize() const override;
    virtual bool InitPerInstanceData(void* PerInstanceData, FNiagaraSystemInstance* SystemInstance) override;
    virtual void DestroyPerInstanceData(void* PerInstanceData, FNiagaraSystemInstance* SystemInstance) override;
    virtual bool PerInstanceTick(void* PerInstanceData, FNiagaraSystemInstance* SystemInstance, float DeltaSeconds) override;

    virtual bool Equals(const UNiagaraDataInterface* Other) const override;

    /** GPU path: fill DataForRenderThread for the matching FNiagaraDataInterfaceProxy. */
    virtual void ProvidePerInstanceDataForRenderThread(void* DataForRenderThread, void* PerInstanceData, const FNiagaraSystemInstanceID& SystemInstance) override;
    virtual int32 PerInstanceDataPassedToRenderThreadSize() const override;

protected:
    virtual bool CopyToInternal(UNiagaraDataInterface* Destination) const override;

#if WITH_EDITORONLY_DATA
    virtual void GetFunctionsInternal(TArray<FNiagaraFunctionSignature>& OutFunctions) const override;
#endif
};
```

Rules that trip people up:

- `GetFunctionsInternal` is `const` and lives under `#if WITH_EDITORONLY_DATA`. Callers use the
  non-virtual `GetFunctionSignatures(TArray<FNiagaraFunctionSignature>&) const`.
- `Equals` and `CopyToInternal` must both account for every new `UPROPERTY`, or duplicated systems
  and editor copies silently lose settings.
- `PerInstanceDataPassedToRenderThreadSize()` must return a 16-byte-aligned size.
- Add `"NiagaraCore"` to the module's `PublicDependencyModuleNames` alongside `"Niagara"`.

---

## Choosing a DI

| Use Case | Recommended DI |
|---|---|
| Spawn particles on a character mesh | `UNiagaraDataInterfaceSkeletalMesh` |
| Scatter particles on a static prop | `UNiagaraDataInterfaceStaticMesh` |
| Drive emitter rate with an authored curve | `UNiagaraDataInterfaceCurve` |
| Push a gameplay position list | `UNiagaraDataInterfaceArrayPosition` |
| Particles collide with world geometry | `UNiagaraDataInterfaceCollisionQuery` |
| Audio-reactive effects | `UNiagaraDataInterfaceAudioSpectrum` |
| Particles follow a spline path | `UNiagaraDataInterfaceSpline` |
| Camera-relative VFX | `UNiagaraDataInterfaceCamera` |
| Export particle positions to gameplay code | `UNiagaraDataInterfaceExport` |
| Send gameplay events into Niagara | Niagara Data Channels (`UNiagaraDataChannelLibrary`) |
| Share data between independent systems | Niagara Data Channels |
| Volumetric or fluid simulation | `UNiagaraDataInterfaceGrid3DCollection` |
