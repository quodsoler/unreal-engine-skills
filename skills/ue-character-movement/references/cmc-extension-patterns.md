# CMC Extension Patterns

Working templates for extending `UCharacterMovementComponent` in UE 5.8: a custom movement mode, a predicted saved move, custom network move data, and movement-base handling. Every signature below is copied from `GameFramework/CharacterMovementComponent.h`, `GameFramework/Character.h` and `GameFramework/CharacterMovementReplication.h`.

---

## 1. Custom CMC subclass header

`GameFramework/CharacterMovementComponent.h` already includes `RootMotionSource.h`, `CharacterMovementReplication.h`, `CharacterMovementComponentAsync.h` and `Interfaces/MovementBaseInterface.h`, so one include covers the whole surface.

```cpp
// MyCharacterMovement.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/CharacterMovementComponent.h"
#include "MyCharacterMovement.generated.h"

UENUM(BlueprintType)
enum class EMyMovementMode : uint8
{
    None      = 0,
    Climbing  = 1,
    Dash      = 2
};

UCLASS()
class MYGAME_API UMyCharacterMovement : public UCharacterMovementComponent
{
    GENERATED_BODY()

public:
    UMyCharacterMovement();

    /** Input intent, replicated through a compressed flag. */
    UPROPERTY(Transient)
    uint8 bWantsToClimb : 1;

    /** Surface normal of the wall being climbed. Too wide for a flag; goes in custom move data. */
    UPROPERTY(Transient)
    FVector ClimbSurfaceNormal;

    UPROPERTY(EditDefaultsOnly, Category = "Character Movement: Climbing")
    float MaxClimbSpeed = 250.f;

    virtual void PhysCustom(float deltaTime, int32 Iterations) override;
    virtual void OnMovementModeChanged(EMovementMode PreviousMovementMode, uint8 PreviousCustomMode) override;
    virtual float GetMaxSpeed() const override;
    virtual void UpdateFromCompressedFlags(uint8 Flags) override;
    virtual class FNetworkPredictionData_Client* GetPredictionData_Client() const override;
    virtual void MoveAutonomous(float ClientTimeStamp, float DeltaTime, uint8 CompressedFlags, const FVector& NewAccel) override;
    virtual void OnClientCorrectionReceived(class FNetworkPredictionData_Client_Character& ClientData, float TimeStamp,
                                            FVector NewLocation, FVector NewVelocity,
                                            FMovementBaseInterfaceData* NewMovementBaseInterfaceData,
                                            FName NewBaseBoneName, bool bHasBase, bool bBaseRelativePosition,
                                            uint8 ServerMovementMode, FVector ServerGravityDirection) override;

    void StartClimbing(const FHitResult& WallHit);
    void StopClimbing();
    void ApplyPlatformBoost();
    void AttachToSurface(const FHitResult& Hit);

protected:
    void PhysClimb(float deltaTime, int32 Iterations);
    bool CanClimb() const;
};
```

---

## 2. PhysCustom and mode transitions

Call `Super::PhysCustom` **first**: the base implementation forwards to `ACharacter::K2_UpdateCustomMovement(float DeltaTime)`.

```cpp
// MyCharacterMovement.cpp
#include "MyCharacterMovement.h"

#include "GameFramework/Character.h"
#include "Components/CapsuleComponent.h"

void UMyCharacterMovement::PhysCustom(float deltaTime, int32 Iterations)
{
    Super::PhysCustom(deltaTime, Iterations);

    if (CustomMovementMode == static_cast<uint8>(EMyMovementMode::Climbing))
    {
        PhysClimb(deltaTime, Iterations);
    }
}

void UMyCharacterMovement::PhysClimb(float deltaTime, int32 Iterations)
{
    if (deltaTime < MIN_TICK_TIME)
    {
        return;
    }

    if (!bWantsToClimb || !CanClimb())
    {
        StopClimbing();
        return;
    }

    // Climb plane: up and sideways along the wall.
    const FVector WallUp = FVector::VectorPlaneProject(-GetGravityDirection(), ClimbSurfaceNormal).GetSafeNormal();
    const FVector WallRight = FVector::CrossProduct(ClimbSurfaceNormal, WallUp);

    const FVector Input = GetLastInputVector();
    FVector ClimbDirection = WallUp * FVector::DotProduct(Input, WallUp)
                           + WallRight * FVector::DotProduct(Input, WallRight);
    ClimbDirection = ClimbDirection.GetClampedToMaxSize(1.f);

    Velocity = ClimbDirection * MaxClimbSpeed;
    Velocity += -ClimbSurfaceNormal * 50.f;   // stay pressed against the wall

    FHitResult Hit(1.f);
    SafeMoveUpdatedComponent(Velocity * deltaTime, UpdatedComponent->GetComponentQuat(), true, Hit);

    if (Hit.bBlockingHit)
    {
        ClimbSurfaceNormal = Hit.ImpactNormal;
        SlideAlongSurface(Velocity * deltaTime, 1.f - Hit.Time, Hit.Normal, Hit, true);
    }
    else if (!CanClimb())
    {
        StopClimbing();
    }
}

void UMyCharacterMovement::StartClimbing(const FHitResult& WallHit)
{
    ClimbSurfaceNormal = WallHit.ImpactNormal;
    bWantsToClimb = 1;
    SetMovementMode(MOVE_Custom, static_cast<uint8>(EMyMovementMode::Climbing));
}

void UMyCharacterMovement::StopClimbing()
{
    bWantsToClimb = 0;
    ClimbSurfaceNormal = FVector::ZeroVector;
    SetMovementMode(MOVE_Falling);
}

void UMyCharacterMovement::OnMovementModeChanged(EMovementMode PreviousMovementMode, uint8 PreviousCustomMode)
{
    Super::OnMovementModeChanged(PreviousMovementMode, PreviousCustomMode);

    const bool bWasClimbing = PreviousMovementMode == MOVE_Custom
        && PreviousCustomMode == static_cast<uint8>(EMyMovementMode::Climbing);

    if (bWasClimbing)
    {
        bOrientRotationToMovement = true;
    }
    else if (MovementMode == MOVE_Custom
        && CustomMovementMode == static_cast<uint8>(EMyMovementMode::Climbing))
    {
        bOrientRotationToMovement = false;
    }
}

float UMyCharacterMovement::GetMaxSpeed() const
{
    if (MovementMode == MOVE_Custom
        && CustomMovementMode == static_cast<uint8>(EMyMovementMode::Climbing))
    {
        return MaxClimbSpeed;
    }

    return Super::GetMaxSpeed();
}

bool UMyCharacterMovement::CanClimb() const
{
    return !ClimbSurfaceNormal.IsNearlyZero();
}
```

---

## 3. FSavedMove_Character subclass

```cpp
// MyCharacterMovement.h, after UMyCharacterMovement
class FMySavedMove : public FSavedMove_Character
{
public:
    using Super = FSavedMove_Character;

    uint8 bSavedWantsToClimb : 1;
    FVector SavedClimbSurfaceNormal;

    virtual void Clear() override;
    virtual void SetMoveFor(ACharacter* C, float InDeltaTime, FVector const& NewAccel,
                            class FNetworkPredictionData_Client_Character& ClientData) override;
    virtual void PrepMoveFor(ACharacter* C) override;
    virtual uint8 GetCompressedFlags() const override;
    virtual bool CanCombineWith(const FSavedMovePtr& NewMove, ACharacter* InCharacter, float MaxDelta) const override;
};
```

```cpp
// MyCharacterMovement.cpp
void FMySavedMove::Clear()
{
    Super::Clear();
    bSavedWantsToClimb = 0;
    SavedClimbSurfaceNormal = FVector::ZeroVector;
}

void FMySavedMove::SetMoveFor(ACharacter* C, float InDeltaTime, FVector const& NewAccel,
                              FNetworkPredictionData_Client_Character& ClientData)
{
    Super::SetMoveFor(C, InDeltaTime, NewAccel, ClientData);

    if (const UMyCharacterMovement* Movement = Cast<UMyCharacterMovement>(C->GetCharacterMovement()))
    {
        bSavedWantsToClimb = Movement->bWantsToClimb;
        SavedClimbSurfaceNormal = Movement->ClimbSurfaceNormal;
    }
}

void FMySavedMove::PrepMoveFor(ACharacter* C)
{
    Super::PrepMoveFor(C);

    if (UMyCharacterMovement* Movement = Cast<UMyCharacterMovement>(C->GetCharacterMovement()))
    {
        Movement->bWantsToClimb = bSavedWantsToClimb;
        Movement->ClimbSurfaceNormal = SavedClimbSurfaceNormal;
    }
}

uint8 FMySavedMove::GetCompressedFlags() const
{
    uint8 Result = Super::GetCompressedFlags();

    if (bSavedWantsToClimb)
    {
        Result |= FLAG_Custom_0;
    }

    return Result;
}

bool FMySavedMove::CanCombineWith(const FSavedMovePtr& NewMove, ACharacter* InCharacter, float MaxDelta) const
{
    const FMySavedMove* Other = static_cast<const FMySavedMove*>(NewMove.Get());

    if (bSavedWantsToClimb != Other->bSavedWantsToClimb)
    {
        return false;
    }

    if (bSavedWantsToClimb && !SavedClimbSurfaceNormal.Equals(Other->SavedClimbSurfaceNormal, 0.1f))
    {
        return false;
    }

    return Super::CanCombineWith(NewMove, InCharacter, MaxDelta);
}
```

The CMC side of the flag:

```cpp
void UMyCharacterMovement::UpdateFromCompressedFlags(uint8 Flags)
{
    Super::UpdateFromCompressedFlags(Flags);

    bWantsToClimb = (Flags & FSavedMove_Character::FLAG_Custom_0) != 0;
}
```

Other virtuals available on `FSavedMove_Character`, verbatim:

```cpp
virtual void SetInitialPosition(ACharacter* C);
virtual bool IsImportantMove(const FSavedMovePtr& LastAckedMove) const;
virtual void PostUpdate(ACharacter* C, EPostUpdateMode PostUpdateMode);
virtual void CombineWith(const FSavedMove_Character* OldMove, ACharacter* InCharacter,
                         APlayerController* PC, const FVector& OldStartLocation);
virtual bool IsMatchingStartControlRotation(const APlayerController* PC) const;
virtual void GetPackedAngles(uint32& YawAndPitchPack, uint8& RollPack) const;
virtual void AddStructReferencedObjects(FReferenceCollector& Collector) const;
```

---

## 4. Prediction data and the lazy-init accessor

```cpp
class FMyNetworkPredictionData : public FNetworkPredictionData_Client_Character
{
public:
    using Super = FNetworkPredictionData_Client_Character;

    FMyNetworkPredictionData(const UCharacterMovementComponent& ClientMovement)
        : Super(ClientMovement)
    {
    }

    virtual FSavedMovePtr AllocateNewMove() override
    {
        return FSavedMovePtr(new FMySavedMove());
    }
};
```

```cpp
FNetworkPredictionData_Client* UMyCharacterMovement::GetPredictionData_Client() const
{
    if (ClientPredictionData == nullptr)
    {
        UMyCharacterMovement* MutableThis = const_cast<UMyCharacterMovement*>(this);
        MutableThis->ClientPredictionData = new FMyNetworkPredictionData(*this);
    }

    return ClientPredictionData;
}
```

`const_cast` is required because the base accessor is `const` while the prediction data is mutable state; this is the engine's own pattern. `GetPredictionData_Client_Character()` returns the same object already typed.

---

## 5. Custom network move data

Use this when the four custom compressed-flag bits (`FLAG_Custom_0` through `FLAG_Custom_3`) are not enough. `ENetworkMoveType` is nested inside `FCharacterNetworkMoveData`.

```cpp
struct FMyNetworkMoveData : public FCharacterNetworkMoveData
{
    using Super = FCharacterNetworkMoveData;

    FVector SavedClimbSurfaceNormal = FVector::ZeroVector;

    virtual void ClientFillNetworkMoveData(const FSavedMove_Character& ClientMove, ENetworkMoveType MoveType) override
    {
        Super::ClientFillNetworkMoveData(ClientMove, MoveType);

        const FMySavedMove& MyMove = static_cast<const FMySavedMove&>(ClientMove);
        SavedClimbSurfaceNormal = MyMove.SavedClimbSurfaceNormal;
    }

    virtual bool Serialize(UCharacterMovementComponent& CharacterMovement, FArchive& Ar,
                           UPackageMap* PackageMap, ENetworkMoveType MoveType) override
    {
        Super::Serialize(CharacterMovement, Ar, PackageMap, MoveType);

        Ar << SavedClimbSurfaceNormal;

        return !Ar.IsError();
    }
};

struct FMyNetworkMoveDataContainer : public FCharacterNetworkMoveDataContainer
{
    FMyNetworkMoveDataContainer()
    {
        NewMoveData     = &MyMoveData[0];
        PendingMoveData = &MyMoveData[1];
        OldMoveData     = &MyMoveData[2];
    }

    FMyNetworkMoveData MyMoveData[3];
};
```

```cpp
UMyCharacterMovement::UMyCharacterMovement()
{
    static FMyNetworkMoveDataContainer MoveDataContainer;
    SetNetworkMoveDataContainer(MoveDataContainer);
}
```

`SetNetworkMoveDataContainer` stores a pointer, not a copy — the container must outlive every component that uses it, hence `static`. On the server, reach the unpacked data with `GetCurrentNetworkMoveData()` from inside `MoveAutonomous` or `UpdateFromCompressedFlags`:

```cpp
void UMyCharacterMovement::MoveAutonomous(float ClientTimeStamp, float DeltaTime, uint8 CompressedFlags, const FVector& NewAccel)
{
    if (const FMyNetworkMoveData* MoveData = static_cast<FMyNetworkMoveData*>(GetCurrentNetworkMoveData()))
    {
        ClimbSurfaceNormal = MoveData->SavedClimbSurfaceNormal;
    }

    Super::MoveAutonomous(ClientTimeStamp, DeltaTime, CompressedFlags, NewAccel);
}
```

The server-to-client response side mirrors this with `FCharacterMoveResponseDataContainer` and `SetMoveResponseDataContainer(...)`.

---

## 6. Movement base handling in a subclass

The base a character stands on is `FMovementBaseInterfaceData` (`Interfaces/MovementBaseInterface.h`), not a `UPrimitiveComponent*`.

```cpp
void UMyCharacterMovement::ApplyPlatformBoost()
{
    const FMovementBaseInterfaceData* BaseData = GetMovementBaseInterfaceData();
    if (!MovementBaseUtility::IsMovementBaseDataValid(BaseData))
    {
        return;
    }

    const FName BoneName = CharacterOwner->GetBasedMovement().BoneName;

    if (MovementBaseUtility::IsDynamicBase(BaseData))
    {
        const FVector BaseVelocity = MovementBaseUtility::GetMovementBaseVelocity(BaseData, BoneName);
        const FVector Tangential = MovementBaseUtility::GetMovementBaseTangentialVelocity(
            BaseData, BoneName, UpdatedComponent->GetComponentLocation());

        Velocity += BaseVelocity + Tangential;
    }

    if (const UPrimitiveComponent* BaseComponent = Cast<UPrimitiveComponent>(BaseData->GetMovementBaseObject()))
    {
        UE_LOG(LogMyGame, Verbose, TEXT("Standing on %s"), *BaseComponent->GetName());
    }
}
```

Assigning a base from a trace or a floor result:

```cpp
void UMyCharacterMovement::AttachToSurface(const FHitResult& Hit)
{
    FMovementBaseInterfaceData NewBase = MovementBaseUtility::GetMovementBaseDataFromHitResult(&Hit);
    SetBase(&NewBase, Hit.BoneName);
}

// Declared in section 1; every declared override needs a body or the module fails to link.
void UMyCharacterMovement::OnClientCorrectionReceived(FNetworkPredictionData_Client_Character& ClientData, float TimeStamp,
    FVector NewLocation, FVector NewVelocity, FMovementBaseInterfaceData* NewMovementBaseInterfaceData,
    FName NewBaseBoneName, bool bHasBase, bool bBaseRelativePosition, uint8 ServerMovementMode, FVector ServerGravityDirection)
{
    Super::OnClientCorrectionReceived(ClientData, TimeStamp, NewLocation, NewVelocity, NewMovementBaseInterfaceData,
        NewBaseBoneName, bHasBase, bBaseRelativePosition, ServerMovementMode, ServerGravityDirection);
}
```

`SetBaseFromFloor(const FFindFloorResult& FloorResult)` does the same from `CurrentFloor`. Space conversions relative to the base use `MovementBaseUtility::TransformLocationToLocal`, `TransformLocationToWorld`, `TransformDirectionToLocal`, `TransformDirectionToWorld` and `GetMovementBaseTransform`, all taking `const FMovementBaseInterfaceData*`.

To react to mounting and dismounting on the client, override `OnClientCorrectionReceived` — the overload taking `FMovementBaseInterfaceData*`, declared in section 1. Overriding the deprecated `UPrimitiveComponent*` overload compiles but is never called. `ACharacter::BaseChange()` and `ACharacter::OnRep_ReplicatedBasedMovement()` are the actor-level notifications.

---

## 7. Wiring the component onto a character

```cpp
// MyCharacter.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/Character.h"
#include "MyCharacter.generated.h"

class UMyCharacterMovement;  // header is included only in MyCharacter.cpp

UCLASS()
class MYGAME_API AMyCharacter : public ACharacter
{
    GENERATED_BODY()

public:
    AMyCharacter(const FObjectInitializer& ObjectInitializer);

    UFUNCTION(BlueprintCallable, Category = "Movement")
    void TryClimb();

    UMyCharacterMovement* GetMyCharacterMovement() const;
};
```

```cpp
// MyCharacter.cpp
#include "MyCharacter.h"
#include "MyCharacterMovement.h"

AMyCharacter::AMyCharacter(const FObjectInitializer& ObjectInitializer)
    : Super(ObjectInitializer.SetDefaultSubobjectClass<UMyCharacterMovement>(ACharacter::CharacterMovementComponentName))
{
}

UMyCharacterMovement* AMyCharacter::GetMyCharacterMovement() const
{
    return Cast<UMyCharacterMovement>(GetCharacterMovement());
}

void AMyCharacter::TryClimb()
{
    UMyCharacterMovement* Movement = GetMyCharacterMovement();
    if (Movement == nullptr)
    {
        return;
    }

    const FVector Start = GetActorLocation();
    const FVector End = Start + GetActorForwardVector() * 60.f;

    FHitResult Hit(1.f);
    FCollisionQueryParams Params(SCENE_QUERY_STAT(MyClimbTrace), false, this);

    if (GetWorld()->LineTraceSingleByChannel(Hit, Start, End, ECC_Visibility, Params))
    {
        Movement->StartClimbing(Hit);
    }
}
```

`SetDefaultSubobjectClass<UMyCharacterMovement>(ACharacter::CharacterMovementComponentName)` is the only supported way to swap the movement component class — the subobject is created by `ACharacter`'s constructor.

---

## 8. Checklist for a predicted custom mode

1. Add the state to the CMC subclass (`bWantsTo*`, plus any vectors).
2. Save and restore it in `FMySavedMove::SetMoveFor` / `PrepMoveFor`, and reset it in `Clear`.
3. Pack booleans into `GetCompressedFlags` and read them back in `UpdateFromCompressedFlags`.
4. Reject combining in `CanCombineWith` whenever the state differs.
5. Return your saved-move type from `AllocateNewMove`, and your prediction data from `GetPredictionData_Client`.
6. Anything wider than four bits goes into `FCharacterNetworkMoveData` plus a `static` container registered with `SetNetworkMoveDataContainer`.
7. Override `GetMaxSpeed` so combine heuristics and animation agree on the speed cap.
8. Test with `p.NetShowCorrections 1` and simulated latency; a mode that only desyncs under lag is a missing `PrepMoveFor`.
