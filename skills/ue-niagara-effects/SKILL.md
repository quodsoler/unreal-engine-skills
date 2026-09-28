---
name: ue-niagara-effects
description: "Use when driving Niagara VFX from C++: spawning a system, overriding User parameters, binding data interfaces, sending gameplay data through Niagara Data Channels, pooling components or handling OnSystemFinished. Also use when the user mentions 'Niagara', 'particle system', 'VFX', 'UNiagaraComponent', 'UNiagaraFunctionLibrary', 'SpawnSystemAtLocation', 'SpawnSystemAttached', 'SetVariableFloat', 'User. parameter', 'data interface', 'ENCPoolMethod', 'sim cache', 'lightweight emitter', 'muzzle flash' or 'impact effect'. For particle materials and dynamic material instances, see ue-materials-rendering; for audio-reactive effects, see ue-audio-system."
metadata:
  version: "2.0.0"
  engine: "5.8"
---

# UE Niagara Effects

Target engine: **UE 5.8**. APIs below are verified against the 5.8 headers; older forms are listed under "Deprecated — do not use".

Niagara ships as the `Niagara` plugin (`Engine/Plugins/FX/Niagara`, enabled by default). Game code spawns and drives systems through `UNiagaraFunctionLibrary` and `UNiagaraComponent`, feeds them structured data through data interfaces and Niagara Data Channels, and controls cost through component pooling and `UNiagaraEffectType` scalability. Add `"Niagara"` to your module's `PublicDependencyModuleNames`, plus `"NiagaraCore"` when you subclass `UNiagaraDataInterface`.

## Context

Read `.agents/ue-project-context.md` if it exists (module names, conventions, enabled plugins, GAS/networking setup). Do not stop if it is missing.

Identify the area from the request and the codebase. Ask only when two plausible readings would produce different code.

| Request is about… | Go to |
|---|---|
| Asset, emitter and component relationships | [System and Component Model](#system-and-component-model) |
| Firing an effect, attaching a trail | [Spawning Systems](#spawning-systems) |
| Activate, reset, pause, seek | [Lifecycle Control](#lifecycle-control) |
| Pushing floats, colors, vectors into an effect | [Setting User Parameters](#setting-user-parameters) |
| Skeletal/static mesh, curve, array, texture DIs | [Data Interfaces](#data-interfaces) |
| Exposing game-specific data to Niagara scripts | [Custom Data Interfaces](#custom-data-interfaces) |
| Gameplay events into Niagara, VFX-to-VFX messaging | [Niagara Data Channels](#niagara-data-channels) |
| Knowing when an effect is done | [Completion Callbacks](#completion-callbacks) |
| Recording and replaying a simulation | [Simulation Caches](#simulation-caches) |
| Cheap high-count emitters | [Lightweight Emitters](#lightweight-emitters) |
| Spawn cost, reuse, priming | [Component Pooling](#component-pooling) |
| Culling, tick order, determinism | [Scalability and Tick Behavior](#scalability-and-tick-behavior) |
| Build.cs and includes | [Build Setup](#build-setup) |

## System and Component Model

```
UNiagaraSystem                    asset; holds TArray<FNiagaraEmitterHandle>
  └── FNiagaraEmitterHandle       GetEmitterBase() -> UNiagaraEmitterBase
        ├── UNiagaraEmitter       ENiagaraEmitterMode::Standard (stateful, scripted)
        └── UNiagaraStatelessEmitter  ENiagaraEmitterMode::Stateless (lightweight)
UNiagaraComponent                 runtime instance; GetSystemInstanceController()
```

Authors expose parameters to C++ by placing them in the `User.` namespace in the Niagara editor. Only `User.*` parameters can be overridden at runtime; `System.*`, `Emitter.*`, `Particle.*` and `Module.*` are simulation-internal.

Discover what an asset exposes without opening the editor:

```cpp
#include "NiagaraFunctionLibrary.h"

// FNiagaraUserParameterInfo: ParameterName, ParameterType, TypeName.
const TArray<FNiagaraUserParameterInfo> Params = UNiagaraFunctionLibrary::GetAllUserParameters(ImpactSystem);
// FNiagaraMinimalEmitterInfo: EmitterName, bIsEnabled, bIsLightweight, UsedRendererMaterials, UsedRendererMeshes.
const TArray<FNiagaraMinimalEmitterInfo> Emitters = UNiagaraFunctionLibrary::GetAllEmitters(ImpactSystem);
```

## Spawning Systems

```cpp
#include "NiagaraComponent.h"
#include "NiagaraFunctionLibrary.h"

// One-shot at a world location. Returns nullptr on dedicated servers and when the
// pre-cull check rejects the spawn, so always null-check.
UNiagaraComponent* ImpactVFX = UNiagaraFunctionLibrary::SpawnSystemAtLocation(
    this,                                            // const UObject* WorldContextObject
    ImpactSystem,                                    // UNiagaraSystem*
    HitLocation,                                     // FVector Location
    FRotator::ZeroRotator,                           // FRotator Rotation
    FVector(1.0f),                                   // FVector Scale
    /*bAutoDestroy=*/ true,
    /*bAutoActivate=*/ true,
    ENCPoolMethod::AutoRelease,
    /*bPreCullCheck=*/ true);

if (ImpactVFX)
{
    ImpactVFX->SetVariableVec3(FName("User.HitNormal"), HitNormal);
    ImpactVFX->SetVariableLinearColor(FName("User.HitColor"), DamageColor);
}
```

Attached, persistent effect. Note the argument order: `bAutoDestroy` comes before `bAutoActivate` here.

```cpp
UNiagaraComponent* TrailVFX = UNiagaraFunctionLibrary::SpawnSystemAttached(
    TrailSystem,
    WeaponMesh,                                      // USceneComponent* AttachToComponent
    FName("MuzzleSocket"),                           // FName AttachPointName
    FVector::ZeroVector,
    FRotator::ZeroRotator,
    EAttachLocation::SnapToTarget,
    /*bAutoDestroy=*/ false,
    /*bAutoActivate=*/ true,
    ENCPoolMethod::ManualRelease,
    /*bPreCullCheck=*/ true);
```

A second `SpawnSystemAttached` overload inserts `FVector Scale` after `Rotation` and then orders the tail as `LocationType, bAutoDestroy, PoolingMethod, bAutoActivate, bPreCullCheck`. Pick one overload deliberately; the two are easy to confuse.

`SpawnSystemAtLocationWithParams` and `SpawnSystemAttachedWithParams` take a single `FFXSystemSpawnParameters` struct (`Particles/ParticleSystemComponent.h`) when you want named fields instead of a long argument list. That struct carries `EPSCPoolMethod PoolingMethod`, not `ENCPoolMethod`.

For an effect that repeats on one actor, own the component instead of spawning per event:

```cpp
// Header:
UPROPERTY(VisibleAnywhere, Category = "VFX")
TObjectPtr<UNiagaraComponent> EngineTrailVFX;

// Constructor:
EngineTrailVFX = CreateDefaultSubobject<UNiagaraComponent>(TEXT("EngineTrailVFX"));
EngineTrailVFX->SetupAttachment(GetRootComponent());
EngineTrailVFX->SetAutoActivate(false);

// Gameplay. SetAsset resets existing override parameters unless you opt out,
// so re-apply User parameters after a swap or pass false.
EngineTrailVFX->SetAsset(BoostTrailSystem, /*bResetExistingOverrideParameters=*/ true);
EngineTrailVFX->Activate(/*bReset=*/ true);
```

## Lifecycle Control

```cpp
NiagaraComp->Activate(/*bReset=*/ false);   // resume
NiagaraComp->Activate(/*bReset=*/ true);    // restart from scratch
NiagaraComp->Deactivate();                  // stop spawning, let particles drain
NiagaraComp->DeactivateImmediate();         // kill particles now
NiagaraComp->ResetSystem();                 // restart from time 0
NiagaraComp->ReinitializeSystem();          // full re-init; more expensive than ResetSystem
NiagaraComp->SetPaused(true);
NiagaraComp->SetAutoDestroy(true);

// Warm-up: advance age before the effect becomes visible.
NiagaraComp->SetDesiredAge(2.5f);
NiagaraComp->SeekToDesiredAge(2.5f);

// Deterministic stepping for cutscenes and tests.
NiagaraComp->AdvanceSimulation(/*TickCount=*/ 30, /*TickDeltaSeconds=*/ 0.0166f);
```

## Setting User Parameters

Setters take an `FName` including the namespace prefix. They are silent no-ops when the name or the type does not match the asset.

```cpp
NiagaraComp->SetVariableFloat(FName("User.DamageAmount"), 150.0f);
NiagaraComp->SetVariableInt(FName("User.ProjectileCount"), 12);
NiagaraComp->SetVariableBool(FName("User.bIsCritical"), bIsCriticalHit);
NiagaraComp->SetVariableVec2(FName("User.UVOffset"), FVector2D(0.5, 0.25));
NiagaraComp->SetVariableVec3(FName("User.TargetDirection"), AimDirection);
NiagaraComp->SetVariableVec4(FName("User.CustomData"), FVector4(1.0, 0.5, 0.0, 1.0));
NiagaraComp->SetVariableLinearColor(FName("User.TintColor"), FLinearColor::Red);
NiagaraComp->SetVariableQuat(FName("User.Orientation"), GetActorQuat());
NiagaraComp->SetVariableMatrix(FName("User.LocalToTarget"), TargetMatrix);

// Positions go through the LWC-aware setter, not SetVariableVec3.
NiagaraComp->SetVariablePosition(FName("User.WorldOrigin"), WorldSpaceOrigin);

// Object-valued parameters.
NiagaraComp->SetVariableObject(FName("User.TargetMesh"), TargetMeshComponent);
NiagaraComp->SetVariableActor(FName("User.SourceActor"), this);
NiagaraComp->SetVariableMaterial(FName("User.FXMaterial"), DynamicMaterial);
NiagaraComp->SetVariableStaticMesh(FName("User.ScatterMesh"), ScatterMesh);
NiagaraComp->SetVariableTexture(FName("User.FlowMap"), FlowTexture);
NiagaraComp->SetVariableTextureRenderTarget(FName("User.SceneRT"), SceneRenderTarget);

// Getters report success through an out parameter and are [[nodiscard]].
bool bIsValid = false;
const float CurrentRate = NiagaraComp->GetVariableFloat(FName("User.EmitRate"), bIsValid);
```

| Namespace prefix | Settable from C++ |
|---|---|
| `User.` | Yes — the only runtime-overridable namespace |
| `System.` | No — engine-owned (Age, DeltaTime, ExecutionState) |
| `Emitter.` | No — per-emitter simulation state |
| `Particle.` | No — per-particle simulation state |
| `Module.` | No — private to a stack node |

Full C++-to-Niagara type table, the array function library matrix and the pool-method values are in [niagara-parameter-types.md](references/niagara-parameter-types.md).

## Data Interfaces

Data interfaces are `UNiagaraDataInterface` subclasses exposed as `User.*` parameters of DI type. Bind them with the typed helpers in `UNiagaraFunctionLibrary`, which take the parameter name as `const FString&`.

```cpp
UNiagaraFunctionLibrary::OverrideSystemUserVariableSkeletalMeshComponent(
    NiagaraComp, TEXT("User.SourceMesh"), GetMesh());

UNiagaraFunctionLibrary::SetSkeletalMeshDataInterfaceFilteredBones(
    NiagaraComp, TEXT("User.SourceMesh"), { FName("hand_l"), FName("hand_r") });

UNiagaraFunctionLibrary::SetSkeletalMeshDataInterfaceSamplingRegions(
    NiagaraComp, TEXT("User.SourceMesh"), { FName("UpperBody") });

UNiagaraFunctionLibrary::OverrideSystemUserVariableStaticMeshComponent(
    NiagaraComp, TEXT("User.ScatterMesh"), ScatterMeshComponent);

UNiagaraFunctionLibrary::SetTextureObject(NiagaraComp, TEXT("User.FlowTexture"), FlowTexture);
```

Array DIs are the workhorse for pushing gameplay data per frame. These helpers take `FName`:

```cpp
#include "NiagaraDataInterfaceArrayFunctionLibrary.h"

TArray<FVector> Waypoints = BuildWaypoints();
UNiagaraDataInterfaceArrayFunctionLibrary::SetNiagaraArrayVector(
    NiagaraComp, FName("User.Waypoints"), Waypoints);

UNiagaraDataInterfaceArrayFunctionLibrary::SetNiagaraArrayFloatValue(
    NiagaraComp, FName("User.HeatData"), /*Index=*/ 5, /*Value=*/ 0.9f, /*bSizeToFit=*/ false);
```

Fetch a DI object when you need to mutate it directly:

```cpp
#include "NiagaraDataInterfaceCurve.h"

UNiagaraDataInterfaceCurve* CurveDI =
    UNiagaraFunctionLibrary::GetDataInterface<UNiagaraDataInterfaceCurve>(
        NiagaraComp, FName("User.SpeedCurve"));

if (CurveDI)
{
    CurveDI->Curve.Reset();
    CurveDI->Curve.AddKey(0.0f, 0.0f);
    CurveDI->Curve.AddKey(1.0f, 500.0f);
#if WITH_EDITORONLY_DATA
    CurveDI->UpdateLUT();    // editor-only
#endif
    // With bUseLUT (default true) CPU and GPU both sample the LUT baked at cook time
    // (NiagaraDataInterfaceCurve.cpp:182), so key edits only take effect in cooked builds
    // if the DI was authored with bUseLUT off.
}
```

The non-template overload `GetDataInterface(UClass* DIClass, UNiagaraComponent*, FName)` covers the case where the class is only known at runtime. The built-in DI catalogue with header paths, properties and sim-target support is in [niagara-data-interfaces.md](references/niagara-data-interfaces.md).

## Custom Data Interfaces

Subclass `UNiagaraDataInterface` (`NiagaraDataInterface.h`). The function list is editor-only data: the override is `GetFunctionsInternal(TArray<FNiagaraFunctionSignature>& OutFunctions) const` declared inside `#if WITH_EDITORONLY_DATA`, plus `GetVMExternalFunction` to bind the CPU implementations and `Equals` / `CopyToInternal` so instances compare and duplicate correctly. GPU support adds `ProvidePerInstanceDataForRenderThread` and `PerInstanceDataPassedToRenderThreadSize`. Full class template in [niagara-data-interfaces.md](references/niagara-data-interfaces.md).

## Niagara Data Channels

Data Channels are the supported bridge between gameplay code and Niagara, and between independent systems. A channel is a `UNiagaraDataChannelAsset` describing a payload; writers publish elements, Niagara emitters and game code read them.

```cpp
#include "NiagaraDataChannelAccessContext.h"
#include "NiagaraDataChannelAccessor.h"
#include "NiagaraDataChannelAsset.h"
#include "NiagaraDataChannelFunctionLibrary.h"

void AMyProjectile::ReportImpact(const FVector& ImpactPoint, const FVector& ImpactNormal)
{
    const FNiagaraDataChannelSearchParameters SearchParams(GetRootComponent());

    UNiagaraDataChannelWriter* Writer = UNiagaraDataChannelLibrary::WriteToNiagaraDataChannel(
        this,
        ImpactChannel,                       // UPROPERTY() TObjectPtr<UNiagaraDataChannelAsset>
        SearchParams,
        /*Count=*/ 1,
        /*bVisibleToGame=*/ false,
        /*bVisibleToCPU=*/ true,
        /*bVisibleToGPU=*/ true,
        TEXT("MyProjectileImpact"));

    if (Writer)
    {
        Writer->WritePosition(FName("Position"), 0, ImpactPoint);
        Writer->WriteVector(FName("Normal"), 0, ImpactNormal);
    }
}
```

Reading back on the game side:

```cpp
void AMyProjectile::DrainImpacts()
{
    const FNiagaraDataChannelSearchParameters SearchParams(GetRootComponent());

    UNiagaraDataChannelReader* Reader = UNiagaraDataChannelLibrary::ReadFromNiagaraDataChannel(
        this, ImpactChannel, SearchParams, /*bReadPreviousFrame=*/ true);

    for (int32 Index = 0; Reader && Index < Reader->Num(); ++Index)
    {
        bool bValid = false;
        const FVector Position = Reader->ReadPosition(FName("Position"), Index, bValid);
        if (bValid)
        {
            SpawnDecalAt(Position);
        }
    }
}
```

`SubscribeToNiagaraDataChannel(WorldContextObject, Channel, SearchParams, UpdateDelegate, UnsubscribeToken)` pushes an `FOnNewNiagaraDataChannelPublish` callback whenever new elements are published; its `FNiagaraDataChannelUpdateContext` carries `Reader`, `FirstNewDataIndex`, `LastNewDataIndex` and `NewElementCount`. Pair every subscription with `UnsubscribeFromNiagaraDataChannel` using the stored token.

`UNiagaraDataChannelLibrary` also offers `GetDataChannelElementCount`, single-element `ReadFromNiagaraDataChannelSingle` / `WriteToNiagaraDataChannelSingle`, and `_WithContext` variants that take an `FNDCAccessContextInst` instead of `FNiagaraDataChannelSearchParameters`. The search-parameter overloads are the legacy path and do not support newer channel types; prefer the context variants for new channels.

## Completion Callbacks

`OnSystemFinished` is `DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnNiagaraSystemFinished, class UNiagaraComponent*, PSystem)`, so the handler must be a `UFUNCTION()` declared in a class body.

```cpp
// MyVFXComponent.h
#pragma once

#include "CoreMinimal.h"
#include "Components/ActorComponent.h"
#include "MyVFXComponent.generated.h"

class UNiagaraComponent;
class UNiagaraSystem;

UCLASS(ClassGroup = (Custom), meta = (BlueprintSpawnableComponent))
class MYGAME_API UMyVFXComponent : public UActorComponent
{
    GENERATED_BODY()

public:
    void PlayBurst(UNiagaraSystem* System, const FVector& Location);

    virtual void EndPlay(const EEndPlayReason::Type EndPlayReason) override;

protected:
    UFUNCTION()
    void HandleVFXFinished(UNiagaraComponent* FinishedComponent);

    UPROPERTY(Transient)
    TObjectPtr<UNiagaraComponent> ActiveVFX;
};
```

```cpp
// MyVFXComponent.cpp
#include "MyVFXComponent.h"

#include "NiagaraComponent.h"
#include "NiagaraFunctionLibrary.h"

void UMyVFXComponent::PlayBurst(UNiagaraSystem* System, const FVector& Location)
{
    ActiveVFX = UNiagaraFunctionLibrary::SpawnSystemAtLocation(
        this, System, Location, FRotator::ZeroRotator, FVector(1.0f),
        /*bAutoDestroy=*/ false, /*bAutoActivate=*/ true, ENCPoolMethod::ManualRelease);
    if (ActiveVFX)
    {
        ActiveVFX->OnSystemFinished.AddDynamic(this, &UMyVFXComponent::HandleVFXFinished);
    }
}

void UMyVFXComponent::HandleVFXFinished(UNiagaraComponent* FinishedComponent)
{
    FinishedComponent->OnSystemFinished.RemoveDynamic(this, &UMyVFXComponent::HandleVFXFinished);
    FinishedComponent->ReleaseToPool();
    ActiveVFX = nullptr;
}

void UMyVFXComponent::EndPlay(const EEndPlayReason::Type EndPlayReason)
{
    if (ActiveVFX)
    {
        ActiveVFX->OnSystemFinished.RemoveDynamic(this, &UMyVFXComponent::HandleVFXFinished);
        ActiveVFX->ReleaseToPool();   // ManualRelease components leak unless released
        ActiveVFX = nullptr;
    }
    Super::EndPlay(EndPlayReason);
}
```

## Simulation Caches

A `UNiagaraSimCache` records simulation frames and plays them back deterministically — useful for cinematics, network-visible hero effects and regression tests.

```cpp
#include "NiagaraComponent.h"
#include "NiagaraSimCache.h"
#include "NiagaraSimCacheFunctionLibrary.h"

UNiagaraSimCache* Cache = UNiagaraSimCacheFunctionLibrary::CreateNiagaraSimCache(this);

FNiagaraSimCacheCreateParameters CreateParameters;
UNiagaraSimCache* OutCache = nullptr;
UNiagaraSimCacheFunctionLibrary::CaptureNiagaraSimCacheImmediate(
    Cache, CreateParameters, NiagaraComp, OutCache,
    /*bAdvanceSimulation=*/ false, /*AdvanceDeltaTime=*/ 0.01666f);

// Play a cache back on any component running the same system, then hand control
// back to the live simulation.
NiagaraComp->SetSimCache(Cache, /*bResetSystem=*/ true);
NiagaraComp->ClearSimCache(/*bResetSystem=*/ true);
```

`UNiagaraSimCache` exposes `IsCacheValid()`, `CanRead(UNiagaraSystem*)`, `GetNumFrames()`, `GetStartSeconds()` and `GetDurationSeconds()`. Data interfaces that hold their own state participate through `INiagaraSimCacheCustomStorageInterface`.

## Lightweight Emitters

Emitters come in two modes, reported by `FNiagaraEmitterHandle::GetEmitterMode()`: `ENiagaraEmitterMode::Standard` (`UNiagaraEmitter`, full scripted simulation) and `ENiagaraEmitterMode::Stateless` (`UNiagaraStatelessEmitter`, a fixed-cost analytic emitter with no per-particle simulation buffers). Stateless emitters are authored in the Niagara editor and cost far less on CPU and memory; prefer them for ambient, high-instance-count effects.

`UNiagaraStatelessEmitter` lives in the module-internal `Internal/Stateless/NiagaraStatelessEmitter.h`, so game modules cannot include it. Drive stateless emitters the same way as any other system — through `UNiagaraComponent` and `User.` parameters. To detect them from game code, read `bIsLightweight` on `FNiagaraMinimalEmitterInfo` from `UNiagaraFunctionLibrary::GetAllEmitters`.

## Component Pooling

`ENCPoolMethod` on every spawn call selects pool behavior: `None` (fresh component), `AutoRelease` (returned automatically on completion; do not keep the pointer past the spawning tick) and `ManualRelease` (you call `ReleaseToPool()`). `ManualRelease_OnComplete` and `FreeInPool` are hidden internal states.

```cpp
#include "NiagaraComponentPool.h"
#include "NiagaraWorldManager.h"

// Prime one world before a gameplay-critical moment. Call the pool directly:
// FNiagaraWorldManager::PrimePool / PrimePoolForAllWorlds are not NIAGARA_API
// (NiagaraWorldManager.h:236-238) and fail to link from a game module.
if (FNiagaraWorldManager* WorldMan = FNiagaraWorldManager::Get(GetWorld()))
{
    WorldMan->GetComponentPool()->PrimePool(ExplosionSystem, GetWorld());
}
```

Pool capacity is a per-system property on the asset (`MaxPoolSize`, `PoolPrimeSize` on `UFXSystemAsset`, `Particles/ParticleSystem.h:126,135`); priming creates `min(PoolPrimeSize, MaxPoolSize)` components and `PoolPrimeSize` defaults to 0, so set it or priming is a no-op (`NiagaraComponentPool.cpp:285`). Global CVars: `FX.NiagaraComponentPool.Enable`, `FX.NiagaraComponentPool.KillUnusedTime`, `FX.NiagaraComponentPool.CleanTime`, `FX.NiagaraComponentPool.Validation`.

## Scalability and Tick Behavior

```cpp
NiagaraComp->SetAllowScalability(true);    // let the scalability manager cull this instance
NiagaraComp->SetTickBehavior(ENiagaraTickBehavior::UsePrereqs);
```

| `ENiagaraTickBehavior` | Meaning |
|---|---|
| `UsePrereqs` | Tick after attachment and data-interface prerequisites. Safest, and the default. |
| `UseComponentTickGroup` | Ignore prerequisites; use the component's own tick group. |
| `ForceTickFirst` | Tick in the first tick group. |
| `ForceTickLast` | Tick in the last tick group. |

Scalability lives in the `UNiagaraEffectType` assigned to the system: `SystemScalabilitySettings`, `EmitterScalabilitySettings`, `CullReaction`, `UpdateFrequency` and an instanced `SignificanceHandler`. It is data-driven; no per-platform C++.

**CPU vs GPU sim**: CPU simulations support every data interface and can be read back; GPU simulations scale to far higher counts but support a narrower DI set and cannot be read back without an explicit readback path.

**Determinism**: for effects that must match across clients, enable fixed tick on the `UNiagaraSystem` asset (`bFixedTickDelta` / `FixedTickDeltaTime`, readable through `HasFixedTickDelta()` and `GetFixedTickDeltaTime()`) and use CPU simulation. Cosmetic effects belong on clients only — guard spawns with `IsRunningDedicatedServer()`.

**Occlusion**: `SetOcclusionQueryMode(ENiagaraOcclusionQueryMode)` / `GetOcclusionQueryMode()` control occlusion queries per component (`Default`, `AlwaysEnabled`, `AlwaysDisabled`).

Niagara Fluids (Beta in 5.8) adds GPU fluid and gas solvers at a high per-frame cost; restrict it to hero effects.

## Build Setup

```csharp
PublicDependencyModuleNames.AddRange(new string[] { "Core", "CoreUObject", "Engine", "Niagara", "NiagaraCore" });
```

`"NiagaraCore"` is only needed when you derive from `UNiagaraDataInterface`. Headers under `Internal/` (for example `Internal/DataInterface/NiagaraDataInterfaceStaticMesh.h`) are not includable from game modules.

## Deprecated — do not use

| Do not emit | Use in 5.8 | Source |
|---|---|---|
| `SetNiagaraVariableFloat(const FString&, float)` and every other `SetNiagaraVariable*` FString setter | `SetVariableFloat(FName, float)` and the matching `SetVariable*` FName setters | `UE_DEPRECATED(5.3)` in `Public/NiagaraComponent.h` |
| `UNiagaraDataInterface::GetFunctions(TArray<FNiagaraFunctionSignature>&)` | `GetFunctionsInternal(TArray<FNiagaraFunctionSignature>&) const` inside `#if WITH_EDITORONLY_DATA` | `UE_DEPRECATED(5.4)` in `Classes/NiagaraDataInterface.h` |
| `UNiagaraDataInterface::RequiresDistanceFieldData()` | `RequiresGlobalDistanceField()` | `UE_DEPRECATED(5.4)` in `Classes/NiagaraDataInterface.h` |
| `UNiagaraDataInterface::ReadsEmitterParticleData(const FString&)` | `GetEmitterReferencesByName()` | `UE_DEPRECATED(5.4)` in `Classes/NiagaraDataInterface.h` |
| `UNiagaraDataInterface::PostCompile()` with no arguments | the `PostCompile` overload taking the owning system | `UE_DEPRECATED(5.7)` in `Classes/NiagaraDataInterface.h` |
| `UNiagaraDataInterface::IsUsedByCPUEmitter()` / `IsUsedByGPUEmitter()` | `IsUsedWithCPUScript()` / `IsUsedWithGPUScript()` | `UE_DEPRECATED(5.3)` in `Classes/NiagaraDataInterface.h` |
| `UNiagaraComponent::GetSystemInstance()` | `GetSystemInstanceController()` | `UE_DEPRECATED(5.0)` in `Public/NiagaraComponent.h` |
| An abstract or submix-named audio data interface base class | `UNiagaraDataInterfaceAudioSpectrum` or `UNiagaraDataInterfaceAudioOscilloscope`, each with a `USoundSubmix* Submix` property | only three concrete audio DIs exist, in `Classes/NiagaraDataInterfaceAudio*.h` |
| Treating `ENCPoolMethod::FreeInPool` as a spawn argument | `None`, `AutoRelease` or `ManualRelease` | `UMETA(Hidden)` in `Public/NiagaraComponentPoolMethodEnum.h` |

## Common Mistakes

**Assuming there is no C++ event path into Niagara:** there is. Use Data Channels (`UNiagaraDataChannelLibrary`) for gameplay-to-Niagara events rather than toggling a `User.` bool and hoping the spawn script samples it on the right tick.

**Spawning a component every tick:** `SpawnSystemAtLocation` inside `Tick` creates and destroys a component per frame. Create the component once, or spawn with `ENCPoolMethod::AutoRelease` on discrete events only.

**Wrong namespace or wrong type:** `SetVariableFloat(FName("Emitter.Speed"), 300.f)` and `SetVariableVec3` on a `Color` parameter both fail silently. The author must expose the value as `User.Speed`, the setter must match the authored type, and world positions go through `SetVariablePosition` so LWC is handled.

**Ignoring the return value of a spawn call:** it is `nullptr` on dedicated servers and when the pre-cull check rejects the spawn. Null-check before touching the component.

**`UFUNCTION()` on an out-of-line definition:** the macro only has meaning inside a `UCLASS` body. A `OnSystemFinished` handler marked `UFUNCTION()` above its `.cpp` definition will not bind; declare it in the header as shown above.

**Holding a pointer to an `AutoRelease` component:** the pool reclaims it as soon as the system completes, and later access is unsafe. Use `ManualRelease` plus `ReleaseToPool()` when you need a lasting handle.

**Treating a data interface like a component:** `UpdateLUT()` is `WITH_EDITORONLY_DATA` and must be guarded, and `MarkRenderStateDirty()` does not exist — `UNiagaraDataInterface` derives from `UObject`. Array changes are picked up on the next simulation tick; curve key changes only if the curve DI has `bUseLUT` off.

## Related Skills

- `ue-materials-rendering` — particle materials, dynamic material instances, render targets fed to DIs
- `ue-actor-component-architecture` — component creation, attachment, activation and lifecycle
- `ue-animation-system` — skeletal mesh sockets, notifies and bone data that drive mesh-sampling DIs
- `ue-gameplay-abilities` — GameplayCue-driven effect spawning and ability-scoped VFX ownership
- `ue-audio-system` — sound submixes and spectrum analysis behind the audio data interfaces
- `ue-async-threading` — render-thread and task-graph rules when feeding DIs from background work
- `ue-procedural-generation` — generating the point and spline data that array and spline DIs consume
- `ue-sequencer-cinematics` — Level Sequences, playback, cine cameras and Movie Render Graph
