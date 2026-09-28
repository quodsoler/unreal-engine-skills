# EQS Reference

Environment Query System generators, tests and query recipes for UE 5.8. Property names are taken from `Engine/Source/Runtime/AIModule/Classes/EnvironmentQuery`; the editor shows the same names.

---

## Architecture

```
UEnvQuery (data asset)
  └── Options (UEnvQueryOption)
        ├── Generator (UEnvQueryGenerator) — produces candidate items (locations or actors)
        └── Tests (UEnvQueryTest array) — filter and/or score the items
```

| Type | Role |
|---|---|
| `UEnvQuery` | the query asset |
| `UEnvQueryManager` | AI subsystem (`UAISubsystem`, `EnvQueryManager.h:207`) that runs queries (`GetCurrent`, `RunQuery`, `RunInstantQuery`, `RunEQSQuery`) |
| `FEnvQueryRequest` | one request: template + owner + named params |
| `FEnvQueryResult` | `Items`, `GetItemAsLocation(int32)`, `GetItemAsActor(int32)`, `GetItemScore(int32)`, `IsSuccessful()`, `IsAborted()` |
| `UEnvQueryContext_Querier` | built-in context: the querier |
| `UEnvQueryContext_Item` | built-in context: the item under test |
| `UEnvQueryContext_NavigationData` | built-in context: the nav data to use |
| `UEnvQueryItemType_Point` / `UEnvQueryItemType_Actor` | item payload types, each with `SetContextHelper` |

Most numeric generator and test properties are `FAIDataProviderFloatValue` / `FAIDataProviderIntValue` / `FAIDataProviderBoolValue`, so each can be a constant or driven by a named query parameter set through `FEnvQueryRequest::SetFloatParam` / `SetIntParam` / `SetBoolParam`.

---

## Generators

### `UEnvQueryGenerator_SimpleGrid`

Uniform grid of points projected onto the navmesh.

| Property | Type | Notes |
|---|---|---|
| `GridSize` | Float provider | Full extent of the grid in cm |
| `SpaceBetween` | Float provider | Spacing between points in cm |
| `GenerateAround` | Context class | Usually `UEnvQueryContext_Querier` |

Cover search: `GridSize=1600, SpaceBetween=150`. Wide patrol search: `GridSize=4000, SpaceBetween=300`. Item count grows with the square of `GridSize / SpaceBetween` — this is the single biggest EQS cost lever.

### `UEnvQueryGenerator_Donut`

Concentric rings (an annulus), optionally restricted to an arc.

| Property | Type | Notes |
|---|---|---|
| `Center` | Context class | Ring centre |
| `InnerRadius` / `OuterRadius` | Float providers | Annulus bounds |
| `NumberOfRings` | Int provider | Rings between inner and outer |
| `PointsPerRing` | Int provider | Points on each ring when `PointOnRingSpacingMethod` is `ByNumberOfPoints` (the default); `BySpaceBetween` uses `SpaceBetweenPoints` instead (`EnvQueryGenerator_Donut.h:32-40`) |
| `bDefineArc` | bool | Restrict to an arc |
| `ArcDirection` | `FEnvDirection` | Arc forward direction (`LineFrom`/`LineTo` or `Rotation` + `DirMode`) |
| `ArcAngle` | Float provider | Arc width in degrees |

Use for flanking positions (centred on the enemy) and retreat points (centred on self).

### `UEnvQueryGenerator_OnCircle`

Points on a single circle.

| Property | Type | Notes |
|---|---|---|
| `CircleRadius` | Float provider | Circle radius |
| `PointOnCircleSpacingMethod` | `EEnvQueryPointSpacingMethod` | `BySpaceBetween` or `ByNumberOfPoints` |
| `SpaceBetween` | Float provider | Used when the method is `BySpaceBetween` |
| `NumberOfPoints` | Int provider | Used when the method is `ByNumberOfPoints` |

`EEnvQueryPointSpacingMethod` lives in `Generators/EnvQueryGenerator_ProjectedPoints.h`.

### `UEnvQueryGenerator_PathingGrid`

`SimpleGrid` that also runs a path query, so results are navigable. Adds:

| Property | Type | Notes |
|---|---|---|
| `PathToItem` | Bool provider | Path direction: to the item, or from it |
| `NavigationFilter` | `TSubclassOf<UNavigationQueryFilter>` | Filter used for the path queries |
| `ScanRangeMultiplier` | Float provider | Widens the scan range used for exploration |

Much more expensive than `SimpleGrid`; prefer a cheap grid plus a `Pathfinding` filter test unless you need true reachability from the generator.

### `UEnvQueryGenerator_ActorsOfClass`

| Property | Type | Notes |
|---|---|---|
| `SearchedActorClass` | `TSubclassOf<AActor>` | Class to gather |
| `GenerateOnlyActorsInRadius` | Bool provider | False returns every actor of the class in the world |
| `SearchRadius` | Float provider | Radius when the bool above is true |
| `SearchCenter` | Context class | Centre of the radius |

Use for patrol waypoints, interaction targets and pickups.

### Other generators

| Class | Produces |
|---|---|
| `UEnvQueryGenerator_Cone` | Points in a cone (`CenterActor`, `ConeDegrees`, `AngleStep`, `Range`, `AlignedPointsDistance`) |
| `UEnvQueryGenerator_CurrentLocation` | A single point at a context |
| `UEnvQueryGenerator_PerceivedActors` | Actors this AI currently perceives (`AllowedActorClass`, `SearchRadius`, `ListenerContext`, `SenseToUse`) |
| `UEnvQueryGenerator_Composite` | Merges the output of its `Generators` array |
| `UEnvQueryGenerator_BlueprintBase` | Blueprint-authored generator |

---

## Test properties shared by every test

| Property | Type | Notes |
|---|---|---|
| `TestPurpose` | `EEnvTestPurpose::{Filter, Score, FilterAndScore}` | What the test does |
| `FilterType` | `EEnvTestFilterType::{Minimum, Maximum, Range, Match}` | `Match` is for boolean tests, the rest are numeric |
| `FloatValueMin` / `FloatValueMax` | Float providers | Numeric filter bounds |
| `BoolValue` | Bool provider | Expected value for `Match` filters |
| `ScoringEquation` | `EEnvTestScoreEquation::{Linear, Square, InverseLinear, SquareRoot, Constant}` | Curve applied to the normalized test value |
| `ScoringFactor` | Float provider | Weight; negative inverts the contribution |
| `ClampMinType` / `ClampMaxType`, `ScoreClampMin` / `ScoreClampMax` | `EEnvQueryTestClamping` + float providers | Clamp the normalization range |
| `ReferenceValue` / `bDefineReferenceValue` | Float provider + bool | Score relative to a target value instead of the min/max of the set |
| `MultipleContextFilterOp` / `MultipleContextScoreOp` | `EEnvTestFilterOperator` / `EEnvTestScoreOperator` | How to combine results when a context resolves to several items |

"Find the nearest" is `ScoringEquation=InverseLinear`; "find the farthest" is `Linear`.

---

## Tests

### `UEnvQueryTest_Distance`

| Property | Type | Notes |
|---|---|---|
| `TestMode` | `EEnvTestDistance::{Distance3D, Distance2D, DistanceZ, DistanceAbsoluteZ}` | |
| `DistanceTo` | Context class | The other end of the measurement |

### `UEnvQueryTest_Trace`

| Property | Type | Notes |
|---|---|---|
| `TraceData` | `FEnvTraceData` | `TraceChannel` (`ETraceTypeQuery`), `TraceShape` (`EEnvTraceShape`), `TraceMode` (`EEnvQueryTrace`), `ExtentX/Y/Z`, `bTraceComplex`, `bOnlyBlockingHits`, `ProjectDown`/`ProjectUp`, `NavigationFilter` |
| `Context` | Context class | The other end of the trace |
| `TraceFromContext` | Bool provider | Trace from the context to the item, or the reverse |
| `ItemHeightOffset` / `ContextHeightOffset` | Float providers | Raise the endpoints to eye height |

Cover: `TestPurpose=Filter`, `FilterType=Match`, `BoolValue=false` (the item must **not** be visible from the enemy context). Firing position: `BoolValue=true`.

### `UEnvQueryTest_Dot`

| Property | Type | Notes |
|---|---|---|
| `LineA` / `LineB` | `FEnvDirection` | Each is either two contexts (`LineFrom`, `LineTo`) or a `Rotation` context, selected by `DirMode` |
| `TestMode` | `EEnvTestDot::{Dot3D, Dot2D}` | `Dot2D` is the heading comparison |
| `bAbsoluteValue` | bool | Treat -1 and 1 alike |

Flanking: score `InverseLinear` on the absolute dot between the enemy's facing and the enemy→item direction, so perpendicular positions win.

### `UEnvQueryTest_Pathfinding` and `UEnvQueryTest_PathfindingBatch`

| Property | Type | Notes |
|---|---|---|
| `TestMode` | `EEnvTestPathfinding::{PathExist, PathCost, PathLength}` | |
| `Context` | Context class | Path endpoint |
| `PathFromContext` | Bool provider | Path direction |
| `SkipUnreachable` | Bool provider | Discard items with no path |
| `FilterClass` | `TSubclassOf<UNavigationQueryFilter>` | Filter for the path query |
| `NavDataOverrideContext` | Context class | Use a specific nav data |

`PathExist` is far cheaper than `PathLength`. Use the batch variant when many items share a start point.

### `UEnvQueryTest_Overlap`

`OverlapData` is an `FEnvOverlapData`: `ExtentX/Y/Z`, `OverlapShape` (`EEnvOverlapShape`), `OverlapChannel` (`ECollisionChannel`), `bOnlyBlockingHits`, `bOverlapComplex`, `bSkipOverlapQuerier`. Use it to check that an agent actually fits at a candidate point.

### `UEnvQueryTest_GameplayTags`

`TagQueryToMatch` is an `FGameplayTagQuery`; items are cast to `IGameplayTagAssetInterface` and tested against it. Author it in the query asset: `SetTagQueryToMatch(FGameplayTagQuery&)` exists but is not exported (`MinimalAPI` class, no `AIMODULE_API`, `EnvQueryTest_GameplayTags.h:29`), so calling it from a game module fails to link, and it has no effect once the query has been cached.

### `UEnvQueryTest_Volume`, `UEnvQueryTest_Project`, `UEnvQueryTest_Random`

`Volume` filters by containment in a `VolumeClass` (`bDoComplexVolumeTest`, `bSkipTestIfNoVolumes`). `Project` re-projects items onto the navmesh or geometry. `Random` adds jitter so equally-scored items are not always resolved identically.

---

## Query recipes

### Nearest cover

```
Generator: SimpleGrid            GridSize = 1600, SpaceBetween = 150, GenerateAround = Querier
Test 1: Distance (Filter)        DistanceTo = Querier, TestMode = Distance2D,
                                 FilterType = Maximum, FloatValueMax = 1500
Test 2: Trace (Filter)           Context = MyEQSContext_Enemy, TraceFromContext = true,
                                 FilterType = Match, BoolValue = false
Test 3: Distance (Score)         DistanceTo = Querier, ScoringEquation = InverseLinear, ScoringFactor = 1.0
Test 4: Pathfinding (Filter)     TestMode = PathExist, Context = Querier, SkipUnreachable = true
```

Order matters: cheap distance filters first, traces next, path tests last.

### Flee point

```
Generator: Donut                 Center = Querier, InnerRadius = 500, OuterRadius = 2000,
                                 NumberOfRings = 3, PointsPerRing = 12
Test 1: Distance (Score)         DistanceTo = MyEQSContext_Enemy, ScoringEquation = Linear
Test 2: Trace (Score)            Context = MyEQSContext_Enemy, FilterType = Match,
                                 BoolValue = false, TestPurpose = Score, ScoringFactor = 0.5
Test 3: Pathfinding (Filter)     TestMode = PathExist, Context = Querier, SkipUnreachable = true
```

### Flanking position

```
Generator: Donut                 Center = MyEQSContext_Enemy, InnerRadius = 500, OuterRadius = 700,
                                 NumberOfRings = 1, PointsPerRing = 12
Test 1: Dot (Score)              LineA = { LineFrom: MyEQSContext_Enemy, LineTo: Item },
                                 LineB = { Rotation: MyEQSContext_Enemy }, TestMode = Dot2D,
                                 bAbsoluteValue = true, ScoringEquation = InverseLinear
Test 2: Trace (Filter)           Context = MyEQSContext_Enemy, FilterType = Match, BoolValue = true
Test 3: Pathfinding (Filter)     TestMode = PathExist, Context = Querier
```

### Patrol waypoint

```
Generator: ActorsOfClass         SearchedActorClass = AMyPatrolPoint,
                                 GenerateOnlyActorsInRadius = true, SearchRadius = 5000,
                                 SearchCenter = Querier
Test 1: Distance (FilterAndScore) DistanceTo = Querier, FilterType = Range,
                                 FloatValueMin = 100, FloatValueMax = 4000, ScoringEquation = Linear
Test 2: GameplayTags (Filter)    TagQueryToMatch = "PatrolPoint.Active"
```

### Ranged attack position

```
Generator: Donut                 Center = Querier, InnerRadius = 400, OuterRadius = 1200,
                                 NumberOfRings = 3, PointsPerRing = 12
Test 1: Distance (Filter)        DistanceTo = MyEQSContext_Enemy, FilterType = Range,
                                 FloatValueMin = 600, FloatValueMax = 1400
Test 2: Trace (Filter)           Context = MyEQSContext_Enemy, FilterType = Match, BoolValue = true
Test 3: Distance (Score)         DistanceTo = Querier, ScoringEquation = InverseLinear, ScoringFactor = 0.3
```

---

## Custom context (C++)

```cpp
// MyEQSContext_Enemy.h
#pragma once

#include "CoreMinimal.h"
#include "EnvironmentQuery/EnvQueryContext.h"
#include "MyEQSContext_Enemy.generated.h"

UCLASS()
class MYGAME_API UMyEQSContext_Enemy : public UEnvQueryContext
{
    GENERATED_BODY()

public:
    virtual void ProvideContext(FEnvQueryInstance& QueryInstance, FEnvQueryContextData& ContextData) const override;
};

// MyEQSContext_Enemy.cpp
#include "MyEQSContext_Enemy.h"

#include "AIController.h"
#include "BehaviorTree/BlackboardComponent.h"
#include "EnvironmentQuery/EnvQueryTypes.h"
#include "EnvironmentQuery/Items/EnvQueryItemType_Actor.h"
#include "GameFramework/Pawn.h"

void UMyEQSContext_Enemy::ProvideContext(FEnvQueryInstance& QueryInstance, FEnvQueryContextData& ContextData) const
{
    UObject* Owner = QueryInstance.Owner.Get();
    const APawn* QuerierPawn = Cast<APawn>(Owner);
    const AAIController* Controller = QuerierPawn ? Cast<AAIController>(QuerierPawn->GetController()) : Cast<AAIController>(Owner);
    const UBlackboardComponent* Blackboard = Controller ? Controller->GetBlackboardComponent() : nullptr;
    if (!Blackboard)
    {
        return;
    }

    if (AActor* Enemy = Cast<AActor>(Blackboard->GetValueAsObject(TEXT("TargetActor"))))
    {
        UEnvQueryItemType_Actor::SetContextHelper(ContextData, Enemy);
    }
}
```

`QueryInstance.Owner` is a `TWeakObjectPtr<UObject>` — the querier passed to `FEnvQueryRequest`. `UBTTask_RunEQSQuery` and `UBTService_RunEQS` swap the controller for its pawn (`BTTask_RunEQSQuery.cpp:43-47`), but a query you run with a controller as querier passes the controller, so handle both. Prefer the pawn as querier: `UEnvQueryContext_Querier` uses the owner actor's location and warns when it is a controller (`EnvQueryContext_Querier.cpp:19-21`). For point contexts use `UEnvQueryItemType_Point::SetContextHelper(ContextData, Location)`.

Blueprint contexts derive from `UEnvQueryContext_BlueprintBase` and implement `ProvideSingleActor`, `ProvideSingleLocation`, `ProvideActorsSet` or `ProvideLocationsSet`.

---

## Running queries from C++

```cpp
#include "EnvironmentQuery/EnvQueryManager.h"
#include "EnvironmentQuery/EnvQueryTypes.h"

void AMyAIController::FindCover()
{
    if (!CoverQuery) { return; }

    FEnvQueryRequest Request(CoverQuery, GetPawn()); // pawn as querier; the delegate still binds to this controller
    Request.SetFloatParam(TEXT("SearchRadius"), 1200.f);
    Request.Execute(EEnvQueryRunMode::SingleResult, this, &AMyAIController::HandleCoverQueryFinished);
}

void AMyAIController::HandleCoverQueryFinished(TSharedPtr<FEnvQueryResult> Result)
{
    if (!Result.IsValid() || !Result->IsSuccessful())
    {
        return;
    }

    GetBlackboardComponent()->SetValueAsVector(TEXT("CoverLocation"), Result->GetItemAsLocation(0));

    // Ranked list, when RunMode is AllMatching:
    for (int32 Index = 0; Index < Result->Items.Num(); ++Index)
    {
        const FVector ItemLocation = Result->GetItemAsLocation(Index);
        const float ItemScore = Result->GetItemScore(Index);
        UE_LOG(LogMyGame, VeryVerbose, TEXT("Item %d at %s scored %f"), Index, *ItemLocation.ToString(), ItemScore);
    }
}
```

`FEnvQueryRequest::Execute` has a `(EEnvQueryRunMode::Type, UserClass*, TMethodPtr)` overload as used above and a `(EEnvQueryRunMode::Type, const FQueryFinishedSignature&)` overload; both return an `int32` query id. Cancel with `UEnvQueryManager::GetCurrent(this)->AbortQuery(QueryID)`. `RunInstantQuery(Request, RunMode)` runs synchronously and returns a `TSharedPtr<FEnvQueryResult>` — it bypasses the frame-time budget, so keep it out of per-frame code.

Name the parameters you pass with `SetFloatParam` / `SetIntParam` / `SetBoolParam` exactly as the asset's data providers name them, otherwise the constant in the asset is used silently.

### From a Behavior Tree

`UBTTask_RunEQSQuery` holds an `FEQSParametrizedQueryExecutionRequest EQSRequest` (its `QueryTemplate`, `RunMode`, `QueryConfig` and `EQSQueryBlackboardKey` live there) plus `BlackboardKey` for the output and `bUpdateBBOnFail`. The task keeps an `FBTEnvQueryTaskMemory` in node memory so it can abort the in-flight query. `UBTService_RunEQS` is the periodic equivalent.

---

## Debugging

Open the Gameplay Debugger (default key `'`) with an AI selected and switch to the EQS category to see per-item scores and which test rejected each item. `ai.debug.EQS.RefreshInterval` sets how often that panel refreshes.

Place an `AEQSTestingPawn` (`EnvironmentQuery/EQSTestingPawn.h`) in the level, set `QueryTemplate` and optionally `QueryConfig`, and the editor previews the scored item cloud without entering PIE. `TimeLimitPerStep` > 0 runs one step per execution instead of the whole query, and `StepToDebugDraw` picks which recorded step is drawn (`EQSTestingPawn.cpp:244-248`). `AEQSTestingPawn::RunEQSQuery()` re-runs it on demand.

---

## Performance

| Concern | Mitigation |
|---|---|
| Too many candidate items | Lower `GridSize` or raise `SpaceBetween`; a donut with explicit `PointsPerRing` gives a fixed, predictable count |
| Frequent queries | Run from `UBTService_RunEQS` with a 0.5 s or longer interval and cache the result in a blackboard key |
| Expensive trace tests | Put a distance filter before every trace so fewer items reach it |
| Pathfinding tests | Prefer `PathExist` over `PathLength`, use `PathfindingBatch`, and keep them last in the test list |
| Many concurrent AI | `UEnvQueryManager` time-slices queries across frames; its `UPROPERTY(config)` members (`config=Game`, `EnvQueryManager.h:340-367`) are `MaxAllowedTestingTime`, `bTestQueriesUsingBreadth`, `QueryCountWarningThreshold` and the `*TimeWarningSeconds` thresholds; at runtime pass an `FEnvQueryManagerConfig` to `Configure()` |
| `AllMatching` mode | Only when you need the ranked list; `SingleResult` stops as soon as a winner is known |
