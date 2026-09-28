---
name: ue-physics-collision
description: "Use when setting up collision channels and profiles, running line/sweep/overlap queries, wiring hit and overlap events, or simulating rigid bodies in Unreal Engine C++. Also use when the user mentions 'collision channel', 'collision profile', 'LineTraceSingleByChannel', 'SweepSingleByChannel', 'OverlapMultiByObjectType', 'FHitResult', 'FCollisionQueryParams', 'FCollisionShape', 'OnComponentHit', 'OnComponentBeginOverlap', 'SetCollisionProfileName', 'SetSimulatePhysics', 'AddRadialImpulse', 'ragdoll', 'Chaos solver', 'ECC_GameTraceChannel1', or 'my overlap never fires'. For component attachment, see ue-actor-component-architecture; for capsule floor checks, see ue-character-movement."
metadata:
  version: "2.0.0"
  engine: "5.8"
---

# UE Physics & Collision

Target engine: **UE 5.8**. APIs below are verified against the 5.8 headers; older forms are listed under "Deprecated — do not use".

Collision and physics live in the `Engine` module (`UPrimitiveComponent`, `UWorld` queries, `FHitResult`, `UCollisionProfile`) and the `PhysicsCore` module (`FCollisionShape`, `UPhysicalMaterial`, `FBodyInstanceCore`, `ECollisionTraceFlag`, `EPhysicalSurface`). The simulator itself is Chaos (`Chaos` / `ChaosSolverEngine` modules); destruction adds `GeometryCollectionEngine` and `FieldSystemEngine`. Most gameplay code only needs `"Engine"` and `"PhysicsCore"` in `Build.cs`.

## Context

Read `.agents/ue-project-context.md` if it exists (module names, conventions, enabled plugins, GAS/networking setup). Do not stop if it is missing.

Identify the area from the request and the codebase. Ask only when two plausible readings would produce different code.

| Request is about… | Go to |
|---|---|
| Channels, profiles, `DefaultEngine.ini`, responses | [Collision Channels and Profiles](#collision-channels-and-profiles) |
| Line traces, sweeps, overlap queries, async traces | [Trace Sweep and Overlap Queries](#trace-sweep-and-overlap-queries) |
| Reading what a query hit | [FHitResult](#fhitresult) |
| `OnComponentHit`, `OnComponentBeginOverlap`, wake/sleep | [Collision Events](#collision-events) |
| Rigid bodies, impulses, forces, mass, damping, sleeping | [Physics Bodies and Forces](#physics-bodies-and-forces) |
| Joints, radial force fields, ragdolls | [Constraints Radial Force and Ragdolls](#constraints-radial-force-and-ragdolls) |
| Friction, restitution, surface types | [Physical Materials](#physical-materials) |
| Solver settings, substepping, CVars, physics-thread ticks | [Chaos Solver and Async Physics](#chaos-solver-and-async-physics) |
| Replicated physics, resimulation, predictive interpolation | [Network Physics](#network-physics) |

## Collision Channels and Profiles

`ECollisionChannel` (`Engine/EngineTypes.h:1098`) has 8 engine channels, 6 hidden engine slots and 50 game slots (64 total; `ECC_GameTraceChannel50` at `:1168`):

```cpp
ECC_WorldStatic, ECC_WorldDynamic, ECC_Pawn, ECC_PhysicsBody,
ECC_Vehicle, ECC_Destructible                 // object channels (what a component IS)
ECC_Visibility, ECC_Camera                    // trace channels (what a query LOOKS FOR)
ECC_GameTraceChannel1 ... ECC_GameTraceChannel50  // project slots, either kind
```

- **Object channel** — every component has exactly one, set with `SetCollisionObjectType`. Matched by `*ByObjectType` queries.
- **Trace channel** — never an object type; a query passes it to `*ByChannel` and each component's response to that channel decides block/overlap/ignore.
- `ECollisionResponse` (`Engine/EngineTypes.h:1346`): `ECR_Ignore`, `ECR_Overlap`, `ECR_Block`.
- `ECollisionEnabled::Type` (`Engine/EngineTypes.h:1805`): `NoCollision`, `QueryOnly`, `PhysicsOnly`, `QueryAndPhysics`, `ProbeOnly`, `QueryAndProbe`.

```cpp
#include "Components/PrimitiveComponent.h"

MyMesh->SetCollisionProfileName(TEXT("BlockAllDynamic"));   // preferred: one call sets everything
MyMesh->SetCollisionEnabled(ECollisionEnabled::QueryAndPhysics);
MyMesh->SetCollisionObjectType(ECC_PhysicsBody);
MyMesh->SetCollisionResponseToAllChannels(ECR_Block);
MyMesh->SetCollisionResponseToChannel(ECC_Pawn, ECR_Overlap);
MyMesh->SetCollisionResponseToChannel(ECC_Camera, ECR_Ignore);

FCollisionResponseContainer Responses(ECR_Ignore);
Responses.SetResponse(ECC_Visibility, ECR_Block);
MyMesh->SetCollisionResponseToChannels(Responses);
```

`SetCollisionProfileName` overwrites every manual response, so apply the profile first and the per-channel overrides after.

Engine profiles shipped in `Config/BaseEngine.ini` under `[/Script/Engine.CollisionProfile]`: `NoCollision`, `BlockAll`, `OverlapAll`, `BlockAllDynamic`, `OverlapAllDynamic`, `IgnoreOnlyPawn`, `OverlapOnlyPawn`, `Pawn`, `Spectator`, `CharacterMesh`, `PhysicsActor`, `Destructible`, `InvisibleWall`, `InvisibleWallDynamic`, `Trigger`, `Ragdoll`, `Vehicle`, `UI`, `WaterBodyCollision`.

Custom channels and profiles are config-only — there is no runtime API to create one:

```ini
[/Script/Engine.CollisionProfile]
+DefaultChannelResponses=(Channel=ECC_GameTraceChannel1,DefaultResponse=ECR_Block,bTraceType=True,bStaticObject=False,Name="Weapon")
+DefaultChannelResponses=(Channel=ECC_GameTraceChannel2,DefaultResponse=ECR_Block,bTraceType=False,bStaticObject=False,Name="Interactable")
+Profiles=(Name="MyInteractable",CollisionEnabled=QueryAndPhysics,ObjectTypeName="Interactable",CustomResponses=((Channel="Weapon",Response=ECR_Ignore),(Channel="Visibility",Response=ECR_Block)),HelpMessage="Interactable props")
+EditProfiles=(Name="Pawn",CustomResponses=((Channel="Weapon",Response=ECR_Block)))
```

`bTraceType=True` declares a trace channel, `False` an object channel. `+Profiles=` adds or replaces a whole profile; `+EditProfiles=` only patches responses on an existing one (`Engine/CollisionProfile.h:160-190`). `UCollisionProfile::Get()` reads the table and `UCollisionProfile::LoadProfileConfig(bool bForceInit)` (`Engine/CollisionProfile.h:237`) re-reads it after a config change.

See [collision-channel-setup.md](references/collision-channel-setup.md) for full profile sets, the ini reference and a debugging checklist.

## Trace Sweep and Overlap Queries

All world queries are `const` members of `UWorld` (`Engine/World.h:2128-2441`). Three families, each in `Test` / `Single` / `Multi` form:

| Suffix | Selects by | Extra parameter |
|---|---|---|
| `ByChannel` | the querier's trace channel vs each component's response | `ECollisionChannel TraceChannel`, optional `FCollisionResponseParams` |
| `ByObjectType` | the target component's object channel | `const FCollisionObjectQueryParams&` |
| `ByProfile` | the channel and responses stored in a named profile | `FName ProfileName` |

```cpp
#include "Engine/World.h"
#include "Engine/HitResult.h"
#include "CollisionQueryParams.h"
#include "CollisionShape.h"

FCollisionQueryParams Params(TEXT("MyWeaponTrace"), /*bTraceComplex=*/false, GetOwner());
Params.bReturnPhysicalMaterial = true;   // fills Hit.PhysMaterial
Params.bReturnFaceIndex        = false;  // fills Hit.FaceIndex; expensive
Params.bIgnoreTouches          = true;   // discard ECR_Overlap results (bIgnoreBlocks is the inverse)
Params.bFindInitialOverlaps    = true;   // report shapes already overlapping at Start
Params.MobilityType            = EQueryMobilityType::Any;  // Any | Static | Dynamic
Params.AddIgnoredActor(GetOwner());
Params.AddIgnoredComponent(MyMesh.Get());   // .Get(): a TObjectPtr member is ambiguous (CollisionQueryParams.h:267-269)

FHitResult Hit;
GetWorld()->LineTraceSingleByChannel(Hit, Start, End, ECC_Visibility, Params);

TArray<FHitResult> Hits;
GetWorld()->LineTraceMultiByChannel(Hits, Start, End, ECC_GameTraceChannel1, Params);

FCollisionObjectQueryParams ObjParams(ECC_Pawn);
ObjParams.AddObjectTypesToQuery(ECC_PhysicsBody);
GetWorld()->LineTraceSingleByObjectType(Hit, Start, End, ObjParams, Params);

GetWorld()->LineTraceSingleByProfile(Hit, Start, End, TEXT("BlockAll"), Params);

const bool bAnythingThere = GetWorld()->LineTraceTestByChannel(Start, End, ECC_Visibility, Params);
```

`TraceTag` is the first constructor argument; it also drives the `TraceTagAll` / named-tag debug drawing in the console.

Sweeps take a rotation and an `FCollisionShape` (`PhysicsCore/Public/CollisionShape.h:284-316`):

```cpp
const FCollisionShape Sphere  = FCollisionShape::MakeSphere(30.f);                  // radius
const FCollisionShape Box     = FCollisionShape::MakeBox(FVector(50.f, 30.f, 80.f));// half-extents
const FCollisionShape Capsule = FCollisionShape::MakeCapsule(34.f, 88.f);           // radius, half-height

GetWorld()->SweepSingleByChannel(Hit, Start, End, FQuat::Identity, ECC_Pawn, Capsule, Params);
GetWorld()->SweepMultiByChannel(Hits, Start, End, FQuat::Identity, ECC_Pawn, Sphere, Params);
GetWorld()->SweepSingleByObjectType(Hit, Start, End, FQuat::Identity, ObjParams, Box, Params);
GetWorld()->SweepSingleByProfile(Hit, Start, End, FQuat::Identity, TEXT("Pawn"), Capsule, Params);
```

`GetCapsuleHalfHeight()` includes the end radius; `GetCapsuleAxisHalfLength()` is the shaft only.

Overlaps are stationary shape tests returning `FOverlapResult` (`Engine/OverlapResult.h:12`):

```cpp
#include "Engine/OverlapResult.h"

TArray<FOverlapResult> Overlaps;
GetWorld()->OverlapMultiByObjectType(Overlaps, Center, FQuat::Identity,
    FCollisionObjectQueryParams(ECC_Pawn), FCollisionShape::MakeSphere(500.f), Params);
GetWorld()->OverlapMultiByChannel(Overlaps, Center, FQuat::Identity, ECC_Pawn,
    FCollisionShape::MakeSphere(500.f), Params);

const bool bBlocked = GetWorld()->OverlapAnyTestByChannel(Center, FQuat::Identity,
    ECC_WorldStatic, FCollisionShape::MakeBox(FVector(40.f)), Params);

for (const FOverlapResult& Result : Overlaps)
{
    AActor* Actor = Result.GetActor();
    UPrimitiveComponent* Comp = Result.GetComponent();
}
```

`OverlapAnyTestByChannel` stops at the first block **or** touch; `OverlapBlockingTestByChannel` ignores touches. When the query is "what is my own component already touching", use `UPrimitiveComponent::GetOverlappingActors` or `ComponentOverlapComponent` instead of a world query.

Async traces queue this frame and are readable next frame (`Engine/World.h:2522`, `WorldCollision.h:148`):

```cpp
#include "WorldCollision.h"

// EAsyncTraceType: Test | Single | Multi
FTraceHandle Handle = GetWorld()->AsyncLineTraceByChannel(
    EAsyncTraceType::Single, Start, End, ECC_Visibility, Params);

FTraceDatum Datum;
if (GetWorld()->QueryTraceData(Handle, Datum) && Datum.OutHits.Num() > 0)
{
    const FHitResult& First = Datum.OutHits[0];
}
```

Passing a `FTraceDelegate*` (`DECLARE_DELEGATE_TwoParams(FTraceDelegate, const FTraceHandle&, FTraceDatum&)`) instead calls you back when the batch completes.

See [trace-patterns.md](references/trace-patterns.md) for hitscan, melee sweep, interaction, ground check, AoE, line-of-sight and async sensor patterns, plus the Blueprint-layer `UKismetSystemLibrary` wrappers and performance guidance.

## FHitResult

`Engine/HitResult.h`. `Actor` is no longer a member — the actor is reached through an accessor.

```cpp
Hit.bBlockingHit;        // :107  false for a touch-only result
Hit.bStartPenetrating;   // :116  query began already overlapping; PenetrationDepth :91
Hit.Time;                // :33   0..1 along Start->End;  Distance :37 in cm
Hit.Location;            // :45   where the swept shape's origin ended up
Hit.ImpactPoint;         // :53   surface contact point
Hit.Normal;              // :61   normal opposing movement at Location
Hit.ImpactNormal;        // :69   geometric surface normal at ImpactPoint
Hit.TraceStart;          // :76;  TraceEnd :83
Hit.Item;                // :99   instance index (ISM/HISM); ElementIndex :103 shape index
Hit.FaceIndex;           // :26   needs Params.bReturnFaceIndex
Hit.BoneName;            // :141  bone hit;  MyBoneName :145 our own bone
Hit.PhysMaterial;        // :123  TWeakObjectPtr, needs bReturnPhysicalMaterial

AActor* HitActor = Hit.GetActor();                 // :211
UPrimitiveComponent* HitComp = Hit.GetComponent(); // :227
FActorInstanceHandle Handle = Hit.GetHitObjectHandle(); // :216, lightweight instances
```

For a `Multi` query only the last element can be a blocking hit; every earlier element is a touch.

## Collision Events

Delegates are sparse dynamic multicast, so handlers must be `UFUNCTION()`-declared **in the header** with exactly the delegate's parameter list, or `AddDynamic` fails to bind at runtime. Signatures from `Components/PrimitiveComponent.h:274-282` and `GameFramework/Actor.h:194-196`:

| Delegate | Parameters |
|---|---|
| `FComponentHitSignature` | `UPrimitiveComponent* HitComponent, AActor* OtherActor, UPrimitiveComponent* OtherComp, FVector NormalImpulse, const FHitResult& Hit` |
| `FComponentBeginOverlapSignature` | `UPrimitiveComponent* OverlappedComponent, AActor* OtherActor, UPrimitiveComponent* OtherComp, int32 OtherBodyIndex, bool bFromSweep, const FHitResult& SweepResult` |
| `FComponentEndOverlapSignature` | `UPrimitiveComponent* OverlappedComponent, AActor* OtherActor, UPrimitiveComponent* OtherComp, int32 OtherBodyIndex` |
| `FComponentWakeSignature` / `FComponentSleepSignature` | `UPrimitiveComponent* WakingComponent` (`SleepingComponent`), `FName BoneName` |
| `FActorBeginOverlapSignature` / `FActorEndOverlapSignature` | `AActor* OverlappedActor, AActor* OtherActor` |
| `FActorHitSignature` | `AActor* SelfActor, AActor* OtherActor, FVector NormalImpulse, const FHitResult& Hit` |

```cpp
// MyPickup.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/Actor.h"
#include "Engine/HitResult.h"
#include "MyPickup.generated.h"

UCLASS()
class MYGAME_API AMyPickup : public AActor
{
	GENERATED_BODY()

public:
	AMyPickup();

protected:
	virtual void BeginPlay() override;

	UPROPERTY(VisibleAnywhere, Category = "Collision")
	TObjectPtr<class USphereComponent> Trigger;

	UPROPERTY(VisibleAnywhere, Category = "Collision")
	TObjectPtr<class UStaticMeshComponent> Mesh;

	UFUNCTION()
	void OnBeginOverlap(UPrimitiveComponent* OverlappedComponent, AActor* OtherActor,
		UPrimitiveComponent* OtherComp, int32 OtherBodyIndex, bool bFromSweep, const FHitResult& SweepResult);

	UFUNCTION()
	void OnEndOverlap(UPrimitiveComponent* OverlappedComponent, AActor* OtherActor,
		UPrimitiveComponent* OtherComp, int32 OtherBodyIndex);

	UFUNCTION()
	void OnHit(UPrimitiveComponent* HitComponent, AActor* OtherActor,
		UPrimitiveComponent* OtherComp, FVector NormalImpulse, const FHitResult& Hit);
};
```

```cpp
// MyPickup.cpp
#include "MyPickup.h"
#include "Components/SphereComponent.h"
#include "Components/StaticMeshComponent.h"

AMyPickup::AMyPickup()
{
	Trigger = CreateDefaultSubobject<USphereComponent>(TEXT("Trigger"));
	SetRootComponent(Trigger);
	Trigger->SetSphereRadius(120.f);
	Trigger->SetCollisionProfileName(TEXT("OverlapAllDynamic"));
	Trigger->SetGenerateOverlapEvents(true);

	Mesh = CreateDefaultSubobject<UStaticMeshComponent>(TEXT("Mesh"));
	Mesh->SetupAttachment(Trigger);
	Mesh->SetCollisionProfileName(TEXT("PhysicsActor"));
	Mesh->SetSimulatePhysics(true);
	Mesh->SetNotifyRigidBodyCollision(true);   // "Simulation Generates Hit Events"
}

void AMyPickup::BeginPlay()
{
	Super::BeginPlay();

	Trigger->OnComponentBeginOverlap.AddDynamic(this, &AMyPickup::OnBeginOverlap);
	Trigger->OnComponentEndOverlap.AddDynamic(this, &AMyPickup::OnEndOverlap);
	Mesh->OnComponentHit.AddDynamic(this, &AMyPickup::OnHit);
}

void AMyPickup::OnBeginOverlap(UPrimitiveComponent* OverlappedComponent, AActor* OtherActor,
	UPrimitiveComponent* OtherComp, int32 OtherBodyIndex, bool bFromSweep, const FHitResult& SweepResult)
{
	if (OtherActor && OtherActor != this)
	{
		Destroy();
	}
}

void AMyPickup::OnEndOverlap(UPrimitiveComponent* OverlappedComponent, AActor* OtherActor,
	UPrimitiveComponent* OtherComp, int32 OtherBodyIndex)
{
}

void AMyPickup::OnHit(UPrimitiveComponent* HitComponent, AActor* OtherActor,
	UPrimitiveComponent* OtherComp, FVector NormalImpulse, const FHitResult& Hit)
{
	const float ImpactStrength = NormalImpulse.Size();
}
```

Requirements: overlaps need `ECR_Overlap` on at least one side, a non-`ECR_Ignore` pair, and `SetGenerateOverlapEvents(true)` on **both** components (`bGenerateOverlapEvents` is private — `Components/PrimitiveComponent.h:429`). Simulation hits need `ECR_Block` on both, `QueryAndPhysics` or `PhysicsOnly`, and `SetNotifyRigidBodyCollision(true)` on the simulating component; a swept move (`SetActorLocation(..., true)`, movement components) fires `OnComponentHit` on any blocking hit without that flag (`PrimitiveComponent.cpp:3540-3550`). `OnComponentWake` / `OnComponentSleep` need `bGenerateWakeEvents` on the body (`PhysicsCore/Public/BodyInstanceCore.h:59`).

## Physics Bodies and Forces

```cpp
MyMesh->SetSimulatePhysics(true);                          // PrimitiveComponent.h:1661
MyMesh->SetEnableGravity(true);                            // :2801
MyMesh->SetMassOverrideInKg(NAME_None, 50.f, true);        // :2857;  GetMass() :2861
MyMesh->SetLinearDamping(0.1f);                            // :2825;  SetAngularDamping :2833
MyMesh->SetUseCCD(true);                                   // :2894 swept collision for fast bodies
MyMesh->SetConstraintMode(EDOFMode::XYPlane);              // :1682 lock to a plane
const bool bSimulating = MyMesh->IsSimulatingPhysics();    // :2644

MyMesh->AddImpulse(FVector(0.f, 0.f, 1000.f), NAME_None, /*bVelChange=*/false);  // :1692
MyMesh->AddImpulseAtLocation(FVector(500.f, 0.f, 0.f), Hit.ImpactPoint);         // :1725 adds spin
MyMesh->AddRadialImpulse(Center, 500.f, 2000.f, RIF_Linear, /*bVelChange=*/false); // :1748
MyMesh->AddForce(FVector(0.f, 0.f, 9800.f), NAME_None, /*bAccelChange=*/false);  // :1759 every tick
MyMesh->AddRadialForce(Center, 500.f, 3000.f, RIF_Constant, false);              // :1793

MyMesh->SetPhysicsLinearVelocity(FVector(0.f, 0.f, 300.f));  // :1825 teleports velocity; use sparingly
const FVector Velocity = MyMesh->GetPhysicsLinearVelocity(); // :1832
MyMesh->WakeRigidBody();                                     // :1938;  PutRigidBodyToSleep() :1945
```

`bVelChange=true` / `bAccelChange=true` ignore mass. `ERadialImpulseFalloff` is `RIF_Constant` or `RIF_Linear` (`PhysicsCore/Public/Chaos/ChaosEngineInterface.h:90`).

`MyMesh->GetBodyInstance()` reaches the body. `FBodyInstanceCore` flags (`PhysicsCore/Public/BodyInstanceCore.h`): `bSimulatePhysics` (:30), `bOverrideMass` (:35), `bEnableGravity` (:39), `bUpdateKinematicFromSimulation` (:43), `bAutoWeld` (:51), `bStartAwake` (:55), `bGenerateWakeEvents` (:59). `FBodyInstance` adds `bUseCCD` (:418), `bNotifyRigidBodyCollision` (:440), `LinearDamping` (:629), `MassScale` (:645), `InertiaTensorScale` (:653), `SleepFamily` (:410) and `CustomSleepThresholdMultiplier` (:696). `ESleepFamily` (`Chaos/ChaosEngineInterface.h:101`) is `Normal`, `Sensitive` or `Custom`; only `Custom` reads the multiplier.

Collision complexity comes from `ECollisionTraceFlag` (`PhysicsCore/Public/BodySetupEnums.h:13-20`): `CTF_UseDefault`, `CTF_UseSimpleAndComplex`, `CTF_UseSimpleAsComplex`, `CTF_UseComplexAsSimple`. Complex-as-simple bodies cannot simulate.

## Constraints Radial Force and Ragdolls

`UPhysicsConstraintComponent` (`PhysicsEngine/PhysicsConstraintComponent.h:24`) wraps an `FConstraintInstance` (`:73`) — the same struct a Physics Asset stores per joint.

```cpp
#include "PhysicsEngine/PhysicsConstraintComponent.h"

UPhysicsConstraintComponent* Joint = NewObject<UPhysicsConstraintComponent>(this);
Joint->SetupAttachment(RootComponent);
Joint->RegisterComponent();
Joint->SetConstrainedComponents(MeshA, NAME_None, MeshB, NAME_None);  // :134
Joint->SetLinearXLimit(ELinearConstraintMotion::LCM_Locked, 0.f);     // :270
Joint->SetAngularSwing1Limit(EAngularConstraintMotion::ACM_Limited, 45.f); // :291
Joint->SetAngularTwistLimit(EAngularConstraintMotion::ACM_Free, 0.f);      // :305
Joint->SetAngularVelocityDriveTwistAndSwing(true, false);             // :199
Joint->SetAngularDriveParams(500.f, 50.f, 0.f);                       // :262
Joint->SetDisableCollision(true);                                     // :374
Joint->BreakConstraint();                                             // :142
```

Motion enums are `ELinearConstraintMotion` (`LCM_Free`, `LCM_Limited`, `LCM_Locked`) and `EAngularConstraintMotion` (`ACM_Free`, `ACM_Limited`, `ACM_Locked`). Editor presets are only combinations of these: Fixed locks all six, Hinge frees one angular axis, Prismatic frees one linear axis, Ball-and-Socket frees all angular. `URadialForceComponent` (`PhysicsEngine/RadialForceComponent.h:16`) is the drop-in explosion/vacuum component: `Radius`, `Falloff`, `ImpulseStrength`, `bImpulseVelChange`, `bIgnoreOwningActor`, `ForceStrength`, then `FireImpulse()` (:50) for a one-shot and `AddObjectTypeToAffect(EObjectTypeQuery)` (:54) to filter targets.

Ragdolls run through the skeletal mesh's Physics Asset (`UPhysicsAsset::SkeletalBodySetups`, `ConstraintSetup` — `PhysicsEngine/PhysicsAsset.h:213-220`):

```cpp
GetMesh()->SetCollisionProfileName(TEXT("Ragdoll"));
GetMesh()->SetAllBodiesSimulatePhysics(true);                  // SkeletalMeshComponent.h:2344
GetMesh()->SetAllBodiesBelowSimulatePhysics(TEXT("spine_01"), true); // :2376 partial ragdoll
GetMesh()->SetPhysicsBlendWeight(1.f);                         // :2353 blend animation <-> sim
```

## Physical Materials

`PhysicsCore/Public/PhysicalMaterials/PhysicalMaterial.h:103`.

```cpp
float Friction;                 // :115 kinetic;  StaticFriction :119
float Restitution;              // :131 0 dead, 1 elastic
float Density;                  // :147 g/cm^3, drives mass from shape volume
float SleepLinearVelocityThreshold;   // :151;  SleepAngularVelocityThreshold :155
int32 SleepCounterThreshold;          // :159;  RaiseMassToPower :167
TEnumAsByte<EPhysicalSurface> SurfaceType;  // :181
// FrictionCombineMode / RestitutionCombineMode are EFrictionCombineMode::Type and only
// apply when bOverrideFrictionCombineMode / bOverrideRestitutionCombineMode is set.

MyMesh->SetPhysMaterialOverride(MyPhysMaterial);   // PrimitiveComponent.h:3008

if (UPhysicalMaterial* PhysMat = Hit.PhysMaterial.Get())   // needs bReturnPhysicalMaterial
{
	const EPhysicalSurface Surface = UPhysicalMaterial::DetermineSurfaceType(PhysMat); // :242
}
```

`EPhysicalSurface` (`PhysicsCore/Public/Chaos/ChaosEngineInterface.h:20`) is `SurfaceType_Default` plus `SurfaceType1..SurfaceType62`, named in Project Settings > Physics.

## Chaos Solver and Async Physics

Chaos is the only physics backend in 5.8. `FChaosScene` (`PhysicsCore/Public/Chaos/ChaosScene.h:84`) owns the solver and is driven by `SetUpForFrame` (:141), `StartFrame` (:140), `EndFrame` (:142); `WaitPhysScenes` (:122) blocks for results.

Project settings live on `UPhysicsSettings` / `UPhysicsSettingsCore` under `[/Script/Engine.PhysicsSettings]`:

| Setting | Header | Meaning |
|---|---|---|
| `DefaultGravityZ` | `PhysicsSettingsCore.h:25` | world gravity, `-980` default |
| `BounceThresholdVelocity` | `:79` | below this, restitution is skipped |
| `MaxDepenetrationVelocity` | `:95` | caps push-out speed; 0 = unlimited |
| `bSimulateSkeletalMeshOnDedicatedServer` | `:114` | off saves ragdoll cost on servers |
| `DefaultShapeComplexity` | `:119` | project-wide `ECollisionTraceFlag` |
| `SolverOptions` | `:123` | `FChaosSolverConfiguration` |
| `bSubstepping` / `MaxSubstepDeltaTime` / `MaxSubsteps` | `PhysicsSettings.h:325,341,345` | fixes tunnelling and jitter for small fast bodies |
| `bTickPhysicsAsync` / `AsyncFixedTimeStepSize` | `:333,337` | fixed-rate physics thread |
| `PhysicsPrediction` | `:267` | `FPhysicsPredictionSettings`, gates resimulation |

`FChaosSolverConfiguration` (`Chaos/Public/ChaosSolverConfiguration.h:51`) exposes `PositionIterations`, `VelocityIterations`, `ProjectionIterations`, `CollisionMarginFraction`, `CollisionCullDistance`, `CollisionMaxPushOutVelocity`.

Diagnostic CVars that exist in 5.8:

```
p.Chaos.DebugDraw.Enabled 1                              ChaosDebugDrawComponent.cpp:29
p.Chaos.DebugDraw.Radius 3000 / p.Chaos.DebugDraw.MaxLines 20000   capture limits
p.Chaos.Solver.Iterations.Position|Velocity|Projection   PBDRigidsSolver.cpp:337,340,343
p.Chaos.Solver.Deterministic :346   p.Chaos.Solver.UseCCD :406 (global CCD switch)
p.Chaos.DedicatedThreadEnabled                           ChaosSolversModule.cpp:23
```

With `bTickPhysicsAsync` on, gameplay code that must run in lockstep with the solver uses the async tick instead of `Tick`: set `bAsyncPhysicsTickEnabled` in an actor constructor (protected, `GameFramework/Actor.h:699`) or call `SetAsyncPhysicsTickEnabled(true)` on a component (`ActorComponent.h:1435`; the component field at `Components/ActorComponent.h:407` is private) and override `AsyncPhysicsTickActor(float DeltaTime, float SimTime)` (`Actor.h:2930`) or `AsyncPhysicsTickComponent(float DeltaTime, float SimTime)` (`ActorComponent.h:985`). Feed inputs across the boundary with `UAsyncPhysicsInputComponent` (`Engine/Public/Physics/AsyncPhysicsInputComponent.h:10`).

Below `FBodyInstance`, bodies are addressed as `Chaos::FPhysicsObject`. `UPrimitiveComponent` implements `IPhysicsComponent` and exposes `GetPhysicsObjectById`, `GetPhysicsObjectByName` and `GetAllPhysicsObjects` (`Components/PrimitiveComponent.h:3248-3250`); `FHitResult::PhysicsObject` and `PhysicsObjectOwner` (`Engine/HitResult.h:134-137`) identify what a query hit when no actor owns it.

## Network Physics

Physics simulates on the server. Hit and overlap events fire wherever the simulation runs, so clients see nothing unless the actor also simulates locally or you replicate the result.

- `AActor::SetPhysicsReplicationMode(EPhysicsReplicationMode)` / `GetPhysicsReplicationMode()` (`GameFramework/Actor.h:924,928`) select `Default`, `PredictiveInterpolation`, `Resimulation` or `None` (`Engine/EngineTypes.h:3614`).
- `Resimulation` requires Project Settings > Physics > Physics Prediction (`UPhysicsSettings::PhysicsPrediction`); `PredictiveInterpolation` runs without it but works best with it enabled (`Engine/EngineTypes.h:3619-3629`).
- `UNetworkPhysicsComponent` (`Engine/Public/Physics/NetworkPhysicsComponent.h:2290`) records input and state history on the physics thread so the client can resimulate from a corrected past state.
- Transform sync for non-predicted actors still rides on `bReplicateMovement` / `FRepMovement`.

Replication rules, conditions and RPC design belong to `ue-networking-replication`; this skill only covers which physics mode to pick.

## Module Dependencies

```csharp
// MyGame.Build.cs
PublicDependencyModuleNames.AddRange(new string[] {
	"Core", "CoreUObject",
	"Engine",       // UPrimitiveComponent, FHitResult, UWorld queries, UCollisionProfile
	"PhysicsCore",  // FCollisionShape, UPhysicalMaterial, FBodyInstanceCore, ECollisionTraceFlag
});
// Add "Chaos" only for Chaos:: types; "GeometryCollectionEngine"/"FieldSystemEngine" for destruction.
```

## Deprecated — do not use

| Do not emit | Use in 5.8 | Source |
|---|---|---|
| `Hit.Actor` | `Hit.GetActor()` / `Hit.GetHitObjectHandle()` | no `Actor` member; `HitObjectHandle` in `Engine/HitResult.h:127` |
| `Comp->bGenerateOverlapEvents = true` | `Comp->SetGenerateOverlapEvents(true)` | private field, `Components/PrimitiveComponent.h:429` |
| `Params.GetIgnoredActors()` | `Params.GetIgnoredSourceObjects()` | `UE_DEPRECATED(5.5)` in `CollisionQueryParams.h:145` |
| `Params.ClearIgnoredActors()` | `Params.ClearIgnoredSourceObjects()` | `UE_DEPRECATED(5.5)` in `CollisionQueryParams.h:180` |
| `ObjParams.ObjectTypesToQuery` | `GetObjectTypesToQuery()` / `SetObjectTypesToQuery()` | `UE_DEPRECATED(5.8)` in `CollisionQueryParams.h:490` |
| `GetAllObjectsQueryFlag()`, `GetQueryBitfield()` | the `...64()` forms | `UE_DEPRECATED(5.8)` in `CollisionQueryParams.h:368,571` |
| `Constraint->SetAngularVelocityDrive(bSwing, bTwist)` | `SetAngularVelocityDriveTwistAndSwing(bTwist, bSwing)` — argument order is swapped | `DeprecatedFunction` meta, `PhysicsEngine/PhysicsConstraintComponent.h:187-199` |
| `PhysMat->DestructibleDamageThresholdScale` | Geometry Collection damage thresholds | renamed `_DEPRECATED`, `UE_DEPRECATED(5.3)` in `PhysicalMaterials/PhysicalMaterial.h:170` |
| `FChaosQueryFilterData` / `FQueryFilterData` | `ChaosInterface::FSceneQueryCommonParams` | `UE_DEPRECATED(5.8)` in `PhysicsInterfaceTypesCore.h:410` / `ChaosInterfaceWrapperCore.h:35` |
| `ICollisionQueryFilterCallbackBase` | `ICollisionQueryFilterCallback` | `UE_DEPRECATED(5.8)` in `CollisionQueryFilterCallbackCore.h:63` |
| `GetQueryFilterData()` / `GetSimulationFilterData()` | `GetCombinedShapeFilterData()` | `UE_DEPRECATED(5.7)` in `ChaosInterfaceWrapperCore.h:98` |
| `BodySetup->ChaosTriMeshes` | `BodySetup->TriMeshGeometries` (`:278`) | `UE_DEPRECATED(5.4)` in `PhysicsEngine/BodySetup.h:308` |
| PhysX types (`PxRigidActor`, `PxScene`, `PhysX` module) | Chaos equivalents | no PhysX module ships in 5.8 |

## Common Mistakes

**`UFUNCTION()` on an out-of-line definition:** UHT only reads headers, so a `UFUNCTION()` written above `void AMyPickup::OnHit` in the `.cpp` generates no reflection data and `AddDynamic` silently binds nothing. Declare the handler inside the class body with `UFUNCTION()`; define it plainly in the `.cpp`.

**Handler signature drift:** `AddDynamic` matches the delegate exactly. A missing `int32 OtherBodyIndex` or a `FHitResult` taken by value will not compile, and a stale copy of an old signature will not bind.

**Overlap events set on one side only:** both components need a non-ignore response *and* `SetGenerateOverlapEvents(true)`. Setting it only on the trigger is the most common "my overlap never fires".

**Hit events without `SetNotifyRigidBodyCollision(true)`:** for simulated bodies `ECR_Block` alone is not enough; `OnComponentHit` stays silent (swept moves dispatch regardless).

**Profile applied after manual responses:** `SetCollisionProfileName` resets the whole response container. Order is profile first, overrides second.

**Mixing channel families:** a component whose object type is `ECC_Pawn` is invisible to `LineTraceSingleByObjectType` querying `ECC_GameTraceChannel3`, and a trace channel can never be an object type. `bTraceType` in the ini decides which one a slot is.

**`bTraceComplex = true` by default:** per-triangle queries are several times the cost of the simple hull and skip simple-only bodies. Enable it only where triangle accuracy matters.

**`SetPhysicsLinearVelocity` as a movement API:** it overwrites the solver's velocity every call and fights contacts. Use `AddForce` / `AddImpulse` unless you are deliberately teleporting velocity.

**Fast bodies tunnelling:** enable `SetUseCCD(true)` on the body, or substepping project-wide, rather than shrinking the timestep globally. For traces every tick on many actors, use `AsyncLineTraceByChannel` or throttle to 5–10 Hz with a timer.

## Related Skills

- `ue-actor-component-architecture` — component creation, attachment, registration and lifecycle for shape and mesh components
- `ue-character-movement` — `UCharacterMovementComponent` floor checks, capsule sweeps and step-up logic
- `ue-networking-replication` — `DOREPLIFETIME`, replication conditions and RPCs for anything you send about a physics result
- `ue-ai-navigation` — perception traces, navmesh queries and `UNavigationSystemV1`
- `ue-procedural-generation` — collision setup on runtime-generated meshes
- Also relevant: `ue-testing-debugging`, `ue-gameplay-framework`, `ue-mover`
