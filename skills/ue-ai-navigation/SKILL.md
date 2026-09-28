---
name: ue-ai-navigation
description: "Use when writing Unreal Engine C++ AI — AI controllers, behavior tree nodes, blackboards, AI perception, navmesh pathfinding, EQS queries or Smart Objects. Also use when the user mentions 'AAIController', 'UBTTaskNode', 'ExecuteTask', 'FinishLatentTask', 'UBlackboardComponent', 'FBlackboardKeySelector', 'UAIPerceptionComponent', 'FAIStimulus', 'UAISenseConfig_Sight', 'MoveTo', 'UNavigationSystemV1', 'ProjectPointToNavigation', 'UNavArea', 'ANavLinkProxy', 'UEnvQueryManager', 'FEnvQueryRequest', 'EQS', 'navmesh', 'USmartObjectSubsystem' or 'ZoneGraph'. For State Tree authoring, see ue-state-trees; for Mass crowds, see ue-mass-entity; for the movement component that executes paths, see ue-character-movement."
metadata:
  version: "2.0.0"
  engine: "5.8"
---

# UE AI and Navigation

Target engine: **UE 5.8**. APIs below are verified against the 5.8 headers; older forms are listed under "Deprecated — do not use".

Classic UE AI runs on a server-only `AAIController` that owns a brain component (`UBehaviorTreeComponent`), a `UBlackboardComponent`, a `UAIPerceptionComponent` and a `UPathFollowingComponent`. Behavior trees, perception and EQS live in the `AIModule`; navmesh generation and queries live in `NavigationSystem`; BT tasks that wrap gameplay tasks need `GameplayTasks`. Smart Objects (`SmartObjectsModule`, plugin `SmartObjects`), ZoneGraph (`ZoneGraph`, Experimental in 5.8), MassAI (Experimental in 5.8) and NavCorridor (Experimental in 5.8) are opt-in plugins. Add what you use to `PublicDependencyModuleNames` in your `.Build.cs`, e.g. `"AIModule", "NavigationSystem", "GameplayTasks"`.

## Context

Read `.agents/ue-project-context.md` if it exists (module names, conventions, enabled plugins, GAS/networking setup). Do not stop if it is missing.

Identify the area from the request and the codebase. Ask only when two plausible readings would produce different code.

| Request is about… | Go to |
|---|---|
| Possessing a pawn, running a tree, move requests, focus | [AI Controller](#ai-controller) |
| Reading/writing AI knowledge, key selectors, observers | [Blackboard](#blackboard) |
| Custom tasks, decorators, services, node memory | [Behavior Tree Nodes](#behavior-tree-nodes) |
| Sight/hearing/damage senses, stimuli sources | [AI Perception](#ai-perception) |
| Path queries, projection, random points, invokers | [Navigation and Pathfinding](#navigation-and-pathfinding) |
| Area costs, query filters, jump/off-mesh links | [Nav Areas, Filters and Off-Mesh Links](#nav-areas-filters-and-off-mesh-links) |
| Cover/flank/patrol point selection, custom contexts | [EQS Environment Query System](#eqs-environment-query-system) |
| State Tree instead of a behavior tree | [State Tree for AI](#state-tree-for-ai) |
| Interactable world objects, slot claiming | [Smart Objects](#smart-objects) |
| Lane-based crowds, large agent counts | [ZoneGraph, MassAI and NavCorridor](#zonegraph-massai-and-navcorridor) |

## AI Controller

```cpp
// MyAIController.h
#pragma once

#include "CoreMinimal.h"
#include "AIController.h"
#include "Perception/AIPerceptionTypes.h"
#include "MyAIController.generated.h"

class UBehaviorTree;

UCLASS()
class MYGAME_API AMyAIController : public AAIController
{
    GENERATED_BODY()

public:
    AMyAIController();

protected:
    UPROPERTY(EditDefaultsOnly, Category = "AI")
    TObjectPtr<UBehaviorTree> BehaviorTreeAsset;

    virtual void OnPossess(APawn* InPawn) override;
    virtual void OnMoveCompleted(FAIRequestID RequestID, const FPathFollowingResult& Result) override;

    UFUNCTION()
    void HandleTargetPerceptionUpdated(AActor* Actor, FAIStimulus Stimulus);
};

// MyAIController.cpp
#include "MyAIController.h"
#include "BehaviorTree/BlackboardComponent.h"
#include "Navigation/PathFollowingComponent.h"
#include "Perception/AIPerceptionComponent.h"

AMyAIController::AMyAIController()
{
    bStartAILogicOnPossess = true;
    bStopAILogicOnUnposses = true;
}

void AMyAIController::OnPossess(APawn* InPawn)
{
    Super::OnPossess(InPawn);
    if (BehaviorTreeAsset)
    {
        RunBehaviorTree(BehaviorTreeAsset); // calls UseBlackboard internally
    }
    if (UAIPerceptionComponent* Perception = GetAIPerceptionComponent())
    {
        Perception->OnTargetPerceptionUpdated.AddDynamic(this, &AMyAIController::HandleTargetPerceptionUpdated);
    }
}

void AMyAIController::OnMoveCompleted(FAIRequestID RequestID, const FPathFollowingResult& Result)
{
    Super::OnMoveCompleted(RequestID, Result);
    if (UBlackboardComponent* BB = GetBlackboardComponent()) { BB->SetValueAsBool(TEXT("ReachedGoal"), Result.Code == EPathFollowingResult::Success); }
}

void AMyAIController::HandleTargetPerceptionUpdated(AActor* Actor, FAIStimulus Stimulus)
{
    UBlackboardComponent* BlackboardComp = GetBlackboardComponent();
    if (BlackboardComp && Stimulus.WasSuccessfullySensed())
    {
        BlackboardComp->SetValueAsObject(TEXT("TargetActor"), Actor);
        BlackboardComp->SetValueAsVector(TEXT("LastKnownLocation"), Stimulus.StimulusLocation);
    }
}
```

```cpp
// AIController.h — signatures you will call
EPathFollowingRequestResult::Type MoveToActor(AActor* Goal, float AcceptanceRadius = -1, bool bStopOnOverlap = true,
    bool bUsePathfinding = true, bool bCanStrafe = true,
    TSubclassOf<UNavigationQueryFilter> FilterClass = {}, bool bAllowPartialPath = true);

EPathFollowingRequestResult::Type MoveToLocation(const FVector& Dest, float AcceptanceRadius = -1, bool bStopOnOverlap = true,
    bool bUsePathfinding = true, bool bProjectDestinationToNavigation = false, bool bCanStrafe = true,
    TSubclassOf<UNavigationQueryFilter> FilterClass = {}, bool bAllowPartialPath = true);

virtual FPathFollowingRequestResult MoveTo(const FAIMoveRequest& MoveRequest, FNavPathSharedPtr* OutPath = nullptr);
virtual void SetFocus(AActor* NewFocus, EAIFocusPriority::Type InPriority = EAIFocusPriority::Gameplay);
virtual void SetFocalPoint(FVector NewFocus, EAIFocusPriority::Type InPriority = EAIFocusPriority::Gameplay);
virtual void ClearFocus(EAIFocusPriority::Type InPriority);
virtual bool RunBehaviorTree(UBehaviorTree* BTAsset);
bool UseBlackboard(UBlackboardData* BlackboardAsset, UBlackboardComponent*& BlackboardComponent);
EPathFollowingStatus::Type GetMoveStatus() const;   // Idle, Waiting, Paused, Moving
bool HasPartialPath() const;
virtual void SetGenericTeamId(const FGenericTeamId& NewTeamID) override;   // IGenericTeamAgentInterface
```

| Need | Use |
|---|---|
| One-liner move with per-call options | `MoveToActor` / `MoveToLocation` — returns `EPathFollowingRequestResult::{Failed, AlreadyAtGoal, RequestSuccessful}` |
| Full control (goal actor + radius + filter + partial path) and the resulting path | `MoveTo(FAIMoveRequest, FNavPathSharedPtr*)` — returns `FPathFollowingRequestResult` with `MoveId` and `Code` |
| Move a controller that has no BT/blackboard, e.g. a click-to-move test | `UAIBlueprintHelperLibrary::SimpleMoveToLocation(AController*, const FVector&)` |
| React to arrival | Override `OnMoveCompleted`, or bind the dynamic delegate `ReceiveMoveCompleted` (`FAIRequestID`, `EPathFollowingResult::Type`) |

Build an `FAIMoveRequest` with its setters: `SetGoalActor`, `SetGoalLocation`, `SetAcceptanceRadius`, `SetUsePathfinding`, `SetAllowPartialPath`, `SetNavigationFilter`, `SetProjectGoalLocation`, `SetCanStrafe`, `SetReachTestIncludesAgentRadius`.

On the pawn: `AIControllerClass = AMyAIController::StaticClass(); AutoPossessAI = EAutoPossessAI::PlacedInWorldOrSpawned;`

## Blackboard

| Type | Get | Set |
|---|---|---|
| Object | `GetValueAsObject` | `SetValueAsObject` |
| Vector | `GetValueAsVector` | `SetValueAsVector` |
| Bool | `GetValueAsBool` | `SetValueAsBool` |
| Float | `GetValueAsFloat` | `SetValueAsFloat` |
| Int | `GetValueAsInt` | `SetValueAsInt` |
| Enum | `GetValueAsEnum` (returns `uint8`) | `SetValueAsEnum` (takes `uint8`) |
| Name / String / Class / Rotator | `GetValueAsName` / `…String` / `…Class` / `…Rotator` | `SetValueAs…` matching |

All accessors take `const FName& KeyName`. `ClearValue(FName)` and `IsVectorValueSet(FName)` have `FBlackboard::FKey` overloads that skip the name lookup.

```cpp
#include "BehaviorTree/BlackboardComponent.h"

UBlackboardComponent* BlackboardComp = GetBlackboardComponent();
BlackboardComp->SetValueAsObject(TEXT("TargetActor"), TargetActor);
BlackboardComp->ClearValue(TEXT("TargetActor"));

// Observers: the delegate returns EBlackboardNotificationResult, it is not void.
const FBlackboard::FKey KeyID = BlackboardComp->GetKeyID(TEXT("TargetActor"));
const FDelegateHandle Handle = BlackboardComp->RegisterObserver(KeyID, this,
    FOnBlackboardChangeNotification::CreateUObject(this, &AMyAIController::HandleTargetKeyChanged));
BlackboardComp->UnregisterObserver(KeyID, Handle);

// Declared in AMyAIController — returning void will not bind:
EBlackboardNotificationResult HandleTargetKeyChanged(const UBlackboardComponent& BlackboardComp, FBlackboard::FKey ChangedKeyID);
```

Return `EBlackboardNotificationResult::ContinueObserving` to stay registered, `RemoveObserver` to unsubscribe. `UnregisterObserversFrom(this)` drops every observer an object registered.

**In BT nodes, never hard-code key names.** Expose a `FBlackboardKeySelector` `UPROPERTY` and read `SelectedKeyName`; the tree resolves it against the asset through `FBlackboardKeySelector::ResolveSelectedKey(const UBlackboardData&)`, and `GetSelectedKeyID()` gives the cached `FBlackboard::FKey`. Mark a key **Instance Synced** (`FBlackboardEntry::bInstanceSynced` in the `UBlackboardData` asset) to share its value across every AI using that asset — squad-wide alerts without extra plumbing; `UBlackboardData::HasSynchronizedKeys()` reports whether any key is synced. For hot paths, cache the key: `FBBKeyCachedAccessor<UBlackboardKeyType_Bool>` built from `(const UBlackboardComponent&, FBlackboard::FKey)` exposes `Get()` and `SetValue(UBlackboardComponent&, Value)` and skips the per-call name lookup.

## Behavior Tree Nodes

| Base class | Override | Signature source |
|---|---|---|
| `UBTTaskNode` | `ExecuteTask`, `AbortTask`, `TickTask`, `OnTaskFinished`, `OnMessage` | `BehaviorTree/BTTaskNode.h` |
| `UBTDecorator` | `CalculateRawConditionValue` (const), `OnNodeActivation`, `OnNodeDeactivation`, `OnNodeProcessed` | `BehaviorTree/BTDecorator.h` |
| `UBTService` | `TickNode`, `OnSearchStart`, `OnBecomeRelevant`, `OnCeaseRelevant` | `BehaviorTree/BTService.h` (relevance hooks inherited from `BTAuxiliaryNode.h`) |
| Any node | `GetInstanceMemorySize`, `InitializeMemory`, `CleanupMemory` | `BehaviorTree/BTNode.h` |

Copy these verbatim (only the class name changes):

```cpp
virtual EBTNodeResult::Type ExecuteTask(UBehaviorTreeComponent& OwnerComp, uint8* NodeMemory) override;
virtual EBTNodeResult::Type AbortTask(UBehaviorTreeComponent& OwnerComp, uint8* NodeMemory) override;
virtual void TickTask(UBehaviorTreeComponent& OwnerComp, uint8* NodeMemory, float DeltaSeconds) override;
virtual void OnTaskFinished(UBehaviorTreeComponent& OwnerComp, uint8* NodeMemory, EBTNodeResult::Type TaskResult) override;
virtual void OnMessage(UBehaviorTreeComponent& OwnerComp, uint8* NodeMemory, FName Message, int32 RequestID, bool bSuccess) override;
virtual bool CalculateRawConditionValue(UBehaviorTreeComponent& OwnerComp, uint8* NodeMemory) const override;
virtual void OnNodeActivation(FBehaviorTreeSearchData& SearchData) override;
virtual void OnNodeDeactivation(FBehaviorTreeSearchData& SearchData, EBTNodeResult::Type NodeResult) override;
virtual void TickNode(UBehaviorTreeComponent& OwnerComp, uint8* NodeMemory, float DeltaSeconds) override;
virtual void OnSearchStart(FBehaviorTreeSearchData& SearchData) override;
virtual uint16 GetInstanceMemorySize() const override;
virtual void InitializeMemory(UBehaviorTreeComponent& OwnerComp, uint8* NodeMemory, EBTMemoryInit::Type InitType) const override;
virtual void CleanupMemory(UBehaviorTreeComponent& OwnerComp, uint8* NodeMemory, EBTMemoryClear::Type CleanupType) const override;
```

`EBTNodeResult::Type` has exactly four values: `Succeeded`, `Failed`, `Aborted`, `InProgress`. There is no `FBTNodeResult`.

### Custom task with node memory

Per-AI runtime state goes in a plain struct (for example `FMyAttackTaskMemory { float ElapsedTime; TWeakObjectPtr<AActor> CachedTarget; }`), never in `UPROPERTY` members of the node. The task returns `sizeof` of that struct from `GetInstanceMemorySize()`. The full `UMyBTTask_Attack` header and `.cpp` (`FBlackboardKeySelector` key, `ExecuteTask` / `TickTask` / `AbortTask`) are in [behavior tree patterns](references/behavior-tree-patterns.md#custom-task-with-node-memory-full-source).

**Node memory rules.** `NodeMemory` is a raw byte block sized by `GetInstanceMemorySize()` and laid out per BT instance, so it is the only safe place for per-AI runtime state on a shared node object. Access it with `CastInstanceNodeMemory<T>()`, which asserts `sizeof(T) <= GetInstanceMemorySize()`. For non-trivial members override `InitializeMemory`/`CleanupMemory` and call the helpers `InitializeNodeMemory<T>(NodeMemory, InitType)` / `CleanupNodeMemory<T>(NodeMemory, CleanupType)`; they placement-new and destroy correctly. Tasks that opt into tick intervals get a `FBTTaskMemory` header (`NextTickRemainingTime`, `AccumulatedDeltaTime`) reachable through `GetSpecialNodeMemory<FBTTaskMemory>()`; the engine places it just before `NodeMemory` (`BTNode.h:390`), so it never overlaps your struct.

Call `INIT_TASK_NODE_NOTIFY_FLAGS()` / `INIT_DECORATOR_NODE_NOTIFY_FLAGS()` / `INIT_SERVICE_NODE_NOTIFY_FLAGS()` in the constructor: they set `bNotifyTick`, `bNotifyTaskFinished`, `bNotifyActivation` and friends from which virtuals you actually overrode. Without them (or setting the `bNotify*` flags by hand) task `TickTask`/`OnTaskFinished` and decorator/service activation and relevance overrides are never called (`BTTaskNode.cpp:12-13`); only `UBTService` defaults `bNotifyTick` and `bNotifyOnSearch` to true (`BTService.cpp:11-12`). Returning `InProgress` means the task is latent — finish it with `FinishLatentTask(OwnerComp, Result)` (or `FinishLatentAbort(OwnerComp)` from an abort). To wait on an external event instead of ticking, call `WaitForMessage(OwnerComp, FName)` and finish inside `OnMessage`; senders use `FAIMessage::Send(Pawn, FAIMessage(TEXT("MontageCompleted"), Sender, true))` or `UAIBlueprintHelperLibrary::SendAIMessage`.

To force a re-evaluation from outside the tree, use `UBehaviorTreeComponent::RequestExecution(const UBTCompositeNode* RequestedOn, int32 InstanceIdx, const UBTNode* RequestedBy, int32 RequestedByChildIndex, EBTNodeResult::Type ContinueWithResult, bool bStoreForDebugger = true)`; from a decorator prefer `ConditionalFlowAbort(OwnerComp, EBTDecoratorAbortRequest::ConditionResultChanged)` and let `FlowAbortMode` decide the scope.

Built-in nodes worth reusing before writing your own: tasks `UBTTask_MoveTo`, `UBTTask_MoveDirectlyToward`, `UBTTask_Wait`, `UBTTask_WaitBlackboardTime`, `UBTTask_RunEQSQuery`, `UBTTask_PlayAnimation`, `UBTTask_PlaySound`, `UBTTask_MakeNoise`, `UBTTask_RotateToFaceBBEntry`, `UBTTask_RunBehavior`, `UBTTask_RunBehaviorDynamic`, `UBTTask_SetKeyValueBool` / `UBTTask_SetKeyValueFloat` / `UBTTask_SetKeyValueObject` (one per key type), `UBTTask_FinishWithResult`; decorators `UBTDecorator_Blackboard`, `UBTDecorator_CompareBBEntries`, `UBTDecorator_Cooldown`, `UBTDecorator_TagCooldown`, `UBTDecorator_Loop`, `UBTDecorator_LoopUntil`, `UBTDecorator_TimeLimit`, `UBTDecorator_DoesPathExist`, `UBTDecorator_IsAtLocation`, `UBTDecorator_ReachedMoveGoal`, `UBTDecorator_CheckGameplayTagsOnActor`, `UBTDecorator_ConeCheck`, `UBTDecorator_ForceSuccess`; services `UBTService_DefaultFocus`, `UBTService_RunEQS`. Composites ship as `UBTComposite_Selector`, `UBTComposite_Sequence` and `UBTComposite_SimpleParallel` only — there is no general-purpose parallel composite.

Tree topologies, a full decorator and service, abort modes and message-driven tasks: [behavior tree patterns](references/behavior-tree-patterns.md).

## AI Perception

```cpp
#include "Perception/AIPerceptionComponent.h"
#include "Perception/AISenseConfig_Hearing.h"
#include "Perception/AISenseConfig_Sight.h"

// In AMyAIController::AMyAIController():
UAIPerceptionComponent* Perception = CreateDefaultSubobject<UAIPerceptionComponent>(TEXT("AIPerception"));
SetPerceptionComponent(*Perception);

UAISenseConfig_Sight* Sight = CreateDefaultSubobject<UAISenseConfig_Sight>(TEXT("Sight"));
Sight->SightRadius = 2000.f;
Sight->LoseSightRadius = 2500.f;
Sight->PeripheralVisionAngleDegrees = 60.f;
Sight->AutoSuccessRangeFromLastSeenLocation = 400.f;
Sight->DetectionByAffiliation.bDetectEnemies = true;
Perception->ConfigureSense(*Sight);
Perception->SetDominantSense(Sight->GetSenseImplementation());

UAISenseConfig_Hearing* Hearing = CreateDefaultSubobject<UAISenseConfig_Hearing>(TEXT("Hearing"));
Hearing->HearingRange = 3000.f;
Hearing->DetectionByAffiliation.bDetectEnemies = true;
Perception->ConfigureSense(*Hearing);
```

Changing sense config at runtime takes effect only after `RequestStimuliListenerUpdate()`.

| Delegate | Signature | Fires |
|---|---|---|
| `OnTargetPerceptionUpdated` | `(AActor* Actor, FAIStimulus Stimulus)` | once per processed stimulus that changed state, i.e. per actor *per sense* (`AIPerceptionComponent.cpp:593`) |
| `OnTargetPerceptionInfoUpdated` | `(const FActorPerceptionUpdateInfo& UpdateInfo)` | same, but carries `TargetId` so it still fires for destroyed/null actors |
| `OnTargetPerceptionForgotten` | `(AActor* Actor)` | after `ForgetActor` / `ForgetAll`, or when all stimuli expire — age-out only with `UAISystem::bForgetStaleActors` enabled (`AISystem.h:84`) |
| `OnPerceptionUpdated` | `(const TArray<AActor*>& UpdatedActors)` | once per stimuli-processing pass that updated anything, batched |

`FAIStimulus` carries `Strength`, `StimulusLocation`, `ReceiverLocation`, `Tag`, `Type` (an `FAISenseID`), plus `WasSuccessfullySensed()`, `IsActive()` and `GetAge()`. Branch per sense with `Stimulus.Type == UAISense::GetSenseID<UAISense_Sight>()`. Queries: `GetCurrentlyPerceivedActors(TSubclassOf<UAISense> SenseToUse, TArray<AActor*>& OutActors)`, `GetPerceivedHostileActors(TArray<AActor*>&)`, `GetPerceivedHostileActorsBySense(TSubclassOf<UAISense>, TArray<AActor*>&)`, `HasActiveStimulus(const AActor&, FAISenseID)`, `ForgetActor(AActor*)`, `ForgetAll()`.

Events are pushed, not polled: `UAISense_Hearing::ReportNoiseEvent(WorldContextObject, NoiseLocation, Loudness, Instigator, MaxRange, Tag)` and `UAISense_Damage::ReportDamageEvent(WorldContextObject, DamagedActor, Instigator, DamageAmount, EventLocation, HitLocation, Tag)`. Sight has no report function — it is trace-driven and only sees registered sources. `UAISense_Prediction::RequestPawnPredictionEvent(APawn* Requestor, AActor* PredictedActor, float PredictionTime)` asks where a target will be.

Make an actor perceivable either by adding `UAIPerceptionStimuliSourceComponent` and setting `bAutoRegisterAsSource = true` (or calling `RegisterWithPerceptionSystem()`) plus `RegisterForSense(UAISense_Sight::StaticClass())` per sense, or by calling `UAIPerceptionSystem::GetCurrent(World)->RegisterSource(*Actor)` (all instantiated senses) / `RegisterSourceForSenseClass(UAISense_Sight::StaticClass(), *Actor)` — both take `AActor&` (`Perception/AIPerceptionSystem.h:139-141`).

Other sense configs: `UAISenseConfig_Touch`, `UAISenseConfig_Team` (propagates awareness between teammates via `IGenericTeamAgentInterface`), `UAISenseConfig_Prediction`.

## Navigation and Pathfinding

```cpp
#include "NavigationPath.h"
#include "NavigationSystem.h"

UNavigationSystemV1* NavSys = UNavigationSystemV1::GetCurrent(GetWorld());
if (!NavSys) { return; }

FNavLocation RandomPoint;
const bool bFoundRandom = NavSys->GetRandomReachablePointInRadius(Origin, 1500.f, RandomPoint);

FNavLocation Projected;
const bool bOnNavMesh = NavSys->ProjectPointToNavigation(WorldLocation, Projected, FVector(500.f, 500.f, 500.f));

// Blueprint-facing helpers, equally usable from C++ (all static, all take a world context):
UNavigationPath* Path = UNavigationSystemV1::FindPathToLocationSynchronously(GetWorld(), Start, End, GetPawn());
const bool bReachable = Path && Path->IsValid() && !Path->IsPartial();

FVector HitLocation;
const bool bBlocked = UNavigationSystemV1::NavigationRaycast(GetWorld(), Start, End, HitLocation);

// Native path, with the full FNavPathPoint list:
const FPathFindingQuery Query(this, *NavSys->GetDefaultNavDataInstance(), Start, End);
const FPathFindingResult Result = NavSys->FindPathSync(Query);
const int32 NumPoints = Result.IsSuccessful() ? Result.Path->GetPathPoints().Num() : 0;
```

`UNavigationPath` exposes `PathPoints`, `GetPathLength()`, `IsValid()` and `IsPartial()`; `FPathFindingResult` exposes `Path`, `IsSuccessful()` and `IsPartial()`.

Async: `FindPathAsync(const FNavAgentProperties&, FPathFindingQuery, const FNavPathQueryDelegate&, EPathFindingMode::Type)`; the delegate is `DECLARE_DELEGATE_ThreeParams(FNavPathQueryDelegate, uint32 QueryID, ENavigationQueryResult::Type, FNavPathSharedPtr)`, so a handler with any other signature will not compile.

| Situation | Do this |
|---|---|
| No navmesh at all | Place an `ANavMeshBoundsVolume`; without one no tiles generate |
| Procedural / runtime geometry | Set the navmesh Runtime Generation to Dynamic; registering or moving navigation-relevant components dirties and rebuilds only the touched tiles (`AddDirtyArea`, `NavigationSystem.h:973`). `NavSys->Build()` rebuilds all nav data — use it once after bulk generation, not per change |
| Open world / streamed levels | Add `UNavigationInvokerComponent` to AI pawns and set `TileGenerationRadius` / `TileRemovalRadius` (or `SetGenerationRadii`) |
| Flying or swimming agents | Set `FNavAgentProperties::bCanFly` / `bCanSwim` on the movement component's `NavAgentProps`; the agent only uses nav data whose flags match |
| Crowd avoidance between many agents | Use `UCrowdFollowingComponent` and `SetCrowdSimulationState`, `SetCrowdAvoidanceQuality`, `SetCrowdSeparationWeight`, `SetCrowdCollisionQueryRange` |

Agent generation settings live on `ARecastNavMesh` (`Cast<ARecastNavMesh>(NavSys->GetDefaultNavDataInstance())`): `AgentRadius`, `AgentHeight`, `AgentMaxSlope`, `TileSizeUU`, and per-resolution step height through `GetAgentMaxStepHeight(ENavigationDataResolution)` / `SetAgentMaxStepHeight(ENavigationDataResolution, float)`. Tiles are identified by `FNavTileRef`, not raw indices — `GetNavMeshTileBounds(FNavTileRef)` and `GetNavMeshTileXY(FNavTileRef, int32&, int32&, int32&)` are the current overloads.

## Nav Areas, Filters and Off-Mesh Links

Subclass `UNavArea` per traversal domain and set `DefaultCost`, `FixedAreaEnteringCost` and `DrawColor`. Built-in areas: `UNavArea_Default`, `UNavArea_Null` (unwalkable), `UNavArea_Obstacle`, `UNavArea_LowHeight`, and `UNavAreaMeta_SwitchByAgent` for per-agent substitution. Paint areas at runtime with `UNavModifierComponent` (`AreaClass`, optional `AreaClassToReplace`, `FailsafeExtent`, `bIncludeAgentHeight`, and the setters `SetAreaClass` / `SetAreaClassToReplace`) or in-level with `ANavModifierVolume`.

**Per-mesh walkable areas.** `UNavCollisionBase` has two mutually exclusive modes: `bIsDynamicObstacle` paints an obstacle modifier after generation, while `bUseSurfaceArea` makes `UNavCollision::AreaClass` flow into rasterization, so the mesh's own triangles become that area. Use the surface mode when a mesh *is* the terrain type (mud, water, rooftop) rather than an obstacle on it.

A query filter is a `UNavigationQueryFilter` subclass — it needs a real class body, not a forward declaration:

```cpp
// MyNavFilter.h
#pragma once

#include "CoreMinimal.h"
#include "NavFilters/NavigationQueryFilter.h"
#include "MyNavFilter.generated.h"

UCLASS()
class MYGAME_API UMyNavFilter : public UNavigationQueryFilter
{
    GENERATED_BODY()

public:
    UMyNavFilter();
};

// MyNavFilter.cpp
#include "MyNavFilter.h"
#include "NavAreas/NavArea_Obstacle.h"

UMyNavFilter::UMyNavFilter()
{
    FNavigationFilterArea& AvoidObstacles = Areas.AddDefaulted_GetRef();
    AvoidObstacles.AreaClass = UNavArea_Obstacle::StaticClass();
    AvoidObstacles.bIsExcluded = true;
}
```

Pass it as the `FilterClass` argument: `MoveToActor(Target, -1.f, true, true, true, UMyNavFilter::StaticClass());`. Each `FNavigationFilterArea` entry can instead override cost with `bOverrideTravelCost` + `TravelCostOverride` or `bOverrideEnteringCost` + `EnteringCostOverride`. `UNavigationQueryFilter::GetQueryFilter(NavData, Querier, FilterClass)` resolves the runtime `FSharedConstNavQueryFilter`; override `InitializeFilter(const ANavigationData&, const UObject*, FNavigationQueryFilter&)` for dynamic rules.

**Off-mesh links.** Drop an `ANavLinkProxy` in the level: its `PointLinks` array holds simple links, `SegmentLinks` holds segment links, and its `UNavLinkCustomComponent` (via `GetSmartLinkComp()`) gives you the smart-link path — `SetLinkData(RelativeStart, RelativeEnd, ENavLinkDirection::Type)`, `SetEnabled(bool)`, `SetEnabledArea(TSubclassOf<UNavArea>)`, and `SetMoveReachedLink(FOnMoveReachedLink const&)` to run code (a jump, a ladder climb) when an agent reaches the link. `ANavLinkProxy::OnSmartLinkReached` is the Blueprint-facing equivalent. Recast can also generate jump links automatically from the `NavLinkJumpConfigs` array of `FNavLinkGenerationJumpConfig` on `ARecastNavMesh`.

## EQS Environment Query System

```cpp
#include "EnvironmentQuery/EnvQueryManager.h"
#include "EnvironmentQuery/EnvQueryTypes.h"

// AMyAIController declares: TObjectPtr<UEnvQuery> FindCoverQuery and
// void HandleCoverQueryFinished(TSharedPtr<FEnvQueryResult> Result).
void AMyAIController::RunCoverQuery()
{
    if (!FindCoverQuery) { return; }

    // Querier = pawn; a controller querier puts the Querier context at the controller (EnvQueryContext_Querier.cpp:19-21)
    FEnvQueryRequest Request(FindCoverQuery, GetPawn());
    Request.SetFloatParam(TEXT("SearchRadius"), 1200.f);
    Request.Execute(EEnvQueryRunMode::SingleResult, this, &AMyAIController::HandleCoverQueryFinished);
}

void AMyAIController::HandleCoverQueryFinished(TSharedPtr<FEnvQueryResult> Result)
{
    UBlackboardComponent* BlackboardComp = GetBlackboardComponent();
    if (BlackboardComp && Result.IsValid() && Result->IsSuccessful())
    {
        BlackboardComp->SetValueAsVector(TEXT("CoverLocation"), Result->GetItemAsLocation(0));
    }
}
```

`FQueryFinishedSignature` is `DECLARE_DELEGATE_OneParam(FQueryFinishedSignature, TSharedPtr<FEnvQueryResult>)`. Run modes: `SingleResult`, `RandomBest5Pct`, `RandomBest25Pct`, `AllMatching`. Results expose `Items`, `GetItemAsLocation(int32)`, `GetItemAsActor(int32)`, `GetItemScore(int32)`, `IsSuccessful()`, `IsAborted()`. Other entry points: `UEnvQueryManager::GetCurrent(WorldContextObject)->RunQuery(Request, RunMode, FinishDelegate)` returns the query id for cancellation; `RunInstantQuery(Request, RunMode)` blocks and returns the result on the spot (use sparingly); `UEnvQueryManager::RunEQSQuery(WorldContextObject, QueryTemplate, Querier, RunMode, WrapperClass)` is the Blueprint wrapper.

Subclass hooks, all const: `UEnvQueryGenerator::GenerateItems(FEnvQueryInstance&)`, `UEnvQueryTest::RunTest(FEnvQueryInstance&)`, `UEnvQueryContext::ProvideContext(FEnvQueryInstance&, FEnvQueryContextData&)`. Blueprint contexts derive from `UEnvQueryContext_BlueprintBase` and implement `ProvideSingleActor`, `ProvideSingleLocation`, `ProvideActorsSet` or `ProvideLocationsSet`.

Place an `AEQSTestingPawn` (`EnvironmentQuery/EQSTestingPawn.h`) in the level to preview a query's scored items in-editor without PIE; set its `QueryTemplate` and `QueryConfig`.

Generator and test property tables, a custom C++ context, scoring recipes and ready-made query configurations: [EQS reference](references/eqs-reference.md).

## State Tree for AI

Behavior trees suit reactive combat with priority-based interrupts; State Trees suit flatter state machines and Smart Object flows. For AI, use `UStateTreeAIComponent` (from `Components/StateTreeAIComponent.h`) rather than the plain `UStateTreeComponent`: it returns `UStateTreeAIComponentSchema`, which guarantees an `AAIController` context value. Build.cs needs `StateTreeModule` **and** `GameplayStateTreeModule`.

Task, condition, evaluator and schema authoring belongs to `ue-state-trees`.

## Smart Objects

`SmartObjects` is an opt-in plugin (`SmartObjectsModule`). Put a `USmartObjectComponent` on world actors and give it a `USmartObjectDefinition` (`GetDefinition()` / `SetDefinition()`); AI finds and claims slots through `USmartObjectSubsystem`.

```cpp
#include "SmartObjectSubsystem.h"
#include "SmartObjectRequestTypes.h" // FSmartObjectRequest, FSmartObjectRequestFilter, FSmartObjectRequestResult

USmartObjectSubsystem* Subsystem = USmartObjectSubsystem::GetCurrent(GetWorld());
if (!Subsystem) { return; }

FSmartObjectRequestFilter Filter;
const FSmartObjectRequest Request(FBox::BuildAABB(Origin, FVector(500.f)), Filter);
const FSmartObjectRequestResult Result = Subsystem->FindSmartObject(Request, GetPawn());
if (Result.IsValid())
{
    const FSmartObjectClaimHandle Claim = Subsystem->MarkSlotAsClaimed(Result.SlotHandle, ESmartObjectClaimPriority::Normal);
    FVector SlotLocation;
    if (Subsystem->GetSlotLocation(Claim, SlotLocation))
    {
        MoveToLocation(SlotLocation);
    }
    Subsystem->MarkSlotAsFree(Claim); // in real code, only once the interaction ends
}
```

`ESmartObjectClaimPriority` values are `Low`, `BelowNormal`, `Normal`, `AboveNormal`, `High`. `FindSmartObjects` returns every match; `FindSmartObjectsInList` restricts the search to actors you already have. The `GameplayBehaviorSmartObjects` plugin (Experimental in 5.8) adds `GameplayBehavior`-driven slot behaviours, and `UBlackboardKeyType_SOClaimHandle` lets a BT hold a claim handle in a blackboard key.

## ZoneGraph, MassAI and NavCorridor

| Plugin | Maturity in 5.8 | Use it for | Entry points |
|---|---|---|---|
| ZoneGraph | Experimental | Lane graphs for traffic and pedestrian flow instead of free navmesh movement | `UZoneGraphSubsystem::FindNearestLane` / `FindOverlappingLanes`, `AZoneGraphData`, `UZoneShapeComponent`, `UE::ZoneGraph::Query` helpers |
| MassAI | Experimental | Thousands of lightweight agents with no actor per agent | `UMassStateTreeProcessor`, `FMassStateTreeExecutionContext`, modules `MassAIBehavior`, `MassNavigation`, `MassZoneGraphNavigation`, `MassNavMeshNavigation` |
| NavCorridor | Experimental | Steering inside a corridor around a navmesh path rather than along its polyline | `FNavCorridor`, `FNavCorridorParams`, `FNavCorridorPortal`, `FNavCorridorLocation` |

Mass agent authoring, fragments and processors belong to `ue-mass-entity`.

## Deprecated — do not use

| Do not emit | Use in 5.8 | Source |
|---|---|---|
| `EPointOnCircleSpacingMethod` | `EEnvQueryPointSpacingMethod` | `UE_DEPRECATED(5.8)` in `EnvironmentQuery/Generators/EnvQueryGenerator_OnCircle.h:15` |
| `USmartObjectComponent::SetSmartObjectEnabled` (BlueprintPure) | `K2_SetSmartObjectEnabled` | `UE_DEPRECATED(5.8)` in `SmartObjectComponent.h:101` |
| `USmartObjectComponent::SetSmartObjectEnabledForReason` (BlueprintPure) | `K2_SetSmartObjectEnabledForReason` | `UE_DEPRECATED(5.8)` in `SmartObjectComponent.h:118` |
| `FNavLinkGenerationJumpDownConfig` | `FNavLinkGenerationJumpConfig` | `UE_DEPRECATED(5.7)` in `NavMesh/LinkGenerationConfig.h:159` |
| `UAIPerceptionComponent::RefreshStimulus` | `ConditionallyStoreSuccessfulStimulus(FAIStimulus&, const FAIStimulus&)` | `UE_DEPRECATED(5.6)` in `Perception/AIPerceptionComponent.h:448` |
| `UPathFollowingComponent::SetMovementComponent(UNavMovementComponent*)` | `SetNavMovementInterface(INavMovementInterface*)` (the deprecation message's `SetNavMoveInterface` does not exist) | `UE_DEPRECATED(5.5)` in `Navigation/PathFollowingComponent.h:282`; replacement at `:286` |
| `ARecastNavMesh` tile functions taking `int32 TileIndex` | overloads taking `FNavTileRef` | `UE_DEPRECATED(5.5)` in `NavMesh/RecastNavMesh.h:1134,1141,1151,1161` |
| `ARecastNavMesh::AgentMaxStepHeight` | `GetAgentMaxStepHeight(ENavigationDataResolution)` / `NavMeshResolutionParams` | `UE_DEPRECATED(all)` in `NavMesh/RecastNavMesh.h:718` |
| `UBehaviorTreeComponent::RequestExecution(const UBTDecorator*)` | `RequestBranchEvaluation` | "replaced by" comment, `BehaviorTree/BehaviorTreeComponent.h:153` |
| `UNavigationSystemV1::K2_GetRandomPointInNavigableRadius` | `K2_GetRandomLocationInNavigableRadius` | `DeprecatedFunction` in `NavigationSystem.h:1484` |

## Common Mistakes

**Forgetting the notify-flags macro:** a `TickTask` or `OnSearchStart` override never runs unless the constructor calls `INIT_TASK_NODE_NOTIFY_FLAGS()` / `INIT_SERVICE_NODE_NOTIFY_FLAGS()` — the flags, not the override, decide what the tree calls.

**Per-instance state as a member:** BT node objects are shared by every AI running the tree, so `float ElapsedTime;` as a `UPROPERTY` is a cross-agent data race. Put it in a struct sized by `GetInstanceMemorySize()` and reach it with `CastInstanceNodeMemory<T>()`.

**Returning `InProgress` and never finishing:** the task hangs forever. Every latent path must end in `FinishLatentTask` or `FinishLatentAbort`.

**Blackboard mistakes:** `FOnBlackboardChangeNotification` returns `EBlackboardNotificationResult`, so a void observer will not bind; and BT nodes should expose a `FBlackboardKeySelector` rather than hard-coding key names, so designers can rebind and `ResolveSelectedKey` can validate against the asset.

**No `ANavMeshBoundsVolume`:** with no bounds volume no tiles generate and every `MoveTo` fails silently. For procedural geometry use Dynamic runtime generation (plus one `NavSys->Build()` after bulk generation), or `UNavigationInvokerComponent`.

**Assuming the AI exists on clients:** `AAIController` is server-only. Replicate results through the pawn's replicated properties; the blackboard and behavior tree are not replicated.

**Polling instead of throttling:** prefer `UBTDecorator_Blackboard` with `FlowAbortMode` or `WaitForMessage` over per-frame checks; keep service `Interval` at or above 0.5 s with a `RandomDeviation`; gate EQS with `UBTDecorator_Cooldown`, prefer `SingleResult` over `AllMatching`, and put cheap filter tests before trace and pathfinding tests.

## Related Skills

- `ue-state-trees` — State Tree tasks, conditions, evaluators, schemas and Mass behaviours
- `ue-mass-entity` — Mass fragments, processors and crowd simulation for large agent counts
- `ue-character-movement` — the movement component that executes the paths this skill requests
- `ue-gameplay-framework` — GameMode spawning, controller/pawn ownership, possession lifecycle
- `ue-physics-collision` — trace channels and collision responses used by sight and EQS trace tests
- `ue-gameplay-tags-messaging` — gameplay tags used by `UBTDecorator_CheckGameplayTagsOnActor` and EQS tag tests
- `ue-actor-component-architecture` — component creation, ownership and replication patterns
- `ue-mover` — the Mover plugin: movement modes, layered moves and rollback networking
- `ue-testing-debugging` — automation tests, logging, assertions, profiling and debug drawing
