# Trace, Sweep and Overlap Patterns

Target engine: **UE 5.8**. Gameplay patterns built on the `UWorld` query API (`Engine/Classes/Engine/World.h:2128-2441`) with `FCollisionQueryParams` (`Engine/Public/CollisionQueryParams.h`) and `FCollisionShape` (`PhysicsCore/Public/CollisionShape.h`). The Blueprint-callable wrappers with built-in debug drawing live in `UKismetSystemLibrary` and are covered at the end.

---

## Pattern 1: Hitscan weapon

One line trace along the aim direction, first blocking hit wins, physical surface returned for decal and sound selection.

```cpp
// MyWeaponComponent.h
#pragma once

#include "CoreMinimal.h"
#include "Components/ActorComponent.h"
#include "Engine/HitResult.h"
#include "MyWeaponComponent.generated.h"

UCLASS(ClassGroup = (Custom), meta = (BlueprintSpawnableComponent))
class MYGAME_API UMyWeaponComponent : public UActorComponent
{
	GENERATED_BODY()

public:
	UFUNCTION(BlueprintCallable, Category = "Weapon")
	bool FireHitscan(const FVector& MuzzleLocation, const FVector& AimDirection, float Range, FHitResult& OutHit);

protected:
	UPROPERTY(EditDefaultsOnly, Category = "Weapon")
	float DamageAmount = 25.f;
};
```

```cpp
// MyWeaponComponent.cpp
#include "MyWeaponComponent.h"
#include "Engine/World.h"
#include "CollisionQueryParams.h"
#include "PhysicalMaterials/PhysicalMaterial.h"
#include "DrawDebugHelpers.h"

bool UMyWeaponComponent::FireHitscan(const FVector& MuzzleLocation, const FVector& AimDirection,
	float Range, FHitResult& OutHit)
{
	const FVector TraceStart = MuzzleLocation;
	const FVector TraceEnd   = MuzzleLocation + AimDirection.GetSafeNormal() * Range;

	FCollisionQueryParams Params(TEXT("MyHitscan"), /*bTraceComplex=*/false, GetOwner());
	Params.bReturnPhysicalMaterial = true;

	const bool bHit = GetWorld()->LineTraceSingleByChannel(
		OutHit, TraceStart, TraceEnd, ECC_GameTraceChannel1, Params);   // "Weapon" trace channel

	if (bHit)
	{
		EPhysicalSurface Surface = SurfaceType_Default;
		if (UPhysicalMaterial* PhysMat = OutHit.PhysMaterial.Get())
		{
			Surface = UPhysicalMaterial::DetermineSurfaceType(PhysMat);
		}
		// Surface picks the impact decal / sound. Damage application belongs to the
		// gameplay layer - see ue-gameplay-framework for TakeDamage / ApplyPointDamage.
	}

#if ENABLE_DRAW_DEBUG
	DrawDebugLine(GetWorld(), TraceStart, bHit ? OutHit.ImpactPoint : TraceEnd,
		FColor::Red, /*bPersistentLines=*/false, /*LifeTime=*/1.f, /*DepthPriority=*/0, /*Thickness=*/1.f);
#endif

	return bHit;
}
```

`ENABLE_DRAW_DEBUG` is 0 in Shipping, where `DrawDebugHelpers.h` supplies empty inline stubs (`Engine/Public/DrawDebugHelpers.h:185`), so the guard is about intent rather than link errors — but keep it so the argument expressions are compiled out too.

---

## Pattern 2: Melee arc — multi sphere sweep

`SweepMultiByChannel` returns every touch plus the final block, so the same actor can appear more than once; deduplicate before acting.

```cpp
// MyMeleeComponent.cpp
#include "MyMeleeComponent.h"
#include "Engine/World.h"
#include "CollisionShape.h"
#include "CollisionQueryParams.h"

void UMyMeleeComponent::PerformSwing(const FVector& SwingStart, const FVector& SwingEnd, float AttackRadius)
{
	const FCollisionShape Sphere = FCollisionShape::MakeSphere(AttackRadius);

	FCollisionQueryParams Params(TEXT("MyMeleeSwing"), /*bTraceComplex=*/false, GetOwner());

	TArray<FHitResult> Hits;
	GetWorld()->SweepMultiByChannel(Hits, SwingStart, SwingEnd, FQuat::Identity,
		ECC_Pawn, Sphere, Params);

	TSet<AActor*> AlreadyHit;
	for (const FHitResult& Hit : Hits)
	{
		AActor* HitActor = Hit.GetActor();
		if (HitActor && !AlreadyHit.Contains(HitActor))
		{
			AlreadyHit.Add(HitActor);
			HitTargets.Add(HitActor);   // consumed by the ability / damage layer
		}
	}
}
```

Sweeping `ECC_Pawn` as a *trace* channel relies on each pawn's response to `ECC_Pawn`, which for the engine `Pawn` profile is Block. If the swing should also register on props, either sweep a dedicated `"Weapon"` trace channel or switch to `SweepMultiByObjectType` with `ECC_Pawn` plus `ECC_PhysicsBody`.

---

## Pattern 3: Interaction trace from the camera

Object-type query so only components whose object channel is the project's `Interactable` slot can answer.

```cpp
// MyCharacter.cpp
#include "MyCharacter.h"
#include "MyInteractable.h"
#include "Engine/World.h"
#include "GameFramework/PlayerController.h"

void AMyCharacter::TryInteract()
{
	APlayerController* PC = Cast<APlayerController>(GetController());
	if (!PC)
	{
		return;
	}

	FVector CameraLocation;
	FRotator CameraRotation;
	PC->GetPlayerViewPoint(CameraLocation, CameraRotation);

	const FVector TraceStart = CameraLocation;
	const FVector TraceEnd   = CameraLocation + CameraRotation.Vector() * InteractRange;

	FCollisionQueryParams Params(TEXT("MyInteract"), /*bTraceComplex=*/false, this);

	FCollisionObjectQueryParams ObjectParams;
	ObjectParams.AddObjectTypesToQuery(ECC_GameTraceChannel3);   // "Interactable" object channel

	FHitResult Hit;
	if (GetWorld()->LineTraceSingleByObjectType(Hit, TraceStart, TraceEnd, ObjectParams, Params))
	{
		if (AMyInteractable* Target = Cast<AMyInteractable>(Hit.GetActor()))
		{
			Target->Interact(this);
		}
	}
}
```

`LineTraceSingleByObjectType` has no response filtering, so anything whose object type is in the list is a blocking hit. Use `LineTraceSingleByChannel` with a dedicated `"Interaction"` trace channel instead when world geometry should occlude the prompt.

---

## Pattern 4: Ground check

```cpp
// MyGroundSensorComponent.cpp
#include "MyGroundSensorComponent.h"
#include "Engine/World.h"

bool UMyGroundSensorComponent::IsGrounded(FVector& OutGroundNormal) const
{
	const FVector Start = GetOwner()->GetActorLocation();
	const FVector End   = Start + FVector::DownVector * (GroundCheckDistance + CapsuleHalfHeight);

	FCollisionQueryParams Params(TEXT("MyGroundCheck"), /*bTraceComplex=*/false, GetOwner());

	FHitResult Hit;
	const bool bHit = GetWorld()->LineTraceSingleByChannel(Hit, Start, End, ECC_Visibility, Params);

	OutGroundNormal = bHit ? Hit.ImpactNormal : FVector::UpVector;
	return bHit && Hit.bBlockingHit;
}
```

A single line trace misses ledges and steps. For characters, `UCharacterMovementComponent` already maintains `CurrentFloor` from a capsule sweep — see `ue-character-movement` rather than duplicating floor logic.

---

## Pattern 5: Explosion — sphere overlap with line-of-sight

```cpp
// MyExplosiveActor.cpp
#include "MyExplosiveActor.h"
#include "Engine/World.h"
#include "Engine/OverlapResult.h"
#include "CollisionShape.h"
#include "Components/PrimitiveComponent.h"
#include "DrawDebugHelpers.h"

void AMyExplosiveActor::Detonate(const FVector& ExplosionCenter, float Radius, float ImpulseStrength)
{
	const FCollisionShape Sphere = FCollisionShape::MakeSphere(Radius);

	FCollisionObjectQueryParams ObjectParams;
	ObjectParams.AddObjectTypesToQuery(ECC_Pawn);
	ObjectParams.AddObjectTypesToQuery(ECC_PhysicsBody);
	ObjectParams.AddObjectTypesToQuery(ECC_WorldDynamic);

	FCollisionQueryParams Params(TEXT("MyExplosion"), /*bTraceComplex=*/false, this);

	TArray<FOverlapResult> Overlaps;
	GetWorld()->OverlapMultiByObjectType(Overlaps, ExplosionCenter, FQuat::Identity,
		ObjectParams, Sphere, Params);

	TSet<AActor*> Affected;
	for (const FOverlapResult& Result : Overlaps)
	{
		AActor* Actor = Result.GetActor();
		if (!Actor || Affected.Contains(Actor))
		{
			continue;
		}
		Affected.Add(Actor);

		// Line of sight: do not push or damage through walls
		FCollisionQueryParams LOSParams(TEXT("MyExplosionLOS"), /*bTraceComplex=*/false, this);
		LOSParams.AddIgnoredActor(Actor);

		FHitResult LOSHit;
		const FVector ActorCenter = Actor->GetActorLocation();
		if (GetWorld()->LineTraceSingleByChannel(LOSHit, ExplosionCenter, ActorCenter, ECC_Visibility, LOSParams))
		{
			continue;
		}

		if (UPrimitiveComponent* PrimComp = Result.GetComponent())
		{
			if (PrimComp->IsSimulatingPhysics())
			{
				PrimComp->AddRadialImpulse(ExplosionCenter, Radius, ImpulseStrength, RIF_Linear, /*bVelChange=*/false);
			}
		}
	}

#if ENABLE_DRAW_DEBUG
	DrawDebugSphere(GetWorld(), ExplosionCenter, Radius, 16, FColor::Orange, false, 2.f);
#endif
}
```

Radial damage with falloff (`UGameplayStatics::ApplyRadialDamageWithFalloff`, `FRadialDamageEvent`, `UDamageType`) is owned by `ue-gameplay-framework`; this pattern covers only the query and the physics push. `URadialForceComponent::FireImpulse()` does the same push with no query code at all when the impulse is all you need.

---

## Pattern 6: Line of sight test

`LineTraceTestByChannel` allocates no `FHitResult` and is the cheapest form when you only need a yes/no.

```cpp
// MyAIComponent.cpp
#include "GameFramework/Pawn.h"

bool UMyAIComponent::HasLineOfSightTo(const AActor* Target) const
{
	const APawn* OwnerPawn = Cast<APawn>(GetOwner());
	if (!Target || !OwnerPawn)
	{
		return false;
	}

	FCollisionQueryParams Params(TEXT("MyLineOfSight"), /*bTraceComplex=*/false, OwnerPawn);
	Params.AddIgnoredActor(Target);

	return !GetWorld()->LineTraceTestByChannel(
		OwnerPawn->GetPawnViewLocation(), Target->GetActorLocation(), ECC_Visibility, Params);
}
```

For real perception (senses, stimuli, teams, forgetting) use `UAIPerceptionComponent` — see `ue-ai-navigation`.

---

## Pattern 7: Capsule sweep before moving

```cpp
// MyMoverComponent.cpp
bool UMyMoverComponent::CanReach(const FVector& TargetLocation, FHitResult& OutBlockingHit) const
{
	const FVector CurrentLocation = GetOwner()->GetActorLocation();
	const FCollisionShape Capsule = FCollisionShape::MakeCapsule(34.f, 88.f);   // radius, half-height

	FCollisionQueryParams Params(TEXT("MyMoveSweep"), /*bTraceComplex=*/false, GetOwner());

	const bool bBlocked = GetWorld()->SweepSingleByChannel(OutBlockingHit,
		CurrentLocation, TargetLocation, FQuat::Identity, ECC_Pawn, Capsule, Params);

	// OutBlockingHit.Distance is how far the capsule got; ImpactNormal is the wall orientation.
	return !bBlocked;
}
```

`FCollisionShape::MakeCapsule(Radius, HalfHeight)` takes the *total* half-height including the end caps, matching `UCapsuleComponent::GetScaledCapsuleHalfHeight()`.

---

## Pattern 8: Async trace for high-frequency sensors

Queue on one frame, read on the next. Cheaper than a synchronous trace only because it leaves the game thread; it does not make the query itself free.

```cpp
// MySensorComponent.h
#pragma once

#include "CoreMinimal.h"
#include "Components/ActorComponent.h"
#include "WorldCollision.h"
#include "MySensorComponent.generated.h"

UCLASS(ClassGroup = (Custom), meta = (BlueprintSpawnableComponent))
class MYGAME_API UMySensorComponent : public UActorComponent
{
	GENERATED_BODY()

public:
	UMySensorComponent();

	virtual void TickComponent(float DeltaTime, ELevelTick TickType,
		FActorComponentTickFunction* ThisTickFunction) override;

protected:
	void OnSensorHit(const FHitResult& Hit);

	UPROPERTY(EditDefaultsOnly, Category = "Sensor")
	float SensorRange = 800.f;

private:
	FTraceHandle PendingTraceHandle;
	bool bHasPendingTrace = false;
};
```

```cpp
// MySensorComponent.cpp
#include "MySensorComponent.h"
#include "Engine/World.h"
#include "CollisionQueryParams.h"

UMySensorComponent::UMySensorComponent()
{
	PrimaryComponentTick.bCanEverTick = true;
}

void UMySensorComponent::TickComponent(float DeltaTime, ELevelTick TickType,
	FActorComponentTickFunction* ThisTickFunction)
{
	Super::TickComponent(DeltaTime, TickType, ThisTickFunction);

	if (bHasPendingTrace)
	{
		FTraceDatum Datum;
		if (GetWorld()->QueryTraceData(PendingTraceHandle, Datum))
		{
			bHasPendingTrace = false;
			if (Datum.OutHits.Num() > 0 && Datum.OutHits[0].bBlockingHit)
			{
				OnSensorHit(Datum.OutHits[0]);
			}
		}
	}

	if (!bHasPendingTrace)
	{
		FCollisionQueryParams Params(TEXT("MySensor"), /*bTraceComplex=*/false, GetOwner());

		const FVector Start = GetOwner()->GetActorLocation();
		const FVector End   = Start + GetOwner()->GetActorForwardVector() * SensorRange;

		PendingTraceHandle = GetWorld()->AsyncLineTraceByChannel(
			EAsyncTraceType::Single, Start, End, ECC_Visibility, Params);
		bHasPendingTrace = true;
	}
}

void UMySensorComponent::OnSensorHit(const FHitResult& Hit)
{
}
```

`EAsyncTraceType` (`WorldCollision.h:148`) is `Test` (bool only), `Single` (first block) or `Multi` (block plus touches). `QueryTraceData` returns false until the batch completes; `QueryOverlapData` is the equivalent for `AsyncOverlapByChannel`. Passing a `FTraceDelegate*` to the async call replaces the polling with a callback (`DECLARE_DELEGATE_TwoParams(FTraceDelegate, const FTraceHandle&, FTraceDatum&)`, `WorldCollision.h:137`).

---

## FCollisionQueryParams quick reference

`Engine/Public/CollisionQueryParams.h:42`.

```cpp
FCollisionQueryParams Params(
	TEXT("MyTraceTag"),   // TraceTag :45   also drives the named-tag trace debugger
	false,                // bTraceComplex :51   true = per-triangle, several times the cost
	GetOwner()            // convenience ignored actor
);

Params.bFindInitialOverlaps    = true;   // :54  report shapes already overlapping at Start
Params.bReturnFaceIndex        = false;  // :57  fills Hit.FaceIndex; expensive
Params.bReturnPhysicalMaterial = false;  // :60  fills Hit.PhysMaterial
Params.bIgnoreBlocks           = false;  // :63  drop blocking results, keep touches
Params.bIgnoreTouches          = false;  // :66  drop touches, keep blocks
Params.bSkipNarrowPhase        = false;  // :69  overlaps only: skip narrow-phase checks
Params.MobilityType            = EQueryMobilityType::Any;   // :78  Any | Static | Dynamic

Params.AddIgnoredActor(SomeActor);            // :243
Params.AddIgnoredActors(ArrayOfActors);       // :257
Params.AddIgnoredComponent(SomeComponent);    // :267
Params.AddIgnoredComponents(ArrayOfComps);    // :272
```

`FCollisionResponseParams` (`:323`) wraps an `FCollisionResponseContainer` and is the optional trailing argument on every `*ByChannel` query — use it to run one query with responses that differ from the querier's component.

`FCollisionObjectQueryParams` (`:453`): construct from a channel or add with `AddObjectTypesToQuery(ECollisionChannel)` (`:557`). Read and write the set through `GetObjectTypesToQuery()` / `SetObjectTypesToQuery()`; the old `int32 ObjectTypesToQuery` field is deprecated.

---

## FCollisionShape quick reference

`PhysicsCore/Public/CollisionShape.h:20`.

```cpp
FCollisionShape Line;                                                  // default ctor => Line (:51)
FCollisionShape Sphere  = FCollisionShape::MakeSphere(50.f);           // :300 radius
FCollisionShape Box     = FCollisionShape::MakeBox(FVector(50.f, 30.f, 80.f)); // :284 half-extents
FCollisionShape Capsule = FCollisionShape::MakeCapsule(34.f, 88.f);    // :308 radius, half-height

const bool bIsSphere    = Sphere.IsSphere();                  // :69   IsBox :63, IsCapsule :75, IsLine :57
const float Radius      = Sphere.GetSphereRadius();           // :198
const float HalfHeight  = Capsule.GetCapsuleHalfHeight();     // :210  includes the end radius
const float ShaftHalf   = Capsule.GetCapsuleAxisHalfLength(); // :185  HalfHeight - Radius
const FVector HalfExtent = Box.GetBox();                      // :192
```

`MakeBox` also accepts an `FVector3f` (`:292`) and `MakeCapsule` an `FVector` extent (`:316`). A sweep with a default-constructed (Line) shape behaves like the corresponding `LineTrace*` call.

---

## Blueprint-layer wrappers

`UKismetSystemLibrary` (`Engine/Classes/Kismet/KismetSystemLibrary.h`) takes `ETraceTypeQuery` instead of `ECollisionChannel` and draws debug shapes for you.

```cpp
#include "Kismet/KismetSystemLibrary.h"

TArray<AActor*> ActorsToIgnore = { this };
FHitResult Hit;

UKismetSystemLibrary::LineTraceSingle(this, Start, End,
	ETraceTypeQuery::TraceTypeQuery1, /*bTraceComplex=*/false, ActorsToIgnore,
	EDrawDebugTrace::ForDuration, Hit, /*bIgnoreSelf=*/true);            // :1270

UKismetSystemLibrary::SphereTraceSingle(this, Start, End, /*Radius=*/50.f,
	ETraceTypeQuery::TraceTypeQuery1, false, ActorsToIgnore,
	EDrawDebugTrace::ForOneFrame, Hit, true);                            // :1300

UKismetSystemLibrary::BoxTraceSingle(this, Start, End, FVector(50.f), FRotator::ZeroRotator,
	ETraceTypeQuery::TraceTypeQuery1, false, ActorsToIgnore,
	EDrawDebugTrace::None, Hit, true);                                   // :1332

UKismetSystemLibrary::CapsuleTraceSingle(this, Start, End, 34.f, 88.f,
	ETraceTypeQuery::TraceTypeQuery1, false, ActorsToIgnore,
	EDrawDebugTrace::None, Hit, true);                                   // :1366

UKismetSystemLibrary::LineTraceSingleByProfile(this, Start, End, TEXT("BlockAll"),
	false, ActorsToIgnore, EDrawDebugTrace::None, Hit, true);            // :1528
```

Each also takes trailing `FLinearColor TraceColor`, `FLinearColor TraceHitColor` and `float DrawTime = 5.0f`. `EDrawDebugTrace::Type` (`:30`) is `None`, `ForOneFrame`, `ForDuration`, `Persistent`.

The raw drawing helpers differ: `UKismetSystemLibrary::DrawDebugLine(WorldContextObject, Start, End, FLinearColor, Duration, Thickness, DepthPriority)` (`:1664`) versus the C++ `DrawDebugLine(const UWorld*, Start, End, FColor, bool bPersistentLines, float LifeTime, uint8 DepthPriority, float Thickness)` (`DrawDebugHelpers.h:22`). Mixing up the bool/float pair is the usual reason a debug line never appears.

---

## Choosing a query

| Need | Call |
|---|---|
| "Is anything between A and B?" | `LineTraceTestByChannel` |
| "What did I hit first?" | `LineTraceSingleByChannel` |
| "Everything along the ray, in order" | `LineTraceMultiByChannel` |
| "Find objects of a given type" | `*ByObjectType` with `FCollisionObjectQueryParams` |
| "Use the responses of a named profile" | `*ByProfile` |
| "Will this shape fit along this path?" | `SweepSingleByChannel` |
| "Everything a moving shape brushes past" | `SweepMultiByChannel` |
| "Everything inside a volume right now" | `OverlapMultiByObjectType` / `OverlapMultiByChannel` |
| "Is this volume occupied at all?" | `OverlapAnyTestByChannel` (blocks or touches) / `OverlapBlockingTestByChannel` (blocks only) |
| "What is my own component already touching?" | `UPrimitiveComponent::GetOverlappingActors` / `ComponentOverlapComponent` |
| "Does this one actor intersect this ray?" | `AActor::ActorLineTraceSingle` (`GameFramework/Actor.h:3350`) |

---

## Performance guidance

| Technique | Relative cost | Use |
|---|---|---|
| `LineTraceTestByChannel` | lowest | visibility and occlusion yes/no |
| `LineTraceSingleByChannel` | low | shooting, interaction |
| `LineTraceMultiByChannel` | medium | penetration, multi-target beams |
| `SweepSingleByChannel` | medium | movement validation, precise melee |
| `SweepMultiByChannel` | medium-high | melee arcs, projectile sweeps |
| `OverlapMultiByObjectType` | medium | AoE, proximity |
| `bTraceComplex = true` | several times the simple-hull cost | only where triangle accuracy matters |
| `AsyncLineTraceByChannel` | deferred | sensors polling many actors per frame |

- Prefer `Test` over `Single`, and `Single` over `Multi`, whenever the extra data is discarded.
- Narrow the candidate set with the channel or object type before narrowing it in C++ — a response of `ECR_Ignore` is rejected in the broad phase, an `if (Actor->IsA(...))` is not.
- `Params.bIgnoreTouches = true` skips overlap bookkeeping on queries that only care about blocks.
- Throttle non-critical traces to 5–10 Hz with a timer; most AI awareness does not need per-frame resolution.
- Use `TraceTag` plus the console trace debugger to confirm a suspect query is actually running before optimising it.
