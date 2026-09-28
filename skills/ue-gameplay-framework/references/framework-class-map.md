# Gameplay Framework Class Map

Target engine: **UE 5.8**. Where each class lives, who owns it, the exact lifecycle order, the full class declarations the skill body abbreviates, and verified member tables.

---

## Authority and Presence Matrix

| Class | Dedicated Server | Listen Server (Host) | Listen Server (Remote Client) | Standalone |
|---|---|---|---|---|
| `AGameModeBase` / `AGameMode` | YES (authority) | YES (authority) | NO — `nullptr` | YES (authority) |
| `AGameStateBase` / `AGameState` | YES | YES | YES (replicated) | YES |
| `APlayerController` (local player) | n/a | YES (authority + local) | YES (own only) | YES |
| `APlayerController` (remote player) | YES (all players) | YES (all players) | NO — not present | n/a |
| `APlayerState` (all players) | YES (all) | YES (all) | YES (all, replicated) | YES |
| `APawn` / `ACharacter` (possessed locally) | YES (authority) | YES (authority + local) | YES (autonomous proxy) | YES |
| `APawn` / `ACharacter` (possessed remotely) | YES (authority) | YES (authority) | YES (simulated proxy) | n/a |
| `UGameInstance` | YES | YES | YES | YES |
| `AHUD` | NO | YES (host player only) | YES (own only) | YES |
| `APlayerCameraManager` | NO | YES (host player only) | YES (own only) | YES |

Roles come from `ENetRole` in `Engine/Classes/Engine/EngineTypes.h`. A simulated proxy interpolates its transform from server updates; an autonomous proxy predicts locally and is corrected.

---

## Class Ownership Chain

```
UGameInstance  [persists across every level load]
  |
  +-- UWorld
        |
        +-- AGameModeBase        [server only]
        |     |
        |     +-- AGameStateBase [server + all clients]
        |           |
        |           +-- PlayerArray[]  --> APlayerState per player
        |
        +-- APlayerController    [server: all; client: own only]
              |
              +-- APlayerState           [server + all clients]
              +-- AHUD                   [local client only]
              +-- APlayerCameraManager   [local client only]
              +-- (possesses) --> APawn / ACharacter
                                    |
                                    +-- UCapsuleComponent  (root)
                                    +-- USkeletalMeshComponent
                                    +-- UCharacterMovementComponent
```

---

## Responsibility Summary

| Class | Primary responsibility | Do NOT put here |
|---|---|---|
| `AGameModeBase` | Rules, join approval, spawn selection, match initialisation | Any data a client must read |
| `AGameMode` | `AGameModeBase` plus the match state machine | Per-player data |
| `AGameStateBase` | Global replicated state: server clock, `PlayerArray`, has-begun-play | Server-only decisions |
| `AGameState` | `AGameStateBase` plus `MatchState` and `ElapsedTime` | Client-only UI state |
| `APlayerController` | Input, camera ownership, HUD ownership, possession, RPC bridge | Data other clients need |
| `APlayerState` | Replicated per-player data: name, score, team, ping | Input processing, UI |
| `APawn` | Minimal possessable actor, custom movement | Anything needing capsule/mesh/CMC |
| `ACharacter` | Humanoid body with capsule, skeletal mesh and CMC prediction | Game rules, scoring |
| `UGameInstance` | Cross-level persistence: session handles, save-game refs, analytics | Per-match state |
| `AHUD` | Local-only canvas overlay and debug display | Any replicated data |

---

## Player Join Sequence (Server Side)

```
1. InitGame(MapName, Options, ErrorMessage)
      Before any player connects. InitGameState() follows.

2. PreLogin(Options, Address, UniqueId, ErrorMessage)
      Set ErrorMessage to a non-empty string to reject the connection.

3. Login(NewPlayer, InRemoteRole, Portal, Options, UniqueId, ErrorMessage)
      Calls SpawnPlayerController(), which creates the APlayerController
      and its APlayerState. Returns nullptr on failure.

4. PostLogin(NewPlayer)
      First point where server-to-client RPCs are safe.
      GameState->PlayerArray now contains this player.

5. OnPostLogin(NewPlayer)                      [protected]
      Called at the end of PostLogin. Broadcasts GameModePostLoginEvent
      and fires the K2_PostLogin Blueprint event.

6. HandleStartingNewPlayer_Implementation(NewPlayer)
      Calls RestartPlayer(NewPlayer) unless bStartPlayersAsSpectators,
      MustSpectate() or !PlayerCanRestart().

7. RestartPlayer(NewPlayer)
      FindPlayerStart() -> ChoosePlayerStart()
      SpawnDefaultPawnFor()
      NewPlayer->Possess(Pawn)
```

`DispatchPostLogin` is deprecated (`UE_DEPRECATED(5.6)`, `GameFramework/GameModeBase.h:329`) and its body is empty — override `OnPostLogin` instead.

---

## Player Logout Sequence

```
1. PlayerController destroyed        -- AController::Destroyed (server, controller has a PlayerState)
2. Logout(Exiting)                    -- GameMode notified
3. CleanupPlayerState()               -- PlayerState->Destroy(); APlayerState::Destroyed
                                         calls GameState->RemovePlayerState(this)
4. UnPossess(), controller removed from the world
```

---

## Seamless Travel Actor Survival

With `bUseSeamlessTravel = true` the travel happens in two legs: current map → transition map, then transition map → destination map. `GetSeamlessTravelActorList(bool bToTransition, TArray<AActor*>& ActorList)` is called for BOTH legs.

| Object | Survives seamless travel | Notes |
|---|---|---|
| `UGameInstance` | YES | Never destroyed |
| `APlayerController` | YES (transferred) | Class may change via `GetPlayerControllerClassToSpawnForSeamlessTravel` |
| `APlayerState` | YES (duplicated) | `CopyProperties` is your hook to carry custom fields |
| `AGameModeBase` | NO | New one spawned in the destination map |
| `AGameStateBase` | NO | New one spawned in the destination map |
| `APawn` / `ACharacter` | NO (by default) | Destroyed; respawned by `RestartPlayer` |
| Custom actors | Optional | Add them in `GetSeamlessTravelActorList` |

`APlayerController::SeamlessTravelCount` and `LastCompletedSeamlessTravelCount` let you detect that travel has just finished.

---

## Full Class Declarations

These are the headers the skill body abbreviates. Example names are `AMy*` / `UMy*` in module `MyGame` with API macro `MYGAME_API`.

### AMyGameMode.h

```cpp
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/GameMode.h"
#include "MyGameMode.generated.h"

UCLASS()
class MYGAME_API AMyGameMode : public AGameMode
{
    GENERATED_BODY()

public:
    AMyGameMode();

    virtual void InitGame(const FString& MapName, const FString& Options, FString& ErrorMessage) override;
    virtual void PreLogin(const FString& Options, const FString& Address, const FUniqueNetIdRepl& UniqueId, FString& ErrorMessage) override;
    virtual void PostLogin(APlayerController* NewPlayer) override;
    virtual void Logout(AController* Exiting) override;

protected:
    virtual void OnPostLogin(AController* NewPlayer) override;
    virtual AActor* ChoosePlayerStart_Implementation(AController* Player) override;
    virtual APawn* SpawnDefaultPawnFor_Implementation(AController* NewPlayer, AActor* StartSpot) override;
    virtual bool ReadyToStartMatch_Implementation() override;
    virtual bool ReadyToEndMatch_Implementation() override;
    virtual void HandleMatchHasStarted() override;

    UPROPERTY(EditDefaultsOnly, Category = "Match")
    int32 MinPlayersToStart = 2;

    UPROPERTY(EditDefaultsOnly, Category = "Match")
    int32 ScoreLimit = 10;
};
```

### Team-aware player start selection

```cpp
// MyGameMode.cpp - also include EngineUtils.h, GameFramework/PlayerStart.h and MyPlayerState.h
AActor* AMyGameMode::ChoosePlayerStart_Implementation(AController* Player)
{
    const AMyPlayerState* PS = Player ? Player->GetPlayerState<AMyPlayerState>() : nullptr;
    const FName WantedTag = (PS && PS->TeamIndex == 0) ? FName("TeamA") : FName("TeamB");

    for (TActorIterator<APlayerStart> It(GetWorld()); It; ++It)
    {
        if (It->PlayerStartTag == WantedTag)
        {
            return *It;
        }
    }
    return Super::ChoosePlayerStart_Implementation(Player);
}

bool AMyGameMode::ReadyToEndMatch_Implementation()
{
    const AMyGameState* GS = GetGameState<AMyGameState>();
    return GS != nullptr && GS->TeamAScore >= ScoreLimit;
}

void AMyGameMode::HandleMatchHasStarted()
{
    Super::HandleMatchHasStarted();
    // Server-side match start work: unlock doors, start the clock.
}
```

### AMyPlayerController.h

```cpp
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/PlayerController.h"
#include "MyPlayerController.generated.h"

class UInputMappingContext;

UCLASS()
class MYGAME_API AMyPlayerController : public APlayerController
{
    GENERATED_BODY()

public:
    UFUNCTION(Server, Reliable, WithValidation)
    void ServerRequestRespawn();

    UFUNCTION(Client, Reliable)
    void ClientNotifyMatchStart(float ServerStartTime);

protected:
    virtual void BeginPlay() override;
    virtual void SetupInputComponent() override;
    virtual void OnPossess(APawn* aPawn) override;
    virtual void OnUnPossess() override;

    UPROPERTY(EditDefaultsOnly, Category = "Input")
    TObjectPtr<UInputMappingContext> DefaultMappingContext;
};
```

`ClientNotifyMatchStart_Implementation(float ServerStartTime)` completes the `Client` RPC; `ServerRequestRespawn_Implementation()` plus `ServerRequestRespawn_Validate()` complete the `Server, WithValidation` RPC. Both `_Implementation` and `_Validate` are plain member functions in the `.cpp` — never re-tag them with `UFUNCTION()`.

### AMyHUD.h

```cpp
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/HUD.h"
#include "MyHUD.generated.h"

UCLASS()
class MYGAME_API AMyHUD : public AHUD
{
    GENERATED_BODY()

public:
    virtual void DrawHUD() override;
    virtual void NotifyHitBoxClick(FName BoxName) override;
    virtual void ShowDebugInfo(float& YL, float& YPos) override;
};
```

### UMyGameInstance.h

```cpp
#pragma once

#include "CoreMinimal.h"
#include "Engine/GameInstance.h"
#include "MyGameInstance.generated.h"

UCLASS()
class MYGAME_API UMyGameInstance : public UGameInstance
{
    GENERATED_BODY()

public:
    virtual void Init() override;
    virtual void Shutdown() override;

protected:
    virtual void OnStart() override;
};
```

---

## Legacy Session Search and Join

```cpp
// Needs OnlineSubsystem.h, OnlineSessionSettings.h and Interfaces/OnlineSessionInterface.h.
// SessionSearch is a TSharedPtr<FOnlineSessionSearch> member; the handles are FDelegateHandle members.
void UMyGameInstance::FindSessions()
{
    IOnlineSubsystem* OSS = IOnlineSubsystem::Get();
    IOnlineSessionPtr Sessions = OSS ? OSS->GetSessionInterface() : nullptr;
    if (!Sessions.IsValid())
    {
        return;
    }

    SessionSearch = MakeShared<FOnlineSessionSearch>();
    SessionSearch->MaxSearchResults = 20;

    FindSessionsHandle = Sessions->AddOnFindSessionsCompleteDelegate_Handle(
        FOnFindSessionsCompleteDelegate::CreateUObject(this, &UMyGameInstance::HandleFindSessionsComplete));
    Sessions->FindSessions(0, SessionSearch.ToSharedRef());
}

void UMyGameInstance::HandleFindSessionsComplete(bool bWasSuccessful)
{
    IOnlineSubsystem* OSS = IOnlineSubsystem::Get();
    IOnlineSessionPtr Sessions = OSS ? OSS->GetSessionInterface() : nullptr;
    if (!Sessions.IsValid())
    {
        return;
    }

    Sessions->ClearOnFindSessionsCompleteDelegate_Handle(FindSessionsHandle);

    if (bWasSuccessful && SessionSearch.IsValid() && SessionSearch->SearchResults.Num() > 0)
    {
        JoinSessionHandle = Sessions->AddOnJoinSessionCompleteDelegate_Handle(
            FOnJoinSessionCompleteDelegate::CreateUObject(this, &UMyGameInstance::HandleJoinSessionComplete));
        Sessions->JoinSession(0, NAME_GameSession, SessionSearch->SearchResults[0]);
    }
}

void UMyGameInstance::HandleJoinSessionComplete(FName SessionName, EOnJoinSessionCompleteResult::Type Result)
{
    IOnlineSubsystem* OSS = IOnlineSubsystem::Get();
    IOnlineSessionPtr Sessions = OSS ? OSS->GetSessionInterface() : nullptr;
    if (!Sessions.IsValid())
    {
        return;
    }

    Sessions->ClearOnJoinSessionCompleteDelegate_Handle(JoinSessionHandle);

    FString ConnectInfo;
    if (Result == EOnJoinSessionCompleteResult::Success &&
        Sessions->GetResolvedConnectString(SessionName, ConnectInfo))
    {
        if (APlayerController* PC = GetFirstLocalPlayerController())
        {
            PC->ClientTravel(ConnectInfo, TRAVEL_Absolute);
        }
    }
}
```

`EOnJoinSessionCompleteResult::Type` values: `Success`, `SessionIsFull`, `SessionDoesNotExist`, `CouldNotRetrieveAddress`, `AlreadyInSession`, `UnknownError`.

---

## Key Properties Quick Reference

### AGameModeBase class assignments

```cpp
TSubclassOf<AGameSession>       GameSessionClass;
TSubclassOf<AGameStateBase>     GameStateClass;
TSubclassOf<APlayerController>  PlayerControllerClass;
TSubclassOf<APlayerState>       PlayerStateClass;
TSubclassOf<AHUD>               HUDClass;
TSubclassOf<APawn>              DefaultPawnClass;
TSubclassOf<ASpectatorPawn>     SpectatorClass;
TSubclassOf<APlayerController>  ReplaySpectatorPlayerControllerClass;
uint32                          bUseSeamlessTravel : 1;
uint32                          bStartPlayersAsSpectators : 1;
uint32                          bPauseable : 1;
```

`AWorldSettings::DefaultGameMode` (a `TSubclassOf<AGameModeBase>`) supplies the per-map override; `AGameModeBase::GetGameSessionClass()` is the virtual that picks the `AGameSession`.

### AGameStateBase replicated members

```cpp
TSubclassOf<AGameModeBase>       GameModeClass;                    // ReplicatedUsing = OnRep_GameModeClass
TSubclassOf<ASpectatorPawn>      SpectatorClass;                   // ReplicatedUsing = OnRep_SpectatorClass
TArray<TObjectPtr<APlayerState>> PlayerArray;                      // Transient, BlueprintReadOnly; NOT replicated, filled locally by AddPlayerState
bool                             bReplicatedHasBegunPlay;          // ReplicatedUsing = OnRep_ReplicatedHasBegunPlay
double                           ReplicatedWorldTimeSecondsDouble; // ReplicatedUsing = OnRep_ReplicatedWorldTimeSecondsDouble
```

`AGameState` adds `FName MatchState` (`ReplicatedUsing = OnRep_MatchState`), `FName PreviousMatchState` and `int32 ElapsedTime` (`replicatedUsing = OnRep_ElapsedTime`).

### APlayerController notable members

```cpp
TObjectPtr<APawn>                AcknowledgedPawn;   // server-confirmed possession
TObjectPtr<AHUD>                 MyHUD;
TObjectPtr<APlayerCameraManager> PlayerCameraManager;
TObjectPtr<UPlayerInput>         PlayerInput;        // valid on local controllers only
TObjectPtr<UCheatManager>        CheatManager;
TSubclassOf<UCheatManager>       CheatClass;
uint32                           bShowMouseCursor : 1;
uint32                           bEnableClickEvents : 1;
uint32                           bEnableStreamingSource : 1;
uint16                           SeamlessTravelCount;
uint16                           LastCompletedSeamlessTravelCount;
```

`UCheatManager::AddCheatManagerExtension(UCheatManagerExtension* CheatObject)` and `RemoveCheatManagerExtension` attach per-feature cheat objects; `virtual void InitCheatManager()` is the setup hook.

### AController members shared by player and AI controllers

```cpp
TObjectPtr<APlayerState> PlayerState;     // replicatedUsing = OnRep_PlayerState
TObjectPtr<APawn>        Pawn;            // replicatedUsing = OnRep_Pawn
FRotator                 ControlRotation;
TWeakObjectPtr<AActor>   StartSpot;
```

Virtuals: `virtual void OnPossess(APawn* InPawn)`, `virtual void OnUnPossess()`, `virtual void SetPawn(APawn* InPawn)`, `virtual void OnRep_Pawn()`, `virtual void OnRep_PlayerState()`, `virtual FRotator GetControlRotation() const`, `virtual void SetControlRotation(const FRotator& NewRotation)`, `virtual void InitPlayerState()`, `virtual void CleanupPlayerState()`. `Possess` and `UnPossess` are `virtual final`.

### APlayerState members

```cpp
uint8 bIsSpectator : 1;
uint8 bOnlySpectator : 1;
```

Accessors: `FString GetPlayerName() const`, `virtual FString GetPlayerNameCustom() const`, `float GetScore() const`, `void SetScore(const float NewScore)`, `int32 GetPlayerId() const`, `APawn* GetPawn() const`, `class APlayerController* GetPlayerController() const`, `virtual class APlayerState* Duplicate()`, `virtual void ClientInitialize(class AController* C)`. Travel hooks: `virtual void CopyProperties(APlayerState* PlayerState)`, `virtual void OverrideWith(APlayerState* PlayerState)`, `virtual void SeamlessTravelTo(class APlayerState* NewPlayerState)`.

### ACharacter notable members

```cpp
uint8 bIsCrouched : 1;             // replicatedUsing = OnRep_IsCrouched
uint8 bProxyIsJumpForceApplied : 1;
uint8 ReplicatedMovementMode;      // GetReplicatedMovementMode() / SetReplicatedMovementMode(const uint8)
float JumpMaxHoldTime;             // Replicated
int32 JumpMaxCount;                // Replicated
int32 JumpCurrentCount;
```

Component getters: `USkeletalMeshComponent* GetMesh() const`, `UCharacterMovementComponent* GetCharacterMovement() const`, `UCapsuleComponent* GetCapsuleComponent() const`, `class UArrowComponent* GetArrowComponent() const` (editor-only direction indicator, declared only under `#if WITH_EDITORONLY_DATA`).

### AHUD notable members

```cpp
TObjectPtr<APlayerController> PlayerOwner;
TObjectPtr<UCanvas>           Canvas;        // valid only inside DrawHUD
TObjectPtr<UCanvas>           DebugCanvas;
uint8                         bShowHUD : 1;
uint8                         bShowDebugInfo : 1;
uint8                         bShowDebugForReticleTarget : 1;
TSubclassOf<AActor>           ShowDebugTargetDesiredClass;
TObjectPtr<AActor>            ShowDebugTargetActor;
```

Exec commands: `virtual void ShowHUD()`, `virtual void ShowDebug(FName DebugType = NAME_None)`. Helpers: `void ShowDebugToggleSubCategory(FName Category)`, `void ShowDebugForReticleTargetToggle(TSubclassOf<AActor> DesiredClass)`, `virtual void PostRender()`.

---

## NetMode Cheat Sheet

```cpp
GetNetMode() == NM_Standalone       // single player, no network
GetNetMode() == NM_DedicatedServer  // server process, no local player
GetNetMode() == NM_ListenServer     // server plus a local player (host)
GetNetMode() == NM_Client           // remote client

HasAuthority()        // true on standalone, dedicated server and listen server
IsLocalController()   // APlayerController: belongs to this machine's player
IsLocallyControlled() // APawn: its controller is local
```

Every mode numerically below `NM_Client` is some kind of server, so `GetNetMode() < NM_Client` is the "am I a server" test.

---

## Class Selection Decision Tree

```
Game-wide rules, or control over who may join?
  --> AGameModeBase (AGameMode if you need the match state machine)

Global data every client must see (scores, match timer)?
  --> AGameStateBase (replicated everywhere)

Per-player data every client must see (kills, team, name)?
  --> APlayerState (always relevant, replicated everywhere)

Input, camera or UI for ONE player?
  --> APlayerController (server plus the owning client only)

A body that walks, jumps and crouches with built-in prediction?
  --> ACharacter (with UCharacterMovementComponent)

A vehicle, drone or other non-humanoid body?
  --> APawn (with a custom UPawnMovementComponent or manual physics)

Data that must survive level transitions?
  --> UGameInstance (one per process, never destroyed)
```

---

## Movement Modes

`EMovementMode` (`MOVE_None`, `MOVE_Walking`, `MOVE_NavWalking`, `MOVE_Falling`, `MOVE_Swimming`, `MOVE_Flying`, `MOVE_Custom`) and everything about `UCharacterMovementComponent` — custom modes, `PhysCustom` (call `Super::PhysCustom(DeltaTime, Iterations)` first), root motion, network prediction and `FMovementBaseInterfaceData` — belong to `ue-character-movement`.
