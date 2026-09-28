# Replication Patterns

Worked examples for UE 5.8. Each pattern gives the header, the implementation and the client callback. Types are named `AMy*` / `UMy*` / `FMy*` and the module API macro is `MYGAME_API`.

---

## Pattern 1: Health with an OnRep Callback

```cpp
// MyCharacter.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/Character.h"
#include "MyCharacter.generated.h"

DECLARE_DYNAMIC_MULTICAST_DELEGATE_TwoParams(FMyHealthChanged, float, OldHealth, float, NewHealth);

UCLASS()
class MYGAME_API AMyCharacter : public ACharacter
{
    GENERATED_BODY()

public:
    virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override;

    /** Server-side entry point for damage. */
    void ApplyDamageOnServer(float Damage);

    UPROPERTY(BlueprintAssignable)
    FMyHealthChanged OnHealthChanged;

protected:
    UPROPERTY(ReplicatedUsing = OnRep_Health)
    float Health = 100.f;

    UPROPERTY(Replicated)
    float MaxHealth = 100.f;

    UFUNCTION()
    void OnRep_Health(float PreviousHealth);

    void PlayHurtReaction();
    void HandleDeathOnServer();
};
```

```cpp
// MyCharacter.cpp
#include "MyCharacter.h"
#include "Net/UnrealNetwork.h"

void AMyCharacter::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);
    DOREPLIFETIME(AMyCharacter, Health);
    DOREPLIFETIME(AMyCharacter, MaxHealth);
}

void AMyCharacter::ApplyDamageOnServer(float Damage)
{
    if (!HasAuthority())
    {
        return;
    }

    Health = FMath::Clamp(Health - Damage, 0.f, MaxHealth);
    if (Health <= 0.f)
    {
        HandleDeathOnServer();
    }
}

void AMyCharacter::OnRep_Health(float PreviousHealth)
{
    // Clients only. Idempotent: relevancy changes re-fire this.
    OnHealthChanged.Broadcast(PreviousHealth, Health);
    if (Health < PreviousHealth)
    {
        PlayHurtReaction();
    }
}
```

The server never runs `OnRep_Health`. If the host of a listen server also needs the reaction, call the same helper from `ApplyDamageOnServer`.

---

## Pattern 2: Public State Plus Owner-Only Private State

`APlayerState` replicates to every client, so private data needs `COND_OwnerOnly`.

```cpp
// MyPlayerState.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/PlayerState.h"
#include "MyPlayerState.generated.h"

UCLASS()
class MYGAME_API AMyPlayerState : public APlayerState
{
    GENERATED_BODY()

public:
    virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override;

    int32 GetTeamId() const { return TeamId; }

protected:
    UPROPERTY(ReplicatedUsing = OnRep_TeamId)
    int32 TeamId = 0;

    UPROPERTY(Replicated)
    int32 Kills = 0;

    /** Only the owning client receives this. */
    UPROPERTY(ReplicatedUsing = OnRep_Currency)
    int32 Currency = 0;

    UFUNCTION()
    void OnRep_TeamId();

    UFUNCTION()
    void OnRep_Currency();

    void RefreshTeamVisuals();
    void RefreshShopUI();
};
```

```cpp
// MyPlayerState.cpp
#include "MyPlayerState.h"
#include "Net/UnrealNetwork.h"

void AMyPlayerState::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    DOREPLIFETIME_CONDITION(AMyPlayerState, TeamId, COND_InitialOrOwner);
    DOREPLIFETIME(AMyPlayerState, Kills);
    DOREPLIFETIME_CONDITION(AMyPlayerState, Currency, COND_OwnerOnly);
}

void AMyPlayerState::OnRep_TeamId()
{
    RefreshTeamVisuals();
}

void AMyPlayerState::OnRep_Currency()
{
    RefreshShopUI();
}
```

---

## Pattern 3: Inventory with FFastArraySerializer

Only changed entries go over the wire instead of the whole array.

```cpp
// MyInventoryTypes.h
#pragma once

#include "CoreMinimal.h"
#include "Net/Serialization/FastArraySerializer.h"
#include "MyInventoryTypes.generated.h"

USTRUCT(BlueprintType)
struct MYGAME_API FMyInventoryItem : public FFastArraySerializerItem
{
    GENERATED_BODY()

    UPROPERTY()
    int32 ItemId = 0;

    UPROPERTY()
    int32 Quantity = 0;

    void PreReplicatedRemove(const struct FMyInventoryList& InArraySerializer);
    void PostReplicatedAdd(const struct FMyInventoryList& InArraySerializer);
    void PostReplicatedChange(const struct FMyInventoryList& InArraySerializer);
};

USTRUCT(BlueprintType)
struct MYGAME_API FMyInventoryList : public FFastArraySerializer
{
    GENERATED_BODY()

    UPROPERTY()
    TArray<FMyInventoryItem> Items;

    bool NetDeltaSerialize(FNetDeltaSerializeInfo& DeltaParms)
    {
        return FFastArraySerializer::FastArrayDeltaSerialize<FMyInventoryItem, FMyInventoryList>(Items, DeltaParms, *this);
    }

    /** Optional array-level hook, called once after each received update. */
    void PostReplicatedReceive(const FFastArraySerializer::FPostReplicatedReceiveParameters& Parameters);
};

template<>
struct TStructOpsTypeTraits<FMyInventoryList> : public TStructOpsTypeTraitsBase2<FMyInventoryList>
{
    enum { WithNetDeltaSerializer = true };
};
```

```cpp
// MyInventoryComponent.h
#pragma once

#include "CoreMinimal.h"
#include "Components/ActorComponent.h"
#include "MyInventoryTypes.h"
#include "MyInventoryComponent.generated.h"

UCLASS(ClassGroup = (Custom), meta = (BlueprintSpawnableComponent))
class MYGAME_API UMyInventoryComponent : public UActorComponent
{
    GENERATED_BODY()

public:
    UMyInventoryComponent();

    virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override;

    void AddItemOnServer(int32 ItemId, int32 Quantity);
    void RemoveItemOnServer(int32 ItemId);

protected:
    UPROPERTY(Replicated)
    FMyInventoryList Inventory;
};
```

```cpp
// MyInventoryComponent.cpp
#include "MyInventoryComponent.h"
#include "Net/UnrealNetwork.h"

UMyInventoryComponent::UMyInventoryComponent()
{
    SetIsReplicatedByDefault(true);
}

void UMyInventoryComponent::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);
    DOREPLIFETIME_CONDITION(UMyInventoryComponent, Inventory, COND_OwnerOnly);
}

void UMyInventoryComponent::AddItemOnServer(int32 ItemId, int32 Quantity)
{
    if (!GetOwner()->HasAuthority())
    {
        return;
    }

    for (FMyInventoryItem& Item : Inventory.Items)
    {
        if (Item.ItemId == ItemId)
        {
            Item.Quantity += Quantity;
            Inventory.MarkItemDirty(Item);
            return;
        }
    }

    FMyInventoryItem& NewItem = Inventory.Items.AddDefaulted_GetRef();
    NewItem.ItemId = ItemId;
    NewItem.Quantity = Quantity;
    Inventory.MarkItemDirty(NewItem);
}

void UMyInventoryComponent::RemoveItemOnServer(int32 ItemId)
{
    if (!GetOwner()->HasAuthority())
    {
        return;
    }

    const int32 Removed = Inventory.Items.RemoveAll(
        [ItemId](const FMyInventoryItem& Item) { return Item.ItemId == ItemId; });

    if (Removed > 0)
    {
        // Mandatory after a removal. MarkItemDirty alone does not cover it.
        Inventory.MarkArrayDirty();
    }
}
```

```cpp
// MyInventoryTypes.cpp
#include "MyInventoryTypes.h"

void FMyInventoryItem::PostReplicatedAdd(const FMyInventoryList& InArraySerializer)
{
    // New item arrived on the client.
}

void FMyInventoryItem::PostReplicatedChange(const FMyInventoryList& InArraySerializer)
{
    // Quantity changed on the client.
}

void FMyInventoryItem::PreReplicatedRemove(const FMyInventoryList& InArraySerializer)
{
    // Item is about to disappear on the client.
}

void FMyInventoryList::PostReplicatedReceive(const FFastArraySerializer::FPostReplicatedReceiveParameters& Parameters)
{
    // Runs once per received update; Parameters.OldArraySize is the size before it.
}
```

---

## Pattern 4: Push Model on a Hot Property

Use when a property is compared every replication tick but changes rarely.

```cpp
// MyTurret.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/Actor.h"
#include "MyTurret.generated.h"

UCLASS()
class MYGAME_API AMyTurret : public AActor
{
    GENERATED_BODY()

public:
    AMyTurret();

    virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override;

    void SetTargetIdOnServer(int32 NewTargetId);

protected:
    UPROPERTY(Replicated)
    int32 TargetId = INDEX_NONE;
};
```

```cpp
// MyTurret.cpp
#include "MyTurret.h"
#include "Net/UnrealNetwork.h"
#include "Net/Core/PushModel/PushModel.h"

AMyTurret::AMyTurret()
{
    bReplicates = true;
    SetNetUpdateFrequency(5.f);
    SetMinNetUpdateFrequency(1.f);
}

void AMyTurret::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    FDoRepLifetimeParams Params;
    Params.bIsPushBased = true;
    DOREPLIFETIME_WITH_PARAMS_FAST(AMyTurret, TargetId, Params);
}

void AMyTurret::SetTargetIdOnServer(int32 NewTargetId)
{
    if (!HasAuthority() || TargetId == NewTargetId)
    {
        return;
    }

    TargetId = NewTargetId;
    MARK_PROPERTY_DIRTY_FROM_NAME(AMyTurret, TargetId, this);
}
```

Every write path to a push-based property must reach a `MARK_PROPERTY_DIRTY_FROM_NAME`. Route writes through one setter so there is a single place to get it right.

---

## Pattern 5: Replicated UObject Subobject

`bReplicateUsingRegisteredSubObjectList` is **false** by default in 5.8. Without the constructor opt-in this pattern replicates nothing.

```cpp
// MyQuestData.h
#pragma once

#include "CoreMinimal.h"
#include "UObject/Object.h"
#include "MyQuestData.generated.h"

UCLASS()
class MYGAME_API UMyQuestData : public UObject
{
    GENERATED_BODY()

public:
    virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override;
    virtual bool IsSupportedForNetworking() const override { return true; }

    UPROPERTY(Replicated)
    int32 QuestId = 0;

    UPROPERTY(Replicated)
    int32 Progress = 0;
};
```

```cpp
// MyQuestData.cpp
#include "MyQuestData.h"
#include "Net/UnrealNetwork.h"

void UMyQuestData::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);
    DOREPLIFETIME(UMyQuestData, QuestId);
    DOREPLIFETIME(UMyQuestData, Progress);
}
```

```cpp
// MyPlayerController.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/PlayerController.h"
#include "MyPlayerController.generated.h"

class UMyQuestData;

UCLASS()
class MYGAME_API AMyPlayerController : public APlayerController
{
    GENERATED_BODY()

public:
    AMyPlayerController();

    virtual void BeginPlay() override;
    virtual void EndPlay(const EEndPlayReason::Type EndPlayReason) override;

protected:
    UPROPERTY(Replicated)
    TObjectPtr<UMyQuestData> ActiveQuest;

public:
    virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override;
};
```

```cpp
// MyPlayerController.cpp
#include "MyPlayerController.h"
#include "MyQuestData.h"
#include "Net/UnrealNetwork.h"

AMyPlayerController::AMyPlayerController()
{
    bReplicateUsingRegisteredSubObjectList = true; // required: the engine default is false
}

void AMyPlayerController::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);
    DOREPLIFETIME_CONDITION(AMyPlayerController, ActiveQuest, COND_OwnerOnly);
}

void AMyPlayerController::BeginPlay()
{
    Super::BeginPlay();

    if (HasAuthority())
    {
        ActiveQuest = NewObject<UMyQuestData>(this);
        ActiveQuest->QuestId = 42;
        AddReplicatedSubObject(ActiveQuest, COND_OwnerOnly);
    }
}

void AMyPlayerController::EndPlay(const EEndPlayReason::Type EndPlayReason)
{
    if (HasAuthority() && ActiveQuest)
    {
        DestroyReplicatedSubObjectOnRemotePeers(ActiveQuest);
    }
    Super::EndPlay(EndPlayReason);
}
```

`RemoveReplicatedSubObject` stops updates but leaves the client copy alive. `DestroyReplicatedSubObjectOnRemotePeers` deletes it remotely; `TearOffReplicatedSubObjectOnRemotePeers` leaves an orphaned local copy that the client owns from then on.

### Legacy path

When `bReplicateUsingRegisteredSubObjectList` is false the engine calls the virtual instead — it is not deprecated in 5.8:

```cpp
bool AMyLegacyActor::ReplicateSubobjects(UActorChannel* Channel, FOutBunch* Bunch, FReplicationFlags* RepFlags)
{
    bool bWroteSomething = Super::ReplicateSubobjects(Channel, Bunch, RepFlags);
    if (ActiveQuest)
    {
        bWroteSomething |= Channel->ReplicateSubobject(ActiveQuest, *Bunch, *RepFlags);
    }
    return bWroteSomething;
}
```

Declared in the header as:

```cpp
virtual bool ReplicateSubobjects(class UActorChannel* Channel, class FOutBunch* Bunch, FReplicationFlags* RepFlags) override;
```

---

## Pattern 6: Replicated Movement Without CMC

```cpp
// MyVehicle.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/Pawn.h"
#include "MyVehicle.generated.h"

UCLASS()
class MYGAME_API AMyVehicle : public APawn
{
    GENERATED_BODY()

public:
    AMyVehicle();

    virtual void OnRep_ReplicatedMovement() override;

protected:
    void SyncWheelPositions();
};
```

```cpp
// MyVehicle.cpp
#include "MyVehicle.h"

AMyVehicle::AMyVehicle()
{
    bReplicates = true;
    SetReplicateMovement(true);
    SetNetUpdateFrequency(50.f);
    NetPriority = 2.5f;
    SetPhysicsReplicationMode(EPhysicsReplicationMode::PredictiveInterpolation);
}

void AMyVehicle::OnRep_ReplicatedMovement()
{
    Super::OnRep_ReplicatedMovement(); // applies FRepMovement to the root component
    SyncWheelPositions();
}
```

`FRepMovement` lives in `Engine/Classes/Engine/ReplicatedState.h`. When the root component simulates physics the engine sets `bRepPhysics` on it and includes linear and angular velocity. Characters using `UCharacterMovementComponent` must keep movement replication on (the `APawn` default, `Pawn.cpp:89`): simulated proxies are positioned from `ReplicatedMovement` via `ACharacter::PostNetReceiveLocationAndRotation` (`Character.cpp:2004`) and CMC smoothing, while the autonomous proxy uses CMC's own move RPCs.

---

## Pattern 7: Custom Relevancy

```cpp
// MyTeamActor.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/Actor.h"
#include "MyTeamActor.generated.h"

UCLASS()
class MYGAME_API AMyTeamActor : public AActor
{
    GENERATED_BODY()

public:
    virtual bool IsNetRelevantFor(const AActor* RealViewer, const AActor* ViewTarget, const FVector& SrcLocation) const override;

protected:
    UPROPERTY(Replicated)
    int32 TeamId = 0;
};
```

```cpp
// MyTeamActor.cpp
#include "MyTeamActor.h"
#include "MyPlayerState.h"
#include "GameFramework/PlayerController.h"
#include "Net/UnrealNetwork.h"

// A UPROPERTY(Replicated) member makes UHT declare this override; it must be defined or linking fails.
void AMyTeamActor::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);
    DOREPLIFETIME(AMyTeamActor, TeamId);
}

bool AMyTeamActor::IsNetRelevantFor(const AActor* RealViewer, const AActor* ViewTarget, const FVector& SrcLocation) const
{
    if (RealViewer == GetOwner())
    {
        return true;
    }

    if (const APlayerController* PC = Cast<APlayerController>(RealViewer))
    {
        if (const AMyPlayerState* PS = PC->GetPlayerState<AMyPlayerState>())
        {
            if (PS->GetTeamId() == TeamId)
            {
                return Super::IsNetRelevantFor(RealViewer, ViewTarget, SrcLocation);
            }
        }
    }

    return false;
}
```

`IsNetRelevantFor` runs per connection per replication tick. Keep it branch-cheap; for large player counts move the policy into a replication graph node instead (see the Replication Graph section of the main skill file).

---

## Pattern 8: Dormancy for Rarely-Updated Actors

```cpp
// MyPickup.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/Actor.h"
#include "MyPickup.generated.h"

UCLASS()
class MYGAME_API AMyPickup : public AActor
{
    GENERATED_BODY()

public:
    AMyPickup();

    void CollectOnServer();

protected:
    UPROPERTY(ReplicatedUsing = OnRep_PickedUp)
    bool bPickedUp = false;

    UPROPERTY(EditDefaultsOnly, Category = "Pickup")
    float RespawnTime = 30.f;

    FTimerHandle RespawnTimer;

    UFUNCTION()
    void OnRep_PickedUp();

    void RespawnOnServer();

public:
    virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override;
};
```

```cpp
// MyPickup.cpp
#include "MyPickup.h"
#include "Net/UnrealNetwork.h"
#include "TimerManager.h"
#include "Engine/World.h"

AMyPickup::AMyPickup()
{
    bReplicates = true;
    NetDormancy = DORM_Initial;   // map-placed pickups start dormant
    SetNetUpdateFrequency(1.f);
}

void AMyPickup::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);
    DOREPLIFETIME(AMyPickup, bPickedUp);
}

void AMyPickup::CollectOnServer()
{
    if (!HasAuthority() || bPickedUp)
    {
        return;
    }

    bPickedUp = true;
    SetActorHiddenInGame(true);
    SetActorEnableCollision(false);
    FlushNetDormancy();   // one update, then dormant again

    GetWorldTimerManager().SetTimer(RespawnTimer, this, &AMyPickup::RespawnOnServer, RespawnTime, false);
}

void AMyPickup::RespawnOnServer()
{
    bPickedUp = false;
    SetActorHiddenInGame(false);
    SetActorEnableCollision(true);
    FlushNetDormancy();
}

void AMyPickup::OnRep_PickedUp()
{
    SetActorHiddenInGame(bPickedUp);
    SetActorEnableCollision(!bPickedUp);
}
```

`FlushNetDormancy()` sends one update and returns the actor to its dormancy state. The flush is diffed against the state kept at dormancy time, but a pickup that leaves and re-enters relevancy is re-sent in full, so `OnRep_PickedUp` must tolerate being called again.

---

## Bandwidth Reference

`FVector_NetQuantize` variants are declared in `Engine/Classes/Engine/NetSerialization.h`:

| Type | Precision | Bits per component |
|---|---|---|
| `FVector_NetQuantize` | 0 decimal places (1 cm) | up to 20 |
| `FVector_NetQuantize10` | 1 decimal place | up to 24 |
| `FVector_NetQuantize100` | 2 decimal places | up to 30 |
| `FVector_NetQuantizeNormal` | unit vector, range -1..+1 | 16 |

Cheapest wins, in order: do not replicate it at all (derive it client-side) → replicate a quantized or packed form → replicate the raw type. Pack booleans into a bitmask before replicating eight separate `bool` properties, and prefer `uint8`/`int16` ids over `FString` names.
