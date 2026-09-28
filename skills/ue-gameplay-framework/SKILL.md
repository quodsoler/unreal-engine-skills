---
name: ue-gameplay-framework
description: "Use when writing or fixing Unreal Engine gameplay framework classes — GameMode, GameState, PlayerController, PlayerState, Pawn, Character, HUD, GameInstance — and the flows they own: player join, pawn spawning, match state, travel and non-GAS damage. Also use when the user mentions 'AGameModeBase', 'PostLogin', 'OnPostLogin', 'RestartPlayer', 'SpawnDefaultPawnFor', 'ChoosePlayerStart', 'match state', 'PossessedBy', 'seamless travel', 'ServerTravel', 'DrawHUD', 'TakeDamage', 'ApplyRadialDamage' or 'create session'. For replication, see ue-networking-replication; for CharacterMovementComponent, see ue-character-movement; for cameras, see ue-gameplay-cameras; for input, see ue-input-system."
metadata:
  version: "2.0.0"
  engine: "5.8"
---

# UE Gameplay Framework

Target engine: **UE 5.8**. APIs below are verified against the 5.8 headers; older forms are listed under "Deprecated — do not use".

The gameplay framework is the set of classes the engine spawns for you — `AGameModeBase`, `AGameStateBase`, `APlayerController`, `APlayerState`, `APawn`, `ACharacter`, `AHUD`, `UGameInstance` — plus the join, spawn, match-state, travel and damage flows that wire them together. All of it ships in the `Engine` module (`Classes/GameFramework`, `Classes/Engine`, `Classes/Kismet`), so Build.cs needs `"Core"`, `"CoreUObject"`, `"Engine"`. Add `"EnhancedInput"` when binding input, and `"OnlineSubsystem"` + `"OnlineSubsystemUtils"` (or `"OnlineServicesInterface"`) only when touching sessions.

## Context

Read `.agents/ue-project-context.md` if it exists (module names, conventions, enabled plugins, GAS/networking setup). Do not stop if it is missing.

Identify the area from the request and the codebase. Ask only when two plausible readings would produce different code.

| Request is about… | Go to |
|---|---|
| Which class holds which data; server vs client presence | [Class Responsibility Map](#class-responsibility-map) |
| Join flow, player starts, pawn spawning, match state | [GameMode](#gamemode-server-only) |
| Replicated scores, timers, connected-player list | [GameState](#gamestate-everywhere) |
| Per-player input, RPC bridge, possession, input mode | [PlayerController](#playercontroller) |
| Per-player replicated stats, team, name | [PlayerState](#playerstate) |
| Possessable bodies, `PossessedBy`, character setup | [Pawn and Character](#pawn-and-character) |
| Canvas overlay, hit boxes, `showdebug` | [HUD](#hud) |
| Health and damage without Gameplay Abilities | [Damage without GAS](#damage-without-gas) |
| Data that survives level load, subsystems, accessors | [GameInstance](#gameinstance) |
| Hosting, finding and joining online sessions | [Sessions](#sessions) |
| `ServerTravel`, `ClientTravel`, seamless travel | [Travel](#travel) |

Presence/authority matrix, the full class declarations these snippets come from, lifecycle order and inherited-property tables: [references/framework-class-map.md](references/framework-class-map.md).

## Class Responsibility Map

| Class | Lives on | Owns | Never put here |
|---|---|---|---|
| `AGameModeBase` / `AGameMode` | Server and standalone only; `nullptr` on clients | Rules, join approval, spawn selection, match state | Anything a client must read |
| `AGameStateBase` / `AGameState` | Everywhere, replicated | Global replicated state, `PlayerArray`, server clock | Server-only decisions |
| `APlayerController` | Server holds all; each client holds only its own | Input, RPC bridge, possession, input mode, view target | Data other clients need |
| `APlayerState` | Everywhere (`bAlwaysRelevant = true`) | Per-player replicated stats, team, name, ping | Input handling, UI state |
| `APawn` / `ACharacter` | Server plus all clients | The body: collision, mesh, movement | Rules, scoring |
| `AHUD` | Local players only; never on a dedicated server | Canvas overlay, hit boxes, debug display | Replicated data |
| `UGameInstance` | One per process, survives every level load | Session handles, cross-level state, subsystems | Per-match state |

`APlayerState` sets `bReplicates = true` and `bAlwaysRelevant = true` in its constructor, which is why scoreboards work at any distance.

## GameMode (Server Only)

`AGameModeBase` is the minimal base. `AGameMode` adds the match state machine — use it when you need `MatchState::EnteringMap → WaitingToStart → InProgress → WaitingPostMatch → LeavingMap` (plus `Aborted` on failure).

### Join sequence (server only)

| Order | Signature to override | Notes |
|---|---|---|
| 1 | `virtual void InitGame(const FString& MapName, const FString& Options, FString& ErrorMessage)` | Before any player joins; also `virtual void InitGameState()` |
| 2 | `virtual void PreLogin(const FString& Options, const FString& Address, const FUniqueNetIdRepl& UniqueId, FString& ErrorMessage)` | Set `ErrorMessage` non-empty to reject. For a non-blocking credential check override `PreLoginAsync(..., const FOnPreLoginCompleteDelegate& OnComplete)` (`GameModeBase.h:303`) instead; it must always call `OnComplete` (empty string = accept) or the connection hangs, and it must still run the `PreLogin` checks |
| 3 | `virtual APlayerController* Login(UPlayer* NewPlayer, ENetRole InRemoteRole, const FString& Portal, const FString& Options, const FUniqueNetIdRepl& UniqueId, FString& ErrorMessage)` | Spawns the `APlayerController` and its `APlayerState` |
| 4 | `virtual void PostLogin(APlayerController* NewPlayer)` | First point where server→client RPCs are safe |
| 5 | `virtual void OnPostLogin(AController* NewPlayer)` | Protected; runs at the end of `PostLogin`, last hook before the pawn spawns |
| 6 | `virtual void HandleStartingNewPlayer_Implementation(APlayerController* NewPlayer)` | `BlueprintNativeEvent`; calls `RestartPlayer` unless the player must spectate |
| 7 | `virtual void RestartPlayer(AController* NewPlayer)` | `FindPlayerStart` → `SpawnDefaultPawnFor` → `Possess` |
| — | `virtual void Logout(AController* Exiting)` | Fires when a controller with a PlayerState leaves |

```cpp
// MyGameMode.cpp - include the header of every class named below
#include "MyGameMode.h"

AMyGameMode::AMyGameMode()
{
    DefaultPawnClass      = AMyCharacter::StaticClass();
    PlayerControllerClass = AMyPlayerController::StaticClass();
    GameStateClass        = AMyGameState::StaticClass();
    PlayerStateClass      = AMyPlayerState::StaticClass();
    HUDClass              = AMyHUD::StaticClass();
    bUseSeamlessTravel    = true;
}

bool AMyGameMode::ReadyToStartMatch_Implementation()
{
    return GetNumPlayers() >= MinPlayersToStart;
}
```

`ChoosePlayerStart`, `FindPlayerStart`, `SpawnDefaultPawnFor`, `PlayerCanRestart`, `MustSpectate`, `ReadyToStartMatch` and `ReadyToEndMatch` are `BlueprintNativeEvent` — override the generated `_Implementation`, never the bare name. Verbatim forms: `virtual AActor* ChoosePlayerStart_Implementation(AController* Player)`, `virtual AActor* FindPlayerStart_Implementation(AController* Player, const FString& IncomingName)`, `virtual APawn* SpawnDefaultPawnFor_Implementation(AController* NewPlayer, AActor* StartSpot)`. Plain virtuals in the same pipeline: `virtual void RestartPlayerAtPlayerStart(AController* NewPlayer, AActor* StartSpot)`, `virtual void RestartPlayerAtTransform(AController* NewPlayer, const FTransform& SpawnTransform)`, `virtual bool ShouldSpawnAtStartSpot(AController* Player)`. Spawn points are `APlayerStart` actors, selected by their `FName PlayerStartTag`.

### Match state (AGameMode only)

Public: `FName GetMatchState() const`, `virtual bool IsMatchInProgress() const`, `virtual void StartMatch()`, `virtual void EndMatch()`, `virtual void AbortMatch()`, `virtual void RestartGame()`.
Protected hooks: `virtual void SetMatchState(FName NewState)`, `virtual void OnMatchStateSet()`, `virtual void HandleMatchIsWaitingToStart()`, `virtual void HandleMatchHasStarted()`, `virtual void HandleMatchHasEnded()`, `virtual void HandleLeavingMap()`, `virtual void HandleMatchAborted()`.

`SetMatchState` is the only sanctioned way to move states; it pushes the new state onto `AGameState::MatchState`, which replicates to clients. Do not write `MatchState` directly.

## GameState (Everywhere)

Clients cannot see the GameMode, so everything they need goes here.

```cpp
// MyGameState.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/GameStateBase.h"
#include "MyGameState.generated.h"

UCLASS()
class MYGAME_API AMyGameState : public AGameStateBase
{
    GENERATED_BODY()

public:
    virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override;

    UPROPERTY(Replicated, BlueprintReadOnly, Category = "Match")
    int32 TeamAScore = 0;

    UPROPERTY(ReplicatedUsing = OnRep_MatchTimeRemaining, BlueprintReadOnly, Category = "Match")
    float MatchTimeRemaining = 0.f;

protected:
    UFUNCTION()
    void OnRep_MatchTimeRemaining();
};

// MyGameState.cpp - also include Net/UnrealNetwork.h

void AMyGameState::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);
    DOREPLIFETIME(AMyGameState, TeamAScore);
    DOREPLIFETIME(AMyGameState, MatchTimeRemaining);
}

void AMyGameState::OnRep_MatchTimeRemaining()
{
    // Client-side reaction: refresh the timer widget.
}
```

Every `ReplicatedUsing = X` needs a matching `UFUNCTION() void X();` declaration, and every `Replicated`/`ReplicatedUsing` property needs a `DOREPLIFETIME` line inside `GetLifetimeReplicatedProps`. Conditions, `COND_*`, push model and RPC rules belong to `ue-networking-replication`.

Inherited: `TArray<TObjectPtr<APlayerState>> PlayerArray` (not itself replicated; each `APlayerState` adds itself on every machine via `AddPlayerState`, which works because PlayerStates are always relevant). Inherited and replicated: `TSubclassOf<AGameModeBase> GameModeClass` (`ReplicatedUsing = OnRep_GameModeClass`), `TSubclassOf<ASpectatorPawn> SpectatorClass`, `bReplicatedHasBegunPlay`, `ReplicatedWorldTimeSecondsDouble`. Accessors: `virtual double GetServerWorldTimeSeconds() const`, `virtual bool HasBegunPlay() const`, `virtual bool HasMatchStarted() const`, `virtual bool HasMatchEnded() const`, `virtual void AddPlayerState(APlayerState* PlayerState)`, `virtual void RemovePlayerState(APlayerState* PlayerState)`. `AGameState` adds `FName MatchState` (`ReplicatedUsing = OnRep_MatchState`) and `int32 ElapsedTime` (`replicatedUsing = OnRep_ElapsedTime`).

## PlayerController

```cpp
// MyPlayerController.cpp
#include "MyPlayerController.h"
#include "EnhancedInputSubsystems.h"
#include "Engine/LocalPlayer.h"
#include "InputMappingContext.h"

void AMyPlayerController::BeginPlay()
{
    Super::BeginPlay();

    if (IsLocalController())
    {
        if (UEnhancedInputLocalPlayerSubsystem* Subsystem =
                ULocalPlayer::GetSubsystem<UEnhancedInputLocalPlayerSubsystem>(GetLocalPlayer()))
        {
            Subsystem->AddMappingContext(DefaultMappingContext, 0);
        }
        SetInputMode(FInputModeGameOnly());
    }
}

void AMyPlayerController::ServerRequestRespawn_Implementation()
{
    // Runs on the server.
}

bool AMyPlayerController::ServerRequestRespawn_Validate()
{
    return true;
}
```

Declarations for the above (full class body in the reference): `virtual void BeginPlay() override;`, `virtual void SetupInputComponent() override;`, `virtual void OnPossess(APawn* aPawn) override;`, `virtual void OnUnPossess() override;`, and `UFUNCTION(Server, Reliable, WithValidation) void ServerRequestRespawn();`.

`AController::Possess` and `AController::UnPossess` are `virtual final`; override `OnPossess` / `OnUnPossess` instead. `APlayerController::SetupInputComponent()` is a protected virtual declared on `APlayerController` itself — use it for input that outlives possession (menus, spectator keys, global shortcuts). Pawn-specific bindings go in `APawn::SetupPlayerInputComponent`; mapping contexts, actions, modifiers and triggers belong to `ue-input-system`.

Notable members: `TObjectPtr<APawn> AcknowledgedPawn` (server-confirmed possession), `TObjectPtr<AHUD> MyHUD`, `TObjectPtr<APlayerCameraManager> PlayerCameraManager`, `TObjectPtr<UPlayerInput> PlayerInput` (local only), `TSubclassOf<UCheatManager> CheatClass`, `uint32 bShowMouseCursor:1`, `uint32 bEnableClickEvents:1`, `uint32 bEnableStreamingSource:1` (drives World Partition loading for this viewport), `uint16 SeamlessTravelCount`. Input modes `FInputModeGameOnly`, `FInputModeUIOnly` and `FInputModeGameAndUI` are applied with `virtual void SetInputMode(const FInputModeDataBase& InData)`.

| Camera work | Owner |
|---|---|
| `PlayerCameraManager`, `SetViewTarget`, `SetViewTargetWithBlend`, camera modifiers, camera shakes, Gameplay Cameras (Experimental in 5.8) | `ue-gameplay-cameras` |

**Listen-server dual role:** the host's PlayerController has both `ROLE_Authority` and a local player. Guard with `IsLocalController()` as well as `HasAuthority()`; code written for a dedicated server often runs twice on a listen server host.

## PlayerState

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
    virtual void CopyProperties(APlayerState* PlayerState) override;

    UPROPERTY(Replicated, BlueprintReadOnly, Category = "Score")
    int32 Kills = 0;

    UPROPERTY(ReplicatedUsing = OnRep_TeamIndex, BlueprintReadOnly, Category = "Team")
    uint8 TeamIndex = 0;

protected:
    UFUNCTION()
    void OnRep_TeamIndex();
};

// MyPlayerState.cpp - also include Net/UnrealNetwork.h

void AMyPlayerState::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);
    DOREPLIFETIME(AMyPlayerState, Kills);
    DOREPLIFETIME(AMyPlayerState, TeamIndex);
}

void AMyPlayerState::OnRep_TeamIndex()
{
    // Client-side reaction: recolour the team badge.
}

void AMyPlayerState::CopyProperties(APlayerState* PlayerState)
{
    Super::CopyProperties(PlayerState);

    if (AMyPlayerState* Target = Cast<AMyPlayerState>(PlayerState))
    {
        Target->Kills     = Kills;
        Target->TeamIndex = TeamIndex;
    }
}
```

Without both the `UFUNCTION()` OnRep declaration and the `DOREPLIFETIME` entries the properties compile, run, and silently never replicate. Replication conditions and push model: `ue-networking-replication`.

Seamless-travel hooks: `virtual void CopyProperties(APlayerState* PlayerState)` (protected; called on the old PlayerState with the surviving one as the argument, and when an inactive PlayerState is saved on disconnect), `virtual void OverrideWith(APlayerState* PlayerState)`, `virtual void SeamlessTravelTo(class APlayerState* NewPlayerState)`. Built-ins: `FString GetPlayerName() const`, `float GetScore() const` / `void SetScore(const float NewScore)`, `int32 GetPlayerId() const`, `APawn* GetPawn() const`, `class APlayerController* GetPlayerController() const`, `virtual void ClientInitialize(class AController* C)`.

Reach one with `MyPawn->GetPlayerState()` or `MyController->PlayerState`; reach all of them through `PlayerArray` on the GameState:

```cpp
if (const AGameStateBase* GS = GetWorld()->GetGameState<AGameStateBase>())
{
    for (APlayerState* PS : GS->PlayerArray) { /* every connected player */ }
}
```

## Pawn and Character

`APawn` is the minimal possessable actor: no mesh, no collision component, no movement component. `ACharacter` adds a capsule root, a skeletal mesh and `UCharacterMovementComponent` with networked prediction. `ADefaultPawn` is the engine's flying placeholder (`GetCollisionComponent()`, `GetMeshComponent()`, `bAddDefaultMovementBindings`); `ASpectatorPawn` derives from it.

```cpp
// Pawn-side overrides
virtual void SetupPlayerInputComponent(UInputComponent* PlayerInputComponent) override;
virtual void PossessedBy(AController* NewController) override;
virtual void UnPossessed() override;
virtual void OnRep_Controller() override;
virtual void NotifyControllerChanged() override;
```

`PossessedBy` runs on the server only. `OnRep_Controller` is the client-side counterpart for the replicated `Controller` pointer; `NotifyControllerChanged` fires on the server and the owning client (not simulated proxies, `Pawn.h:396`), which makes it the right place for owner-side logic such as refreshing the input mapping context.

In the Character constructor, size the capsule with `GetCapsuleComponent()->SetCapsuleSize(42.f, 96.f)` and place the mesh with `GetMesh()->SetRelativeLocation(...)` / `SetRelativeRotation(...)`; that needs `#include "Components/CapsuleComponent.h"` and `#include "Components/SkeletalMeshComponent.h"`.

| Area | Owner |
|---|---|
| `UCharacterMovementComponent` tuning, `EMovementMode`, `PhysCustom` (call `Super::PhysCustom(DeltaTime, Iterations)` **first**), root motion, prediction, `FMovementBaseInterfaceData`, Mover (Experimental in 5.8) | `ue-character-movement` |

Character API you will reach for directly: `virtual void Jump()`, `virtual void StopJumping()`, `bool CanJump() const`, `virtual void LaunchCharacter(FVector LaunchVelocity, bool bXYOverride, bool bZOverride)`, `virtual void Crouch(bool bClientSimulation = false)`, `virtual void UnCrouch(bool bClientSimulation = false)`, `virtual void OnRep_IsCrouched()`. Replicated state: `float JumpMaxHoldTime`, `int32 JumpMaxCount`, `uint8 bIsCrouched:1`. Component getters: `GetCapsuleComponent()`, `GetMesh()`, `GetCharacterMovement()`, and `GetArrowComponent()` (declared only under `#if WITH_EDITORONLY_DATA`; guard any call).

## HUD

`AHUD` is the immediate-mode canvas overlay for one local player, spawned from `AGameModeBase::HUDClass`. It never exists on a dedicated server. For retained UI (menus, health bars, layout) use UMG — see `ue-ui-umg-slate`.

```cpp
// MyHUD.cpp — declares: virtual void DrawHUD() override; virtual void NotifyHitBoxClick(FName BoxName) override;
#include "Engine/Canvas.h"   // plus MyHUD.h and GameFramework/PlayerController.h

static const FName MenuBoxName(TEXT("MenuButton"));

void AMyHUD::DrawHUD()
{
    Super::DrawHUD();

    if (Canvas == nullptr)
    {
        return;
    }

    const float CentreX = Canvas->SizeX * 0.5f;
    const float CentreY = Canvas->SizeY * 0.5f;

    DrawRect(FLinearColor::White, CentreX - 2.f, CentreY - 2.f, 4.f, 4.f);
    DrawText(TEXT("Menu"), FLinearColor::White, 40.f, 40.f);
    AddHitBox(FVector2D(40.f, 40.f), FVector2D(120.f, 24.f), MenuBoxName, true, 0);
}

void AMyHUD::NotifyHitBoxClick(FName BoxName)
{
    Super::NotifyHitBoxClick(BoxName);

    if (BoxName == MenuBoxName)
    {
        if (APlayerController* PC = GetOwningPlayerController())
        {
            PC->SetInputMode(FInputModeUIOnly());
        }
    }
}
```

`TObjectPtr<UCanvas> Canvas` is only valid inside `DrawHUD` — never cache it. Drawing helpers: `DrawText`, `DrawLine`, `DrawRect`, `DrawTexture`, `DrawTextureSimple`. Hit boxes added with `void AddHitBox(FVector2D Position, FVector2D Size, FName InName, bool bConsumesInput, int32 Priority = 0)` raise the native `virtual void NotifyHitBoxClick(FName BoxName)`, `NotifyHitBoxRelease`, `NotifyHitBoxBeginCursorOver` and `NotifyHitBoxEndCursorOver`, plus the `ReceiveHitBoxClick` / `ReceiveHitBoxRelease` / `ReceiveHitBoxBeginCursorOver` Blueprint events. Owner access: `TObjectPtr<APlayerController> PlayerOwner`, `APlayerController* GetOwningPlayerController() const`, `APawn* GetOwningPawn() const`.

Debug display: `uint8 bShowHUD:1` gates all drawing and is toggled by the `ShowHUD` exec; `virtual void ShowDebug(FName DebugType = NAME_None)` backs the `showdebug` console command (engine-supported values include `AI`, `physics`, `net`, `camera`, `collision`); override `virtual void ShowDebugInfo(float& YL, float& YPos)` to add your own lines.

## Damage without GAS

Engine damage is one virtual plus four `UGameplayStatics` entry points. Use it when you do not have GAS; attributes, execution calculations and cues belong to `ue-gameplay-abilities`.

```cpp
// Dealing damage - server authority only. Needs Kismet/GameplayStatics.h.
// Target, Hit, ShotDirection, InstigatorController and IgnoreActors are locals of the caller.
UGameplayStatics::ApplyDamage(Target, 25.f, InstigatorController, this, UMyDamageType::StaticClass());
UGameplayStatics::ApplyPointDamage(Target, 40.f, ShotDirection, Hit,
    InstigatorController, this, UMyDamageType::StaticClass());
UGameplayStatics::ApplyRadialDamage(this, 100.f, Hit.ImpactPoint, 500.f,
    UMyDamageType::StaticClass(), IgnoreActors, this, InstigatorController, false, ECC_Visibility);
UGameplayStatics::ApplyRadialDamageWithFalloff(this, 100.f, 10.f, Hit.ImpactPoint, 200.f, 600.f, 1.f,
    UMyDamageType::StaticClass(), IgnoreActors, this, InstigatorController, ECC_Visibility);
```

```cpp
// Receiving damage. AMyCharacter declares UPROPERTY(Replicated) float Health = 100.f; and FName LastHitBone;
// Declaration:
// virtual float TakeDamage(float DamageAmount, struct FDamageEvent const& DamageEvent, class AController* EventInstigator, AActor* DamageCauser) override;
#include "Engine/DamageEvents.h"

float AMyCharacter::TakeDamage(float DamageAmount, FDamageEvent const& DamageEvent, AController* EventInstigator, AActor* DamageCauser)
{
    const float Applied = Super::TakeDamage(DamageAmount, DamageEvent, EventInstigator, DamageCauser);

    if (!HasAuthority() || Applied <= 0.f)
    {
        return 0.f;
    }

    if (DamageEvent.IsOfType(FPointDamageEvent::ClassID))
    {
        const FPointDamageEvent& Point = static_cast<const FPointDamageEvent&>(DamageEvent);
        LastHitBone = Point.HitInfo.BoneName;
    }

    Health = FMath::Max(0.f, Health - Applied);
    return Applied;
}
```

`FDamageEvent` carries `TSubclassOf<UDamageType> DamageTypeClass` and dispatches through `virtual bool IsOfType(int32 InID) const`. `FPointDamageEvent` adds `float Damage`, `FVector_NetQuantizeNormal ShotDirection` and `FHitResult HitInfo`; `FRadialDamageEvent` adds `FRadialDamageParams Params`, `FVector Origin` and `TArray<FHitResult> ComponentHits`. `UDamageType` is a `UObject` used as a class, never instanced per hit — subclass it to carry `float DamageImpulse`, `float DestructibleImpulse`, `float DamageFalloff`, `uint32 bCausedByWorld:1`, `uint32 bScaleMomentumByMass:1`.

Dynamic multicast alternatives to overriding `TakeDamage`, all on `AActor`:

| Delegate | Handler signature |
|---|---|
| `OnTakeAnyDamage` | `(AActor* DamagedActor, float Damage, const UDamageType* DamageType, AController* InstigatedBy, AActor* DamageCauser)` |
| `OnTakePointDamage` | `(AActor* DamagedActor, float Damage, AController* InstigatedBy, FVector HitLocation, UPrimitiveComponent* FHitComponent, FName BoneName, FVector ShotFromDirection, const UDamageType* DamageType, AActor* DamageCauser)` |
| `OnTakeRadialDamage` | `(AActor* DamagedActor, float Damage, const UDamageType* DamageType, FVector Origin, const FHitResult& HitInfo, AController* InstigatedBy, AActor* DamageCauser)` |

Bind with `OnTakeAnyDamage.AddDynamic(this, &AMyCharacter::HandleAnyDamage);` and mark the handler `UFUNCTION()`. Gate damage with `void SetCanBeDamaged(bool bInCanBeDamaged)` / `bool CanBeDamaged() const`. Falling below `AWorldSettings::KillZ` calls `virtual void FellOutOfWorld(const class UDamageType& dmgType)` with `AWorldSettings::KillZDamageType`.

## GameInstance

One per process; survives every level load, including seamless travel. Override `virtual void Init()` (once at startup; `Super::Init()` initializes the game instance subsystems last, so use them only after that call), the protected `virtual void OnStart()` (once the instance is ready), and `virtual void Shutdown()` (on exit). `UGameInstance::GetWorld()` is `final` — do not override it. Game instance subsystems (`UGameInstanceSubsystem`) are the right home for cross-level services; the full subsystem table lives in `ue-cpp-foundations`.

| Need | Call |
|---|---|
| Game instance from an actor | `AActor::GetGameInstance<UMyGameInstance>()` or `UWorld::GetGameInstance<UMyGameInstance>()` |
| Game instance from any UObject with a world | `UGameplayStatics::GetGameInstance(const UObject* WorldContextObject)` |
| A game instance subsystem | `UGameInstance::GetSubsystem<UMyGameInstanceSubsystem>()` |
| First local player controller | `UWorld::GetFirstPlayerController()` or `UGameInstance::GetFirstLocalPlayerController(const UWorld* World = nullptr)` |
| A specific player controller | `UGameplayStatics::GetPlayerController(const UObject* WorldContextObject, int32 PlayerIndex)` |
| Game mode (server only) | `UWorld::GetAuthGameMode<AMyGameMode>()` or `UGameplayStatics::GetGameMode(const UObject* WorldContextObject)` |
| Game state | `UWorld::GetGameState<AMyGameState>()` or `UGameplayStatics::GetGameState(const UObject* WorldContextObject)` |

## Sessions

Two stacks ship in 5.8. Legacy `OnlineSubsystem` (modules `"OnlineSubsystem"`, `"OnlineSubsystemUtils"`; backends Null, Steam, EOS) is what most existing projects use. Online Services v2 (`Plugins/Online/OnlineServices`, module `"OnlineServicesInterface"`) is the modern path for new code, with `OnlineServicesOSSAdapter` bridging the two. Session code belongs on `UGameInstance` or a game instance subsystem, because it must outlive the level.

```cpp
// Legacy OnlineSubsystem - host. Needs OnlineSubsystem.h, OnlineSessionSettings.h and
// Interfaces/OnlineSessionInterface.h; CreateSessionHandle is an FDelegateHandle member.
IOnlineSubsystem* OSS = IOnlineSubsystem::Get();
IOnlineSessionPtr Sessions = OSS ? OSS->GetSessionInterface() : nullptr;
if (Sessions.IsValid())
{
    FOnlineSessionSettings Settings;
    Settings.NumPublicConnections = 4;
    Settings.bShouldAdvertise = true;
    Settings.bIsLANMatch = false;
    Settings.bUsesPresence = true;

    CreateSessionHandle = Sessions->AddOnCreateSessionCompleteDelegate_Handle(
        FOnCreateSessionCompleteDelegate::CreateUObject(this, &UMyGameInstance::HandleCreateSessionComplete));
    Sessions->CreateSession(0, NAME_GameSession, Settings);
}
```

Search with `FOnlineSessionSearch` and `Sessions->FindSessions(0, SessionSearch.ToSharedRef())`; join with `Sessions->JoinSession(0, NAME_GameSession, SessionSearch->SearchResults[Index])`; then resolve the address with `Sessions->GetResolvedConnectString(NAME_GameSession, ConnectInfo)` and call `PlayerController->ClientTravel(ConnectInfo, TRAVEL_Absolute)`. Handlers are `void HandleCreateSessionComplete(FName SessionName, bool bWasSuccessful)`, `void HandleFindSessionsComplete(bool bWasSuccessful)` and `void HandleJoinSessionComplete(FName SessionName, EOnJoinSessionCompleteResult::Type Result)`. Release every stored `FDelegateHandle` with the matching `Clear...Delegate_Handle`. Full search/join code: [references/framework-class-map.md](references/framework-class-map.md).

```cpp
// Online Services (v2). Needs Online/OnlineServices.h and Online/Sessions.h.
using namespace UE::Online;
TSharedPtr<IOnlineServices> Services = GetServices(EOnlineServices::Default);
ISessionsPtr SessionsV2 = Services.IsValid() ? Services->GetSessionsInterface() : nullptr;
```

v2 calls return `TOnlineAsyncOpHandle` / `TOnlineResult` instead of delegate handles; the sibling interfaces alongside `ISessions` are `IAuth`, `ILobbies`, `IAchievements`, `IPresence` and `ILeaderboards`.

## Travel

| Pattern | Call | Clients | GameMode / GameState |
|---|---|---|---|
| Server travel, non-seamless | `UWorld::ServerTravel(const FString& InURL, bool bAbsolute = false, bool bShouldSkipGameNotify = false)` | Disconnect and reconnect | Recreated |
| Server travel, seamless | Same call with `AGameModeBase::bUseSeamlessTravel = true` | Stay connected | Recreated |
| Move one client to another server | `APlayerController::ClientTravel(const FString& URL, ETravelType TravelType, bool bSeamless = false, FGuid MapPackageGuid = FGuid())` | That client only | n/a |
| Load a map locally | `UGameplayStatics::OpenLevel(const UObject* WorldContextObject, FName LevelName, bool bAbsolute = true, FString Options = FString(TEXT("")))` | n/a | Recreated |

From the GameMode, a listen-server map change is `GetWorld()->ServerTravel(TEXT("/Game/Maps/NewMap?listen"))`.

Seamless travel runs in two legs (current map → transition map → destination), and `virtual void GetSeamlessTravelActorList(bool bToTransition, TArray<AActor*>& ActorList)` is called for both. `UGameInstance`, `APlayerController` and `APlayerState` are carried across; `AGameModeBase`, `AGameStateBase`, pawns and level actors are not. `AGameModeBase::ProcessServerTravel(const FString& URL, bool bAbsolute = false)` is the override point for custom URL handling. Level streaming, World Partition and Data Layers belong to `ue-world-level-streaming`.

## Deprecated — do not use

| Do not emit | Use in 5.8 | Source |
|---|---|---|
| `DispatchPostLogin(NewPlayer)` | override `OnPostLogin(AController* NewPlayer)`; it runs at the end of `PostLogin` | `UE_DEPRECATED(5.6)` in `GameFramework/GameModeBase.h:329` |
| `virtual void Possess(APawn*) override` / `virtual void UnPossess() override` | `OnPossess(APawn* aPawn)` / `OnUnPossess()` | `virtual final` in `GameFramework/Controller.h:284,288` |
| `Pawn->IsControlled()` | `IsPawnControlled()` or `IsPlayerControlled()` | `UE_DEPRECATED(4.24)` in `GameFramework/Pawn.h:254` |
| `Pawn->GetMovementBase()` | `GetMovementBaseObject()` or `GetMovementBaseInterfaceData()` | `UE_DEPRECATED(5.8)` in `GameFramework/Pawn.h:59`, `GameFramework/Character.h:797` |
| `Character->SetBase(UPrimitiveComponent*, BoneName, bNotifyActor)` | `SetBase(FMovementBaseInterfaceData*, const FName, bool)` | `UE_DEPRECATED(5.8)` in `GameFramework/Character.h:515` |
| `Pawn->RemoteViewPitch` | `GetRemoteViewPitch()` / `RemoteViewPitch16` | `UE_DEPRECATED(5.6)` in `GameFramework/Pawn.h:134` |
| `Actor->NetUpdateFrequency = 10.f` | `SetNetUpdateFrequency(10.f)` / `GetNetUpdateFrequency()` | `UE_DEPRECATED(5.5)` in `GameFramework/Actor.h:903` |
| `Actor->MinNetUpdateFrequency` / `Actor->NetCullDistanceSquared` direct access | `Set/GetMinNetUpdateFrequency()`, `Set/GetNetCullDistanceSquared()` | `UE_DEPRECATED(5.5)` in `GameFramework/Actor.h:898,908` |
| `PlayerController::InputKey(...)` old overload | overload taking `FInputKeyEventArgs` | `UE_DEPRECATED(5.6)` in `GameFramework/PlayerController.h:1785` |
| `PlayerController::InputTouch(...)` old overload | overload taking `FTouchId` | `UE_DEPRECATED(5.8)` in `GameFramework/PlayerController.h:1794` |
| `PlayerController::ProcessTouchHitResult(FInputDeviceId, uint32, ...)` | overload taking `FTouchId` | `UE_DEPRECATED(5.8)` in `GameFramework/PlayerController.h:2005` |
| `SetDeprecatedInputYawScale()` and the Pitch/Roll pair | Enhanced Input `UInputModifierScalar` | `UE_DEPRECATED(5.0)` in `GameFramework/PlayerController.h:511` |
| `UCheatManager::DebugCameraControllerRef` | `FDebugCameraManager` | `UE_DEPRECATED(5.8)` in `GameFramework/CheatManager.h:107` |

## Common Mistakes

**GameMode dereferenced on a client:** `GetAuthGameMode()` returns `nullptr` everywhere except the server, so `GetWorld()->GetAuthGameMode<AMyGameMode>()->EndMatch()` crashes there. Write `if (AMyGameMode* GM = GetWorld()->GetAuthGameMode<AMyGameMode>()) { GM->EndMatch(); }` instead.

**`ReplicatedUsing` with no OnRep declaration:** UHT fails the build. Every `ReplicatedUsing = OnRep_X` needs `UFUNCTION() void OnRep_X();` in the same class body.

**Replicated property with no `DOREPLIFETIME`:** it compiles, runs, and silently never replicates. Add `GetLifetimeReplicatedProps` with `Super::` first, then one `DOREPLIFETIME` per property.

**Overriding `Possess` / `UnPossess`:** both are `virtual final` on `AController`. Override `OnPossess(APawn* aPawn)` and `OnUnPossess()`.

**Overriding a `BlueprintNativeEvent` by its bare name:** `ChoosePlayerStart`, `FindPlayerStart`, `SpawnDefaultPawnFor`, `HandleStartingNewPlayer`, `ReadyToStartMatch`, `ReadyToEndMatch`, `PlayerCanRestart` and `MustSpectate` all need the `_Implementation` suffix, and the matching `Super::` call carries that suffix too when you chain.

**HUD, camera or audio work on a dedicated server:** those objects do not exist there. Guard presentation code with `if (GetNetMode() != NM_DedicatedServer)`.

**Wrong class for the data:** a score every client can see belongs on `APlayerState`, not `APlayerController`; a match timer clients read belongs on a replicated `AGameStateBase` property, not `AGameModeBase`; data that outlives a level belongs on `UGameInstance`, not `AGameStateBase`; pawn input binding belongs in `APawn::SetupPlayerInputComponent`, not the Character constructor.

**`GetPawn()` before the server acknowledges possession:** use `AcknowledgedPawn` when you need the pawn the server has confirmed.

**PIE with multiple players:** each player gets its own PlayerController but they share one GameMode instance. Set PIE "Number of Players" to 2 or more before trusting any multiplayer path.

## Related Skills

- `ue-gameplay-abilities` — GAS attributes, gameplay effects and ability-driven damage, the alternative to
- `ue-networking-replication` — `DOREPLIFETIME` conditions, `COND_*`, RPC rules, push model, Iris (Beta in 5.8)
- `ue-character-movement` — `UCharacterMovementComponent`, movement modes, `PhysCustom`, prediction, Mover
- `ue-gameplay-cameras` — `APlayerCameraManager`, view targets and blends, camera modifiers, Gameplay
- `ue-input-system` — Enhanced Input mapping contexts, input actions, modifiers and triggers
- `ue-actor-component-architecture` — actor lifecycle, components, attachment, tick groups
- `ue-cpp-foundations` — `UCLASS`/`UPROPERTY`/`UFUNCTION` specifiers, `TObjectPtr`, the subsystem table
- `ue-ui-umg-slate` — UMG widgets and retained UI, the usual alternative to canvas HUD drawing
- `ue-serialization-savegames` — `USaveGame`, slots, what to persist off `UGameInstance`
- `ue-world-level-streaming` — level streaming, World Partition, Data Layers, streaming sources
- `ue-ai-navigation` — AI controllers, behavior trees, perception, EQS and navmesh queries
- `ue-audio-system` — UAudioComponent, MetaSounds, submixes, attenuation and concurrency
- Also relevant: `ue-blueprint-cpp-interop`, `ue-game-features`, `ue-mover`, `ue-physics-collision`, `ue-state-trees`, `ue-testing-debugging`
