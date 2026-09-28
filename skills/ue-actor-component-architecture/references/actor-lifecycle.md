# Actor Lifecycle Reference (UE 5.8)

Event order for `AActor` and `UActorComponent` from spawn or level load through destruction, with the header line that backs each claim. Source of truth: the `AActor` class comment at `Engine/Source/Runtime/Engine/Classes/GameFramework/Actor.h:243-278` plus the declarations cited below (`Actor.h`, `Components/ActorComponent.h`, `Engine/EngineTypes.h`, `Engine/EngineBaseTypes.h`, `Engine/World.h`).

## Initialization order

| # | Hook (verbatim declaration) | Declared at | Fires |
|---|---|---|---|
| 1 | `AActor()` | `Actor.h:288` | On the Class Default Object and on every instance. No world, no other actors. `CreateDefaultSubobject`, `SetRootComponent`, `PrimaryActorTick` config, default values. |
| 2 | `virtual void PostLoad() override` | `Actor.h:2348` | Actors statically placed in a level, in editor and gameplay. Not called for newly spawned actors. |
| 3 | `virtual void OnComponentCreated()` (component) | `ActorComponent.h:1331` | Native components when the actor is spawned in editor or gameplay; Blueprint-created components during construction. Not for components loaded from a level. |
| 4 | `virtual void PreRegisterAllComponents()` | `Actor.h:3232` | Level-placed actors and spawned actors with a native root component; Blueprint actors without a native root get registration later, during construction. |
| 5 | `void RegisterComponent()` → `virtual void OnRegister()` (component) | `ActorComponent.h:1322`, `:830` | Every component, editor and runtime. Creates render and physics state. May be spread over several frames, always after `PreRegisterAllComponents`. Fires again after `UnregisterComponent()` + re-register. |
| 6 | `virtual void PostRegisterAllComponents()` | `Actor.h:3238` | All actors, editor and gameplay. |
| 7 | `virtual void PostActorCreated()` | `Actor.h:2937` | Spawned actors only (editor or gameplay), before construction scripts and after native components exist. Location and rotation are already set when there is a root component. |
| 8 | `void UserConstructionScript()` | `Actor.h:2318` | Blueprint construction script. |
| 9 | `virtual void OnConstruction(const FTransform& Transform)` | `Actor.h:3445` | End of `ExecuteConstruction`, after all Blueprint-created components are created and registered. Gameplay: spawned actors only. Editor: re-runs when the Blueprint changes. |
| 10 | `virtual void PreInitializeComponents()` | `Actor.h:3124` | Gameplay (and some editor previews) only, before `InitializeComponent` on any component. |
| 11 | `virtual void Activate(bool bReset = false)` (component) | `ActorComponent.h:580` | Components with `bAutoActivate` (`:317`) set; also on any later manual activation. |
| 12 | `virtual void InitializeComponent()` (component) | `ActorComponent.h:919` | Components with `bWantsInitializeComponent` (`:340`) set; once per gameplay session. |
| 13 | `virtual void PostInitializeComponents()` | `Actor.h:3127` | Gameplay and some editor previews. All components initialized; world available. |
| 14 | `virtual void BeginPlay()`; components: `virtual void BeginPlay()` | `Actor.h:2125`; `ActorComponent.h:936` | When the level starts ticking, gameplay only. Normally right after `PostInitializeComponents`, "but can be delayed for networked or child actors" (`Actor.h:275`). |

`PostInitProperties()` (`Actor.h:2343`) is the `UObject` hook that runs after the constructor and property initialization, on the CDO as well; keep it for property fix-ups that do not need the world.

## Play

| Hook | Declared at | Notes |
|---|---|---|
| `virtual void Tick(float DeltaSeconds)` | `Actor.h:3060` | Only when `PrimaryActorTick.bCanEverTick` (`Actor.h:3056` comment). Enabled state via `SetActorTickEnabled(bool)` / `IsActorTickEnabled()` (`:2898, :2902`); rate via `SetActorTickInterval(float)` (`:2909`). |
| `virtual void TickComponent(float DeltaTime, enum ELevelTick TickType, FActorComponentTickFunction* ThisTickFunction)` | `ActorComponent.h:976` | Requires registration and `PrimaryComponentTick.bCanEverTick` (`:970` comment). `SetComponentTickEnabled(bool)` (`:1002`). |
| `virtual void AsyncPhysicsTickActor(float DeltaTime, float SimTime)` | `Actor.h:2930` | Fixed-step physics callback when `bAsyncPhysicsTickEnabled` (`Actor.h:699`). Owned by `ue-physics-collision`. |
| `bool HasActorBegunPlay() const`, `bool IsActorInitialized() const` | `Actor.h:2145, :2139` | State queries. |

## Termination

Explicit destruction: `bool Destroy(bool bNetForce = false, bool bShouldModifyLevel = true)` (`Actor.h:2327`) leads to `virtual void Destroyed()` (`Actor.h:3569`, "called when this actor is explicitly being destroyed during gameplay or in the editor, not called during level streaming or gameplay ending"). `AActor::Destroyed()` runs `RouteEndPlay(EEndPlayReason::Destroyed)`, then the Blueprint `ReceiveDestroyed()` event and the `OnDestroyed` delegate (`Actor.cpp:3311-3316`).

`virtual void EndPlay(const EEndPlayReason::Type EndPlayReason)` (`Actor.h:2132`) runs for every reason below. The base implementation calls `EndPlay(EndPlayReason)` on each component that has begun play (`Actor.cpp:3277-3281`), so a missing `Super::EndPlay` skips component cleanup; `RouteEndPlay` ensures against exactly that (`Actor.cpp:3229-3230`). Component teardown continues with `virtual void UninitializeComponent()` (`ActorComponent.h:955`), `virtual void OnUnregister()` (`:835`) and `virtual void OnComponentDestroyed(bool bDestroyingHierarchy)` (`:1338`).

Delegates: `OnEndPlay` (`FActorEndPlaySignature`, `Actor.h:208, :2339`) and `OnDestroyed` (`FActorDestroyedSignature`, `Actor.h:207, :2335`).

### `EEndPlayReason` (`Engine/EngineTypes.h:3670-3685`)

```cpp
namespace EEndPlayReason
{
    enum Type : int
    {
        Destroyed,          // the Actor or Component is explicitly destroyed
        LevelTransition,    // the world is being unloaded for a level transition
        EndPlayInEditor,    // the world is being unloaded because PIE is ending
        RemovedFromWorld,   // the level it is a member of is streamed out
        Quit,               // the application is being exited
    };
}
```

| Reason | Play effects? | Persist state? | Release resources, clear timers, unbind delegates? |
|---|---|---|---|
| `Destroyed` | Optional | If the object is persistent | Yes |
| `LevelTransition` | No | If needed | Yes |
| `EndPlayInEditor` | No | No | Yes |
| `RemovedFromWorld` | No | If needed | Yes |
| `Quit` | No | If needed | Yes |

```cpp
void AMyActor::EndPlay(const EEndPlayReason::Type EndPlayReason)
{
    GetWorldTimerManager().ClearAllTimersForObject(this);   // Actor.h:3769, TimerManager.h
    Super::EndPlay(EndPlayReason);                          // forwards EndPlay to components
}
```

## Tick functions and groups

`FTickFunction` (`Engine/EngineBaseTypes.h:183`): `TickGroup` (`:196`), `EndTickGroup` (`:204`), `bTickEvenWhenPaused` (`:209`), `bCanEverTick` (`:213`), `bStartWithTickEnabled` (`:217`), `bAllowTickOnDedicatedServer` (`:221`), `bHighPriority` (`:227`), `bRunOnAnyThread` (`:230`), `TickInterval` (`:258`); `SetTickFunctionEnable(bool bInEnabled)` (`:364`), `AddPrerequisite(UObject* TargetObject, struct FTickFunction& TargetTickFunction)` (`:426`). Subtypes `FActorTickFunction` (`:566`) and `FActorComponentTickFunction` (`:612`).

`ETickingGroup` (`Engine/EngineBaseTypes.h:83-109`):

```
Frame start
  ├─ TG_PrePhysics       default; input, movement requests, AI decisions
  ├─ TG_StartPhysics     UMETA(Hidden): engine starts the physics step
  ├─ TG_DuringPhysics    runs while physics simulates; do not read this frame's physics results
  ├─ TG_EndPhysics       UMETA(Hidden): engine finishes the physics step
  ├─ TG_PostPhysics      camera, IK, cloth, anything reading final physics transforms
  ├─ TG_PostUpdateWork   last gameplay group
  ├─ TG_LastDemotable    UMETA(Hidden): engine-internal
  └─ TG_NewlySpawned     UMETA(Hidden): engine-internal (actors spawned mid-frame)
Frame end
```

`ELevelTick` (`EngineBaseTypes.h:69-78`): `LEVELTICK_TimeOnly`, `LEVELTICK_ViewportsOnly`, `LEVELTICK_All`, `LEVELTICK_PauseTick`; passed to `TickComponent` as `TickType`.

```cpp
AMyActor::AMyActor()
{
    PrimaryActorTick.bCanEverTick = true;
    PrimaryActorTick.TickGroup = TG_PostPhysics;
}

UMyComponent::UMyComponent()
{
    PrimaryComponentTick.bCanEverTick = true;
    PrimaryComponentTick.TickGroup = TG_PostPhysics;     // or SetTickGroup(TG_PostPhysics) at runtime (ActorComponent.h:1351)
}
```

Cross-object ordering: `AddTickPrerequisiteActor(AActor* PrerequisiteActor)` / `AddTickPrerequisiteComponent(UActorComponent* PrerequisiteComponent)` (`Actor.h:2093, :2097`; `ActorComponent.h:1355, :1359`) and the matching `RemoveTickPrerequisite*` (`Actor.h:2101`; `ActorComponent.h:1363, :1367`).

## Network timing

The single-player order holds on the server. On clients:

| Hook | Declared at | Header comment |
|---|---|---|
| `virtual void PreNetReceive() override` | `Actor.h:2943` | "Always called immediately before properties are received from the remote." |
| `virtual void PostNetReceive() override` | `Actor.h:2946` | "Always called immediately after properties are received from the remote." Fires on every property batch, not only the first. |
| `virtual void PostNetReceiveRole()` | `Actor.h:2949` | "Always called immediately after a new Role is received from the remote." |
| `virtual void PostNetInit()` | `Actor.h:2958` | "Always called immediately after spawning and reading in replicated properties." One-time client-side init hook. |
| `virtual void OnRep_ReplicatedMovement()`, `OnRep_Owner()`, `OnRep_Instigator()`, `OnRep_AttachmentReplication()` | `Actor.h:2962, :600, :1015, :859` | Engine `OnRep_` notifies you may override (call `Super`). |
| `BeginPlay()` | `Actor.h:275` comment | "Can be delayed for networked or child actors." |

Do not assume client `BeginPlay` sees the same initial property values as the server did; initialize from `OnRep_` functions or `PostNetInit` when the logic depends on replicated state. Actor replication is enabled with `SetReplicates(bool bInReplicates)` (`Actor.h:759`; `bReplicates` is protected at `:587-593`), component replication with `SetIsReplicated(bool ShouldReplicate)` (`ActorComponent.h:634`; `bReplicates` is private at `:264-267`) or `SetIsReplicatedByDefault(const bool bNewReplicates)` in the constructor (`:1471`). Property rules, `DOREPLIFETIME`, RPCs and Iris are owned by `ue-networking-replication`.

## Deferred spawn sequence

`UWorld::SpawnActorDeferred<T>(UClass* Class, FTransform const& Transform, AActor* Owner = nullptr, APawn* Instigator = nullptr, ESpawnActorCollisionHandlingMethod CollisionHandlingOverride = ESpawnActorCollisionHandlingMethod::Undefined, ESpawnActorScaleMethod TransformScaleMethod = ESpawnActorScaleMethod::MultiplyWithRoot)` (`World.h:3851-3870`) fills an `FActorSpawnParameters` with `bDeferConstruction = true` and calls `SpawnActor(Class, &Transform, SpawnInfo)`.

```
SpawnActorDeferred<T>()
  → AActor::AActor()                         constructor, CreateDefaultSubobject calls
  → AActor::PostActorCreated()               Actor.h:2937
  [construction deferred: OnConstruction has NOT run, BeginPlay has NOT run]
  [set properties on the actor here]

Actor->FinishSpawning(Transform)             Actor.h:3117
  → UserConstructionScript / OnConstruction(const FTransform&)
  → PreInitializeComponents → InitializeComponent (per component) → PostInitializeComponents
  → BeginPlay                                if the world has begun play
```

`FinishSpawning(const FTransform& Transform, bool bIsDefaultTransform = false, const FComponentInstanceDataCache* InstanceDataCache = nullptr, ESpawnActorScaleMethod TransformScaleMethod = ESpawnActorScaleMethod::OverrideRootScale)` (`Actor.h:3117`). `bHasFinishedSpawning` (`Actor.h:632-633`): "If it has not, the Actor is in a malformed state." The Blueprint-callable twin is `UGameplayStatics::FinishSpawningActor(AActor* Actor, const FTransform& SpawnTransform, ESpawnActorScaleMethod TransformScaleMethod = ESpawnActorScaleMethod::MultiplyWithRoot)` (`Kismet/GameplayStatics.h:71`). Note the three different scale defaults (`MultiplyWithRoot` for the spawn templates and the Kismet helper, `OverrideRootScale` for `AActor::FinishSpawning`); pass the method explicitly when the root has a non-unit default scale.

`ESpawnActorScaleMethod` (`Actor.h:87-94`): `OverrideRootScale`, `MultiplyWithRoot`, `SelectDefaultAtRuntime` (hidden).

## Child actors

`UChildActorComponent` (`Components/ChildActorComponent.h:84`, derives `USceneComponent`) spawns and owns a nested actor at the component transform.

| Member | Line |
|---|---|
| `void SetChildActorClass(TSubclassOf<AActor> InClass)` / `void SetChildActorClass(TSubclassOf<AActor> InClass, AActor* NewChildActorTemplate)` | `:95, :110` |
| `TSubclassOf<AActor> GetChildActorClass() const` | `:112` |
| `AActor* GetChildActor() const`, `AActor* GetChildActorTemplate() const` | `:216, :217` |
| `virtual void CreateChildActor(TFunction<void(AActor*)> CustomizerFunc = nullptr)`, `void DestroyChildActor()` | `:211, :227` |
| `FOnChildActorCreated& OnChildActorCreated()` | `:214` |
| Overrides: `OnRegister`, `OnUnregister`, `BeginPlay`, `OnComponentDestroyed(bool)` | `:196-200` |

From the child's side: `bool IsChildActor() const` (`Actor.h:3185`), `UChildActorComponent* GetParentComponent() const` (`:3222`), `AActor* GetParentActor() const` (`:3226`). Child-actor `BeginPlay` "can be delayed" (`Actor.h:275`); read `GetChildActor()` from the parent's `BeginPlay` or later, not from `PostInitializeComponents`.

## Component lifecycle inside an actor

```
CreateDefaultSubobject<T>(FName) / NewObject<T>(Owner, FName)     Object.h:151, UObjectGlobals.h:1973
  → OnComponentCreated()                 ActorComponent.h:1331   native components on spawn
  → RegisterComponent()                  :1322                   (or automatic for default subobjects)
  → OnRegister()                         :830
  → CreateRenderState_Concurrent(FRegisterComponentContext* Context)   :850   if ShouldCreateRenderState()
  → OnCreatePhysicsState()               :871                    if ShouldCreatePhysicsState()
  → Activate(bool bReset = false)        :580                    if bAutoActivate
  → InitializeComponent()                :919                    if bWantsInitializeComponent
  → BeginPlay()                          :936
  → TickComponent(...)                   :976                    every frame while ticking

  → EndPlay(const EEndPlayReason::Type)  :949
  → UninitializeComponent()              :955
  → OnUnregister()                       :835
  → DestroyRenderState_Concurrent()      :868
  → OnDestroyPhysicsState()              :874
  → DestroyComponent(bool bPromoteChildren = false)   :1328
  → OnComponentDestroyed(bool bDestroyingHierarchy)   :1338
```

State queries (the bit fields behind them are private, `ActorComponent.h:357-371`): `HasBeenCreated()` (`:505`), `HasBeenInitialized()` (`:508`), `HasBegunPlay()` (`:514`), `IsBeingDestroyed()` (`:520`), `IsRegistered()` (`:1316`), `IsActive()` (`:607`), `IsRenderStateCreated()` (`:1172`), `IsPhysicsStateCreated()` (`:1184`).

Re-creating state without destroying the component: `ReregisterComponent()` (`:1347`), `RecreateRenderState_Concurrent()` (`:1166`), `RecreatePhysicsState()` (`:1169`), `MarkRenderStateDirty()` (`:1128`).

How a component came to exist: `EComponentCreationMethod CreationMethod` (`ActorComponent.h:431`) with values `Native`, `SimpleConstructionScript`, `UserConstructionScript`, `Instance` (`Engine/Public/ComponentInstanceDataCache.h:28-34`); helper `bool IsCreatedByConstructionScript() const` (`:526`). Components you add with `NewObject` + `RegisterComponent` + `AActor::AddInstanceComponent(UActorComponent* Component)` (`Actor.h:4348`) land in `InstanceComponents` (`Actor.h:4340`).

## What is safe at each stage

| Stage | `GetWorld()` | Other actors | Own components | Typical work |
|---|---|---|---|---|
| Constructor | `nullptr` on the CDO | No | Create only (`CreateDefaultSubobject`, `SetupAttachment`) | Defaults, tick config |
| `PostLoad` | Yes | No | Read | Data fix-ups for level-placed actors |
| `PostActorCreated` / `OnConstruction` | Yes | Not guaranteed initialized | Yes, including Blueprint-created ones in `OnConstruction` | Procedural component setup that must re-run in the editor |
| `PreInitializeComponents` | Yes | Not guaranteed initialized | Created, not yet initialized | Pre-init setup that cannot live in the constructor |
| `PostInitializeComponents` | Yes | Level-load actors exist; do not assume their `BeginPlay` ran | Initialized | Bind to own components |
| `BeginPlay` | Yes | Yes | Yes | Gameplay logic, spawning, timers |
| `Tick` | Yes | Yes | Yes | Per-frame only |
| `EndPlay` | Yes | May already be ending | Yes | Clear timers, unbind, save |
| `Destroyed` | Yes | Avoid | Partially torn down | Minimal |
