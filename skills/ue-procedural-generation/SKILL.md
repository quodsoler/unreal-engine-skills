---
name: ue-procedural-generation
description: "Use when generating world content or geometry procedurally in Unreal Engine: PCG graphs, custom PCG nodes in C++, runtime mesh building, instancing and seeded noise. Also use when the user mentions 'PCG', 'UPCGComponent', 'UPCGSettings', 'IPCGElement', 'UPCGBasePointData', 'surface sampler', 'static mesh spawner', 'runtime generation', 'CreateMeshSection', 'UProceduralMeshComponent', 'UDynamicMeshComponent', 'Geometry Script', 'AddInstance', 'HISM', 'spline mesh', 'PerlinNoise', 'FRandomStream', 'scatter foliage' or 'marching cubes'. For collision on generated geometry, see ue-physics-collision; for instance materials, see ue-materials-rendering; for background work, see ue-async-threading."
metadata:
  version: "2.0.0"
  engine: "5.8"
---

# UE Procedural Generation

Target engine: **UE 5.8**. APIs below are verified against the 5.8 headers; older forms are listed under "Deprecated — do not use".

Covers the PCG framework (plugin `PCG` at `Engine/Plugins/PCG`, enabled by default, Build.cs module `PCG`), runtime mesh building with `UProceduralMeshComponent` (plugin `ProceduralMeshComponent`, module `ProceduralMeshComponent`) and `UDynamicMeshComponent` + Geometry Script (modules `GeometryFramework` and `GeometryScriptingCore`), instancing with `UInstancedStaticMeshComponent`/`UHierarchicalInstancedStaticMeshComponent`, spline-driven placement, and deterministic noise/random from `Core`.

## Context

Read `.agents/ue-project-context.md` if it exists (module names, conventions, enabled plugins, GAS/networking setup). Do not stop if it is missing.

Identify the area from the request and the codebase. Ask only when two plausible readings would produce different code.

| Request is about… | Go to |
|---|---|
| Enabling PCG, module/plugin wiring | [PCG setup](#pcg-setup) |
| Driving generation from an actor, runtime generation, partitioning | [PCG component and runtime generation](#pcg-component-and-runtime-generation) |
| Reading/writing points inside a graph | [Point data](#point-data) |
| Writing a new PCG node in C++ | [Custom PCG node in C++](#custom-pcg-node-in-c) |
| Which node does X, pin labels, node settings fields | [PCG node reference](references/pcg-node-reference.md) |
| Building triangles at runtime | [ProceduralMeshComponent](#proceduralmeshcomponent) |
| Boolean ops, primitives, baking to a static mesh | [Dynamic Mesh and Geometry Script](#dynamic-mesh-and-geometry-script) |
| Thousands of repeated meshes | [Instanced static meshes](#instanced-static-meshes) |
| Roads, rivers, fences, cables | [Splines](#splines) |
| Heightfields, scatter, reproducible results | [Noise and deterministic random](#noise-and-deterministic-random) |
| Marching cubes, BSP dungeons, WFC, Poisson disc | [Procedural mesh patterns](references/procedural-mesh-patterns.md) |

## PCG setup

```csharp
// MyGame.Build.cs
PublicDependencyModuleNames.AddRange(new string[] { "Core", "CoreUObject", "Engine", "PCG" });
```

```json
{ "Name": "PCG", "Enabled": true }
```

| Class | Header | Role |
|---|---|---|
| `UPCGComponent` | `PCGComponent.h` | Actor component that owns a graph and drives generation |
| `UPCGGraph` | `PCGGraph.h` | Graph asset: nodes, edges, `UserParameters` |
| `UPCGGraphInstance` | `PCGGraph.h` | Graph instance with per-instance parameter overrides |
| `UPCGGraphInterface` | `PCGGraph.h` | Common base of graph and graph instance |
| `UPCGSettings` | `PCGSettings.h` | Node settings base class |
| `IPCGElement` | `PCGElement.h` | The executable half of a node |
| `FPCGContext` | `PCGContext.h` | Per-execution state: `InputData`, `OutputData`, `Node`, `ExecutionSource` |
| `UPCGBasePointData` | `Data/PCGBasePointData.h` | Abstract point collection |
| `UPCGPointArrayData` | `Data/PCGPointArrayData.h` | Structure-of-arrays point data |
| `UPCGSubsystem` | `Subsystems/PCGSubsystem.h` | Scheduling, partitioning, runtime generation |
| `APCGVolume` | `PCGVolume.h` | Volume actor carrying a `UPCGComponent` |

## PCG component and runtime generation

```cpp
#include "PCGComponent.h"
#include "PCGGraph.h"

void AMyGenerator::StartGeneration(UPCGGraphInterface* Graph, int32 InSeed)
{
    UPCGComponent* PCG = FindComponentByClass<UPCGComponent>();
    if (!PCG)
    {
        return;
    }

    PCG->Seed = InSeed;
    PCG->bActivated = true;
    PCG->SetGraph(Graph);      // NetMulticast, Reliable
    PCG->Generate(/*bForce=*/true);  // NetMulticast, Reliable
}
```

| Call | Replication | Use for |
|---|---|---|
| `Generate(bool bForce)` / `Cleanup(bool bRemoveComponents)` | `NetMulticast, Reliable` | Server-driven generation that must appear on clients |
| `GenerateLocal(bool bForce)` / `CleanupLocal(bool bRemoveComponents)` | none | Client-side or single-player generation |
| `NotifyPropertiesChangedFromBlueprint()` | none | Mark dirty and conditionally regenerate after editing exposed properties |
| `CancelGeneration()` | none | Abort an in-flight generation |
| `GetGeneratedGraphOutput()` | none | Read back the `FPCGDataCollection` the graph produced |

`EPCGComponentGenerationTrigger` (`PCGComponent.h:77`): `GenerateOnLoad`, `GenerateOnDemand`, `GenerateAtRuntime`.

Partitioning and runtime generation:

- `SetIsPartitioned(bool)` / `IsPartitioned()` back the `bIsComponentPartitioned` property; partitioned components dispatch work to local components on a grid.
- `GenerateAtRuntime` hands the component to the runtime-gen scheduler. Tune it with `SchedulingPolicyClass` / `SchedulingPolicy` (`UPCGSchedulingPolicyBase`) and `bOverrideGenerationRadii` + `GenerationRadii` (`FPCGRuntimeGenerationRadii`).
- Scheduler CVars: `pcg.RuntimeGeneration.Enable`, `pcg.RuntimeGeneration.NumGeneratingComponents`, `pcg.RuntimeGeneration.GlobalRadiusMultiplier`, `pcg.RuntimeGeneration.EnablePooling`, `pcg.RuntimeGeneration.BasePoolSize`, `pcg.RuntimeGeneration.FramesBeforeFirstGenerate`, `pcg.RuntimeGeneration.EnableChangeDetection`, `pcg.RuntimeGeneration.EnableDebugging`.
- `UPCGSubsystem::GetSubsystemForCurrentWorld()` returns the subsystem; `RefreshAllComponentsFiltered(Filter, ChangeType)` forces a refresh of a subset (`WITH_EDITOR` only, `Subsystems/PCGSubsystem.h:227-230`).

Hierarchical generation lives on `UPCGGraph`: `bUseHierarchicalGeneration`, `HiGenGridSize` (`EPCGHiGenGrid::Grid4` … `Grid2048`, plus `Unbounded`), `HiGenGridSizeMultiplier`, `bUse2DGrid`.

Graph parameters are an `FInstancedPropertyBag UserParameters` on `UPCGGraph`, read and written through `UPCGGraphInterface`:

```cpp
TValueOrError<double, EPropertyBagResult> Result = Graph->GetGraphParameter<double>(TEXT("SpawnRadius"));
if (Result.HasValue())
{
    const double Radius = Result.GetValue();
    Graph->SetGraphParameter<double>(TEXT("SpawnRadius"), Radius * 2.0);
}
```

## Point data

Point collections are `UPCGBasePointData`. `UPCGPointArrayData` is the structure-of-arrays implementation; `UPCGPointData` is the array-of-`FPCGPoint` implementation kept for compatibility. Allocate through the context so the project-configured class is used:

```cpp
UPCGBasePointData* Points = FPCGContext::NewPointData_AnyThread(Context);
```

Never iterate `GetPoints()`/`GetMutablePoints()` in new code — that only exists on `UPCGPointData` and forces a conversion. Read and write through value ranges instead:

```cpp
#include "Data/PCGBasePointData.h"

const FConstPCGPointValueRanges ReadRanges(InputPoints);
FPCGPointValueRanges WriteRanges(OutputPoints, /*bAllocate=*/false);

WriteRanges.TransformRange[Index] = ReadRanges.TransformRange[Index];
WriteRanges.DensityRange[Index]   = ReadRanges.DensityRange[Index];
```

Per-point native properties (`EPCGPointNativeProperties` in `PCGPointPropertiesTraits.h`): `Transform`, `Density`, `BoundsMin`, `BoundsMax`, `Color`, `Steepness`, `Seed`, `MetadataEntry`, plus `All` and `AllProperties`. Sizing and allocation:

```cpp
Output->SetNumPoints(Input->GetNumPoints(), /*bInitializeValues=*/false);
Output->AllocateProperties(Input->GetAllocatedProperties() | EPCGPointNativeProperties::Density);
Output->CopyUnallocatedPropertiesFrom(Input);
```

Other data types: `UPCGSpatialData` (base), `UPCGSplineData`, `UPCGLandscapeData`, `UPCGVolumeData`, `UPCGTextureData`, `UPCGPrimitiveData`, `UPCGDynamicMeshData`, and `UPCGParamData` for attribute sets. `UPCGSpatialData::ToBasePointData(FPCGContext*, const FBox&)` discretizes any spatial data into points.

## Custom PCG node in C++

A node is a `UPCGSettings` subclass plus an `IPCGElement`. Settings hold data; the element is const and stateless and reads everything from `FPCGContext`.

```cpp
// MyPCGJitter.h
#pragma once

#include "PCGElement.h"
#include "PCGSettings.h"
#include "MyPCGJitter.generated.h"

UCLASS(BlueprintType, ClassGroup = (Procedural))
class MYGAME_API UMyPCGJitterSettings : public UPCGSettings
{
    GENERATED_BODY()

public:
#if WITH_EDITOR
    virtual FName GetDefaultNodeName() const override { return FName(TEXT("MyJitter")); }
    virtual FText GetDefaultNodeTitle() const override { return NSLOCTEXT("MyPCGJitter", "NodeTitle", "My Jitter"); }
    virtual EPCGSettingsType GetType() const override { return EPCGSettingsType::PointOps; }
#endif

    UPROPERTY(BlueprintReadWrite, EditAnywhere, Category = Settings, meta = (PCG_Overridable))
    double JitterRadius = 100.0;

protected:
    virtual TArray<FPCGPinProperties> InputPinProperties() const override { return Super::DefaultPointInputPinProperties(); }
    virtual TArray<FPCGPinProperties> OutputPinProperties() const override { return Super::DefaultPointOutputPinProperties(); }
    virtual FPCGElementPtr CreateElement() const override;
};

class FMyPCGJitterElement : public IPCGElement
{
protected:
    virtual bool ExecuteInternal(FPCGContext* Context) const override;
    virtual bool IsCacheable(const UPCGSettings* InSettings) const override { return true; }
    virtual bool CanExecuteOnlyOnMainThread(FPCGContext* Context) const override { return false; }
    virtual bool SupportsBasePointDataInputs(FPCGContext* InContext) const override { return true; }
    virtual EPCGElementExecutionLoopMode ExecutionLoopMode(const UPCGSettings* Settings) const override { return EPCGElementExecutionLoopMode::SinglePrimaryPin; }
};
```

```cpp
// MyPCGJitter.cpp
#include "MyPCGJitter.h"

#include "PCGContext.h"
#include "Data/PCGBasePointData.h"
#include "Data/PCGSpatialData.h"
#include "Math/RandomStream.h"

FPCGElementPtr UMyPCGJitterSettings::CreateElement() const
{
    return MakeShared<FMyPCGJitterElement>();
}

bool FMyPCGJitterElement::ExecuteInternal(FPCGContext* Context) const
{
    const UMyPCGJitterSettings* Settings = Context->GetInputSettings<UMyPCGJitterSettings>();
    check(Settings);

    const double JitterRadius = Settings->JitterRadius;
    const TArray<FPCGTaggedData> Inputs = Context->InputData.GetInputsByPin(PCGPinConstants::DefaultInputLabel);
    TArray<FPCGTaggedData>& Outputs = Context->OutputData.TaggedData;

    for (const FPCGTaggedData& Input : Inputs)
    {
        const UPCGSpatialData* SpatialData = Cast<UPCGSpatialData>(Input.Data);
        if (!SpatialData)
        {
            continue;
        }

        const UPCGBasePointData* InputPoints = SpatialData->ToBasePointData(Context);
        if (!InputPoints)
        {
            continue;
        }

        UPCGBasePointData* OutputPoints = FPCGContext::NewPointData_AnyThread(Context);
        OutputPoints->InitializeFromDataWithParams(FPCGInitializeFromDataParams(InputPoints));
        OutputPoints->SetNumPoints(InputPoints->GetNumPoints(), /*bInitializeValues=*/false);
        OutputPoints->AllocateProperties(InputPoints->GetAllocatedProperties() | EPCGPointNativeProperties::Transform);
        OutputPoints->CopyUnallocatedPropertiesFrom(InputPoints);

        const FConstPCGPointValueRanges ReadRanges(InputPoints);
        FPCGPointValueRanges WriteRanges(OutputPoints, /*bAllocate=*/false);

        for (int32 Index = 0; Index < InputPoints->GetNumPoints(); ++Index)
        {
            WriteRanges.SetFromValueRanges(Index, ReadRanges, Index);

            const FRandomStream Stream(ReadRanges.SeedRange[Index]);
            FTransform Jittered = ReadRanges.TransformRange[Index];
            Jittered.AddToTranslation(Stream.VRand() * (Stream.FRand() * JitterRadius));
            WriteRanges.TransformRange[Index] = Jittered;
        }

        FPCGTaggedData& Output = Outputs.Add_GetRef(Input);
        Output.Data = OutputPoints;
    }

    return true;
}
```

Rules that fall out of the headers:

- `SupportsBasePointDataInputs` returning `false` (the default) makes PCG convert every input to `UPCGPointData` before your element runs. Return `true` and use value ranges.
- `IsCacheable` must return `false` if the node spawns actors or components, or reads untracked data.
- `CanExecuteOnlyOnMainThread` returning `true` serializes the node onto the game thread; keep it `false` unless you touch `UWorld` or components.
- Long loops belong in `FPCGAsync::AsyncProcessingRangeEx(&Context->AsyncState, NumIterations, Initialize, ProcessRange, MoveDataRange, Finished, bEnableTimeSlicing)` (`Helpers/PCGAsync.h`), which time-slices and multithreads.
- Pin labels come from `PCGPinConstants::DefaultInputLabel` (`"In"`), `DefaultOutputLabel` (`"Out"`), `DefaultParamsLabel` (`"Overrides"`), `DefaultExecutionDependencyLabel`.
- Blueprint nodes derive from `UPCGBlueprintBaseElement` and override the `Execute(const FPCGDataCollection&, FPCGDataCollection&)` BlueprintNativeEvent; seed helpers are `GetSeedWithContext(GetContextHandle())` and `GetRandomStreamWithContext(GetContextHandle())`.

See [PCG node reference](references/pcg-node-reference.md) for node settings classes, pin behaviour, metadata attributes and GPU nodes.

## ProceduralMeshComponent

Triangle-level control at runtime. No Nanite support, no automatic LODs.

```csharp
PublicDependencyModuleNames.Add("ProceduralMeshComponent");
```

```cpp
// Full form (ProceduralMeshComponent.h:190) also takes UV1, UV2 and UV3 between UV0 and VertexColors
// UV0-only convenience overload (ProceduralMeshComponent.h:193)
void CreateMeshSection_LinearColor(int32 SectionIndex, const TArray<FVector>& Vertices, const TArray<int32>& Triangles,
    const TArray<FVector>& Normals, const TArray<FVector2D>& UV0, const TArray<FLinearColor>& VertexColors,
    const TArray<FProcMeshTangent>& Tangents, bool bCreateCollision, bool bSRGBConversion = false);

void UpdateMeshSection_LinearColor(int32 SectionIndex, const TArray<FVector>& Vertices, const TArray<FVector>& Normals,
    const TArray<FVector2D>& UV0, const TArray<FLinearColor>& VertexColors,
    const TArray<FProcMeshTangent>& Tangents, bool bSRGBConversion = true);   // true here, false on Create (ProceduralMeshComponent.h:230)

void ClearMeshSection(int32 SectionIndex);
void ClearAllMeshSections();
void SetMeshSectionVisible(int32 SectionIndex, bool bNewVisibility);
int32 GetNumSections() const;
void AddCollisionConvexMesh(TArray<FVector> ConvexVerts);
void ClearCollisionConvexMeshes();
```

Materials come from `UMeshComponent::SetMaterial(int32 ElementIndex, UMaterialInterface* Material)` — one material slot per section index.

Collision:

- `bUseComplexAsSimpleCollision` (default true) uses the rendered triangles for collision. Accurate, expensive, and cannot be simulated — set it to `false` and feed `AddCollisionConvexMesh` when the mesh must be dynamic.
- `bUseAsyncCooking` moves physics cooking off the game thread. Collision lags a frame or more behind the visual mesh; use it for far-away streamed geometry.

`UpdateMeshSection_LinearColor` moves existing vertices and refreshes collision, but cannot change vertex or triangle count — call `CreateMeshSection_LinearColor` when topology changes. Build the arrays on a worker thread, then call the component on the game thread; see [async mesh generation](references/procedural-mesh-patterns.md#async-mesh-generation-pattern).

## Dynamic Mesh and Geometry Script

`UDynamicMeshComponent` (`GeometryFramework`, header `Components/DynamicMeshComponent.h`) plus the Geometry Script libraries (`GeometryScriptingCore`) are the modern path: boolean operations, remeshing, normals recomputation, and baking to a `UStaticMesh` asset (editor only — `CopyMeshToStaticMesh` errors "Not currently supported at Runtime" outside `WITH_EDITOR`, `MeshAssetFunctions.cpp:471`).

```csharp
PublicDependencyModuleNames.AddRange(new string[] { "GeometryFramework", "GeometryScriptingCore" });
```

| Library | Representative functions |
|---|---|
| `UGeometryScriptLibrary_MeshPrimitiveFunctions` | `AppendBox`, `AppendSphereLatLong`, `AppendSphereBox`, `AppendCapsule`, `AppendBoxWithCollision` |
| `UGeometryScriptLibrary_MeshBooleanFunctions` | `ApplyMeshBoolean(TargetMesh, TargetTransform, ToolMesh, ToolTransform, Operation, Options, Debug)` |
| `UGeometryScriptLibrary_MeshDeformFunctions` | `ApplyPerlinNoiseToMesh2(TargetMesh, Selection, Options, Debug)` |
| `UGeometryScriptLibrary_MeshNormalsFunctions` | `RecomputeNormals(TargetMesh, CalculateOptions, bDeferChangeNotifications, Debug)`, `SetPerFaceNormals` |
| `UGeometryScriptLibrary_StaticMeshFunctions` | `CopyMeshToStaticMesh`, `CopyMeshFromStaticMeshV2` |

A worked example is in [dynamic mesh with Geometry Script](references/procedural-mesh-patterns.md#dynamic-mesh-with-geometry-script).

Collision on `UDynamicMeshComponent`: `EnableComplexAsSimpleCollision()`, `SetComplexAsSimpleCollisionEnabled(bool bEnabled, bool bImmediateUpdate)`, `SetSimpleCollisionShapes(const FKAggregateGeom&, bool bUpdateCollision)`, `bDeferCollisionUpdates` + `UpdateCollision(bool bOnlyIfPending)`. `ADynamicMeshActor` ships a component at the root via `GetDynamicMeshComponent()`.

## Instanced static meshes

| | `UInstancedStaticMeshComponent` | `UHierarchicalInstancedStaticMeshComponent` |
|---|---|---|
| Header | `Components/InstancedStaticMeshComponent.h` | `Components/HierarchicalInstancedStaticMeshComponent.h` |
| Best for | Small, frequently mutated sets | Large, mostly static sets |
| Culling | Start/end cull distance | Hierarchical tree plus cull distance |
| Removal | Cheap | Triggers a tree rebuild (`bAutoRebuildTreeOnInstanceChanges`, `BuildTreeIfOutdated`) |

```cpp
virtual int32 AddInstance(const FTransform& InstanceTransform, bool bWorldSpace = false);
virtual TArray<int32> AddInstances(const TArray<FTransform>& InstanceTransforms, bool bShouldReturnIndices,
    bool bWorldSpace = false, bool bUpdateNavigation = true);
virtual bool UpdateInstanceTransform(int32 InstanceIndex, const FTransform& NewInstanceTransform,
    bool bWorldSpace = false, bool bMarkRenderStateDirty = false, bool bTeleport = false);
virtual bool BatchUpdateInstancesTransforms(int32 StartInstanceIndex, const TArray<FTransform>& NewInstancesTransforms,
    bool bWorldSpace = false, bool bMarkRenderStateDirty = false, bool bTeleport = false);
bool GetInstanceTransform(int32 InstanceIndex, FTransform& OutInstanceTransform, bool bWorldSpace = false) const;
virtual bool RemoveInstance(int32 InstanceIndex);
virtual bool RemoveInstances(const TArray<int32>& InstancesToRemove);
virtual void PreAllocateInstancesMemory(int32 AddedInstanceCount);
int32 GetNumInstances() const;
virtual void SetNumCustomDataFloats(int32 InNumCustomDataFloats);
virtual bool SetCustomDataValue(int32 InstanceIndex, int32 CustomDataIndex, float CustomDataValue,
    bool bMarkRenderStateDirty = false);
void SetCullDistances(int32 StartCullDistance, int32 EndCullDistance);
const TArray<FBodyInstance*>& GetInstanceBodies() const;
```

Per-instance floats set with `SetNumCustomDataFloats` / `SetCustomDataValue` are read in materials through the PerInstanceCustomData node. Cull properties: `InstanceStartCullDistance`, `InstanceEndCullDistance`, `InstanceLODDistanceScale`, `bUseGpuLodSelection`.

Batch large populations: `PreAllocateInstancesMemory` first, build the whole `TArray<FTransform>`, then one `AddInstances` call. A full seeded scatter that traces onto terrain and fills per-instance custom data is in [vegetation scatter](references/procedural-mesh-patterns.md#vegetation-scatter-hism).

Foliage (module `Foliage`): painted foliage lives on `AInstancedFoliageActor` backed by `UFoliageInstancedStaticMeshComponent`; simulation-driven placement uses `UProceduralFoliageComponent` with a `UProceduralFoliageSpawner`. PCG's Static Mesh Spawner node is usually the better fit for graph-driven scatter.

## Splines

```cpp
// USplineComponent — Components/SplineComponent.h
void AddSplinePoint(const FVector& Position, ESplineCoordinateSpace::Type CoordinateSpace, bool bUpdateSpline = true);
void SetSplinePoints(const TArray<FVector>& Points, ESplineCoordinateSpace::Type CoordinateSpace, bool bUpdateSpline = true);
void SetSplinePointType(int32 PointIndex, ESplinePointType::Type Type, bool bUpdateSpline = true);
void SetTangentsAtSplinePoint(int32 PointIndex, const FVector& InArriveTangent, const FVector& InLeaveTangent,
    ESplineCoordinateSpace::Type CoordinateSpace, bool bUpdateSpline = true);
void SetClosedLoop(bool bInClosedLoop, bool bUpdateSpline = true);
virtual void UpdateSpline();
float GetSplineLength() const;
FVector GetLocationAtDistanceAlongSpline(float Distance, ESplineCoordinateSpace::Type CoordinateSpace) const;
FTransform GetTransformAtDistanceAlongSpline(float Distance, ESplineCoordinateSpace::Type CoordinateSpace,
    bool bUseScale = false) const;
void GetLocationAndTangentAtSplinePoint(int32 PointIndex, FVector& Location, FVector& Tangent,
    ESplineCoordinateSpace::Type CoordinateSpace) const;
float FindInputKeyClosestToWorldLocation(const FVector& WorldLocation) const;
```

`ESplinePointType::Type`: `Linear`, `Curve`, `Constant`, `CurveClamped`, `CurveCustomTangent`. `ESplineCoordinateSpace::Type`: `Local`, `World`.

Pass `bUpdateSpline = false` while batching edits and call `UpdateSpline()` once — each update rebuilds the reparameterization table. Distance along the spline is arc length; the input key is not, so always space instances by distance.

```cpp
// USplineMeshComponent — one deformed mesh per spline segment
void AMyRoadActor::BuildSegment(USplineComponent* Spline, UStaticMesh* RoadMesh, int32 SegmentIndex)
{
    USplineMeshComponent* SegmentMesh = NewObject<USplineMeshComponent>(this);
    SegmentMesh->SetMobility(EComponentMobility::Movable);
    SegmentMesh->SetupAttachment(Spline);
    SegmentMesh->SetStaticMesh(RoadMesh);
    SegmentMesh->SetForwardAxis(ESplineMeshAxis::X, /*bUpdateMesh=*/false);
    SegmentMesh->RegisterComponent();

    FVector StartPos, StartTangent, EndPos, EndTangent;
    Spline->GetLocationAndTangentAtSplinePoint(SegmentIndex, StartPos, StartTangent, ESplineCoordinateSpace::Local);
    Spline->GetLocationAndTangentAtSplinePoint(SegmentIndex + 1, EndPos, EndTangent, ESplineCoordinateSpace::Local);
    SegmentMesh->SetStartAndEnd(StartPos, StartTangent, EndPos, EndTangent, /*bUpdateMesh=*/true);
}
```

## Noise and deterministic random

```cpp
#include "Math/RandomStream.h"
#include "Math/UnrealMathUtility.h"

// All three return a continuous value in [-1, 1]
float SampleNoise(const FVector& Position, float Frequency)
{
    const float N1 = FMath::PerlinNoise1D(Position.X * Frequency);
    const float N2 = FMath::PerlinNoise2D(FVector2D(Position.X, Position.Y) * Frequency);
    const float N3 = FMath::PerlinNoise3D(Position * Frequency);
    return (N1 + N2 + N3) / 3.f;
}

FTransform MakeSeededTransform(int32 Seed, const FVector& Origin)
{
    FRandomStream Stream(Seed);
    const float Unit = Stream.GetFraction();              // [0, 1), same as FRand()
    const double Offset = Stream.FRandRange(-50.0, 50.0);
    const int32 Variant = Stream.RandRange(0, 3);         // inclusive on both ends
    const FVector Direction = Stream.VRand();             // uniform unit vector

    return FTransform(FRotator(0.0, Unit * 360.0, 0.0),
                      Origin + Direction * Offset,
                      FVector(1.0 + Variant * 0.1));
}
```

Determinism rules:

- Derive every stream from one project seed. `FRandomStream::Initialize(int32)` resets a stream; `GetCurrentSeed()` / `GetInitialSeed()` let you checkpoint one.
- Inside PCG, seed per point from `ReadRanges.SeedRange[Index]`, or take the node seed from `FPCGContext::GetSeed()`. Do not call `FMath::Rand`.
- For networked generation, replicate the seed (GameState or spawn parameter) and use `Generate(bForce)`; `GenerateLocal` never replicates.
- Sort inputs before consuming them when order affects the result — iteration order of gathered actor data is not guaranteed stable.

Octave/fractal noise, Poisson disc sampling, marching cubes, BSP dungeons and wave function collapse are implemented in [procedural mesh patterns](references/procedural-mesh-patterns.md).

## Deprecated — do not use

| Do not emit | Use in 5.8 | Source |
|---|---|---|
| `FSimplePCGElement` | `IPCGElement` | `UE_DEPRECATED(5.4)` in `PCGElement.h:271` |
| `IPCGElement::Initialize(const FPCGDataCollection&, TWeakObjectPtr<UPCGComponent>, const UPCGNode*)` | `Initialize(const FPCGInitializeElementParams&)` | `UE_DEPRECATED(5.6)` in `PCGElement.h:147` |
| `FPCGContext::SourceComponent` | `FPCGContext::ExecutionSource` | `UE_DEPRECATED(5.6)` in `PCGContext.h:122` |
| `FPCGContext::GetComponentName()` | `GetExecutionSourceName()` | `UE_DEPRECATED(5.6)` in `PCGContext.h:207` |
| `FPCGContext::Stack` | `GetStack()` | `UE_DEPRECATED(5.6)` in `PCGContext.h:147` |
| `UPCGSpatialData::IntersectWith/ProjectOn/UnionWith/Subtract/CopyInternal` without a context | overloads taking `FPCGContext*` | `UE_DEPRECATED(5.5)` in `Data/PCGSpatialData.h:208-280` |
| `ToPointData()` (no context) | `ToBasePointData(FPCGContext*)` / `ToBasePointDataWithContext` | `DeprecatedFunction` in `Data/PCGSpatialData.h:159` |
| `UPCGPointData::GetOctree()` / `IsOctreeDirty()` | `GetPointOctree()` / `IsPointOctreeDirty()` | `UE_DEPRECATED(5.6)` in `Data/PCGPointData.h:145,147` |
| `UPCGComponent::CleanupLocal(bRemoveComponents, bSave)` | `CleanupLocal(bRemoveComponents)` | `UE_DEPRECATED(5.6)` in `PCGComponent.h:236` |
| `UPCGComponent::GenerateLocal(..., EPCGHiGenGrid Grid, ...)` | overload taking a `uint32` grid size | `UE_DEPRECATED(5.8)` in `PCGComponent.h:872` |
| `UPCGGraph::HiGenExponential` / `GetGridExponential()` | `HiGenGridSizeMultiplier` / `GetGridSizeMultiplier()` | `UE_DEPRECATED(5.8)` in `PCGGraph.h:866,872` |
| `UPCGGraph::bIsEditorOnly` | `ShouldCook` (`FPerPlatformBool`) | `UE_DEPRECATED(5.8)` in `PCGGraph.h:868` |
| `UPCGSettings::BP_GetTypeUnionOfIncidentEdges` | `GetTypeUnionIDOfIncidentEdges` | `UE_DEPRECATED(5.7)` in `PCGSettings.h:520` |
| `UInstancedStaticMeshComponent::InstanceBodies` | `GetInstanceBodies()` | `UE_DEPRECATED(5.8)` in `Components/InstancedStaticMeshComponent.h:524` |
| `UInstancedStaticMeshComponent::InitInstanceBody` | `InstancePhysicsBodies` | `UE_DEPRECATED(5.8)` in `Components/InstancedStaticMeshComponent.h:771` |
| `ApplyPerlinNoiseToMesh` | `ApplyPerlinNoiseToMesh2` | `UE_DEPRECATED(5.7)` in `GeometryScript/MeshDeformFunctions.h:401` |
| `CopyMeshToStaticMesh` without `bUseSectionMaterials` | overload taking `bUseSectionMaterials` | `UE_DEPRECATED(5.5)` in `GeometryScript/MeshAssetFunctions.h:311` |

## Common Mistakes

**Iterating `FPCGPoint` arrays in a custom element:** `GetPoints()` only exists on `UPCGPointData`, so PCG silently converts every input and you pay a full copy. Override `SupportsBasePointDataInputs` to return `true` and read `FConstPCGPointValueRanges`.

**Forgetting `AllocateProperties` before writing:** `FPCGPointValueRanges` built with `bAllocate = false` leaves unallocated ranges empty, so indexing them fails the `checkf` range check (`Utils/PCGValueRange.h:139`). Call `SetNumPoints` then `AllocateProperties` for every property you intend to write.

**`GenerateLocal` in multiplayer:** it is not a network function, so clients never generate. Use `Generate(bool bForce)` (`NetMulticast, Reliable`) and replicate the seed.

**Generating from `Tick`:** PCG generation schedules graph tasks. Use `GenerateOnDemand` and call `Generate` only when inputs change, or `GenerateAtRuntime` and let the scheduler budget it.

**Caching a node that spawns actors:** leaving `IsCacheable` at `true` for a node that creates actors or components produces duplicated or missing artifacts on regeneration. Return `false`.

**Expecting `UpdateMeshSection_LinearColor` to change topology:** it only rewrites existing vertices. Adding or removing triangles requires `CreateMeshSection_LinearColor`.

**Clockwise triangle winding:** front faces are counter-clockwise, so clockwise triangles vanish under back-face culling.

**Calling `AddSplinePoint` with `bUpdateSpline = true` in a loop:** each call rebuilds the whole reparameterization table. Pass `false` and call `UpdateSpline()` once.

**Spacing instances by spline input key:** the key is not proportional to arc length. Step by distance and use `GetTransformAtDistanceAlongSpline`.

**`bMarkRenderStateDirty = true` on every instance update:** each call re-uploads the instance buffer. Leave it `false` in the loop and call `MarkRenderStateDirty()` once.

**Expecting Nanite from `UProceduralMeshComponent`:** it has no Nanite path. Bake to a `UStaticMesh` in the editor with `CopyMeshToStaticMesh` (editor-only) or scatter Nanite static meshes with PCG and instanced components.

## Related Skills

- `ue-actor-component-architecture` — component construction, registration, attachment and lifecycle for the components created here
- `ue-physics-collision` — collision profiles, body setup, complex vs simple collision, traces used to project scatter onto terrain
- `ue-materials-rendering` — material instances, PerInstanceCustomData, Nanite and virtual texturing for generated geometry
- `ue-world-level-streaming` — World Partition, data layers and HLODs that PCG partitioning and runtime generation build on
- `ue-async-threading` — `ParallelFor`, `Async`, task graph and thread-safety rules for background mesh and point computation
- `ue-mass-entity` — large agent populations, an alternative to instanced components for crowds
- `ue-data-assets-tables` — data assets and data tables that drive generation parameters and mesh/prop tables
- `ue-niagara-effects` — Niagara systems, user parameters, data interfaces and data channels
