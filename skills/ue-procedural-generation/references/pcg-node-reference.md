# PCG Node Reference

Target engine: **UE 5.8**. Plugin `PCG` at `Engine/Plugins/PCG`, module `PCG`, headers under `Source/PCG/Public`.

A node is a `UPCGSettings` subclass (data, pins, editor identity) paired with an `IPCGElement` (the const, thread-safe execution body). Data travels between nodes as `UPCGData` subclasses wrapped in `FPCGTaggedData`.

---

## Data flow model

```cpp
// PCGData.h:195
struct FPCGTaggedData
{
    FPCGDataPtrWrapper Data;       // wraps TObjectPtr<const UPCGData>
    TSet<FString>      Tags;
    FName              Pin = NAME_None;
    bool               bPinlessData = false;
    bool               bIsUsedMultipleTimes = true;
    int32              OriginalIndex = INDEX_NONE;
};
```

`FPCGDataCollection` (`PCGData.h:236`) holds `TArray<FPCGTaggedData> TaggedData` and the lookup helpers:

| Call | Returns |
|---|---|
| `GetAllInputs()` | `const TArray<FPCGTaggedData>&`, everything |
| `GetInputs()` | `UE_DEPRECATED(5.6)` (`PCGData.h:241`) — use `GetAllSpatialInputs()` or `GetAllInputs()` |
| `GetInputsByPin(const FName&)` | entries whose `Pin` matches |
| `GetSpatialInputsByPin(const FName&)` | same, filtered to `UPCGSpatialData` |
| `GetAllSpatialInputs()` | every spatial entry |
| `GetTaggedInputs(const FString&)` | entries carrying a tag |
| `GetParamsByPin(const FName&)` / `GetAllParams()` | `UPCGParamData` entries |
| `GetSettings<T>()` | the settings object carried in the collection |

Add data with `AddData(const FPCGTaggedData&, const FPCGCrc&)` or by appending to `Context->OutputData.TaggedData` directly.

---

## Pin labels

`PCGPinConstants` (`PCGCommon.h:350`, values in `Private/PCGCommon.cpp:513`):

| Constant | Value |
|---|---|
| `PCGPinConstants::DefaultInputLabel` | `"In"` |
| `PCGPinConstants::DefaultOutputLabel` | `"Out"` |
| `PCGPinConstants::DefaultParamsLabel` | `"Overrides"` |
| `PCGPinConstants::DefaultExecutionDependencyLabel` | execution-order-only pin |
| `PCGPinConstants::DefaultInFilterLabel` / `DefaultOutFilterLabel` | filter node in/out |

`FPCGPinProperties` (`PCGPin.h:52`):

```cpp
explicit FPCGPinProperties(
    const FName& InLabel,
    FPCGDataTypeIdentifier InAllowedTypes = FPCGDataTypeIdentifier{EPCGDataType::Any},
    bool bInAllowMultipleConnections = true,
    bool bAllowMultipleData = true,
    const FText& InTooltip = FText::GetEmpty());
```

Fields: `Label`, `Usage` (`EPCGPinUsage`), `AllowedTypes` (`FPCGDataTypeIdentifier`), `bAllowMultipleData`, `PinStatus` (`EPCGPinStatus::Normal | Required | Advanced | OverrideOrUserParam`), `bInvisiblePin`. Helpers: `SetRequiredPin()`, `SetAdvancedPin()`, `SetAllowMultipleConnections(bool)`, `AllowsMultipleConnections()`.

`UPCGSettings::DefaultPointInputPinProperties()` and `DefaultPointOutputPinProperties()` give the standard single `In`/`Out` point pins.

---

## Settings categories

`EPCGSettingsType` (`PCGSettings.h:58`) — returned from `GetType()` and used to colour and group nodes:

`InputOutput`, `Spatial`, `Density`, `Blueprint`, `Metadata`, `Filter`, `Sampler`, `Spawner`, `Subgraph`, `Debug`, `Generic`, `Param`, `HierarchicalGeneration`, `ControlFlow`, `PointOps`, `GraphParameters`, `Reroute`, `GPU`, `DynamicMesh`, `DataLayers`, `Resource`.

---

## Node catalogue

Class names below are the real `UPCGSettings` subclasses; use them with `UPCGGraph::AddNodeOfType(TSubclassOf<UPCGSettings>, UPCGSettings*&)`.

### Input / output

| Node | Settings class | Header |
|---|---|---|
| Get Actor Data | `UPCGDataFromActorSettings` | `Elements/PCGDataFromActor.h` |
| Get Actor Property | `UPCGGetActorPropertySettings` | `Elements/PCGGetActorProperty.h` |
| Data Table Row To Attribute Set | `UPCGDataTableRowToParamDataSettings` | `Elements/PCGDataTableRowToParamData.h` |
| Get Spline Control Points | `UPCGGetSplineControlPointsSettings` | `Elements/PCGGetSplineControlPoints.h` |

`UPCGDataFromActorSettings` key fields: `ActorSelector` (`FPCGActorSelectorSettings` from `Elements/PCGActorSelector.h`), `Mode` (`EPCGGetDataFromActorMode`), `bAlwaysRequeryActors`, `ExpectedPins`.

### Samplers (`EPCGSettingsType::Sampler`)

**Surface Sampler** — `UPCGSurfaceSamplerSettings` (`Elements/PCGSurfaceSampler.h`)
`PointsPerSquaredMeter` (default 0.1), `PointExtents` (default `FVector(50)`), `Looseness` (default 1.0), `bUnbounded`, `bApplyDensityToPoints`, `PointSteepness`, `bUseLegacyGridCreationMethod`.

**Spline Sampler** — `UPCGSplineSamplerSettings` (`Elements/PCGSplineSampler.h`)
`Dimension` (`EPCGSplineSamplingDimension::OnSpline`), `Mode` (`EPCGSplineSamplingMode::Subdivision`), `Fill`, `SubdivisionsPerSegment`, `DistanceIncrement`, `NumSamples`, `NumPlanarSubdivisions`, `NumHeightSubdivisions`, `StartOffset`, `EndOffset`, `bFitToCurve`, `InteriorSampleSpacing`, `InteriorBorderSampleSpacing`, `InteriorOrientation`, `bProjectOntoSurface`, plus opt-in outputs `bComputeTangents`, `bComputeCurvature`, `bComputeAlpha`, `bComputeDistance`, `bComputeInputKey`, `bComputeSegmentIndex`, `bComputeDirectionDelta`. Seeding: `SeedingMode` (`EPCGSplineSamplingSeedingMode`), `bSeedFromLocalPosition`, `bSeedFrom2DPosition`.

**Volume Sampler** — `UPCGVolumeSamplerSettings` (`Elements/PCGVolumeSampler.h`)
`VoxelSize`, `bUnbounded`, `PointSteepness`.

**Create Points Grid** — `UPCGCreatePointsGridSettings` (`Elements/PCGCreatePointsGrid.h`)
`GridExtents`, `CellSize`, `PointSteepness`, `CoordinateSpace` (`EPCGCoordinateSpace`), `PointPosition` (`EPCGPointPosition`), `bSetPointsBounds`, `bCullPointsOutsideVolume`.

**Create Points Sphere** — `UPCGCreatePointsSphereSettings` (`Elements/PCGCreatePointsSphere.h`)
`SphereGeneration` (`EPCGSphereGeneration::Geodesic`), `PointOrientation`, `Origin`, `Radius`, `GeodesicSubdivisions`, `Theta`/`Phi`, `LatitudinalSegments`/`LongitudinalSegments`, `SampleCount`, `PoissonDistance`, `PoissonMaxAttempts`, `Jitter`, `PointLimit`.

Also available: `UPCGTextureSamplerSettings`, `UPCGSampleTextureSettings`, `UPCGWorldRayHitSettings`-family in `Elements/PCGWorldQuery.h` and `Elements/PCGWorldRaycast.h`.

### Filters (`EPCGSettingsType::Filter`)

| Node | Settings class | Key fields |
|---|---|---|
| Density Filter | `UPCGDensityFilterSettings` | `LowerBound`, `UpperBound`, `bInvertFilter`, `bNormalizeOutputDensity` |
| Attribute Filter | `UPCGAttributeFilteringSettings` | `Operator` (`EPCGAttributeFilterOperator`), `TargetAttribute`, `bUseConstantThreshold`, `ThresholdAttribute`, `AttributeTypes` |
| Attribute Range Filter | `UPCGAttributeFilteringRangeSettings` | `bInclusive`, min/max threshold selectors |
| Filter By Attribute | `UPCGFilterByAttributeSettings` | attribute-name based data filtering |
| Filter By Tag | `UPCGFilterByTagSettings` | tag-based data filtering |
| Filter By Type | `UPCGFilterByTypeSettings` | data-type based filtering |
| Filter By Index | `UPCGFilterByIndexSettings` / `UPCGFilterElementsByIndexSettings` | index expressions |
| Cull Points Outside Actor Bounds | `UPCGCullPointsOutsideActorBoundsSettings` | no required configuration |

Attribute filters and the Filter By Attribute/Tag/Type/Index nodes emit on `PCGPinConstants::DefaultInFilterLabel` and `DefaultOutFilterLabel` (Density Filter has a single `Out` pin, `Elements/PCGDensityFilter.h:25`); the data-filter base is `UPCGFilterDataBaseSettings` (`Elements/PCGFilterDataBase.h`).

### Density and attribute math

| Node | Settings class | Key fields |
|---|---|---|
| Attribute Noise | `UPCGAttributeNoiseSettings` | `InputSource`, `OutputTarget`, `Mode` (`EPCGAttributeNoiseMode`), `NoiseMin`, `NoiseMax`, `bInvertSource`, `bClampResult`, `bHasCustomSeedSource`, `CustomSeedSource` |
| Density Remap | `UPCGDensityRemapSettings` — `UE_DEPRECATED(5.5)` (`Elements/PCGDensityRemapElement.h:11`), use Attribute Remap | `InRangeMin`, `InRangeMax`, `OutRangeMin`, `OutRangeMax`, `bExcludeValuesOutsideInputRange` |
| Attribute Remap | `UPCGAttributeRemapSettings` | `Elements/Metadata/PCGAttributeRemap.h`; same `InRangeMin`…`OutRangeMax` fields on any attribute |
| Blur | `UPCGBlurSettings` | `InputSource`, `OutputTarget`, `NumIterations`, `SearchDistance`, `BlurMode` (`EPCGBlurElementMode`), `bUseCustomStandardDeviation`, `CustomStandardDeviation` |
| Spatial Noise | `UPCGSpatialNoiseSettings` | `Elements/PCGSpatialNoise.h`, type `EPCGSettingsType::Spatial` |
| Normal To Density | `UPCGNormalToDensitySettings` | `Elements/PCGNormalToDensity.h` |

### Spawners (`EPCGSettingsType::Spawner`)

**Static Mesh Spawner** — `UPCGStaticMeshSpawnerSettings` (`Elements/PCGStaticMeshSpawner.h`)

```cpp
TSubclassOf<UPCGMeshSelectorBase>       MeshSelectorType;
TObjectPtr<UPCGMeshSelectorBase>        MeshSelectorParameters;
TSubclassOf<UPCGInstanceDataPackerBase> InstanceDataPackerType;
TObjectPtr<UPCGInstanceDataPackerBase>  InstanceDataPackerParameters;
TArray<FPCGObjectPropertyOverrideDescription> StaticMeshComponentPropertyOverrides;
FName OutAttributeName;
bool  bApplyMeshBoundsToPoints;
bool  bAllowDescriptorChanges;
bool  bAllowMergeDifferentDataInSameInstancedComponents;
bool  bSynchronousLoad;
bool  bSetupCullingCells;
TSoftObjectPtr<AActor> TargetActor;
TArray<FName> PostProcessFunctionNames;
```

Mesh selectors (`MeshSelectors/`): `UPCGMeshSelectorWeighted`, `UPCGMeshSelectorWeightedByCategory`, `UPCGMeshSelectorByAttribute`, `UPCGMeshSelectorPrimitiveData`. `UPCGMeshSelectorWeighted::MeshEntries` is a `TArray<FPCGMeshSelectorWeightedEntry>`, each holding a `FPCGSoftISMComponentDescriptor Descriptor` and an `int Weight`.

Instance data packers (`InstanceDataPackers/`): `UPCGInstanceDataPackerByAttribute`, `UPCGInstanceDataPackerByRegex` — these fill the per-instance custom float channels read by materials.

**Spawn Actor** — `UPCGSpawnActorSettings` (`Elements/PCGSpawnActor.h`)
`TemplateActor`, `Option` (`EPCGSpawnActorOption::CollapseActors`), `GenerationTrigger` (`EPCGSpawnActorGenerationTrigger`), `AttachOptions` (`EPCGAttachOptions`), `RootActor`, `bSpawnByAttribute` + `SpawnAttribute`, `bInheritActorTags`, `TagsToAddOnActors`, `PostSpawnFunctionNames`, `SpawnedActorPropertyOverrideDescriptions`, `bForceDisableActorParsing`.

**Create Spline** — `UPCGCreateSplineSettings` (`Elements/PCGCreateSpline.h`)
`Mode` (`EPCGCreateSplineMode::CreateDataOnly`), `bClosedLoop`, `bLinear`, `bApplyCustomTangents` + `ArriveTangentAttribute`/`LeaveTangentAttribute`, `bUseInterpTypeAttribute` + `InterpTypeAttribute`, `TargetActor`, `TagsToAddOnComponents`.

Related: `UPCGSpawnSplineSettings`, `UPCGSpawnSplineMeshSettings` (`Elements/PCGSpawnSpline.h`, `Elements/PCGSpawnSplineMesh.h` with `FPCGSplineMeshParams`), `UPCGSkinnedMeshSpawnerSettings`.

### Metadata (`EPCGSettingsType::Metadata`)

| Node | Settings class | Key fields |
|---|---|---|
| Add Attribute | `UPCGAddAttributeSettings` | `InputSource`, `OutputTarget`, `bCopyAllAttributes`, `bCopyAllDomains`, `MetadataDomainsMapping` |
| Create Attribute Set | `UPCGCreateAttributeSetSettings` | `AttributeTypes` (`FPCGMetadataTypesConstantStruct`), `OutputTarget` |
| Copy Attributes | `UPCGCopyAttributesSettings` | `Elements/PCGCopyAttributes.h` (the `PCGMetadataElement.h` alias `UPCGMetadataOperationSettings` is `UE_DEPRECATED(5.5)`) |
| Attribute Cast | `UPCGAttributeCastSettings` | target type conversion |
| Maths Op | `UPCGMetadataMathsSettings` | `Operation` (`EPCGMetadataMathsOperation`), `InputSource1..3`, `bForceRoundingOpToInt`, `bForceOpToDouble` |
| Compare / Boolean / Bitwise | `UPCGMetadataCompareSettings`, `UPCGMetadataBooleanSettings`, `UPCGMetadataBitwiseSettings` | `Elements/Metadata/` |
| Break / Make Transform | `UPCGMetadataBreakTransformSettings`, `UPCGMetadataMakeTransformSettings` | transform decomposition |
| Break / Make Vector, Make Rotator | `UPCGMetadataBreakVectorSettings`, `UPCGMetadataMakeVectorSettings`, `UPCGMetadataMakeRotatorSettings` | component packing |
| Rename / Delete / Hash attribute | `UPCGMetadataRenameSettings`, `UPCGDeleteAttributesSettings`, `UPCGHashAttributeSettings` | attribute housekeeping |
| Partition | `UPCGMetadataPartitionSettings` | split data by attribute value |

`EPCGMetadataMathsOperation` (`Elements/Metadata/PCGMetadataMathsOpElement.h:10`) — unary: `Sign`, `Frac`, `Truncate`, `Round`, `Sqrt`, `Abs`, `Floor`, `Ceil`, `OneMinus`, `Inc`, `Dec`, `Negate`; binary: `Add`, `Subtract`, `Multiply`, `Divide`, `Max`, `Min`, `Pow`, `ClampMin`, `ClampMax`, `Modulo`, `Set`; ternary: `Clamp`, `Lerp`, `MulAdd`, `AddModulo`.

Attribute selection uses `FPCGAttributePropertyInputSelector` / `FPCGAttributePropertyOutputSelector` (`Metadata/PCGAttributePropertySelector.h`), which address either a native point property or a named metadata attribute.

### Control flow (`EPCGSettingsType::ControlFlow`)

| Node | Settings class | Key fields |
|---|---|---|
| Branch | `UPCGBranchSettings` | `bOutputToB` |
| Switch | `UPCGSwitchSettings` | `SelectionMode` (`EPCGControlFlowSelectionMode::Integer`), `IntegerSelection`, `IntOptions`, `StringSelection`, `StringOptions`, `EnumSelection` |
| Boolean Select | `UPCGBooleanSelectSettings` | picks between two data inputs |
| Quality Branch | `UPCGQualityBranchSettings` | branches on scalability level |
| Wait | `UPCGWaitSettings` | forces ordering on upstream tasks |
| Loop / Subgraph | `UPCGLoopSettings`, `UPCGSubgraphSettings` | `SubgraphInstance` (`UPCGGraphInstance`), `SubgraphOverride` |

### Point ops (`EPCGSettingsType::PointOps`)

| Node | Settings class | Key fields |
|---|---|---|
| Copy Points | `UPCGCopyPointsSettings` | `RotationInheritance`/`ScaleInheritance`/`ColorInheritance`/`SeedInheritance` (`EPCGCopyPointsInheritanceMode`), `AttributeInheritance`, `TagInheritance`, `bCopyEachSourceOnEveryTarget`, `bMatchBasedOnAttribute` + `MatchAttribute` |
| Combine Points | `UPCGCombinePointsSettings` | merge point datasets |
| Collapse Points | `UPCGCollapsePointsSettings` | merge nearby points |
| Attract | `UPCGAttractSettings` | `Mode` (`EPCGAttractMode::Closest`), `Distance`, `Weight`, `bRemoveUnattractedPoints`, `SourceAndTargetAttributeMapping`, `bOutputAttractIndex` |
| Bounds Modifier | `UPCGBoundsModifierSettings` | `Mode` (`EPCGBoundsModifierMode::Scale`), `BoundsMin`, `BoundsMax`, `bAffectSteepness`, `Steepness` |
| Apply Scale To Bounds | `UPCGApplyScaleToBoundsSettings` | folds point scale into bounds |
| Transform Points | `UPCGTransformPointsSettings` | offsets, rotation and scale ranges |
| Align Points | `UPCGAlignPointsSettings` | orient points to a direction or attribute |
| Self Pruning | `UPCGSelfPruningSettings` | overlap-based thinning |
| Point Neighborhood | `UPCGPointNeighborhoodSettings` | neighbour queries |

### Grammar (`Elements/Grammar/`)

`UPCGSplineToSegmentSettings`, `UPCGSubdivideSplineSettings`, `UPCGSubdivideSegmentSettings`, `UPCGSelectGrammarSettings`, `UPCGDuplicateCrossSectionsSettings` — spline-driven modular assembly (fences, buildings, road furniture).

### Blueprint nodes (`EPCGSettingsType::Blueprint`)

Derive from `UPCGBlueprintBaseElement` (`Elements/Blueprint/PCGBlueprintBaseElement.h`); the point-processing conveniences are `UPCGBlueprintPointProcessorElement` and `UPCGBlueprintPointProcessorSimpleElement`.

```cpp
// PCGBlueprintBaseElement.h:47
UFUNCTION(BlueprintNativeEvent, BlueprintCallable, Category = "PCG|Execution")
void Execute(const FPCGDataCollection& Input, FPCGDataCollection& Output);
```

| Member | Purpose |
|---|---|
| `bIsCacheable` (default `false`) | set `true` only when output depends purely on inputs and seed |
| `bComputeFullDataCrc` (default `false`) | deep CRC so downstream nodes can still hit the cache |
| `bRequiresGameThread` (default `true`) | set `false` when the node touches no components or actors |
| `CustomInputPins` / `CustomOutputPins` | extra `FPCGPinProperties` |
| `bHasDefaultInPin` | keep or drop the standard `In` pin |
| `GetContextHandle()` | `FPCGBlueprintContextHandle` for the seed helpers |
| `GetSeedWithContext(Handle)` / `GetRandomStreamWithContext(Handle)` | deterministic per-node randomness |
| `NodeTitleOverride()`, `NodeColorOverride()`, `NodeTypeOverride()`, `IsCacheableOverride()`, `DynamicPinTypesOverride()` | BlueprintNativeEvent customisation hooks |

---

## Element execution contract

```cpp
// PCGElement.h
virtual FPCGContext* Initialize(const FPCGInitializeElementParams& InParams);   // :145
virtual bool CanExecuteOnlyOnMainThread(FPCGContext* Context) const;            // :151
virtual bool IsCacheable(const UPCGSettings* InSettings) const;                 // :157
virtual bool PrepareDataInternal(FPCGContext* Context) const;                   // :214
virtual bool ExecuteInternal(FPCGContext* Context) const = 0;                   // :216
virtual void PostExecuteInternal(FPCGContext* Context) const;                   // :220
virtual void AbortInternal(FPCGContext* Context) const;                         // :222
virtual bool IsPassthrough(const UPCGSettings* InSettings) const;               // :227
virtual EPCGElementExecutionLoopMode ExecutionLoopMode(const UPCGSettings* Settings) const; // :233
virtual FPCGContext* CreateContext();                                           // :242
virtual bool SupportsGPUResidentData(FPCGContext* InContext) const;             // :245
virtual bool SupportsBasePointDataInputs(FPCGContext* InContext) const;         // :248
```

`FPCGInitializeElementParams` (`PCGElement.h:106`) carries `const FPCGDataCollection* InputData`, `TWeakInterfacePtr<IPCGGraphExecutionSource> ExecutionSource`, `const UPCGNode* Node`.

`EPCGElementExecutionLoopMode` (`PCGElement.h:73`): `NotALoop`, `SinglePrimaryPin`, `MatchingPrimaryPins`, `PrimaryPinAndBroadcastablePins`.

`ExecuteInternal` returning `false` means "not finished" — PCG will call it again next frame. Use that with `FPCGContext::ShouldStop()` for time-sliced work, or use `FPCGAsync::AsyncProcessingRangeEx` / `AsyncProcessingOneToOneRangeEx` (`Helpers/PCGAsync.h`).

For an element that needs extra per-execution state, derive a struct from `FPCGContext` and inherit from `IPCGElementWithCustomContext<FMyContext>` (`PCGElement.h:265`).

---

## Point data API

`UPCGBasePointData` (`Data/PCGBasePointData.h:62`) is abstract. `UPCGPointArrayData` stores properties as parallel arrays; `UPCGPointData` stores `TArray<FPCGPoint>`.

```cpp
int32 GetNumPoints() const;
void  SetNumPoints(int32 InNumPoints, bool bInitializeValues = true);
void  AllocateProperties(EPCGPointNativeProperties Properties);
void  FreeProperties(EPCGPointNativeProperties Properties);
EPCGPointNativeProperties GetAllocatedProperties(bool bWithInheritance = true) const;
void  CopyUnallocatedPropertiesFrom(const UPCGBasePointData* InPointData);
void  CopyPropertiesTo(UPCGBasePointData* To, int32 ReadStartIndex, int32 WriteStartIndex,
                       int32 Count, EPCGPointNativeProperties Properties) const;
void  MoveRange(int32 RangeStartIndex, int32 MoveToIndex, int32 NumElements);
void  SetExtents(const FVector& InExtents);
const PCGPointOctree::FPointOctree& GetPointOctree() const;
```

Value ranges (`Utils/PCGValueRange.h`, wrappers at `Data/PCGBasePointData.h:658,696`):

```cpp
FConstPCGPointValueRanges Read(InputPoints);
// TransformRange, DensityRange, SteepnessRange, BoundsMinRange, BoundsMaxRange,
// ColorRange, SeedRange, MetadataEntryRange
FPCGPointValueRanges Write(OutputPoints, /*bAllocate=*/true);
Write.SetFromValueRanges(WriteIndex, Read, ReadIndex);
Write.SetFromPoint(WriteIndex, SomePoint);   // interop with FPCGPoint
FPCGPoint Point = Read.GetPoint(ReadIndex);  // interop the other way
```

`FPCGPoint` (`PCGPoint.h`) remains the per-point value type: `Transform`, `Density`, `BoundsMin`, `BoundsMax`, `Color`, `Steepness`, `Seed`, `MetadataEntry`.

Creating output data inside an element:

```cpp
UPCGBasePointData* Out = FPCGContext::NewPointData_AnyThread(Context);
Out->InitializeFromDataWithParams(FPCGInitializeFromDataParams(In));
```

`FPCGInitializeFromDataParams` (`Data/PCGSpatialData.h:34`) exposes `bInheritSpatialData`, metadata inheritance and attribute-filtering switches — set `bInheritSpatialData = false` when the output point count differs from the input.

---

## Metadata and attributes

`UPCGMetadata` (`Metadata/PCGMetadata.h`):

```cpp
template<typename T>
FPCGMetadataAttribute<T>* CreateAttribute(FPCGAttributeIdentifier AttributeName, const T& DefaultValue,
                                          bool bAllowsInterpolation, bool bOverrideParent);
template<typename T>
FPCGMetadataAttribute<T>* FindOrCreateAttribute(FPCGAttributeIdentifier AttributeName, const T& DefaultValue = T{},
                                                bool bAllowsInterpolation = true, bool bOverrideParent = true,
                                                bool bOverwriteIfTypeMismatch = true);
FPCGMetadataAttributeBase*       GetMutableAttribute(FPCGAttributeIdentifier AttributeName);
const FPCGMetadataAttributeBase* GetConstAttribute(FPCGAttributeIdentifier AttributeName) const;
bool  HasAttribute(FPCGAttributeIdentifier AttributeName) const;
void  DeleteAttribute(FPCGAttributeIdentifier AttributeName);
int64 AddEntry(int64 ParentEntryKey = -1);
int64 AddEntryPlaceholder();
```

`AddEntryPlaceholder()` plus a single batched commit is the multithreaded-safe way to add entries; `AddEntry` reorders indices and must not run concurrently.

Generic reading and writing goes through accessors (`Metadata/Accessors/PCGAttributeAccessorHelpers.h`):

```cpp
TUniquePtr<const IPCGAttributeAccessor> Accessor =
    PCGAttributeAccessorHelpers::CreateConstAccessor(Data, Selector);
TUniquePtr<const IPCGAttributeAccessorKeys> Keys =
    PCGAttributeAccessorHelpers::CreateConstKeys(Data, Selector);
```

`CreateAccessorWithAttributeCreation(const FPCGCreateAccessorWithAttributeCreationParams&)` creates the attribute on demand when writing.

`UPCGParamData` (`PCGParamData.h`) is the attribute-set data type; its default metadata domain is `PCGMetadataDomainID::Elements`, and `FindOrAddMetadataKey(FName)` maps a name to an entry key.

---

## Graph parameters

`UPCGGraph::UserParameters` is an `FInstancedPropertyBag`. Read and write through `UPCGGraphInterface`:

```cpp
template<typename T> TValueOrError<T, EPropertyBagResult> GetGraphParameter(const FName PropertyName) const;
template<typename T> EPropertyBagResult SetGraphParameter(const FName PropertyName, const T& Value);
EPropertyBagResult SetGraphParameter(const FName PropertyName, const uint64 Value, const UEnum* Enum);
bool UpdateSetGraphParameter(const FName PropertyName, TFunctionRef<bool(FPropertyBagSetRef&)> Callback);
```

Per-instance overrides on `UPCGGraphInstance` use `FPCGOverrideInstancedPropertyBag` (`PCGGraph.h:82`):

```cpp
bool UpdatePropertyOverride(const FProperty* InProperty, bool bMarkAsOverridden, const FInstancedPropertyBag* ParentUserParameters);
bool ResetPropertyToDefault(const FProperty* InProperty, const FInstancedPropertyBag* ParentUserParameters);
bool IsPropertyOverridden(const FProperty* InProperty) const;
```

`EPCGGraphParameterEvent` (`PCGGraph.h:39`): `GraphChanged`, `GraphPostLoad`, `Added`, `RemovedUnused`, `RemovedUsed`, `PropertyMoved`, `PropertyRenamed`, `PropertyTypeModified`, `ValueModifiedLocally`, `ValueModifiedByParent`, `MultiplePropertiesAdded`, `UndoRedo`, `CategoryChanged`, `MetadataModified`, `None`.

Node-level overrides come in on the `Overrides` pin: mark a `UPROPERTY` with `meta = (PCG_Overridable)` and PCG wires it automatically.

---

## Component generation

`EPCGComponentGenerationTrigger` (`PCGComponent.h:77`):

| Value | Behaviour |
|---|---|
| `GenerateOnLoad` | generates when the component is loaded into the level |
| `GenerateOnDemand` | generates only when requested (`Generate` / `GenerateLocal`) |
| `GenerateAtRuntime` | generates when the Runtime Generation Scheduler says so |

`EPCGComponentInput` (`PCGComponent.h:67`): `Actor`, `Landscape`, `Other`.
`EPCGComponentDirtyFlag` (`PCGComponent.h:85`, bitflags): `None`, `Actor`, `Landscape`, `Input`, `Data`, `All`.

Runtime generation extras on `UPCGComponent`: `SetSchedulingPolicyClass(TSubclassOf<UPCGSchedulingPolicyBase>)`, `GetGenerationRadii()`, `GetGenerationRadiusFromGrid(uint32)`, `GetCleanupRadiusFromGrid(uint32)`, `UseActorComponentlessGeneration()`, `AreProceduralInstancesInUse()`.

Managed output: `AddToManagedResources(UPCGManagedResource*)`, `AddComponentsToManagedResources(const TArray<UActorComponent*>&)`, `AddActorsToManagedResources(const TArray<AActor*>&)`, and `ClearPCGLink(UClass* TemplateActor)` to hand generated content to a standalone actor.

---

## Hierarchical generation

On `UPCGGraph`:

```cpp
bool           bUseHierarchicalGeneration = false;
EPCGHiGenGrid  HiGenGridSize = EPCGHiGenGrid::Grid256;
double         HiGenGridSizeMultiplier = 1.0;
bool           bUse2DGrid = true;
```

`EPCGHiGenGrid` (`PCGCommon.h:520`) values: `Grid4`, `Grid8`, `Grid16`, `Grid32`, `Grid64`, `Grid128`, `Grid256`, `Grid512`, `Grid1024`, `Grid2048` (the editor display name is the grid size in centimetres, i.e. the enum value × 100), plus `Uninitialized`, `Unbounded` and larger hidden grids. `UPCGHiGenGridSizeSettings` (`Elements/PCGHiGenGridSize.h`) forces a subtree to execute at a chosen resolution; a node runs at the smallest grid among its inputs.

Grid descriptors are `FPCGGridDescriptor` (`Grid/PCGGridDescriptor.h`); partition actors live in `Grid/PCGPartitionActor.h`.

---

## GPU nodes (`EPCGSettingsType::GPU`)

`UPCGSettings` exposes `bExecuteOnGPU` with `ShouldExecuteOnGPU()` and `SetExecuteOnGPU(bool)`; a settings class opts into the checkbox by overriding `DisplayExecuteOnGPUSetting()`.

```cpp
// PCGSettings.h:656 (the overload without InNode at :653 is UE_DEPRECATED(5.8))
virtual void CreateKernels(FPCGGPUCompilationContext& InOutContext, UObject* InObjectOuter,
                           const UPCGNode* InNode, TArray<UPCGComputeKernel*>& OutKernels,
                           TArray<FPCGKernelEdge>& OutEdges) const;
```

Kernel classes derive from `UPCGComputeKernel` (`Compute/PCGComputeKernel.h:101`) and implement `ComputeThreadCount(const UPCGDataBinding*)`, `ComputeOutputBindingDataDesc(...)`, `IsKernelDataValid(const UPCGDataBinding*, FPCGContext*)` and `GetKernelAttributeKeys(TArray<FPCGKernelAttributeKey>&)`. Shader text is supplied by `UPCGComputeSource` (`Compute/PCGComputeSource.h`).

The runtime data plumbing is `UPCGDataBinding` (`Compute/PCGDataBinding.h:78`) and `FPCGDataCollectionDesc` (`Compute/PCGDataDescription.h:249`), with GPU pin descriptions in `Compute/PCGPinPropertiesGPU.h`. Shipped GPU kernels include `UPCGStaticMeshSpawnerKernel`, `UPCGCopyPointsKernel` and `UPCGMetadataPartitionKernel`.

`UPCGDownloadFromGPUSettings` (`Elements/PCGDownloadFromGPU.h`) reads GPU-resident data back to the CPU; an element that can consume GPU-resident data directly overrides `IPCGElement::SupportsGPUResidentData`.

---

## Determinism checklist

1. Set `UPCGComponent::Seed` explicitly; the default is `42` and every node derives from it plus its own settings seed.
2. Inside an element, take randomness from `FPCGContext::GetSeed()` or from the per-point seed (`ReadRanges.SeedRange[Index]`), never from `FMath::Rand`.
3. In Blueprint nodes use `GetRandomStreamWithContext(GetContextHandle())`.
4. `IsCacheable` may only return `true` when output depends solely on inputs plus seed — otherwise the cache will serve stale artifacts.
5. Sort gathered inputs before consuming them; actor-gathering order is not guaranteed stable across runs or platforms.
6. Replicate the seed and use `Generate(bool bForce)` for multiplayer; `GenerateLocal` is not a network function.
