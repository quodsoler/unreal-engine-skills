---
name: ue-actor-component-architecture
description: "Use when writing or debugging AActor and UActorComponent C++: constructor vs BeginPlay, PostInitializeComponents, EndPlay cleanup, SpawnActor/SpawnActorDeferred/FinishSpawning, CreateDefaultSubobject, NewObject + RegisterComponent, SetupAttachment/AttachToComponent, tick groups, UChildActorComponent, UINTERFACE, actor pooling. Also use when the user mentions 'BeginPlay', 'Tick', 'SpawnActor', 'CreateDefaultSubobject', 'attach a component', 'root component', 'lifecycle', 'PrimaryActorTick', 'FAttachmentTransformRules'. For collision and overlaps, see ue-physics-collision; for GameMode/Pawn/Controller, see ue-gameplay-framework; for replicated actors, see ue-networking-replication."
metadata:
  version: "2.0.0"
  engine: "5.8"
---

# UE Actor-Component Architecture

Target engine: **UE 5.8**. APIs below are verified against the 5.8 headers; older forms are listed under "Deprecated — do not use".

Build actors out of components: lifecycle hooks in the right order, spawning (immediate, deferred, pooled), component creation and attachment, ticking, and C++ interfaces. Everything here is in the `Engine` module (`GameFramework/Actor.h`, `Components/ActorComponent.h`, `Components/SceneComponent.h`, `Engine/World.h`) on top of `CoreUObject`; a game module's Build.cs needs `"Core", "CoreUObject", "Engine"`.

## Context

Read `.agents/ue-project-context.md` if it exists (module names, conventions, enabled plugins, GAS/networking setup). Do not stop if it is missing.

Identify the area from the request and the codebase. Ask only when two plausible readings would produce different code.

| Request is about… | Go to |
|---|---|
| Where code belongs: constructor, PostInitializeComponents, BeginPlay, EndPlay | [Actor Lifecycle](#actor-lifecycle) |
| Creating, attaching, activating components | [Component System](#component-system) |
| Overriding component hooks (OnRegister, InitializeComponent, TickComponent) | [Component Lifecycle Hooks](#component-lifecycle-hooks) |
| SpawnActor, deferred spawn, pooling | [Spawning](#spawning) |
| Tick groups, tick interval, prerequisites, tick vs timer | [Ticking](#ticking) |
| Cross-actor capability without casting | [Interfaces](#interfaces) |
| Splitting a fat actor into components, data-driven loadouts | [Composition Patterns](#composition-patterns) |
| Full event order, EEndPlayReason, network and child-actor timing | [references/actor-lifecycle.md](references/actor-lifecycle.md) |
| Which built-in component to use, header paths, member APIs | [references/component-types.md](references/component-types.md) |

## Mental Model

```
UObject
  └── AActor                            placeable/spawnable; owns components      GameFramework/Actor.h
        └── UActorComponent             logic only, no transform                  Components/ActorComponent.h
              └── USceneComponent       transform + attachment tree               Components/SceneComponent.h
                    └── UPrimitiveComponent  rendering + collision + physics      Components/PrimitiveComponent.h
```

Never `new`/`delete` an actor or component. Actors: `UWorld::SpawnActor` and `AActor::Destroy(bool bNetForce = false, bool bShouldModifyLevel = true)` (`Actor.h:2327`). Components: `CreateDefaultSubobject`/`NewObject` and `UActorComponent::DestroyComponent(bool bPromoteChildren = false)` (`ActorComponent.h:1328`).

## Actor Lifecycle

Order from the `AActor` class comment (`Actor.h:243-278`); full table with citations in [references/actor-lifecycle.md](references/actor-lifecycle.md).

```
AActor::AActor()                       CDO + every instance. No world. CreateDefaultSubobject, tick config, defaults
UObject::PostLoad()                    level-placed actors only (not spawned)
UActorComponent::OnComponentCreated()  native components on spawn (not when loaded from a level)
AActor::PreRegisterAllComponents()  →  RegisterComponent() per component  →  AActor::PostRegisterAllComponents()
AActor::PostActorCreated()             spawned actors only, before construction scripts
UserConstructionScript()  →  AActor::OnConstruction(const FTransform&)
AActor::PreInitializeComponents()      gameplay only
UActorComponent::Activate()            if bAutoActivate
UActorComponent::InitializeComponent() if bWantsInitializeComponent, once per session
AActor::PostInitializeComponents()     components initialized; bind to own components here
AActor::BeginPlay()                    level is ticking; may be delayed for networked or child actors
AActor::Tick(float DeltaSeconds)       if PrimaryActorTick.bCanEverTick
AActor::EndPlay(const EEndPlayReason::Type)  base implementation forwards EndPlay to components
AActor::Destroyed()                    explicit Destroy() only; not on level streaming or game end
```

### Constructor vs BeginPlay

The constructor runs on the Class Default Object, where `GetWorld()` is `nullptr`. Anything that needs the world or other actors goes in `PostInitializeComponents` (own components) or `BeginPlay` (gameplay).

```cpp
// MyActor.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/Actor.h"
#include "MyActor.generated.h"

class UStaticMeshComponent;
class UMyHealthComponent;

UCLASS()
class MYGAME_API AMyActor : public AActor
{
    GENERATED_BODY()
public:
    AMyActor();
protected:
    virtual void PostInitializeComponents() override;
    virtual void BeginPlay() override;
    virtual void EndPlay(const EEndPlayReason::Type EndPlayReason) override;

    UFUNCTION()
    void HandleDeath();

    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category="Components")
    TObjectPtr<UStaticMeshComponent> MeshComp;

    UPROPERTY(VisibleAnywhere, Category="Components")
    TObjectPtr<UMyHealthComponent> HealthComp;
};
```

```cpp
// MyActor.cpp
#include "MyActor.h"
#include "Components/StaticMeshComponent.h"
#include "TimerManager.h"
#include "MyHealthComponent.h"

AMyActor::AMyActor()
{
    PrimaryActorTick.bCanEverTick = true;
    PrimaryActorTick.bStartWithTickEnabled = false;

    MeshComp = CreateDefaultSubobject<UStaticMeshComponent>(TEXT("Mesh"));
    SetRootComponent(MeshComp);                      // SetupAttachment(MeshComp) parents a scene component here; no world needed

    HealthComp = CreateDefaultSubobject<UMyHealthComponent>(TEXT("Health")); // logic-only: no attachment
}

void AMyActor::PostInitializeComponents()
{
    Super::PostInitializeComponents();
    HealthComp->OnDeath.AddDynamic(this, &AMyActor::HandleDeath); // own components are initialized
}

void AMyActor::BeginPlay()
{
    Super::BeginPlay();
    SetActorTickEnabled(true);                        // gameplay starts here
}

void AMyActor::EndPlay(const EEndPlayReason::Type EndPlayReason)
{
    GetWorldTimerManager().ClearAllTimersForObject(this);
    Super::EndPlay(EndPlayReason);                    // base forwards EndPlay to components
}

void AMyActor::HandleDeath() { Destroy(); }
```

`EEndPlayReason::Type` (`Engine/EngineTypes.h:3670-3685`): `Destroyed`, `LevelTransition`, `EndPlayInEditor`, `RemovedFromWorld`, `Quit`. Clear timers and unbind delegates for every reason, then call `Super::EndPlay(EndPlayReason)`.

### Network note

On clients, `BeginPlay` "can be delayed for networked or child actors" (`Actor.h:275`). Property arrival hooks are `PreNetReceive()`/`PostNetReceive()` (`Actor.h:2943-2946`) and `PostNetInit()`, "always called immediately after spawning and reading in replicated properties" (`Actor.h:2958`). Replication rules, `OnRep_` functions and RPCs are owned by `ue-networking-replication`.

## Component System

| Class | Transform | Render/Collision | Use for |
|---|---|---|---|
| `UActorComponent` | No | No | Pure logic and data: health, inventory, cooldowns |
| `USceneComponent` | Yes | No | Anchors, pivots, grouping, attach points |
| `UPrimitiveComponent` | Yes | Yes | Meshes, shapes, anything visible or collidable |

Built-in subclasses, header paths and member APIs: [references/component-types.md](references/component-types.md).

### Writing a component

```cpp
// MyHealthComponent.h
#pragma once

#include "CoreMinimal.h"
#include "Components/ActorComponent.h"
#include "MyHealthComponent.generated.h"

DECLARE_DYNAMIC_MULTICAST_DELEGATE(FMyOnDeathSignature);

UCLASS(ClassGroup=(MyGame), meta=(BlueprintSpawnableComponent))
class MYGAME_API UMyHealthComponent : public UActorComponent
{
    GENERATED_BODY()
public:
    UMyHealthComponent();

    UPROPERTY(BlueprintAssignable, Category="Health")
    FMyOnDeathSignature OnDeath;

    UFUNCTION(BlueprintCallable, Category="Health")
    void ApplyDamage(float Amount);
protected:
    virtual void InitializeComponent() override;

    UPROPERTY(EditAnywhere, Category="Health")
    float MaxHealth = 100.f;

    UPROPERTY(VisibleInstanceOnly, Transient, Category="Health")
    float CurrentHealth = 0.f;
};
```

```cpp
// MyHealthComponent.cpp
#include "MyHealthComponent.h"

UMyHealthComponent::UMyHealthComponent()
{
    PrimaryComponentTick.bCanEverTick = false;   // event-driven: no per-frame work
    bWantsInitializeComponent = true;            // opt in to InitializeComponent()
}

void UMyHealthComponent::InitializeComponent()
{
    Super::InitializeComponent();
    CurrentHealth = MaxHealth;                   // once per gameplay session, before BeginPlay
}

void UMyHealthComponent::ApplyDamage(float Amount)
{
    CurrentHealth = FMath::Max(0.f, CurrentHealth - Amount);
    if (CurrentHealth <= 0.f) { OnDeath.Broadcast(); }
}
```

Component replication: `bReplicates` is private on `UActorComponent` (`ActorComponent.h:267`); call `SetIsReplicatedByDefault(true)` in the constructor or `SetIsReplicated(true)` later, and the owning actor must replicate. Details in `ue-networking-replication`.

### Creating components at runtime

```cpp
// In AMyActor (declared: void AddRuntimeLight();)
#include "Components/PointLightComponent.h"

void AMyActor::AddRuntimeLight()
{
    // NewObject creates the UObject only; RegisterComponent creates render/physics state and enables tick.
    UPointLightComponent* Light = NewObject<UPointLightComponent>(this, TEXT("RuntimeLight"));
    Light->SetupAttachment(GetRootComponent());  // allowed before RegisterComponent
    Light->RegisterComponent();
    AddInstanceComponent(Light);                 // adds to InstanceComponents: visible in Details, serialized per instance
    Light->SetIntensity(5000.f);
}
```

`UnregisterComponent()` removes world presence but keeps the object (reversible); `DestroyComponent()` unregisters and marks it for GC. A component whose Outer is an actor is added to that actor's `OwnedComponents` (`ActorComponent.cpp:600` in `PostInitProperties`, `:951` on register), which `AActor::AddReferencedObjects` reports to GC (`Actor.cpp:574`), so it stays alive until `DestroyComponent()` or the actor dies, with or without a `UPROPERTY`. Still hold your own pointer in a `UPROPERTY` (`TObjectPtr`) so it is tracked and nulled safely; a raw pointer dangles after `DestroyComponent()`.

### Attachment

| Call | Where | Header |
|---|---|---|
| `SetupAttachment(USceneComponent* InParent, FName InSocketName = NAME_None)` | Constructor, or any unregistered component | `SceneComponent.h:734` |
| `AttachToComponent(USceneComponent* InParent, const FAttachmentTransformRules& AttachmentRules, FName InSocketName = NAME_None)` | Runtime; valid registered or not, but prefer `SetupAttachment` before registration. Returns `bool` (false = rejected, parent unchanged) | `SceneComponent.h:752` |
| `DetachFromComponent(const FDetachmentTransformRules& DetachmentRules)` | Runtime | `SceneComponent.h:786` |
| `AActor::AttachToComponent(...)`, `AttachToActor(AActor* ParentActor, const FAttachmentTransformRules&, FName SocketName = NAME_None)`, `DetachFromActor(const FDetachmentTransformRules&)` | Attach the whole actor via its root; both attach calls return `bool` | `Actor.h:2015, 2029, 2062` |
| `K2_AttachToComponent(...)`, `K2_AttachToActor(...)` | Blueprint nodes only; C++ calls the non-`K2_` forms | `Actor.h:2006, 2042` |

`FAttachmentTransformRules` presets (`Engine/EngineTypes.h:78-81`): `KeepRelativeTransform`, `KeepWorldTransform`, `SnapToTargetNotIncludingScale`, `SnapToTargetIncludingScale`; per-axis constructor `FAttachmentTransformRules(EAttachmentRule InLocationRule, EAttachmentRule InRotationRule, EAttachmentRule InScaleRule, bool bInWeldSimulatedBodies)`. `FDetachmentTransformRules` presets (`:125-126`): `KeepRelativeTransform`, `KeepWorldTransform`.

```cpp
void AMyCharacter::EquipWeapon(UStaticMeshComponent* WeaponMesh)
{
    WeaponMesh->AttachToComponent(GetMesh(), FAttachmentTransformRules::SnapToTargetNotIncludingScale, TEXT("WeaponSocket"));
}
```

### Activation (`ActorComponent.h`)

`uint8 bAutoActivate:1` is public (`:317`): set it in the constructor. Runtime: `virtual void Activate(bool bReset = false)` (`:580`), `virtual void Deactivate()` (`:586`), `virtual void SetActive(bool bNewActive, bool bReset = false)` (`:594`), `virtual void ToggleActive()` (`:600`), `bool IsActive() const` (`:607`; `bIsActive` is private). Gate activation by overriding `virtual bool ShouldActivate() const` (`:807`). Delegates `OnComponentActivated` / `OnComponentDeactivated` (`:559, :563`).

## Component Lifecycle Hooks

Override these on `UActorComponent` subclasses; signatures are verbatim from `Components/ActorComponent.h`.

| Hook | Line | Fires |
|---|---|---|
| `virtual void OnComponentCreated()` | `:1331` | Native components when the actor is spawned; not for components loaded from a level |
| `virtual void OnRegister()` / `virtual void OnUnregister()` | `:830, :835` | Around world registration; render/physics state is created after `OnRegister` |
| `virtual void InitializeComponent()` / `virtual void UninitializeComponent()` | `:919, :955` | Only when `bWantsInitializeComponent` (`:340`) is true; once per session |
| `virtual void BeginPlay()` / `virtual void EndPlay(const EEndPlayReason::Type EndPlayReason)` | `:936, :949` | Play start/stop |
| `virtual void TickComponent(float DeltaTime, enum ELevelTick TickType, FActorComponentTickFunction* ThisTickFunction)` | `:976` | Requires `PrimaryComponentTick.bCanEverTick` and registration |
| `virtual void OnComponentDestroyed(bool bDestroyingHierarchy)` | `:1338` | From `DestroyComponent` |
| `HasBeenCreated()`, `HasBeenInitialized()`, `HasBegunPlay()`, `IsRegistered()`, `IsBeingDestroyed()` | `:505-520, :1316` | State queries; the bit fields are private |

`GetOwner()` (`:538`) and `GetOwner<T>()` (`:542`) return the owning actor; `GetWorld()` is valid once registered.

## Spawning

`FActorSpawnParameters` (`Engine/World.h:420-519`): `Name`, `Template`, `Owner`, `Instigator`, `OverrideLevel`, `SpawnCollisionHandlingOverride`, `TransformScaleMethod`, `bNoFail`, `bDeferConstruction`, `bAllowDuringConstructionScript`, `NameMode`, `ObjectFlags`, `CustomPreSpawnInitialization`. Collision handling values (`Engine/EngineTypes.h:4411-4423`): `Undefined`, `AlwaysSpawn`, `AdjustIfPossibleButAlwaysSpawn`, `AdjustIfPossibleButDontSpawnIfColliding`, `DontSpawnIfColliding`.

```cpp
// AMyEnemySpawner member: UPROPERTY(EditDefaultsOnly) TSubclassOf<AMyEnemy> EnemyClass;
AMyEnemy* AMyEnemySpawner::SpawnEnemy(const FVector& Location, const FRotator& Rotation)
{
    FActorSpawnParameters Params;
    Params.Owner = this;
    Params.Instigator = GetInstigator();
    Params.SpawnCollisionHandlingOverride = ESpawnActorCollisionHandlingMethod::AdjustIfPossibleButAlwaysSpawn;
    return GetWorld()->SpawnActor<AMyEnemy>(EnemyClass, Location, Rotation, Params);
}
```

### Deferred spawn: configure before construction and BeginPlay

`SpawnActorDeferred<T>(UClass* Class, FTransform const& Transform, AActor* Owner = nullptr, APawn* Instigator = nullptr, ESpawnActorCollisionHandlingMethod CollisionHandlingOverride = ESpawnActorCollisionHandlingMethod::Undefined, ESpawnActorScaleMethod TransformScaleMethod = ESpawnActorScaleMethod::MultiplyWithRoot)` (`World.h:3851-3870`) sets `bDeferConstruction = true`. Finish with `FinishSpawning(const FTransform& Transform, bool bIsDefaultTransform = false, const FComponentInstanceDataCache* InstanceDataCache = nullptr, ESpawnActorScaleMethod TransformScaleMethod = ESpawnActorScaleMethod::OverrideRootScale)` (`Actor.h:3117`). Until then the actor "is in a malformed state" (`Actor.h:632`).

```cpp
// AMyEnemy declares: void SetEnemyData(UMyEnemyData* Data);
AMyEnemy* AMyEnemySpawner::SpawnEnemyFromData(const FTransform& SpawnTransform, UMyEnemyData* Data)
{
    AMyEnemy* Enemy = GetWorld()->SpawnActorDeferred<AMyEnemy>(EnemyClass, SpawnTransform, this, GetInstigator(),
        ESpawnActorCollisionHandlingMethod::AlwaysSpawn);
    if (!Enemy) { return nullptr; }
    Enemy->SetEnemyData(Data);                     // runs before OnConstruction and BeginPlay
    Enemy->FinishSpawning(SpawnTransform, false, nullptr, ESpawnActorScaleMethod::MultiplyWithRoot);
    return Enemy;
}
```

The two defaults differ: `SpawnActorDeferred` uses `MultiplyWithRoot`, `FinishSpawning` uses `OverrideRootScale`. Pass the scale method explicitly when the root component has a non-unit default scale.

### Object pooling

`SpawnActor`/`Destroy` churn for projectiles or casings costs GC time. Park actors instead: hide, disable collision and tick; reuse by reversing. `AActor` has no `IsActive()`; use `IsHidden()` (`Actor.h:4523`) as the parked flag.

Full pooled-actor class (`Acquire`/`Release` with GC-safe storage): [references/component-types.md](references/component-types.md#actor-pooling).

## Ticking

`FTickFunction` fields (`Engine/EngineBaseTypes.h`): `bCanEverTick` (`:213`), `bStartWithTickEnabled` (`:217`), `bTickEvenWhenPaused` (`:209`), `bAllowTickOnDedicatedServer` (`:221`), `TickGroup` (`:196`), `EndTickGroup` (`:204`), `TickInterval` (`:258`). Actors use `PrimaryActorTick` (`Actor.h:318`), components `PrimaryComponentTick` (`ActorComponent.h:177`).

```cpp
AMyEnemy::AMyEnemy()
{
    PrimaryActorTick.bCanEverTick = true;
    PrimaryActorTick.bStartWithTickEnabled = false;   // later: SetActorTickEnabled(true)
    PrimaryActorTick.TickInterval = 0.1f;             // later: SetActorTickInterval(0.1f)
    PrimaryActorTick.TickGroup = TG_PostPhysics;
}
```

| `ETickingGroup` (`EngineBaseTypes.h:83-109`) | Use |
|---|---|
| `TG_PrePhysics` | Default. Input, movement requests, AI decisions |
| `TG_DuringPhysics` | Work that does not read this frame's physics results |
| `TG_PostPhysics` | Camera, IK, anything reading final physics transforms |
| `TG_PostUpdateWork` | Last gameplay group |
| `TG_StartPhysics`, `TG_EndPhysics`, `TG_LastDemotable`, `TG_NewlySpawned` | Engine-internal; never assign |

Ordering between specific objects: `AddTickPrerequisiteActor(AActor* PrerequisiteActor)` and `AddTickPrerequisiteComponent(UActorComponent* PrerequisiteComponent)` exist on both `AActor` (`Actor.h:2093, 2097`) and `UActorComponent` (`ActorComponent.h:1355, 1359`). `SetTickableWhenPaused(bool bTickableWhenPaused)` (`Actor.h:2113`, `ActorComponent.h:618`). Change group at runtime with `SetTickGroup(ETickingGroup NewTickGroup)` (`Actor.h:3566`, `ActorComponent.h:1351`). Fixed-step physics callbacks: `bAsyncPhysicsTickEnabled` (`Actor.h:699`) plus `virtual void AsyncPhysicsTickActor(float DeltaTime, float SimTime)` (`Actor.h:2930`); see `ue-physics-collision`.

### When not to tick

| Need | Use instead |
|---|---|
| Delayed or repeating work | `GetWorldTimerManager().SetTimer(FTimerHandle& InOutHandle, UserClass* InObj, TMethodPtr InTimerMethod, float InRate, bool InbLoop = false, float InFirstDelay = -1.f)` (`TimerManager.h:167`) |
| React to state change | Dynamic multicast delegate on the component, bound in `PostInitializeComponents` |
| Collision | `OnComponentBeginOverlap`, `OnComponentHit` (`PrimitiveComponent.h:1457-1475`), `OnActorBeginOverlap` (`Actor.h:1368`); see `ue-physics-collision` |
| Cheap per-frame updates | Keep tick but throttle with `TickInterval` and disable with `SetActorTickEnabled(false)` when idle |

## Interfaces

Use a `UINTERFACE` for a capability ("can be interacted with") that unrelated actors share; use a component when the behavior owns state or needs to tick. Blueprint-facing exposure rules (`BlueprintImplementableEvent`, `meta=` specifiers, latent actions) are owned by `ue-blueprint-cpp-interop`.

```cpp
// MyInteractable.h
#pragma once

#include "CoreMinimal.h"
#include "UObject/Interface.h"
#include "MyInteractable.generated.h"

UINTERFACE(MinimalAPI, Blueprintable)
class UMyInteractable : public UInterface
{
    GENERATED_BODY()
};

class MYGAME_API IMyInteractable
{
    GENERATED_BODY()
public:
    // C++ default + Blueprint override; UHT also generates Execute_OnInteract(UObject*, AActor*) for safe calls
    UFUNCTION(BlueprintNativeEvent, BlueprintCallable, Category="Interaction")
    void OnInteract(AActor* InstigatorActor);
};
```

Implementers inherit both classes, `class MYGAME_API AMyChest : public AActor, public IMyInteractable`, and override `virtual void OnInteract_Implementation(AActor* InstigatorActor) override;`. Callers never invoke `OnInteract` directly:

```cpp
// Caller (AMyCharacter, declared: void TryInteract(AActor* Target);)
void AMyCharacter::TryInteract(AActor* Target)
{
    if (Target && Target->Implements<UMyInteractable>())        // U-class for the reflection check
    {
        IMyInteractable::Execute_OnInteract(Target, this);       // works for C++ and Blueprint implementers
    }
    // Components implementing the interface: FindComponentByInterface<IMyInteractable>() takes the I-class (Actor.h:3852)
}
```

## Composition Patterns

Prefer a flat actor plus components over deep hierarchies (`AMyCharacter` + `UMyHealthComponent` + `UMyInventoryComponent`), with one class and many data assets instead of one subclass per variant. Components talk through the owner (`GetOwner()->FindComponentByClass<UMyHealthComponent>()`) or through delegates, never through cached raw pointers to siblings. Data-driven loadouts add components at runtime from a `TArray<TSubclassOf<UActorComponent>>` on a data asset:

```cpp
void AMyEnemy::ApplyLoadout(const UMyEnemyData* Data)
{
    for (const TSubclassOf<UActorComponent>& CompClass : Data->ExtraComponents)
    {
        UActorComponent* Comp = NewObject<UActorComponent>(this, CompClass);
        if (!Comp) { continue; }
        Comp->RegisterComponent();          // starts ticking and registers with the world
        AddInstanceComponent(Comp);         // shows up in the details panel and is saved with the actor
    }
}
```

For components injected into actors by plugins or Game Features, use `UGameFrameworkComponentManager::AddReceiver(AActor* Receiver, bool bAddOnlyInGameWorlds = true)` from ModularGameplay (Beta in 5.8); see `ue-game-features`.

## Deprecated — do not use

| Do not emit | Use in 5.8 | Source |
|---|---|---|
| `FindComponentByInterface<UMyInterface>()` | `FindComponentByInterface<IMyInterface>()` | `UE_DEPRECATED(5.5)` in `GameFramework/Actor.h:3844` |
| `PreDuplicate(FObjectDuplicationParameters&)` | `PreDuplicateFromRoot(FObjectDuplicationParameters& DupParams)` | `UE_DEPRECATED(5.6)` in `GameFramework/Actor.h:2413` |
| `Actor->LastRenderTime` | `GetLastRenderTime()` / `SetLastRenderTime(float)` | `UE_DEPRECATED(5.6)` in `GameFramework/Actor.h:138` |
| `NetUpdateFrequency = 30.f`, `MinNetUpdateFrequency = ...`, `NetCullDistanceSquared = ...` | `SetNetUpdateFrequency()`, `SetMinNetUpdateFrequency()`, `SetNetCullDistanceSquared()` | `UE_DEPRECATED(5.5)` in `GameFramework/Actor.h:898-909` |
| `BeginReplication()` / `EndReplication()` overrides | `OnReplicationStartedForIris` / `OnStopReplicationForIris` / `FillReplicationParams`; see `ue-networking-replication` | `UE_DEPRECATED(5.7)` in `GameFramework/Actor.h:3493-3506` |
| `APawn::GetMovementBase()` | `GetMovementBaseObject()` / `GetMovementBaseInterfaceData()`; see `ue-character-movement` | `UE_DEPRECATED(5.8)` in `GameFramework/Pawn.h:59` |
| `AttachTo(...)`, `K2_AttachTo(...)`, `AttachRootComponentTo(...)` | `AttachToComponent(Parent, FAttachmentTransformRules, SocketName)` | `UE_DEPRECATED(4.17)` in `GameFramework/Actor.h:1991`, `Components/SceneComponent.h:740` |
| `AttachRootComponentToActor(...)` | `AttachToActor(ParentActor, FAttachmentTransformRules, SocketName)` | `UE_DEPRECATED(4.17)` in `GameFramework/Actor.h:2018` |
| `DetachFromParent(bool)`, `DetachRootComponentFromParent(bool)` | `DetachFromComponent(FDetachmentTransformRules)`, `DetachFromActor(FDetachmentTransformRules)` | `UE_DEPRECATED(4.12)` in `Components/SceneComponent.h:768`; `UE_DEPRECATED(4.17)` in `GameFramework/Actor.h:2045` |
| `Comp->bReplicates = true` | `SetIsReplicatedByDefault(true)` (constructor) or `SetIsReplicated(true)` | private member, `Components/ActorComponent.h:264-267` |
| `Comp->bCreatedByConstructionScript` | `Comp->CreationMethod == EComponentCreationMethod::UserConstructionScript` or `IsCreatedByConstructionScript()` | `@deprecated Replaced by CreationMethod`, `Components/ActorComponent.h:306-310` |
| `SceneComp->RelativeLocation = ...`, `SceneComp->bVisible = ...`, `SceneComp->AttachParent` | `SetRelativeLocation()`, `SetVisibility()`, `GetAttachParent()` | private members, `Components/SceneComponent.h:109-183` |
| `Actor->bHidden` | `IsHidden()` / `SetActorHiddenInGame(bool)` | private member, `GameFramework/Actor.h:365` |
| `GetComponentsByClass(...)` in C++ (the C++ name is `K2_GetComponentsByClass`) | `GetComponents<T>(OutArray)` or `TInlineComponentArray<T*>` | "intended to only be used by blueprints", `GameFramework/Actor.h:3801-3807` |
| `LightmapType` on primitives | `GetLightmapType()` / `SetLightmapType()` | `UE_DEPRECATED(5.5)` in `Components/PrimitiveComponent.h:359` |

## Common Mistakes

**World access in the constructor:** `GetWorld()` is `nullptr` on the CDO; `SpawnActor`, timers and lookups belong in `BeginPlay`, component binding in `PostInitializeComponents`.

**Skipping `Super::`:** every lifecycle override calls its `Super::` (`BeginPlay` first thing; `EndPlay(EndPlayReason)` last). `AActor::EndPlay` is what forwards `EndPlay` to components; `RouteEndPlay` ensures against a missing `Super::EndPlay` (`Actor.cpp:3230`).

**Pooled or cached actors outside a `UPROPERTY`:** a plain `TArray<AMyProjectile*>` member is invisible to GC; declare `UPROPERTY() TArray<TObjectPtr<AMyProjectile>>`. For non-owning references in non-UObject code use `TWeakObjectPtr<AActor>` and check `IsValid()`.

**`AttachToComponent` in the constructor:** the header allows it on unregistered components, but `SetupAttachment` is the intended call there (`SceneComponent.h:745-746`); use `AttachToComponent` from `BeginPlay` onward and check its `bool` result.

**`NewObject` without `RegisterComponent`:** the component never gets render or physics state and never ticks. Also call `AddInstanceComponent` if it should appear in the Details panel and survive serialization.

**Deferred spawn without `FinishSpawning`:** the actor stays in a malformed state (`Actor.h:632`); always pair `SpawnActorDeferred` with `FinishSpawning`.

**Calling an interface method directly:** `Target->OnInteract(...)` bypasses Blueprint implementers. Test with `Implements<UMyInteractable>()`, call through `IMyInteractable::Execute_OnInteract(Target, ...)`.

**Polling in `Tick`:** `if (HealthComp->IsDead())` every frame is replaced by binding `OnDeath` once; disable tick with `SetActorTickEnabled(false)` when nothing needs per-frame work.

**Assuming child actors exist in `PostInitializeComponents`:** `UChildActorComponent` children may begin play late (`Actor.h:275`); read `GetChildActor()` in or after `BeginPlay`.

**Writing private component flags:** `bReplicates`, `bIsActive`, `bHasBegunPlay`, `RelativeLocation` and `bVisible` are private; use the setters and queries listed in the tables above.

## Related Skills

- `ue-cpp-foundations` — UCLASS/UPROPERTY/UFUNCTION specifiers, TObjectPtr vs TWeakObjectPtr, subsystems, GC rules
- `ue-gameplay-framework` — GameMode, GameState, PlayerController, Pawn, Character, damage without GAS
- `ue-physics-collision` — collision channels and profiles, overlap and hit events, sweeps, async physics tick
- `ue-networking-replication` — SetReplicates, property replication, OnRep, RPCs, subobject replication, Iris
- `ue-gameplay-cameras` — USpringArmComponent, UCameraComponent, PlayerCameraManager, Gameplay Cameras plugin
- `ue-blueprint-cpp-interop` — Blueprint-implementable interfaces, BlueprintNativeEvent rules, latent actions, meta specifiers
- `ue-character-movement` — CharacterMovementComponent, movement base API, CMC vs Mover
- `ue-animation-system` — USkeletalMeshComponent animation, AnimInstance, montages
- `ue-audio-system` — UAudioComponent playback, attenuation, MetaSounds
- `ue-niagara-effects` — UNiagaraComponent, system spawning, user parameters
- `ue-game-features` — ModularGameplay components, UGameFrameworkComponentManager, Game Feature actions
- `ue-gameplay-abilities` — UAbilitySystemComponent placement on actors
- `ue-state-trees` — UStateTreeComponent on actors and AI controllers
- `ue-ai-navigation` — perception and navigation components
- `ue-procedural-generation` — UProceduralMeshComponent, UDynamicMeshComponent, instancing at scale
- `ue-editor-tools` — detail customizations, editor utility widgets, UToolMenus and editor subsystems
- `ue-input-system` — Enhanced Input actions, mapping contexts, triggers, modifiers and user settings
- `ue-mass-entity` — Mass processors, fragments, queries and entity traits
- `ue-materials-rendering` — material instances, parameter collections, render targets and post process
- `ue-sequencer-cinematics` — Level Sequences, playback, cine cameras and Movie Render Graph
- `ue-world-level-streaming` — World Partition, level streaming, data layers and travel
