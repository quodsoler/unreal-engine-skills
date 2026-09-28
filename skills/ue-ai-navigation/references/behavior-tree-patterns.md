# Behavior Tree Patterns

Reusable Behavior Tree structures and the C++ that backs them, for UE 5.8 (`AIModule`). Every class, virtual and macro below exists in `Engine/Source/Runtime/AIModule/Classes/BehaviorTree`.

---

## Blackboard Keys (Common Set)

Define these in your `UBlackboardData` asset as a baseline and extend per character:

| Key Name | Type | Description |
|---|---|---|
| `TargetActor` | Object (AActor) | Current enemy/target |
| `LastKnownLocation` | Vector | Last confirmed target location |
| `PatrolLocation` | Vector | Current patrol waypoint |
| `HomeLocation` | Vector | Spawn/home position |
| `InvestigateLocation` | Vector | Sound/sight disturbance location |
| `CoverLocation` | Vector | EQS-found cover position |
| `IsAlerted` | Bool | Has the AI been alerted |
| `IsInCombat` | Bool | Actively fighting |
| `PatrolIndex` | Int | Index into the patrol spline |
| `PatrolWaitTime` | Float | Seconds to wait at the current waypoint |

Nodes should reach these through a `FBlackboardKeySelector` `UPROPERTY`, never a hard-coded `FName`.

---

## Custom task with node memory (full source)

The attack task referenced in the skill body, header and `.cpp`. Per-AI state lives in `FMyAttackTaskMemory`, not in node members.

```cpp
// MyBTTask_Attack.h
#pragma once

#include "CoreMinimal.h"
#include "BehaviorTree/BTTaskNode.h"
#include "BehaviorTree/BehaviorTreeTypes.h"
#include "MyBTTask_Attack.generated.h"

struct FMyAttackTaskMemory
{
    float ElapsedTime = 0.f;
    TWeakObjectPtr<AActor> CachedTarget;
};

UCLASS()
class MYGAME_API UMyBTTask_Attack : public UBTTaskNode
{
    GENERATED_BODY()

public:
    UMyBTTask_Attack();

    UPROPERTY(EditAnywhere, Category = "Attack")
    FBlackboardKeySelector TargetKey;

    UPROPERTY(EditAnywhere, Category = "Attack", meta = (ClampMin = "0.0"))
    float AttackDuration = 1.5f;

protected:
    virtual EBTNodeResult::Type ExecuteTask(UBehaviorTreeComponent& OwnerComp, uint8* NodeMemory) override;
    virtual void TickTask(UBehaviorTreeComponent& OwnerComp, uint8* NodeMemory, float DeltaSeconds) override;
    virtual EBTNodeResult::Type AbortTask(UBehaviorTreeComponent& OwnerComp, uint8* NodeMemory) override;
    virtual uint16 GetInstanceMemorySize() const override { return sizeof(FMyAttackTaskMemory); }
};
```

```cpp
// MyBTTask_Attack.cpp
#include "MyBTTask_Attack.h"

#include "AIController.h"
#include "BehaviorTree/BlackboardComponent.h"

UMyBTTask_Attack::UMyBTTask_Attack()
{
    NodeName = TEXT("Attack");
    INIT_TASK_NODE_NOTIFY_FLAGS();
}

EBTNodeResult::Type UMyBTTask_Attack::ExecuteTask(UBehaviorTreeComponent& OwnerComp, uint8* NodeMemory)
{
    UBlackboardComponent* Blackboard = OwnerComp.GetBlackboardComponent();
    AActor* Target = Blackboard ? Cast<AActor>(Blackboard->GetValueAsObject(TargetKey.SelectedKeyName)) : nullptr;
    if (!IsValid(Target))
    {
        return EBTNodeResult::Failed;
    }

    FMyAttackTaskMemory* Memory = CastInstanceNodeMemory<FMyAttackTaskMemory>(NodeMemory);
    Memory->ElapsedTime = 0.f;
    Memory->CachedTarget = Target;
    return EBTNodeResult::InProgress;
}

void UMyBTTask_Attack::TickTask(UBehaviorTreeComponent& OwnerComp, uint8* NodeMemory, float DeltaSeconds)
{
    FMyAttackTaskMemory* Memory = CastInstanceNodeMemory<FMyAttackTaskMemory>(NodeMemory);
    Memory->ElapsedTime += DeltaSeconds;

    if (!Memory->CachedTarget.IsValid())
    {
        FinishLatentTask(OwnerComp, EBTNodeResult::Failed);
        return;
    }
    if (Memory->ElapsedTime >= AttackDuration)
    {
        FinishLatentTask(OwnerComp, EBTNodeResult::Succeeded);
    }
}

EBTNodeResult::Type UMyBTTask_Attack::AbortTask(UBehaviorTreeComponent& OwnerComp, uint8* NodeMemory)
{
    FMyAttackTaskMemory* Memory = CastInstanceNodeMemory<FMyAttackTaskMemory>(NodeMemory);
    Memory->CachedTarget.Reset();
    return EBTNodeResult::Aborted;
}
```

`CastInstanceNodeMemory<T>()` checks `sizeof(T) <= GetInstanceMemorySize()` and fails the check otherwise. When the memory struct holds non-trivially-destructible members, also override:

```cpp
virtual void InitializeMemory(UBehaviorTreeComponent& OwnerComp, uint8* NodeMemory, EBTMemoryInit::Type InitType) const override;
virtual void CleanupMemory(UBehaviorTreeComponent& OwnerComp, uint8* NodeMemory, EBTMemoryClear::Type CleanupType) const override;
```

and implement them with the `UBTNode` helpers, which placement-new and destroy correctly:

```cpp
void UMyBTTask_Attack::InitializeMemory(UBehaviorTreeComponent& OwnerComp, uint8* NodeMemory, EBTMemoryInit::Type InitType) const
{
    InitializeNodeMemory<FMyAttackTaskMemory>(NodeMemory, InitType);
}

void UMyBTTask_Attack::CleanupMemory(UBehaviorTreeComponent& OwnerComp, uint8* NodeMemory, EBTMemoryClear::Type CleanupType) const
{
    CleanupNodeMemory<FMyAttackTaskMemory>(NodeMemory, CleanupType);
}
```

---

## Custom decorator

```cpp
// MyBTDecorator_HealthBelow.h
#pragma once

#include "CoreMinimal.h"
#include "BehaviorTree/BTDecorator.h"
#include "MyBTDecorator_HealthBelow.generated.h"

UCLASS()
class MYGAME_API UMyBTDecorator_HealthBelow : public UBTDecorator
{
    GENERATED_BODY()

public:
    UMyBTDecorator_HealthBelow();

    UPROPERTY(EditAnywhere, Category = "Condition", meta = (ClampMin = "0.0", ClampMax = "1.0"))
    float HealthThresholdNormalized = 0.3f;

protected:
    virtual bool CalculateRawConditionValue(UBehaviorTreeComponent& OwnerComp, uint8* NodeMemory) const override;
};

// MyBTDecorator_HealthBelow.cpp
#include "MyBTDecorator_HealthBelow.h"

#include "AIController.h"
#include "MyHealthInterface.h"

UMyBTDecorator_HealthBelow::UMyBTDecorator_HealthBelow()
{
    NodeName = TEXT("Health Below");
    INIT_DECORATOR_NODE_NOTIFY_FLAGS();
    bAllowAbortLowerPri = true;
    bAllowAbortChildNodes = true;
    FlowAbortMode = EBTFlowAbortMode::Both;
}

bool UMyBTDecorator_HealthBelow::CalculateRawConditionValue(UBehaviorTreeComponent& OwnerComp, uint8* NodeMemory) const
{
    const AAIController* Controller = OwnerComp.GetAIOwner();
    const APawn* Pawn = Controller ? Controller->GetPawn() : nullptr;
    const IMyHealthInterface* Health = Cast<IMyHealthInterface>(Pawn);
    return Health && Health->GetNormalizedHealth() < HealthThresholdNormalized;
}
```

`CalculateRawConditionValue` is `const` — it must not mutate the blackboard or the node. `bAllowAbortLowerPri` and `bAllowAbortChildNodes` decide which `FlowAbortMode` values the editor details panel offers; both already default to `true` in `UBTDecorator` (`BTDecorator.cpp:14-15`), so setting them here only documents intent (set them `false` to forbid a mode).

Line-of-sight variant: `OwnerComp.GetAIOwner()->LineOfSightTo(Target)` (`AAIController::LineOfSightTo(const AActor* Other, FVector ViewPoint = FVector(ForceInit), bool bAlternateChecks = false) const`).

---

## Custom service

```cpp
// MyBTService_UpdateTarget.h
#pragma once

#include "CoreMinimal.h"
#include "BehaviorTree/BTService.h"
#include "BehaviorTree/BehaviorTreeTypes.h"
#include "MyBTService_UpdateTarget.generated.h"

UCLASS()
class MYGAME_API UMyBTService_UpdateTarget : public UBTService
{
    GENERATED_BODY()

public:
    UMyBTService_UpdateTarget();

    UPROPERTY(EditAnywhere, Category = "Target")
    FBlackboardKeySelector TargetKey;

protected:
    virtual void TickNode(UBehaviorTreeComponent& OwnerComp, uint8* NodeMemory, float DeltaSeconds) override;
};

// MyBTService_UpdateTarget.cpp
#include "MyBTService_UpdateTarget.h"

#include "AIController.h"
#include "BehaviorTree/BlackboardComponent.h"
#include "GameFramework/Pawn.h"
#include "Perception/AIPerceptionComponent.h"

UMyBTService_UpdateTarget::UMyBTService_UpdateTarget()
{
    NodeName = TEXT("Update Target");
    Interval = 0.5f;
    RandomDeviation = 0.1f;
    INIT_SERVICE_NODE_NOTIFY_FLAGS();
}

void UMyBTService_UpdateTarget::TickNode(UBehaviorTreeComponent& OwnerComp, uint8* NodeMemory, float DeltaSeconds)
{
    Super::TickNode(OwnerComp, NodeMemory, DeltaSeconds);

    AAIController* Controller = OwnerComp.GetAIOwner();
    UAIPerceptionComponent* Perception = Controller ? Controller->GetAIPerceptionComponent() : nullptr;
    UBlackboardComponent* Blackboard = OwnerComp.GetBlackboardComponent();
    if (!Perception || !Blackboard || !Controller->GetPawn())
    {
        return;
    }

    TArray<AActor*> Hostiles;
    Perception->GetPerceivedHostileActors(Hostiles);

    const FVector MyLocation = Controller->GetPawn()->GetActorLocation();
    AActor* Best = nullptr;
    double BestDistSq = TNumericLimits<double>::Max();
    for (AActor* Hostile : Hostiles)
    {
        const double DistSq = FVector::DistSquared(MyLocation, Hostile->GetActorLocation());
        if (DistSq < BestDistSq)
        {
            BestDistSq = DistSq;
            Best = Hostile;
        }
    }
    Blackboard->SetValueAsObject(TargetKey.SelectedKeyName, Best);
}
```

`UBTService::TickNode` already has an implementation in the base class, so call `Super::TickNode(OwnerComp, NodeMemory, DeltaSeconds)`. Override `OnSearchStart(FBehaviorTreeSearchData& SearchData)` when the service must refresh data at the moment the branch is selected rather than on its interval.

---

## Pattern 1: Patrol / Chase / Attack

The root Selector tries Combat first (highest priority), then the alerted search, then idle patrol.

```
Root [Selector]
├── [Sequence] "Combat" — BTDecorator_Blackboard(TargetActor, IsSet, AbortBoth)
│   ├── [Service] MyBTService_UpdateTarget (0.5 s interval)
│   ├── BTTask_MoveTo (TargetActor key, AcceptableRadius=150)
│   └── [Sequence] "Attack Sequence"
│       ├── BTTask_RotateToFaceBBEntry (TargetActor)
│       └── MyBTTask_Attack
│           └── BTDecorator_Cooldown (2.0 s)
│
├── [Sequence] "Investigate" — BTDecorator_Blackboard(InvestigateLocation, IsSet, AbortBoth)
│   ├── BTTask_MoveTo (InvestigateLocation, AcceptableRadius=100)
│   ├── BTTask_Wait (2.0 s)
│   └── MyBTTask_ClearInvestigateLocation
│
└── [Sequence] "Patrol"
    ├── MyBTTask_GetNextSplinePoint (writes PatrolLocation + PatrolWaitTime)
    ├── BTTask_MoveTo (PatrolLocation, AcceptableRadius=50)
    └── BTTask_WaitBlackboardTime (BlackboardKey=PatrolWaitTime)
```

Perception feeds the keys from the controller:

```cpp
void AMyAIController::HandleTargetPerceptionUpdated(AActor* Actor, FAIStimulus Stimulus)
{
    UBlackboardComponent* BlackboardComp = GetBlackboardComponent();
    if (!BlackboardComp)
    {
        return;
    }

    if (Stimulus.WasSuccessfullySensed() && Stimulus.Type == UAISense::GetSenseID<UAISense_Sight>())
    {
        BlackboardComp->SetValueAsObject(TEXT("TargetActor"), Actor);
        BlackboardComp->SetValueAsVector(TEXT("LastKnownLocation"), Stimulus.StimulusLocation);
    }
    else if (Stimulus.WasSuccessfullySensed() && Stimulus.Type == UAISense::GetSenseID<UAISense_Hearing>())
    {
        if (!BlackboardComp->GetValueAsObject(TEXT("TargetActor")))
        {
            BlackboardComp->SetValueAsVector(TEXT("InvestigateLocation"), Stimulus.StimulusLocation);
        }
    }
    else
    {
        // Lost the target: keep the last known location so the search branch can run.
        BlackboardComp->SetValueAsVector(TEXT("LastKnownLocation"), Stimulus.StimulusLocation);
    }
}
```

Name the local `BlackboardComp`, not `Blackboard`: `AAIController` has a protected `Blackboard` member (`AIController.h:148`), and a local that shadows it fails the build (C4458, which UE treats as an error).

---

## Pattern 2: Combat with Cover and Repositioning

```
Root [Selector]
├── [Sequence] "Take Cover" — BTDecorator_Blackboard(IsInCombat, IsSet, AbortBoth)
│   ├── [Service] BTService_RunEQS (CoverQuery → CoverLocation, 1.0 s)
│   ├── BTTask_MoveTo (CoverLocation, AcceptableRadius=80)
│   └── [Selector] "Attack or Wait"
│       ├── [Sequence] "Peek and Shoot"
│       │   ├── BTDecorator_Blackboard(TargetActor, IsSet)
│       │   ├── BTTask_RotateToFaceBBEntry (TargetActor)
│       │   └── MyBTTask_Attack
│       │       └── BTDecorator_Cooldown (1.5 s)
│       └── BTTask_Wait (1.0 s)
│
└── [Sequence] "Find Enemy"
    ├── BTTask_RunEQSQuery (LastKnownLocationSearch → PatrolLocation)
    ├── BTTask_MoveTo (PatrolLocation, AcceptableRadius=100)
    └── BTTask_Wait (3.0 s)
```

`UBTService_RunEQS` ships with the engine (`BehaviorTree/Services/BTService_RunEQS.h`) — prefer it over a hand-written service that calls `FEnvQueryRequest::Execute` on a timer. If you do write your own, capture nothing raw: use `FQueryFinishedSignature::CreateWeakLambda(Controller, Lambda)` so the callback dies with the controller.

---

## Pattern 3: Flee (health threshold trigger)

```
Root [Selector]
├── [Sequence] "Flee" — MyBTDecorator_HealthBelow(0.3, AbortBoth)
│   ├── BTTask_RunEQSQuery (FleePointQuery → PatrolLocation)
│   │    ← Donut generator around self + Trace test (no LoS from TargetActor)
│   │      + Pathfinding test (PathExist)
│   ├── BTTask_MoveTo (PatrolLocation, AcceptableRadius=100)
│   └── BTTask_Wait (5.0 s)
│
└── [Sequence] "Normal Combat" (the combat branch from Pattern 1)
```

---

## Pattern 4: Investigate sound / disturbance

```
Root [Selector]
├── [Sequence] "Investigate" — BTDecorator_Blackboard(InvestigateLocation, IsSet, AbortBoth)
│   ├── BTTask_MoveTo (InvestigateLocation, AcceptableRadius=100)
│   ├── MyBTTask_LookAround
│   │   └── BTDecorator_TimeLimit (5.0 s)
│   └── MyBTTask_ClearBBKey (InvestigateLocation)
│
├── [Sequence] "Combat" (as Pattern 1)
└── [Sequence] "Patrol" (as Pattern 1)
```

A look-around task rotates the pawn by driving focus, then finishes latently:

```cpp
void UMyBTTask_LookAround::TickTask(UBehaviorTreeComponent& OwnerComp, uint8* NodeMemory, float DeltaSeconds)
{
    FMyLookAroundMemory* Memory = CastInstanceNodeMemory<FMyLookAroundMemory>(NodeMemory);
    Memory->ElapsedTime += DeltaSeconds;

    AAIController* Controller = OwnerComp.GetAIOwner();
    APawn* Pawn = Controller ? Controller->GetPawn() : nullptr;
    if (!Pawn)
    {
        FinishLatentTask(OwnerComp, EBTNodeResult::Failed);
        return;
    }

    const float NewYaw = Memory->StartYaw + (Memory->ElapsedTime / LookDuration) * 360.f;
    Controller->SetFocalPoint(Pawn->GetActorLocation() + FRotator(0.f, NewYaw, 0.f).Vector() * 500.f);

    if (Memory->ElapsedTime >= LookDuration)
    {
        Controller->ClearFocus(EAIFocusPriority::Gameplay);
        FinishLatentTask(OwnerComp, EBTNodeResult::Succeeded);
    }
}
```

---

## Pattern 5: Patrol along a spline

```
Root [Sequence]
└── [Sequence + BTDecorator_Loop(bInfiniteLoop=true)] "Patrol Loop"
    ├── MyBTTask_GetNextSplinePoint (writes PatrolLocation + PatrolWaitTime)
    ├── BTTask_MoveTo (PatrolLocation, AcceptableRadius=50)
    └── BTTask_WaitBlackboardTime (BlackboardKey=PatrolWaitTime)
```

```cpp
EBTNodeResult::Type UMyBTTask_GetNextSplinePoint::ExecuteTask(UBehaviorTreeComponent& OwnerComp, uint8* NodeMemory)
{
    UBlackboardComponent* Blackboard = OwnerComp.GetBlackboardComponent();
    if (!Blackboard || !IsValid(PatrolSplineActor))
    {
        return EBTNodeResult::Failed;
    }

    USplineComponent* Spline = PatrolSplineActor->FindComponentByClass<USplineComponent>();
    if (!Spline || Spline->GetNumberOfSplinePoints() == 0)
    {
        return EBTNodeResult::Failed;
    }

    const int32 NextIndex = (Blackboard->GetValueAsInt(TEXT("PatrolIndex")) + 1) % Spline->GetNumberOfSplinePoints();
    Blackboard->SetValueAsInt(TEXT("PatrolIndex"), NextIndex);
    Blackboard->SetValueAsVector(TEXT("PatrolLocation"),
        Spline->GetLocationAtSplinePoint(NextIndex, ESplineCoordinateSpace::World));
    Blackboard->SetValueAsFloat(TEXT("PatrolWaitTime"), WaitTimeAtPoint);
    return EBTNodeResult::Succeeded;
}
```

---

## Pattern 6: Squad AI with a shared blackboard

Mark `TargetActor` and `AlertLevel` **Instance Synced** in the `UBlackboardData` asset (`FBlackboardEntry::bInstanceSynced`). Every `UBlackboardComponent` using that asset then receives the value when any one of them sets it, so a squad leader writing `TargetActor` alerts the whole squad, and each member's tree reacts independently through `UBTDecorator_Blackboard`. `UBlackboardData::HasSynchronizedKeys()` reports whether an asset has any synced key.

---

## Pattern 7: Message-driven task (`WaitForMessage`)

Use this when a task waits on an external event (animation notify, ability end, timer) instead of polling.

```cpp
EBTNodeResult::Type UMyBTTask_PlayMontage::ExecuteTask(UBehaviorTreeComponent& OwnerComp, uint8* NodeMemory)
{
    AAIController* Controller = OwnerComp.GetAIOwner();
    ACharacter* Character = Controller ? Cast<ACharacter>(Controller->GetPawn()) : nullptr;
    UAnimInstance* AnimInstance = Character ? Character->GetMesh()->GetAnimInstance() : nullptr;
    if (!AnimInstance || !MontageToPlay)
    {
        return EBTNodeResult::Failed;
    }

    WaitForMessage(OwnerComp, TEXT("MontageCompleted"));
    WaitForMessage(OwnerComp, TEXT("MontageFailed"));
    AnimInstance->Montage_Play(MontageToPlay);
    return EBTNodeResult::InProgress;
}

void UMyBTTask_PlayMontage::OnMessage(UBehaviorTreeComponent& OwnerComp, uint8* NodeMemory,
    FName Message, int32 RequestID, bool bSuccess)
{
    StopWaitingForMessages(OwnerComp);
    FinishLatentTask(OwnerComp, Message == TEXT("MontageCompleted") ? EBTNodeResult::Succeeded : EBTNodeResult::Failed);
}
```

Senders:

```cpp
// From any gameplay code that has the pawn:
FAIMessage::Send(Pawn, FAIMessage(TEXT("MontageCompleted"), this, true));

// Blueprint-facing equivalent:
UAIBlueprintHelperLibrary::SendAIMessage(Pawn, TEXT("MontageCompleted"), nullptr, true);
```

`FAIMessage::Send` has overloads for `AController*`, `APawn*` and `UBrainComponent*`; `UBehaviorTreeComponent::HandleMessage(const FAIMessage&)` is the receiving end.

---

## Decorator abort modes

| `EBTFlowAbortMode::Type` | Behavior |
|---|---|
| `None` | Never aborts a running task |
| `Self` | Aborts tasks inside its own subtree when the condition changes |
| `LowerPriority` | Aborts lower-priority branches when the condition becomes true |
| `Both` | Self plus LowerPriority |

```cpp
bAllowAbortLowerPri = true;
bAllowAbortChildNodes = true;
FlowAbortMode = EBTFlowAbortMode::Both;
```

`UBTDecorator::UpdateFlowAbortMode()` clamps the mode to what the parent composite allows (`CanAbortLowerPriority()` / `CanAbortSelf()`), and `IsFlowAbortModeValid()` reports whether the current mode fits; both only do work in editor builds (`BTDecorator.cpp:147-203`). From an event handler inside the decorator, trigger a re-evaluation with:

```cpp
ConditionalFlowAbort(OwnerComp, EBTDecoratorAbortRequest::ConditionResultChanged);
```

From outside the tree, call `UBehaviorTreeComponent::RequestBranchEvaluation(EBTNodeResult::Type)` or the full `RequestExecution(const UBTCompositeNode*, int32, const UBTNode*, int32, EBTNodeResult::Type, bool)`.

---

## Composites and control flow

| Composite | Succeeds when | Notes |
|---|---|---|
| `UBTComposite_Selector` | the first child succeeds | Fallback chain; ordering is priority |
| `UBTComposite_Sequence` | every child succeeds | Stops at the first failure |
| `UBTComposite_SimpleParallel` | the main task finishes | One main task plus one background subtree; `FinishMode` decides whether the background branch is allowed to complete |

There is no general-purpose parallel composite in 5.8. For N concurrent branches, chain `SimpleParallel` nodes or write a `UBTCompositeNode` subclass.

---

## Debugging

Open the Gameplay Debugger (default key `'`) with a possessed pawn and switch categories to see the behavior tree, blackboard, perception, navmesh and EQS panels. `ai.debug.DrawPaths` and `ai.debug.DrawOverheadIcons` toggle AI path and icon drawing; `ai.debug.nav.DisplaySize` and `ai.debug.nav.DrawDistance` control navmesh drawing; `ai.debug.EQS.RefreshInterval` controls the EQS panel refresh rate.
