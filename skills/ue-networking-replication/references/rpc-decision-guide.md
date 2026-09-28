# RPC Decision Guide

Choosing the RPC type (`Server`, `Client`, `NetMulticast`) and reliability (`Reliable`, `Unreliable`) in UE 5.8. Engine examples are quoted verbatim from `Engine/Source/Runtime/Engine/Classes/GameFramework/PlayerController.h` with line numbers.

---

## Quick Reference

| Specifier | Called on | Executes on | Typical use |
|---|---|---|---|
| `Server` | owning client | server (authority) | player intent: fire, purchase, activate |
| `Client` | server | owning client only | private feedback, UI, connection-scoped setup |
| `NetMulticast` | server | server + every client the actor is relevant to | shared one-shot cosmetics |

`Reliable` and `Unreliable` are only valid alongside one of the three above (`UObject/ObjectMacros.h`). `WithValidation` requires a `_Validate` body; UHT accepts it on any RPC (the engine even uses it on `Client` RPCs, `Controller.h:198`), but it only earns its keep on `Server` RPCs.

---

## Decision Flow

```
Is this a request from a player to change authoritative state?
  yes -> Server RPC
  no  -> continue

Does exactly one player's machine need to react?
  yes -> Client RPC
  no  -> continue

Must every machine react at the same moment, with no recovery for late joiners?
  yes -> NetMulticast RPC
  no  -> use a replicated property instead
```

---

## Server RPC

```cpp
// MyWeapon.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/Actor.h"
#include "Engine/NetSerialization.h"
#include "MyWeapon.generated.h"

UCLASS()
class MYGAME_API AMyWeapon : public AActor
{
    GENERATED_BODY()

public:
    UFUNCTION(Server, Reliable, WithValidation)
    void ServerFireWeapon(FVector_NetQuantize MuzzleLocation, FVector_NetQuantizeNormal Direction);

    UFUNCTION(NetMulticast, Unreliable)
    void MulticastSpawnTracer(FVector_NetQuantize Start, FVector_NetQuantize End);

protected:
    UPROPERTY(EditDefaultsOnly, Category = "Weapon")
    float WeaponDamage = 20.f;

    UPROPERTY(EditDefaultsOnly, Category = "Weapon")
    float MaxRange = 10000.f;
};
```

```cpp
// MyWeapon.cpp
#include "MyWeapon.h"
#include "MyCharacter.h"
#include "Engine/World.h"

void AMyWeapon::ServerFireWeapon_Implementation(FVector_NetQuantize MuzzleLocation, FVector_NetQuantizeNormal Direction)
{
    FHitResult Hit;
    FCollisionQueryParams Params;
    Params.AddIgnoredActor(this);
    Params.AddIgnoredActor(GetOwner());

    const FVector End = MuzzleLocation + Direction * MaxRange;
    if (GetWorld()->LineTraceSingleByChannel(Hit, MuzzleLocation, End, ECC_Pawn, Params))
    {
        if (AMyCharacter* Victim = Cast<AMyCharacter>(Hit.GetActor()))
        {
            Victim->ApplyDamageOnServer(WeaponDamage);
        }
    }

    MulticastSpawnTracer(MuzzleLocation, Hit.bBlockingHit ? FVector_NetQuantize(Hit.ImpactPoint) : FVector_NetQuantize(End));
}

bool AMyWeapon::ServerFireWeapon_Validate(FVector_NetQuantize MuzzleLocation, FVector_NetQuantizeNormal Direction)
{
    if (Direction.IsNearlyZero())
    {
        return false;
    }
    return FVector::Dist(MuzzleLocation, GetActorLocation()) <= 500.f;
}

void AMyWeapon::MulticastSpawnTracer_Implementation(FVector_NetQuantize Start, FVector_NetQuantize End)
{
    // Cosmetic tracer; every declared RPC needs its _Implementation or the module fails to link.
}
```

`AMyCharacter` declares the server-side entry point `void ApplyDamageOnServer(float Damage);` — see [replication patterns, Pattern 1](replication-patterns.md).

Returning `false` from `_Validate` closes the calling client's connection, so validate only what a legitimate client can never violate. Reject impossible values; do not use `_Validate` for gameplay rules like cooldowns — put those in `_Implementation`.

### Verbatim engine examples

```cpp
// PlayerController.h:1472-1473 — possession handshake must not be lost
UFUNCTION(reliable, server, WithValidation)
ENGINE_API void ServerAcknowledgePossession(class APawn* P);

// PlayerController.h:1495-1496 — respawn request
UFUNCTION(reliable, server, WithValidation)
ENGINE_API void ServerRestartPlayer();

// PlayerController.h:1499-1500 — spectator position, sent constantly, safe to drop
UFUNCTION(unreliable, server, WithValidation)
ENGINE_API void ServerSetSpectatorLocation(FVector NewLoc, FRotator NewRot);

// PlayerController.h:1522-1523 — camera update, superseded every tick
UFUNCTION(unreliable, server, WithValidation)
ENGINE_API void ServerUpdateCamera(FVector_NetQuantize CamLoc, int32 CamPitchAndYaw);
```

Note the pattern: handshakes and one-off state transitions are `reliable`; anything re-sent next tick is `unreliable`.

| Scenario | Reliability |
|---|---|
| Purchase, ability activation, respawn | Reliable |
| Possession acknowledgement, level-load notification | Reliable |
| Settings or name change | Reliable |
| Camera / spectator position | Unreliable |
| Per-tick input relay | Unreliable |

---

## Client RPC

`Client` RPCs rarely need `WithValidation` — the server is already authoritative (a few engine ones use it, e.g. `ClientSetLocation`, `Controller.h:198-199`).

```cpp
// MyPlayerController.h
UFUNCTION(Client, Reliable)
void ClientShowMatchResult(bool bWon, int32 FinalScore);
```

```cpp
// MyPlayerController.cpp
#include "MyPlayerController.h"
#include "MyHUD.h"

void AMyPlayerController::ClientShowMatchResult_Implementation(bool bWon, int32 FinalScore)
{
    // Owning client only — local UI is safe to touch here.
    // Not "MyHUD": that name shadows APlayerController::MyHUD (C4458, an error in UE builds).
    if (AMyHUD* MatchHUD = Cast<AMyHUD>(GetHUD()))
    {
        MatchHUD->DisplayMatchResult(bWon, FinalScore);
    }
}
```

`AMyHUD` declares `void DisplayMatchResult(bool bWon, int32 FinalScore);` in its header; `AHUD` only exists on clients, so this path never runs on a dedicated server.

### Verbatim engine examples

```cpp
// PlayerController.h:944-945 — mute notification
UFUNCTION(Reliable, Client)
ENGINE_API virtual void ClientMutePlayer(FUniqueNetIdRepl PlayerId);

// PlayerController.h:1468-1469 — localized message to one player
UFUNCTION(Reliable, Client)
ENGINE_API void ClientReceiveLocalizedMessage(TSubclassOf<ULocalMessage> Message, int32 Switch = 0, class APlayerState* RelatedPlayerState_1 = nullptr, class APlayerState* RelatedPlayerState_2 = nullptr, class UObject* OptionalObject = nullptr);

// PlayerController.h:453-454 — spectator state change
UFUNCTION(client, reliable, Category=PlayerController)
ENGINE_API void ClientSetSpectatorWaiting(bool bWaiting);

// PlayerController.h:2334-2335 — network physics timestamp setup
UFUNCTION(Client, Reliable)
ENGINE_API void ClientSetupNetworkPhysicsTimestamp(FAsyncPhysicsTimestamp Timestamp);

// PlayerController.h:2338-2339 — continuously corrected, safe to drop
UFUNCTION(Client, Unreliable)
ENGINE_API void ClientAckTimeDilation(float TimeDilation, int32 ServerStep);
```

| Scenario | Reliability |
|---|---|
| Match result, kill feed, kick notice | Reliable |
| View target change, spectator state | Reliable |
| Connection or physics setup handshake | Reliable |
| Time dilation / clock correction | Unreliable |
| Per-frame positional feedback | Unreliable |

---

## NetMulticast RPC

```cpp
// MyExplosive.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/Actor.h"
#include "Engine/NetSerialization.h"
#include "MyExplosive.generated.h"

class UNiagaraSystem;
class USoundBase;

UCLASS()
class MYGAME_API AMyExplosive : public AActor
{
    GENERATED_BODY()

public:
    UFUNCTION(NetMulticast, Unreliable)
    void MulticastPlayExplosion(FVector_NetQuantize Location);

protected:
    UPROPERTY(EditDefaultsOnly, Category = "FX")
    TObjectPtr<UNiagaraSystem> ExplosionSystem;

    UPROPERTY(EditDefaultsOnly, Category = "FX")
    TObjectPtr<USoundBase> ExplosionSound;
};
```

```cpp
// MyExplosive.cpp
#include "MyExplosive.h"
#include "NiagaraFunctionLibrary.h"
#include "Kismet/GameplayStatics.h"
#include "Engine/World.h"

void AMyExplosive::MulticastPlayExplosion_Implementation(FVector_NetQuantize Location)
{
    // Runs on the server too; skip cosmetics there.
    if (GetNetMode() != NM_DedicatedServer)
    {
        UNiagaraFunctionLibrary::SpawnSystemAtLocation(GetWorld(), ExplosionSystem, Location);
        UGameplayStatics::PlaySoundAtLocation(GetWorld(), ExplosionSound, Location);
    }
}
```

Guard with `GetNetMode() != NM_DedicatedServer` rather than `!HasAuthority()`: a listen-server host is the authority and still needs to see the effect.

The multicast only reaches clients for whom the actor is currently relevant — an explosion on the far side of the map is silently skipped. Build.cs needs `"Niagara"` for `UNiagaraFunctionLibrary`.

| Data | Mechanism |
|---|---|
| Plays once, loss acceptable | `NetMulticast, Unreliable` |
| Plays once, must not be missed | `NetMulticast, Reliable` |
| Must be correct for late joiners | `UPROPERTY(Replicated)` |
| Different per client | `Client` RPC |

---

## Reliable vs Unreliable

Use **Reliable** when the call has permanent state consequences, when the receiver must acknowledge it to proceed, when no later update carries the same information, or when dropping it causes a visible desync.

Use **Unreliable** when the next update supersedes this one, when the payload is purely cosmetic, or when the call happens more than once per second.

Reliable bunches are ordered and held until acknowledged. Flooding them — a reliable RPC called every frame — delays everything queued behind a lost packet, and once a channel has more than `RELIABLE_BUFFER` (512, `NetConnection.h:82`) unacked reliable bunches the connection is closed with "Outgoing reliable buffer overflow" (`DataChannel.cpp:1440-1445`). Keep per-tick traffic unreliable.

---

## Ownership Routing

RPCs are dropped when the ownership chain does not reach a `UNetConnection`; the only sign is a `LogNet` Warning.

### Server RPC

```
Client calls MyWeapon->ServerFireWeapon(...)
  MyWeapon->GetOwner()            -> AMyCharacter
  AMyCharacter->GetOwner()        -> AMyPlayerController
  AMyPlayerController->NetConnection == the calling client's connection
  -> routed
```

If `GetOwner()` is `nullptr`, or resolves to a different player's controller, the call is dropped on the client with only `LogNet: Warning: UNetDriver::ProcessRemoteFunction: No owning connection for actor …` (`NetDriver.cpp:8274`). Fix it by calling `SetOwner(OwningPlayerController)` on the server after spawning, and by keeping `AActor::SetOwner` in sync when the pawn changes hands.

### Client RPC

The actor must resolve to one player's connection, which is why most `Client` RPCs are declared on `APlayerController`:

```cpp
// Server-side
if (AMyPlayerController* PC = Cast<AMyPlayerController>(TargetController))
{
    PC->ClientShowMatchResult(true, FinalScore);
}
```

`AActor::GetNetConnection()` walks the owner chain; `APlayerController` overrides it to return its own `NetConnection` (`PlayerController.h:478`, `PlayerController.h:1891`).

---

## Common RPC Mistakes

### Calling a Server RPC on the server

It executes locally with no routing. Harmless but misleading; call the plain helper instead and keep the RPC for the client path.

### Calling a Client or NetMulticast RPC on a client

It is never sent — it just runs locally on that client (`Actor.cpp:5569-5573`, `5587-5592`), so other machines never see it. Use a `Server` RPC to reach the authority, and have the authority issue the `Client`/`NetMulticast` call.

### Using NetMulticast for state

```cpp
// Wrong: a player joining after this never learns the door is open
MulticastSetDoorOpen(true);

// Right: replicated state is correct for late joiners
bDoorOpen = true;                                    // server only
MARK_PROPERTY_DIRTY_FROM_NAME(AMyDoor, bDoorOpen, this); // only if bIsPushBased was set
```

The `MARK_PROPERTY_DIRTY_FROM_NAME` line is required only when the property was declared push-based via `FDoRepLifetimeParams::bIsPushBased`. It is unrelated to Iris and works with the classic net driver.

### Server RPC without validation that mutates state

```cpp
// Insecure: the client dictates the quantity
UFUNCTION(Server, Reliable)
void ServerAddItem(int32 ItemId, int32 Quantity);

// Secure: the server looks price and quantity up from its own data
UFUNCTION(Server, Reliable, WithValidation)
void ServerRequestPurchase(int32 ItemId);
```

### Non-net-addressable parameters

A `UObject*` parameter only survives the trip if the object replicates and is already known to the receiver, or is stably named (an asset, a class, a map-placed actor). A locally spawned effect or a freshly created data object arrives as `nullptr`. Pass an integer id, an `FName`, a `TSubclassOf<>` or a `FGameplayTag` and resolve it on the remote side.

Large payloads do not belong in RPC parameters: a `TArray` over roughly a kilobyte should be a replicated property (or a fast array) instead.

---

## RPC vs Property Replication

| Situation | Mechanism |
|---|---|
| Server changes a value, clients only read it | `UPROPERTY(Replicated)` |
| Clients need a callback when the value lands | `UPROPERTY(ReplicatedUsing = OnRep_Func)` |
| One-shot event, everyone reacts together | `NetMulticast` RPC |
| One-shot event, one player reacts | `Client` RPC |
| Client asks the server to do something | `Server` RPC |
| Late joiners must see the current state | `UPROPERTY(Replicated)` — always |
| Value changes rarely but is compared every tick | `UPROPERTY(Replicated)` + push model |
| Large collection with small per-frame deltas | `FFastArraySerializer` |

---

## Generated Names

| Declared | You implement |
|---|---|
| `ServerFireWeapon` (`WithValidation`) | `ServerFireWeapon_Implementation`, `ServerFireWeapon_Validate` |
| `ClientShowMatchResult` | `ClientShowMatchResult_Implementation` |
| `MulticastPlayExplosion` | `MulticastPlayExplosion_Implementation` |

Call sites always use the declared name; the generated thunk decides whether to route or to run locally. Never declare a body for the plain name — UHT generates it.
