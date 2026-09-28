# Mover C++ Recipes (UE 5.8)

Complete, compilable templates for the Mover plugin (Experimental in 5.8). Every signature is copied from the 5.8 headers under `Plugins/Experimental/Mover/Source/Mover/Public`. Module: `"Mover"`.

## 1. A pawn with a Character Mover Component that produces input

`UMoverComponent` calls `IMoverInputProducerInterface::ProduceInput` on its `InputProducer` and on every component of the owner that implements the interface (the header says `bGatherInputFromAllInputProducerComponents` gates this, but the 5.8 `BeginPlay` gathers them unconditionally). Implementing the interface on the pawn itself works because `UMoverComponent::BeginPlay` adopts the owning actor as `InputProducer` when none is set and the actor implements the interface (`MoverComponent.cpp:293`).

```cpp
// MyMoverPawn.h
#pragma once

#include "GameFramework/Pawn.h"
#include "MoverSimulationTypes.h"
#include "MyMoverPawn.generated.h"

class UCapsuleComponent;
class UCharacterMoverComponent;
class UInputAction;
struct FInputActionValue;

UCLASS()
class MYGAME_API AMyMoverPawn : public APawn, public IMoverInputProducerInterface
{
    GENERATED_BODY()

public:
    AMyMoverPawn(const FObjectInitializer& ObjectInitializer);

    virtual void BeginPlay() override;
    virtual void SetupPlayerInputComponent(UInputComponent* PlayerInputComponent) override;

    UFUNCTION(BlueprintPure, Category = "My Game|Movement")
    UCharacterMoverComponent* GetMoverComponent() const { return MoverComponent; }

protected:
    /** Entry point called by UMoverComponent::ProduceInput just before a simulation frame. */
    virtual void ProduceInput_Implementation(int32 SimTimeMs, FMoverInputCmdContext& InputCmdResult) override;

    UPROPERTY(EditDefaultsOnly, Category = "My Game|Input")
    TObjectPtr<UInputAction> MoveAction;

    UPROPERTY(EditDefaultsOnly, Category = "My Game|Input")
    TObjectPtr<UInputAction> JumpAction;

    UPROPERTY(VisibleAnywhere, Category = "My Game|Movement")
    TObjectPtr<UCapsuleComponent> Capsule;

    UPROPERTY(VisibleAnywhere, Category = "My Game|Movement")
    TObjectPtr<UCharacterMoverComponent> MoverComponent;

private:
    void OnMoveTriggered(const FInputActionValue& Value);
    void OnMoveCompleted(const FInputActionValue& Value);
    void OnJumpStarted(const FInputActionValue& Value);
    void OnJumpReleased(const FInputActionValue& Value);

    FVector CachedMoveInputIntent = FVector::ZeroVector;
    bool bJumpHeld = false;
    bool bJumpPressedThisFrame = false;
};
```

```cpp
// MyMoverPawn.cpp
#include "MyMoverPawn.h"

#include "Components/CapsuleComponent.h"
#include "DefaultMovementSet/CharacterMoverComponent.h"
#include "EnhancedInputComponent.h"
#include "MoverDataModelTypes.h"

AMyMoverPawn::AMyMoverPawn(const FObjectInitializer& ObjectInitializer)
    : Super(ObjectInitializer)
{
    Capsule = CreateDefaultSubobject<UCapsuleComponent>(TEXT("Capsule"));
    Capsule->InitCapsuleSize(40.f, 90.f);
    SetRootComponent(Capsule);

    MoverComponent = CreateDefaultSubobject<UCharacterMoverComponent>(TEXT("MoverComponent"));
}

void AMyMoverPawn::BeginPlay()
{
    Super::BeginPlay();

    // Redundant but harmless: UMoverComponent::BeginPlay (run inside Super::BeginPlay) already
    // adopted this pawn as InputProducer and built its InputProducers list. Assigning a different
    // producer this late would not add it to that list.
    MoverComponent->InputProducer = this;
}

void AMyMoverPawn::SetupPlayerInputComponent(UInputComponent* PlayerInputComponent)
{
    Super::SetupPlayerInputComponent(PlayerInputComponent);

    if (UEnhancedInputComponent* EnhancedInput = Cast<UEnhancedInputComponent>(PlayerInputComponent))
    {
        EnhancedInput->BindAction(MoveAction, ETriggerEvent::Triggered, this, &AMyMoverPawn::OnMoveTriggered);
        EnhancedInput->BindAction(MoveAction, ETriggerEvent::Completed, this, &AMyMoverPawn::OnMoveCompleted);
        EnhancedInput->BindAction(JumpAction, ETriggerEvent::Started, this, &AMyMoverPawn::OnJumpStarted);
        EnhancedInput->BindAction(JumpAction, ETriggerEvent::Completed, this, &AMyMoverPawn::OnJumpReleased);
    }
}

void AMyMoverPawn::OnMoveTriggered(const FInputActionValue& Value)
{
    const FVector2D Axis = Value.Get<FVector2D>();
    CachedMoveInputIntent = FVector(Axis.Y, Axis.X, 0.0);
}

void AMyMoverPawn::OnMoveCompleted(const FInputActionValue& Value)
{
    CachedMoveInputIntent = FVector::ZeroVector;
}

void AMyMoverPawn::OnJumpStarted(const FInputActionValue& Value)
{
    bJumpHeld = true;
    bJumpPressedThisFrame = true;
}

void AMyMoverPawn::OnJumpReleased(const FInputActionValue& Value)
{
    bJumpHeld = false;
}

void AMyMoverPawn::ProduceInput_Implementation(int32 SimTimeMs, FMoverInputCmdContext& InputCmdResult)
{
    FCharacterDefaultInputs& CharacterInputs = InputCmdResult.InputCollection.FindOrAddMutableDataByType<FCharacterDefaultInputs>();

    CharacterInputs.ControlRotation = FRotator::ZeroRotator;
    if (const APlayerController* PC = Cast<APlayerController>(GetController()))
    {
        CharacterInputs.ControlRotation = PC->GetControlRotation();
    }

    const FVector WorldIntent = CharacterInputs.ControlRotation.RotateVector(CachedMoveInputIntent);
    CharacterInputs.SetMoveInput(EMoveInputType::DirectionalIntent, WorldIntent);
    CharacterInputs.OrientationIntent = CharacterInputs.GetMoveInput().GetSafeNormal();

    CharacterInputs.bIsJumpPressed = bJumpHeld;
    CharacterInputs.bIsJumpJustPressed = bJumpPressedThisFrame;

    // "Just pressed" is edge-triggered: consume it so it lasts exactly one simulation frame.
    bJumpPressedThisFrame = false;
}
```

`ProduceInput` runs only on the controlling instance (autonomous proxy or authority) and is **not** re-run during resimulation. Put aim assist and lock-on here; put movement rules in a mode or layered move.

## 2. A custom movement mode

```cpp
// MyClimbingMode.h
#pragma once

#include "MovementMode.h"
#include "MyClimbingMode.generated.h"

class UCommonLegacyMovementSettings;

namespace MyModeNames
{
    const FName Climbing = TEXT("Climbing");
}

UCLASS(Blueprintable, BlueprintType)
class MYGAME_API UMyClimbingMode : public UBaseMovementMode
{
    GENERATED_BODY()

public:
    UMyClimbingMode(const FObjectInitializer& ObjectInitializer);

    virtual void GenerateMove_Implementation(const FMoverSimContext& SimContext, const FMoverTickStartData& StartState, const FMoverTimeStep& TimeStep, FProposedMove& OutProposedMove) const override;
    virtual void SimulationTick_Implementation(const FSimulationTickParams& Params, FMoverTickEndData& OutputState) override;

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "My Game|Climbing", meta = (ClampMin = "1", ForceUnits = "cm/s"))
    float ClimbSpeed = 250.f;

protected:
    virtual void Activate(const FMoverEventContext& Context, FName PrevModeName, const FMoverSimContext& SimContext, const FMoverTickStartData& StartState, FMoverSyncState* OutSyncState, FMoverAuxStateContext* OutAuxState) override;
    virtual void Deactivate(const FMoverEventContext& Context, FName NextModeName, const FMoverSimContext& SimContext) override;
    virtual void OnRegistered(const FName ModeName, const FMoverSimContext& SimContext) override;
    virtual void OnUnregistered(const FMoverSimContext& SimContext) override;

    /** Cached shared settings instance, owned by the Mover component. Never per-frame state. */
    TObjectPtr<const UCommonLegacyMovementSettings> CommonLegacySettings;
};
```

```cpp
// MyClimbingMode.cpp
#include "MyClimbingMode.h"

#include "DefaultMovementSet/Settings/CommonLegacyMovementSettings.h"
#include "MoveLibrary/MovementRecord.h"
#include "MoveLibrary/MovementUtils.h"
#include "MoveLibrary/MoverBlackboard.h"
#include "MoverComponent.h"
#include "MoverDataModelTypes.h"

UMyClimbingMode::UMyClimbingMode(const FObjectInitializer& ObjectInitializer)
    : Super(ObjectInitializer)
{
    SharedSettingsClasses.Add(UCommonLegacyMovementSettings::StaticClass());
    GameplayTags.AddTag(Mover_IsInAir);
}

void UMyClimbingMode::OnRegistered(const FName ModeName, const FMoverSimContext& SimContext)
{
    Super::OnRegistered(ModeName, SimContext);

    CommonLegacySettings = GetMoverComponent()->FindSharedSettings<UCommonLegacyMovementSettings>();
    ensureMsgf(CommonLegacySettings, TEXT("UMyClimbingMode found no UCommonLegacyMovementSettings on %s."), *GetPathNameSafe(this));
}

void UMyClimbingMode::OnUnregistered(const FMoverSimContext& SimContext)
{
    CommonLegacySettings = nullptr;

    Super::OnUnregistered(SimContext);
}

void UMyClimbingMode::Activate(const FMoverEventContext& Context, FName PrevModeName, const FMoverSimContext& SimContext, const FMoverTickStartData& StartState, FMoverSyncState* OutSyncState, FMoverAuxStateContext* OutAuxState)
{
    Super::Activate(Context, PrevModeName, SimContext, StartState, OutSyncState, OutAuxState);

    // Cached floor data is meaningless while climbing.
    if (UMoverBlackboard* SimBlackboard = GetMoverComponent()->GetSimBlackboard_Mutable())
    {
        SimBlackboard->Invalidate(CommonBlackboard::LastFloorResult);
    }
}

void UMyClimbingMode::Deactivate(const FMoverEventContext& Context, FName NextModeName, const FMoverSimContext& SimContext)
{
    Super::Deactivate(Context, NextModeName, SimContext);
}

void UMyClimbingMode::GenerateMove_Implementation(const FMoverSimContext& SimContext, const FMoverTickStartData& StartState, const FMoverTimeStep& TimeStep, FProposedMove& OutProposedMove) const
{
    const UMoverComponent* MoverComp = GetMoverComponent();
    const FCharacterDefaultInputs* CharacterInputs = StartState.InputCmd.InputCollection.FindDataByType<FCharacterDefaultInputs>();
    const FMoverDefaultSyncState* StartingSyncState = StartState.SyncState.SyncStateCollection.FindDataByType<FMoverDefaultSyncState>();
    check(StartingSyncState);

    const FVector UpDir = MoverComp->GetUpDirection();
    const FVector MoveIntent = CharacterInputs ? CharacterInputs->GetMoveInput_WorldSpace() : FVector::ZeroVector;
    const double VerticalIntent = FVector::DotProduct(MoveIntent, UpDir);

    OutProposedMove.MixMode = EMoveMixMode::OverrideAll;
    OutProposedMove.bHasDirIntent = !MoveIntent.IsNearlyZero();
    OutProposedMove.DirectionIntent = MoveIntent.GetSafeNormal();
    OutProposedMove.LinearVelocity = UpDir * VerticalIntent * ClimbSpeed;
    OutProposedMove.AngularVelocityDegrees = FVector::ZeroVector;
}

void UMyClimbingMode::SimulationTick_Implementation(const FSimulationTickParams& Params, FMoverTickEndData& OutputState)
{
    UMoverComponent* MoverComp = GetMoverComponent();
    USceneComponent* UpdatedComponent = Params.MovingComps.UpdatedComponent.Get();
    if (!UpdatedComponent)
    {
        return;
    }

    const FMoverDefaultSyncState* StartingSyncState = Params.StartState.SyncState.SyncStateCollection.FindDataByType<FMoverDefaultSyncState>();
    check(StartingSyncState);

    FMoverDefaultSyncState& OutputSyncState = OutputState.SyncState.SyncStateCollection.FindOrAddMutableDataByType<FMoverDefaultSyncState>();

    const float DeltaSeconds = Params.TimeStep.StepMs * 0.001f;

    FMovementRecord MoveRecord;
    MoveRecord.SetDeltaSeconds(DeltaSeconds);

    const FVector MoveDelta = Params.ProposedMove.LinearVelocity * DeltaSeconds;
    const FQuat TargetOrient = StartingSyncState->GetOrientation_WorldSpace().Quaternion();

    FHitResult Hit(1.f);
    if (!MoveDelta.IsNearlyZero())
    {
        UMovementUtils::TrySafeMoveUpdatedComponent(Params.MovingComps, MoveDelta, TargetOrient, true, Hit, ETeleportType::None, MoveRecord);
    }

    if (Hit.IsValidBlockingHit())
    {
        FMoverOnImpactParams ImpactParams(MyModeNames::Climbing, Hit, MoveDelta);
        MoverComp->HandleImpact(ImpactParams);

        UMovementUtils::TryMoveToSlideAlongSurface(Params.MovingComps, MoveDelta, 1.f - Hit.Time, TargetOrient, Hit.Normal, Hit, true, MoveRecord);
    }

    const FVector FinalVelocity = MoveRecord.GetRelevantVelocity();

    OutputSyncState.SetTransforms_WorldSpace(UpdatedComponent->GetComponentLocation(),
                                             UpdatedComponent->GetComponentRotation(),
                                             FinalVelocity,
                                             FVector::ZeroVector,
                                             nullptr);

    UpdatedComponent->ComponentVelocity = FinalVelocity;
}
```

Register and enter the mode:

```cpp
static void MyEnterClimbing(UMoverComponent& MoverComp)
{
    MoverComp.AddMovementModeFromClass(MyModeNames::Climbing, UMyClimbingMode::StaticClass());
    MoverComp.QueueNextMode(MyModeNames::Climbing);
}
```

Set `bSupportsAsync = true` in the constructor only when `GenerateMove` and `SimulationTick` touch no scene components — the recipe above does, so it must stay false.

## 3. A custom sync-state data struct

Everything the simulation must remember between frames belongs here, not on the mode object.

```cpp
// MyClimbState.h
#pragma once

#include "MoverTypes.h"
#include "MyClimbState.generated.h"

USTRUCT(BlueprintType)
struct FMyClimbState : public FMoverDataStructBase
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadWrite, Category = "My Game|Climbing")
    FVector SurfaceNormal = FVector::ZeroVector;

    UPROPERTY(BlueprintReadWrite, Category = "My Game|Climbing")
    float StaminaSeconds = 0.f;

    virtual ~FMyClimbState() = default;

    virtual FMoverDataStructBase* Clone() const override;
    virtual bool NetSerialize(FArchive& Ar, UPackageMap* Map, bool& bOutSuccess) override;
    virtual UScriptStruct* GetScriptStruct() const override { return StaticStruct(); }
    virtual void ToString(FAnsiStringBuilderBase& Out) const override;
    virtual bool ShouldReconcile(const FMoverDataStructBase& AuthorityState) const override;
    virtual void Interpolate(const FMoverDataStructBase& From, const FMoverDataStructBase& To, float Pct) override;
};

template<>
struct TStructOpsTypeTraits<FMyClimbState> : public TStructOpsTypeTraitsBase2<FMyClimbState>
{
    enum
    {
        WithNetSerializer = true,
        WithCopy = true
    };
};
```

```cpp
// MyClimbState.cpp
#include "MyClimbState.h"

#include "Engine/NetSerialization.h"

FMoverDataStructBase* FMyClimbState::Clone() const
{
    return new FMyClimbState(*this);
}

bool FMyClimbState::NetSerialize(FArchive& Ar, UPackageMap* Map, bool& bOutSuccess)
{
    Super::NetSerialize(Ar, Map, bOutSuccess);

    SerializePackedVector<10, 16>(SurfaceNormal, Ar);
    Ar << StaminaSeconds;

    bOutSuccess = true;
    return true;
}

void FMyClimbState::ToString(FAnsiStringBuilderBase& Out) const
{
    Super::ToString(Out);
    Out.Appendf("ClimbStamina: %.2f\n", StaminaSeconds);
}

bool FMyClimbState::ShouldReconcile(const FMoverDataStructBase& AuthorityState) const
{
    const FMyClimbState& TypedAuthority = static_cast<const FMyClimbState&>(AuthorityState);

    return !SurfaceNormal.Equals(TypedAuthority.SurfaceNormal, 0.01)
        || !FMath::IsNearlyEqual(StaminaSeconds, TypedAuthority.StaminaSeconds, 0.01f);
}

void FMyClimbState::Interpolate(const FMoverDataStructBase& From, const FMoverDataStructBase& To, float Pct)
{
    const FMyClimbState& TypedFrom = static_cast<const FMyClimbState&>(From);
    const FMyClimbState& TypedTo = static_cast<const FMyClimbState&>(To);

    SurfaceNormal = FMath::Lerp(TypedFrom.SurfaceNormal, TypedTo.SurfaceNormal, Pct);
    StaminaSeconds = FMath::Lerp(TypedFrom.StaminaSeconds, TypedTo.StaminaSeconds, Pct);
}
```

Read and write it inside the simulation, and make it survive frames where nobody writes it:

```cpp
// Inside UMyClimbingMode::SimulationTick_Implementation
static void MyAdvanceClimbStamina(const FSimulationTickParams& Params, FMoverTickEndData& OutputState)
{
    const float DeltaSeconds = Params.TimeStep.StepMs * 0.001f;
    const FMyClimbState* PriorClimb = Params.StartState.SyncState.SyncStateCollection.FindDataByType<FMyClimbState>();

    FMyClimbState& OutClimb = OutputState.SyncState.SyncStateCollection.FindOrAddMutableDataByType<FMyClimbState>();
    OutClimb.StaminaSeconds = PriorClimb ? PriorClimb->StaminaSeconds - DeltaSeconds : 5.f;
}

// Once, at setup time, so the struct survives frames where nobody writes it:
static void MyRegisterPersistentClimbState(UMoverComponent& MoverComp)
{
    MoverComp.PersistentSyncStateDataTypes.Add(FMoverDataPersistence(FMyClimbState::StaticStruct(), /*bShouldCopyBetweenFrames=*/true));
}
```

## 4. A custom layered move with NetSerialize

```cpp
// MyGrappleMove.h
#pragma once

#include "LayeredMove.h"
#include "MyGrappleMove.generated.h"

USTRUCT(BlueprintType)
struct FMyGrappleMove : public FLayeredMoveBase
{
    GENERATED_BODY()

    FMyGrappleMove();
    virtual ~FMyGrappleMove() = default;

    /** Worldspace point the actor is pulled towards. */
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "My Game|Grapple")
    FVector AnchorLocation = FVector::ZeroVector;

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "My Game|Grapple", meta = (ClampMin = "0", ForceUnits = "cm/s"))
    float PullSpeed = 1500.f;

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "My Game|Grapple", meta = (ClampMin = "1", ForceUnits = "cm"))
    float ArrivalTolerance = 50.f;

    /** Simulation state: set in GenerateMove, read by IsFinished, so it must be serialized. */
    UPROPERTY(VisibleInstanceOnly, BlueprintReadOnly, Category = "My Game|Grapple")
    bool bReachedAnchor = false;

    virtual bool GenerateMove(const FMoverTickStartData& StartState, const FMoverTimeStep& TimeStep, const UMoverComponent* MoverComp, UMoverBlackboard* SimBlackboard, FProposedMove& OutProposedMove) override;
    virtual bool IsFinished(double CurrentSimTimeMs) const override;
    virtual FLayeredMoveBase* Clone() const override;
    virtual void NetSerialize(FArchive& Ar) override;
    virtual UScriptStruct* GetScriptStruct() const override;
    virtual FString ToSimpleString() const override;
};

template<>
struct TStructOpsTypeTraits<FMyGrappleMove> : public TStructOpsTypeTraitsBase2<FMyGrappleMove>
{
    enum
    {
        WithCopy = true
    };
};
```

```cpp
// MyGrappleMove.cpp
#include "MyGrappleMove.h"

#include "Engine/NetSerialization.h"
#include "MoverDataModelTypes.h"
#include "MoverSimulationTypes.h"

FMyGrappleMove::FMyGrappleMove()
{
    DurationMs = 1500.f;
    MixMode = EMoveMixMode::OverrideVelocity;
    FinishVelocitySettings.FinishVelocityMode = ELayeredMoveFinishVelocityMode::SetVelocity;
    FinishVelocitySettings.SetVelocity = FVector::ZeroVector;
}

bool FMyGrappleMove::GenerateMove(const FMoverTickStartData& StartState, const FMoverTimeStep& TimeStep, const UMoverComponent* MoverComp, UMoverBlackboard* SimBlackboard, FProposedMove& OutProposedMove)
{
    const FMoverDefaultSyncState* SyncState = StartState.SyncState.SyncStateCollection.FindDataByType<FMoverDefaultSyncState>();
    check(SyncState);

    const FVector ToAnchor = AnchorLocation - SyncState->GetLocation_WorldSpace();
    bReachedAnchor = ToAnchor.SizeSquared() <= FMath::Square(ArrivalTolerance);

    if (bReachedAnchor)
    {
        return false;
    }

    OutProposedMove.MixMode = MixMode;
    OutProposedMove.LinearVelocity = ToAnchor.GetSafeNormal() * PullSpeed;
    return true;
}

bool FMyGrappleMove::IsFinished(double CurrentSimTimeMs) const
{
    return bReachedAnchor || Super::IsFinished(CurrentSimTimeMs);
}

FLayeredMoveBase* FMyGrappleMove::Clone() const
{
    return new FMyGrappleMove(*this);
}

void FMyGrappleMove::NetSerialize(FArchive& Ar)
{
    Super::NetSerialize(Ar);

    SerializePackedVector<10, 16>(AnchorLocation, Ar);
    Ar << PullSpeed;
    Ar << ArrivalTolerance;
    Ar << bReachedAnchor;
}

UScriptStruct* FMyGrappleMove::GetScriptStruct() const
{
    return FMyGrappleMove::StaticStruct();
}

FString FMyGrappleMove::ToSimpleString() const
{
    return FString::Printf(TEXT("MyGrapple"));
}
```

Queueing (the queue keeps your shared pointer until the simulation flushes it, so finish setting it up first and leave it alone afterwards):

```cpp
// UE_DEFINE_GAMEPLAY_TAG_STATIC(TAG_MyGame_Grapple, "MyGame.Movement.Grapple");

static void MyStartGrapple(UMoverComponent& MoverComp, const FVector& HookTarget)
{
    TSharedPtr<FMyGrappleMove> Grapple = MakeShared<FMyGrappleMove>();
    Grapple->AnchorLocation = HookTarget;
    Grapple->PullSpeed = 2000.f;
    MoverComp.QueueLayeredMove(Grapple);
}

static void MyInspectAndCancelGrapple(UMoverComponent& MoverComp)
{
    if (const FMyGrappleMove* Active = MoverComp.FindActiveLayeredMoveByType<FMyGrappleMove>())
    {
        UE_LOG(LogMyGame, Verbose, TEXT("Grappling towards %s"), *Active->AnchorLocation.ToString());
    }

    MoverComp.CancelFeaturesWithTag(TAG_MyGame_Grapple, /*bRequireExactMatch=*/false);
}
```

`CancelFeaturesWithTag` only matches moves whose `HasGameplayTag` override returns true, so implement `virtual bool HasGameplayTag(FGameplayTag TagToFind, bool bExactMatch) const override;` if you want tag-based cancellation.

## 5. A custom movement modifier

```cpp
// MySlowModifier.h
#pragma once

#include "MovementModifier.h"
#include "MySlowModifier.generated.h"

USTRUCT(BlueprintType)
struct FMySlowModifier : public FMovementModifierBase
{
    GENERATED_BODY()

    FMySlowModifier();
    virtual ~FMySlowModifier() = default;

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "My Game|Slow", meta = (ClampMin = "0.01", ClampMax = "1.0"))
    float SpeedMultiplier = 0.5f;

    virtual void OnStart(UMoverComponent* MoverComp, const FMoverTimeStep& TimeStep, const FMoverSyncState& SyncState, const FMoverAuxStateContext& AuxState) override;
    virtual void OnEnd(UMoverComponent* MoverComp, const FMoverTimeStep& TimeStep, const FMoverSyncState& SyncState, const FMoverAuxStateContext& AuxState) override;
    virtual FMovementModifierBase* Clone() const override;
    virtual void NetSerialize(FArchive& Ar) override;
    virtual UScriptStruct* GetScriptStruct() const override;
    virtual FString ToSimpleString() const override;

protected:
    /** Max speed to restore when this modifier ends. */
    UPROPERTY()
    float CachedMaxSpeed = 0.f;
};

template<>
struct TStructOpsTypeTraits<FMySlowModifier> : public TStructOpsTypeTraitsBase2<FMySlowModifier>
{
    enum
    {
        WithCopy = true
    };
};
```

```cpp
// MySlowModifier.cpp
#include "MySlowModifier.h"

#include "DefaultMovementSet/Settings/CommonLegacyMovementSettings.h"
#include "MoverComponent.h"

FMySlowModifier::FMySlowModifier()
{
    DurationMs = 3000.f;
}

void FMySlowModifier::OnStart(UMoverComponent* MoverComp, const FMoverTimeStep& TimeStep, const FMoverSyncState& SyncState, const FMoverAuxStateContext& AuxState)
{
    if (UCommonLegacyMovementSettings* Settings = MoverComp->FindSharedSettings_Mutable<UCommonLegacyMovementSettings>())
    {
        CachedMaxSpeed = Settings->MaxSpeed;
        Settings->MaxSpeed = CachedMaxSpeed * SpeedMultiplier;
    }
}

void FMySlowModifier::OnEnd(UMoverComponent* MoverComp, const FMoverTimeStep& TimeStep, const FMoverSyncState& SyncState, const FMoverAuxStateContext& AuxState)
{
    if (UCommonLegacyMovementSettings* Settings = MoverComp->FindSharedSettings_Mutable<UCommonLegacyMovementSettings>())
    {
        Settings->MaxSpeed = CachedMaxSpeed;
    }
}

FMovementModifierBase* FMySlowModifier::Clone() const
{
    return new FMySlowModifier(*this);
}

void FMySlowModifier::NetSerialize(FArchive& Ar)
{
    Super::NetSerialize(Ar);

    Ar << SpeedMultiplier;
    Ar << CachedMaxSpeed;
}

UScriptStruct* FMySlowModifier::GetScriptStruct() const
{
    return FMySlowModifier::StaticStruct();
}

FString FMySlowModifier::ToSimpleString() const
{
    return FString::Printf(TEXT("MySlow"));
}
```

```cpp
static FMovementModifierHandle MyApplySlow(UMoverComponent& MoverComp)
{
    return MoverComp.QueueMovementModifier(MakeShared<FMySlowModifier>());
}

static void MyClearSlow(UMoverComponent& MoverComp, const FMovementModifierHandle& SlowHandle)
{
    if (MoverComp.IsModifierActiveOrQueued(SlowHandle))
    {
        MoverComp.CancelModifierFromHandle(SlowHandle);
    }
}
```

Mover keeps one modifier per type, because the default `Matches` compares only the struct type. To allow several at once, override `virtual bool Matches(const FMovementModifierBase* Other) const override;` using fields that `NetSerialize` also writes.

## 6. A custom instant movement effect

```cpp
// MyForceModeEffect.h
#pragma once

#include "InstantMovementEffect.h"
#include "MyForceModeEffect.generated.h"

USTRUCT(BlueprintType)
struct FMyForceModeEffect : public FInstantMovementEffect
{
    GENERATED_BODY()

    virtual ~FMyForceModeEffect() = default;

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "My Game|Effects")
    FName ModeToForce = NAME_None;

    virtual bool ApplyMovementEffect(FApplyMovementEffectParams& ApplyEffectParams, FMoverSyncState& OutputState) override;
    virtual FInstantMovementEffect* Clone() const override;
    virtual void NetSerialize(FArchive& Ar) override;
    virtual UScriptStruct* GetScriptStruct() const override;
    virtual FString ToSimpleString() const override;
};

template<>
struct TStructOpsTypeTraits<FMyForceModeEffect> : public TStructOpsTypeTraitsBase2<FMyForceModeEffect>
{
    enum
    {
        WithCopy = true
    };
};
```

```cpp
// MyForceModeEffect.cpp
#include "MyForceModeEffect.h"

#include "MoverSimulationTypes.h"

bool FMyForceModeEffect::ApplyMovementEffect(FApplyMovementEffectParams& ApplyEffectParams, FMoverSyncState& OutputState)
{
    if (ModeToForce.IsNone())
    {
        return false;
    }

    OutputState.MovementMode = ModeToForce;
    return true;
}

FInstantMovementEffect* FMyForceModeEffect::Clone() const
{
    return new FMyForceModeEffect(*this);
}

void FMyForceModeEffect::NetSerialize(FArchive& Ar)
{
    Super::NetSerialize(Ar);

    Ar << ModeToForce;
}

UScriptStruct* FMyForceModeEffect::GetScriptStruct() const
{
    return FMyForceModeEffect::StaticStruct();
}

FString FMyForceModeEffect::ToSimpleString() const
{
    return FString::Printf(TEXT("MyForceMode"));
}
```

```cpp
static void MyForceClimbingNow(UMoverComponent& MoverComp)
{
    TSharedPtr<FMyForceModeEffect> ForceMode = MakeShared<FMyForceModeEffect>();
    ForceMode->ModeToForce = MyModeNames::Climbing;
    MoverComp.QueueInstantMovementEffect(ForceMode);
}
```

## 7. A custom transition

```cpp
// MyLedgeGrabTransition.h
#pragma once

#include "MovementModeTransition.h"
#include "MyLedgeGrabTransition.generated.h"

UCLASS(Blueprintable, BlueprintType)
class MYGAME_API UMyLedgeGrabTransition : public UBaseMovementModeTransition
{
    GENERATED_BODY()

public:
    virtual FTransitionEvalResult Evaluate_Implementation(const FSimulationTickParams& Params) const override;
    virtual void Trigger_Implementation(const FSimulationTickParams& Params) override;

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "My Game|Climbing", meta = (ClampMin = "0", ForceUnits = "cm/s"))
    float MaxUpwardSpeedToGrab = 100.f;
};
```

```cpp
// MyLedgeGrabTransition.cpp
#include "MyLedgeGrabTransition.h"

#include "MoverComponent.h"
#include "MoverDataModelTypes.h"
#include "MoverSimulationTypes.h"
#include "MyClimbingMode.h"

FTransitionEvalResult UMyLedgeGrabTransition::Evaluate_Implementation(const FSimulationTickParams& Params) const
{
    const FMoverDefaultSyncState* SyncState = Params.StartState.SyncState.SyncStateCollection.FindDataByType<FMoverDefaultSyncState>();
    if (!SyncState)
    {
        return FTransitionEvalResult::NoTransition;
    }

    const UMoverComponent* MoverComp = GetMoverComponent();
    const FVector UpDir = MoverComp->GetUpDirection();
    const double UpwardSpeed = FVector::DotProduct(SyncState->GetVelocity_WorldSpace(), UpDir);

    if (UpwardSpeed > 0.0 && UpwardSpeed <= MaxUpwardSpeedToGrab)
    {
        return FTransitionEvalResult(MyModeNames::Climbing);
    }

    return FTransitionEvalResult::NoTransition;
}

void UMyLedgeGrabTransition::Trigger_Implementation(const FSimulationTickParams& Params)
{
    // Side effects only. The state machine performs the mode change itself.
    UE_LOG(LogMyGame, Verbose, TEXT("Ledge grab at sim frame %d"), Params.TimeStep.ServerFrame);
}
```

Add the instance to a mode's `Transitions` array (mode-scoped) or `UMoverComponent::Transitions` (global). Mode-owned transitions are evaluated first, in array order, stopping at the first hit.

## 8. Subscribing to Mover events

Delegates on `UMoverComponent` are dynamic multicast, so handlers need `UFUNCTION()`.

```cpp
// MyMoverListenerComponent.h
#pragma once

#include "Components/ActorComponent.h"
#include "MoverSimulationTypes.h"
#include "MyMoverListenerComponent.generated.h"

class UMoverComponent;

UCLASS(ClassGroup = (Custom), meta = (BlueprintSpawnableComponent))
class MYGAME_API UMyMoverListenerComponent : public UActorComponent
{
    GENERATED_BODY()

public:
    virtual void BeginPlay() override;
    virtual void EndPlay(const EEndPlayReason::Type EndPlayReason) override;

protected:
    UFUNCTION()
    void HandleMovementModeChanged(const FName& PreviousMovementModeName, const FName& NewMovementModeName);

    UFUNCTION()
    void HandlePostSimulationRollback(const FMoverTimeStep& CurrentTimeStep, const FMoverTimeStep& ExpungedTimeStep);

    UFUNCTION()
    void HandlePostFinalize(const FMoverSyncState& SyncState, const FMoverAuxStateContext& AuxState);

private:
    UPROPERTY(Transient)
    TObjectPtr<UMoverComponent> MoverComponent;
};
```

```cpp
// MyMoverListenerComponent.cpp
#include "MyMoverListenerComponent.h"

#include "MoverComponent.h"

void UMyMoverListenerComponent::BeginPlay()
{
    Super::BeginPlay();

    MoverComponent = GetOwner()->FindComponentByClass<UMoverComponent>();
    if (!MoverComponent)
    {
        return;
    }

    MoverComponent->OnMovementModeChanged.AddDynamic(this, &UMyMoverListenerComponent::HandleMovementModeChanged);
    MoverComponent->OnPostSimulationRollback.AddDynamic(this, &UMyMoverListenerComponent::HandlePostSimulationRollback);
    MoverComponent->OnPostFinalize.AddDynamic(this, &UMyMoverListenerComponent::HandlePostFinalize);
}

void UMyMoverListenerComponent::EndPlay(const EEndPlayReason::Type EndPlayReason)
{
    if (MoverComponent)
    {
        MoverComponent->OnMovementModeChanged.RemoveDynamic(this, &UMyMoverListenerComponent::HandleMovementModeChanged);
        MoverComponent->OnPostSimulationRollback.RemoveDynamic(this, &UMyMoverListenerComponent::HandlePostSimulationRollback);
        MoverComponent->OnPostFinalize.RemoveDynamic(this, &UMyMoverListenerComponent::HandlePostFinalize);
    }

    Super::EndPlay(EndPlayReason);
}

void UMyMoverListenerComponent::HandleMovementModeChanged(const FName& PreviousMovementModeName, const FName& NewMovementModeName)
{
    // Safe place for cosmetic reactions: this fires on the game thread after the simulation.
    UE_LOG(LogMyGame, Verbose, TEXT("Mover mode %s -> %s"), *PreviousMovementModeName.ToString(), *NewMovementModeName.ToString());
}

void UMyMoverListenerComponent::HandlePostSimulationRollback(const FMoverTimeStep& CurrentTimeStep, const FMoverTimeStep& ExpungedTimeStep)
{
    UE_LOG(LogMyGame, Verbose, TEXT("Rolled back to frame %d from %d"), CurrentTimeStep.ServerFrame, ExpungedTimeStep.ServerFrame);
}

void UMyMoverListenerComponent::HandlePostFinalize(const FMoverSyncState& SyncState, const FMoverAuxStateContext& AuxState)
{
    // State is final for this frame and cannot be changed here.
}
```

`OnPostMovement` is not `BlueprintAssignable` (it has mutable parameters) but can still be bound from C++ when you need to amend the outgoing state. To adjust the mixed `FProposedMove` before it executes, bind `FMover_ProcessGeneratedMovement` via `BindProcessGeneratedMovement` and clear it with `UnbindProcessGeneratedMovement`.

## 9. Reading state from gameplay code

```cpp
void AMyMoverPawn::LogMovementSnapshot() const
{
    const UCharacterMoverComponent* Mover = GetMoverComponent();

    const FName CurrentMode = Mover->GetMovementModeName();
    const FName NextMode    = Mover->GetNextMovementModeName();
    const FVector Velocity  = Mover->GetVelocity();
    const FMoverTimeStep& TimeStep = Mover->GetLastTimeStep();

    FHitResult FloorHit;
    const bool bHasFloor = Mover->TryGetFloorCheckHitResult(FloorHit);

    UE_LOG(LogMyGame, Verbose, TEXT("[frame %d] %s -> %s, speed %.1f, grounded %d, floor %d"),
        TimeStep.ServerFrame, *CurrentMode.ToString(), *NextMode.ToString(),
        Velocity.Size(), Mover->IsOnGround() ? 1 : 0, bHasFloor ? 1 : 0);
}
```

For motion matching, sample ahead instead of extrapolating by hand:

```cpp
#include "MoveLibrary/MovementUtils.h"   // defines FTrajectorySampleInfo; MoverComponent.h does not

static TArray<FTrajectorySampleInfo> MyPredictTrajectory(UMoverComponent& MoverComp)
{
    FMoverPredictTrajectoryParams PredictionParams;
    PredictionParams.NumPredictionSamples = 10;
    PredictionParams.SecondsPerSample = 0.1f;
    PredictionParams.bUseVisualComponentRoot = true;

    return MoverComp.GetPredictedTrajectory(PredictionParams);
}
```
