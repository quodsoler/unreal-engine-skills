# Sequencer Runtime Patterns

Complete, compilable patterns for Unreal Engine 5.8 Sequencer. Every API here is verified against
`LevelSequenceActor.h`, `LevelSequencePlayer.h`, `LevelSequenceDirector.h`,
`MovieSceneSequencePlayer.h`, `MovieSceneSequencePlaybackSettings.h`, `MovieSceneObjectBindingID.h`,
`CineCameraActor.h`, `CineCameraComponent.h`, `CineCameraSettings.h`, `CameraRig_Rail.h`,
`CameraRig_Crane.h` and the `MovieRenderPipelineCore` public headers in UE 5.8.

Build.cs modules, the runtime-vs-editor boundary and the deprecation table live in the skill body.

---

## Pattern 1: Full-Screen Cutscene With Skip

Disables input, plays a sequence, restores the player camera and input whether the sequence
finishes or the player skips it.

### Header

```cpp
// MyCutsceneManager.h
#pragma once

#include "GameFramework/Actor.h"
#include "MyCutsceneManager.generated.h"

class ALevelSequenceActor;
class ULevelSequence;
class ULevelSequencePlayer;

UCLASS()
class MYGAME_API AMyCutsceneManager : public AActor
{
    GENERATED_BODY()

public:
    /** LevelSequence asset to play. */
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Cinematics")
    TObjectPtr<ULevelSequence> CutsceneAsset;

    /** Binding tag applied to the hero track in the Sequencer editor. */
    UPROPERTY(EditAnywhere, Category = "Cinematics")
    FName HeroBindingTag = FName("Hero");

    UFUNCTION(BlueprintCallable, Category = "Cinematics")
    void PlayCutscene(AActor* HeroActor);

    UFUNCTION(BlueprintCallable, Category = "Cinematics")
    void SkipCutscene();

protected:
    virtual void EndPlay(const EEndPlayReason::Type EndPlayReason) override;

private:
    UFUNCTION()
    void HandleCutsceneFinished();

    void RestorePlayerControl();

    UPROPERTY()
    TObjectPtr<ALevelSequenceActor> ActiveSequenceActor;

    UPROPERTY()
    TObjectPtr<ULevelSequencePlayer> ActivePlayer;
};
```

### Implementation

```cpp
// MyCutsceneManager.cpp
#include "MyCutsceneManager.h"

#include "Camera/PlayerCameraManager.h"
#include "GameFramework/PlayerController.h"
#include "LevelSequenceActor.h"
#include "LevelSequencePlayer.h"
#include "MovieSceneSequencePlaybackSettings.h"

void AMyCutsceneManager::PlayCutscene(AActor* HeroActor)
{
    if (!CutsceneAsset || ActivePlayer)
    {
        return;
    }

    FMovieSceneSequencePlaybackSettings Settings;
    Settings.bAutoPlay             = false;
    Settings.PlayRate              = 1.0f;
    Settings.LoopCount.Value       = 0;
    Settings.bDisableMovementInput = true;
    Settings.bDisableLookAtInput   = true;
    Settings.bHidePlayer           = true;
    Settings.bHideHud              = true;
    Settings.bPauseAtEnd           = false;
    // Skipping mid-sequence then restores every animated actor to its pre-sequence state.
    Settings.FinishCompletionStateOverride =
        EMovieSceneCompletionModeOverride::ForceRestoreState;

    ALevelSequenceActor* OutActor = nullptr;
    ULevelSequencePlayer* Player = ULevelSequencePlayer::CreateLevelSequencePlayer(
        this, CutsceneAsset, Settings, OutActor);

    if (!Player || !OutActor)
    {
        return;
    }

    ActiveSequenceActor = OutActor;
    ActivePlayer        = Player;

    // Bind before Play() so frame 0 evaluates against the gameplay actor.
    if (HeroActor && !HeroBindingTag.IsNone())
    {
        OutActor->SetBindingByTag(HeroBindingTag, TArray<AActor*>{ HeroActor },
                                  /*bAllowBindingsFromAsset=*/ false);
    }

    Player->OnFinished.AddDynamic(this, &AMyCutsceneManager::HandleCutsceneFinished);
    Player->Play();
}

void AMyCutsceneManager::SkipCutscene()
{
    if (!ActivePlayer)
    {
        return;
    }

    // Stop() fires OnStop, not OnFinished, so restore control here.
    ActivePlayer->SetCompletionModeOverride(EMovieSceneCompletionModeOverride::ForceRestoreState);
    ActivePlayer->Stop();

    ActivePlayer->OnFinished.RemoveDynamic(this, &AMyCutsceneManager::HandleCutsceneFinished);
    ActivePlayer        = nullptr;
    ActiveSequenceActor = nullptr;

    RestorePlayerControl();
}

void AMyCutsceneManager::HandleCutsceneFinished()
{
    if (ActivePlayer)
    {
        ActivePlayer->OnFinished.RemoveDynamic(this, &AMyCutsceneManager::HandleCutsceneFinished);
    }

    ActivePlayer        = nullptr;
    ActiveSequenceActor = nullptr;

    RestorePlayerControl();
}

void AMyCutsceneManager::EndPlay(const EEndPlayReason::Type EndPlayReason)
{
    if (ActivePlayer && ActivePlayer->IsPlaying())
    {
        ActivePlayer->Stop();
    }

    Super::EndPlay(EndPlayReason);
}

void AMyCutsceneManager::RestorePlayerControl()
{
    APlayerController* PC = GetWorld() ? GetWorld()->GetFirstPlayerController() : nullptr;
    if (!PC)
    {
        return;
    }

    if (APawn* PlayerPawn = PC->GetPawn())
    {
        PC->SetViewTargetWithBlend(PlayerPawn, 0.5f, VTBlend_Cubic);
    }

    // PlayerController.h:2085 — SetCinematicMode(bInCinematicMode, bHidePlayer, bAffectsHUD,
    //                                            bAffectsMovement, bAffectsTurning)
    PC->SetCinematicMode(false, /*bHidePlayer=*/ false, true, true, true);
    PC->SetIgnoreMoveInput(false);
    PC->SetIgnoreLookInput(false);
}
```

`SetViewTargetWithBlend` is declared in `PlayerController.h:1667`; blend functions are
`VTBlend_Linear`, `VTBlend_Cubic`, `VTBlend_EaseIn`, `VTBlend_EaseOut`, `VTBlend_EaseInOut`
(`Camera/PlayerCameraManager.h:29`).

---

## Pattern 2: In-Game Camera Beat Without Input Lockout

A scripted camera move during gameplay: the player keeps control, the HUD stays up, and audio
reacts to each camera cut.

```cpp
// MyEncounterDirector.h
#pragma once

#include "GameFramework/Actor.h"
#include "MyEncounterDirector.generated.h"

class ALevelSequenceActor;
class ULevelSequence;
class ULevelSequencePlayer;
class UCameraComponent;
class USoundBase;

UCLASS()
class MYGAME_API AMyEncounterDirector : public AActor
{
    GENERATED_BODY()

public:
    UFUNCTION(BlueprintCallable, Category = "Cinematics")
    void PlayBossArrival();

private:
    UFUNCTION()
    void HandleCameraCut(UCameraComponent* NewCamera);

    UFUNCTION()
    void HandleArrivalFinished();

    UPROPERTY(EditAnywhere, Category = "Cinematics")
    TObjectPtr<ULevelSequence> BossArrivalSequence;

    UPROPERTY(EditAnywhere, Category = "Cinematics")
    TObjectPtr<USoundBase> CameraCutStinger;

    UPROPERTY()
    TObjectPtr<ALevelSequenceActor> ArrivalSequenceActor;

    UPROPERTY()
    TObjectPtr<ULevelSequencePlayer> ArrivalPlayer;
};
```

```cpp
// MyEncounterDirector.cpp
#include "MyEncounterDirector.h"

#include "Camera/CameraComponent.h"
#include "GameFramework/PlayerController.h"
#include "Kismet/GameplayStatics.h"
#include "LevelSequenceActor.h"
#include "LevelSequencePlayer.h"
#include "MovieSceneSequencePlaybackSettings.h"

void AMyEncounterDirector::PlayBossArrival()
{
    if (!BossArrivalSequence)
    {
        return;
    }

    FMovieSceneSequencePlaybackSettings Settings;
    Settings.bAutoPlay             = false;
    Settings.PlayRate              = 1.0f;
    Settings.LoopCount.Value       = 0;
    Settings.bDisableMovementInput = false;   // player keeps control
    Settings.bDisableLookAtInput   = false;
    Settings.bHidePlayer           = false;
    Settings.bHideHud              = false;
    Settings.bDisableCameraCuts    = false;   // the sequence owns the camera for its duration
    Settings.bPauseAtEnd           = false;

    ALevelSequenceActor* OutActor = nullptr;
    ArrivalPlayer = ULevelSequencePlayer::CreateLevelSequencePlayer(
        this, BossArrivalSequence, Settings, OutActor);
    ArrivalSequenceActor = OutActor;

    if (!ArrivalPlayer)
    {
        return;
    }

    ArrivalPlayer->OnCameraCut.AddDynamic(this, &AMyEncounterDirector::HandleCameraCut);
    ArrivalPlayer->OnFinished.AddDynamic(this, &AMyEncounterDirector::HandleArrivalFinished);
    ArrivalPlayer->Play();
}

void AMyEncounterDirector::HandleCameraCut(UCameraComponent* NewCamera)
{
    if (CameraCutStinger)
    {
        UGameplayStatics::PlaySound2D(this, CameraCutStinger);
    }
}

void AMyEncounterDirector::HandleArrivalFinished()
{
    if (APlayerController* PC = GetWorld()->GetFirstPlayerController())
    {
        if (APawn* PlayerPawn = PC->GetPawn())
        {
            PC->SetViewTargetWithBlend(PlayerPawn, 1.0f, VTBlend_EaseInOut, 2.0f);
        }
    }

    ArrivalPlayer        = nullptr;
    ArrivalSequenceActor = nullptr;
}
```

To leave the gameplay camera untouched entirely, set `Settings.bDisableCameraCuts = true` (or call
`Player->SetDisableCameraCuts(true)`) and let the sequence animate only world actors.

---

## Pattern 3: Director-Driven Scripted Event

Two NPCs perform an authored interaction. The trigger actor binds them by tag; the timeline calls
back into a `ULevelSequenceDirector` subclass, which is the only legal home for an event endpoint.

### Editor setup

1. Create `LS_NPCHandshake` and add two possessable bindings, tagged `NPC_A` and `NPC_B`.
2. Add transform and skeletal animation tracks to each binding.
3. Add an Event Track on the `NPC_A` binding, and key it at the handshake frame.
4. Open the Director Blueprint from the Sequencer toolbar, reparent it to `UMyHandshakeDirector`,
   and pick `OnHandshakeContact` as the event endpoint.

### Director

```cpp
// MyHandshakeDirector.h
#pragma once

#include "LevelSequenceDirector.h"
#include "MyHandshakeDirector.generated.h"

class AActor;

UCLASS(Blueprintable)
class MYGAME_API UMyHandshakeDirector : public ULevelSequenceDirector
{
    GENERATED_BODY()

public:
    /**
     * Event endpoint. The track sits on an object binding, so the bound actor is passed in.
     * Endpoints take no parameters, or exactly one pass-by-value object/interface parameter.
     */
    UFUNCTION(BlueprintCallable, CallInEditor, Category = "Cinematics")
    void OnHandshakeContact(AActor* BoundActor);
};
```

```cpp
// MyHandshakeDirector.cpp
#include "MyHandshakeDirector.h"

#include "GameFramework/Actor.h"
#include "LevelSequencePlayer.h"

void UMyHandshakeDirector::OnHandshakeContact(AActor* BoundActor)
{
    if (!BoundActor)
    {
        return;
    }

    // ULevelSequenceDirector::Player is a UPROPERTY, valid while the sequence evaluates.
    const FQualifiedFrameTime EventTime = GetCurrentTime();
    UE_LOG(LogMyGame, Log, TEXT("Handshake contact on %s at frame %d"),
           *BoundActor->GetName(), EventTime.Time.FrameNumber.Value);
}
```

### Trigger

```cpp
// MyHandshakeTrigger.cpp
#include "MyHandshakeTrigger.h"

#include "LevelSequenceActor.h"
#include "LevelSequencePlayer.h"
#include "MovieSceneSequencePlaybackSettings.h"

void AMyHandshakeTrigger::TriggerHandshake(AActor* NpcA, AActor* NpcB)
{
    if (!HandshakeSequence)
    {
        return;
    }

    FMovieSceneSequencePlaybackSettings Settings;
    Settings.bAutoPlay             = false;
    Settings.LoopCount.Value       = 0;
    Settings.bDisableMovementInput = false;
    Settings.bHideHud              = false;
    Settings.FinishCompletionStateOverride =
        EMovieSceneCompletionModeOverride::ForceRestoreState;

    ALevelSequenceActor* OutActor = nullptr;
    HandshakePlayer = ULevelSequencePlayer::CreateLevelSequencePlayer(
        this, HandshakeSequence, Settings, OutActor);
    HandshakeSequenceActor = OutActor;

    if (!HandshakePlayer || !HandshakeSequenceActor)
    {
        return;
    }

    HandshakeSequenceActor->SetBindingByTag(FName("NPC_A"), TArray<AActor*>{ NpcA }, false);
    HandshakeSequenceActor->SetBindingByTag(FName("NPC_B"), TArray<AActor*>{ NpcB }, false);

    HandshakePlayer->OnFinished.AddDynamic(this, &AMyHandshakeTrigger::HandleHandshakeFinished);
    HandshakePlayer->Play();
}

void AMyHandshakeTrigger::HandleHandshakeFinished()
{
    HandshakePlayer        = nullptr;
    HandshakeSequenceActor = nullptr;
}
```

`AMyHandshakeTrigger` declares `HandshakeSequence`, `HandshakePlayer`, `HandshakeSequenceActor` and
the `UFUNCTION() void HandleHandshakeFinished()` in its header, following the shape of
`AMyCutsceneManager` above.

---

## Pattern 4: Looping Ambient Sequence

Drives ambient elements (light flicker, foliage sway) forever without touching the player.

```cpp
// MyAtmosphereManager.cpp
#include "MyAtmosphereManager.h"

#include "LevelSequenceActor.h"
#include "LevelSequencePlayer.h"
#include "MovieSceneSequencePlaybackSettings.h"

void AMyAtmosphereManager::BeginPlay()
{
    Super::BeginPlay();

    if (!AmbientSequence)
    {
        return;
    }

    FMovieSceneSequencePlaybackSettings Settings;
    Settings.bAutoPlay             = false;
    Settings.LoopCount.Value       = -1;     // infinite
    Settings.bRandomStartTime      = true;   // desynchronise multiple instances
    Settings.bDisableMovementInput = false;
    Settings.bDisableLookAtInput   = false;
    Settings.bHidePlayer           = false;
    Settings.bHideHud              = false;
    Settings.bDisableCameraCuts    = true;   // never steal the gameplay camera

    ALevelSequenceActor* OutActor = nullptr;
    AmbientPlayer = ULevelSequencePlayer::CreateLevelSequencePlayer(
        this, AmbientSequence, Settings, OutActor);
    AmbientSequenceActor = OutActor;

    if (AmbientPlayer)
    {
        AmbientPlayer->Play();
    }
}

void AMyAtmosphereManager::EndPlay(const EEndPlayReason::Type EndPlayReason)
{
    if (AmbientPlayer && AmbientPlayer->IsPlaying())
    {
        AmbientPlayer->Stop();
    }

    Super::EndPlay(EndPlayReason);
}
```

An ambient sequence that only ever loops can also be a placed `ALevelSequenceActor` with
`PlaybackSettings.bAutoPlay = true` and `PlaybackSettings.LoopCount.Value = -1`, with no C++ at all.

---

## Pattern 5: Cine Camera and Rig Setup

Spawns a cine camera on a rail, ready to be possessed by a sequence.

```cpp
// MyCameraRigBuilder.cpp
#include "MyCameraRigBuilder.h"

#include "CameraRig_Rail.h"
#include "CineCameraActor.h"
#include "CineCameraComponent.h"
#include "CineCameraSettings.h"
#include "Components/SplineComponent.h"

ACineCameraActor* AMyCameraRigBuilder::SpawnDollyCamera(ACameraRig_Rail* Rail, AActor* SubjectActor)
{
    if (!Rail)
    {
        return nullptr;
    }

    FActorSpawnParameters SpawnParams;
    SpawnParams.Owner = this;

    ACineCameraActor* Camera = GetWorld()->SpawnActor<ACineCameraActor>(
        ACineCameraActor::StaticClass(), Rail->GetActorTransform(), SpawnParams);
    if (!Camera)
    {
        return nullptr;
    }

    Camera->AttachToComponent(Rail->GetDefaultAttachComponent(),
                              FAttachmentTransformRules::SnapToTargetIncludingScale);

    UCineCameraComponent* CineComp = Camera->GetCineCameraComponent();

    // 35mm full-frame sensor.
    FCameraFilmbackSettings Filmback;
    Filmback.SensorWidth  = 36.0f;
    Filmback.SensorHeight = 24.0f;
    CineComp->SetFilmback(Filmback);

    // Fixed 35mm prime.
    FCameraLensSettings Lens;
    Lens.MinFocalLength       = 35.0f;
    Lens.MaxFocalLength       = 35.0f;
    Lens.MinFStop             = 1.4f;
    Lens.MaxFStop             = 16.0f;
    Lens.MinimumFocusDistance = 30.0f;
    Lens.DiaphragmBladeCount  = 7;
    CineComp->SetLensSettings(Lens);

    // Track focus on the subject rather than keying focus distance by hand.
    FCameraFocusSettings Focus;
    Focus.FocusMethod                              = ECameraFocusMethod::Tracking;
    Focus.TrackingFocusSettings.ActorToTrack       = SubjectActor;
    Focus.TrackingFocusSettings.RelativeOffset     = FVector(0.f, 0.f, 80.f);
    Focus.bSmoothFocusChanges                      = true;
    Focus.FocusSmoothingInterpSpeed                = 8.0f;
    CineComp->SetFocusSettings(Focus);

    CineComp->SetCurrentFocalLength(35.0f);
    CineComp->SetCurrentAperture(2.0f);

    Camera->LookatTrackingSettings.bEnableLookAtTracking     = true;
    Camera->LookatTrackingSettings.ActorToTrack              = SubjectActor;
    Camera->LookatTrackingSettings.RelativeOffset            = FVector(0.f, 0.f, 80.f);
    Camera->LookatTrackingSettings.LookAtTrackingInterpSpeed = 4.0f;
    Camera->LookatTrackingSettings.bAllowRoll                = false;

    // Key CurrentPositionOnRail in Sequencer to dolly; it is UPROPERTY(Interp).
    Rail->CurrentPositionOnRail  = 0.0f;
    Rail->bLockOrientationToRail = false;

    return Camera;
}
```

`ACameraRig_Crane` works the same way: attach to `GetDefaultAttachComponent()` and key
`CraneArmLength`, `CranePitch` and `CraneYaw`, with `bLockMountPitch` / `bLockMountYaw` to keep the
camera level while the arm swings.

---

## Pattern 6: Partial Playback and Segment Chaining

Plays frames 0-60 as an intro, then loops frames 61-119.

```cpp
// MySegmentController.cpp
#include "MySegmentController.h"

#include "LevelSequenceActor.h"
#include "LevelSequencePlayer.h"
#include "MovieSceneSequencePlaybackSettings.h"

void AMySegmentController::PlayIntroSegment()
{
    if (!FullSequence)
    {
        return;
    }

    FMovieSceneSequencePlaybackSettings Settings;
    Settings.bAutoPlay       = false;
    Settings.LoopCount.Value = 0;

    ALevelSequenceActor* OutActor = nullptr;
    SegmentPlayer = ULevelSequencePlayer::CreateLevelSequencePlayer(
        this, FullSequence, Settings, OutActor);
    SegmentSequenceActor = OutActor;

    if (!SegmentPlayer)
    {
        return;
    }

    // SetFrameRange(StartFrame, Duration) — display-rate frames.
    SegmentPlayer->SetFrameRange(0, 60);
    SegmentPlayer->OnFinished.AddDynamic(this, &AMySegmentController::HandleIntroFinished);
    SegmentPlayer->Play();
}

void AMySegmentController::HandleIntroFinished()
{
    if (!SegmentPlayer)
    {
        return;
    }

    SegmentPlayer->OnFinished.RemoveDynamic(this, &AMySegmentController::HandleIntroFinished);

    SegmentPlayer->SetFrameRange(61, 59);   // frames 61..119
    SegmentPlayer->PlayLooping(-1);
}
```

---

## Pattern 7: Reading the Active Shot

A root sequence made of sub-sequences reports which shot is playing through
`ULevelSequencePlayer::TakeFrameSnapshot`.

```cpp
// MyShotHud.cpp
#include "MyShotHud.h"

#include "DrawDebugHelpers.h"
#include "LevelSequenceActor.h"
#include "LevelSequencePlayer.h"

void AMyShotHud::Tick(float DeltaTime)
{
    Super::Tick(DeltaTime);

    ULevelSequencePlayer* Player = WatchedSequenceActor
        ? WatchedSequenceActor->GetSequencePlayer()
        : nullptr;

    if (!Player || !Player->IsPlaying())
    {
        return;
    }

    FLevelSequencePlayerSnapshot Snapshot;
    Player->TakeFrameSnapshot(Snapshot);

    const FString ShotInfo = FString::Printf(
        TEXT("%s / %s | root frame %d"),
        *Snapshot.RootName,
        *Snapshot.CurrentShotName,
        Snapshot.RootTime.Time.FrameNumber.Value);

    DrawDebugString(GetWorld(), FVector::ZeroVector, ShotInfo, nullptr, FColor::White, 0.0f, true);
}
```

`FLevelSequencePlayerSnapshot` (`LevelSequencePlayer.h:33`) fields: `RootName`, `RootTime`,
`SourceTime`, `CurrentShotName`, `CurrentShotLocalTime`, `CurrentShotSourceTime`, `SourceTimecode`,
`CameraComponent` (a `TSoftObjectPtr<UCameraComponent>`) and `ActiveShot` (a `ULevelSequence*`).
`Player->GetActiveCameraComponent()` returns the live camera directly.

---

## Pattern 8: Replicated Cutscene

The server owns playback time; clients follow. Camera cuts are still applied locally on each client.

```cpp
// MyNetworkCutsceneGameMode.cpp
#include "MyNetworkCutsceneGameMode.h"

#include "GameFramework/PlayerController.h"
#include "LevelSequenceActor.h"
#include "LevelSequencePlayer.h"

void AMyNetworkCutsceneGameMode::ServerPlayCutscene(ULevelSequence* Sequence)
{
    if (!HasAuthority() || !Sequence)
    {
        return;
    }

    FActorSpawnParameters SpawnParams;
    SpawnParams.Owner = this;

    // AReplicatedLevelSequenceActor is always net-relevant, so distant clients still receive it.
    ALevelSequenceActor* SeqActor = GetWorld()->SpawnActor<AReplicatedLevelSequenceActor>(
        AReplicatedLevelSequenceActor::StaticClass(), FTransform::Identity, SpawnParams);
    if (!SeqActor)
    {
        return;
    }

    // Configure before the player starts. The actor copied PlaybackSettings into its player during
    // SpawnActor (LevelSequenceActor.cpp:231), so push the edited copy; the player's PlaybackSettings
    // replicate (MovieSceneSequencePlayer.cpp:231) and the actor replicates LevelSequenceAsset.
    SeqActor->SetReplicatePlayback(true);
    SeqActor->PlaybackSettings.bAutoPlay             = false;
    SeqActor->PlaybackSettings.LoopCount.Value       = 0;
    SeqActor->PlaybackSettings.bDisableMovementInput = true;
    SeqActor->PlaybackSettings.bHidePlayer           = false;
    SeqActor->GetSequencePlayer()->SetPlaybackSettings(SeqActor->PlaybackSettings);
    SeqActor->SetSequence(Sequence);

    NetworkSequenceActor = SeqActor;

    if (ULevelSequencePlayer* Player = SeqActor->GetSequencePlayer())
    {
        Player->OnFinished.AddDynamic(
            this, &AMyNetworkCutsceneGameMode::HandleNetworkCutsceneFinished);
        Player->Play();   // the server broadcasts playback state to clients
    }
}

void AMyNetworkCutsceneGameMode::HandleNetworkCutsceneFinished()
{
    MulticastRestoreInput();
}

void AMyNetworkCutsceneGameMode::MulticastRestoreInput_Implementation()
{
    APlayerController* PC = GetWorld()->GetFirstPlayerController();
    if (!PC)
    {
        return;
    }

    PC->SetIgnoreMoveInput(false);
    PC->SetIgnoreLookInput(false);

    if (APawn* PlayerPawn = PC->GetPawn())
    {
        PC->SetViewTargetWithBlend(PlayerPawn, 0.5f, VTBlend_Cubic);
    }
}
```

The header declares `UFUNCTION(NetMulticast, Reliable) void MulticastRestoreInput();` and
`UFUNCTION() void HandleNetworkCutsceneFinished();`. Replication is owned by
`ue-networking-replication`; only the Sequencer-specific parts appear here. Internally the player
replicates through its own reliable multicast RPCs and re-syncs clients that join late, so
`Play()` must be called on the server, never on a client.

---

## Pattern 9: Runtime Render With Movie Render Graph

Renders one shot from a packaged build or a `-game` session using the runtime queue subsystem.
The editor-only `UMoviePipelineQueueSubsystem` must not appear in runtime code.

```cpp
// MyRenderDirector.cpp
#include "MyRenderDirector.h"

#include "Engine/Engine.h"
#include "Graph/MovieGraphConfig.h"
#include "MoviePipelineQueue.h"
#include "MoviePipelineQueueEngineSubsystem.h"

void AMyRenderDirector::RenderShot()
{
    UMoviePipelineQueueEngineSubsystem* RenderSubsystem =
        GEngine->GetEngineSubsystem<UMoviePipelineQueueEngineSubsystem>();

    if (!RenderSubsystem || RenderSubsystem->IsRendering() || !ShotSequence || !RenderGraphAsset)
    {
        return;
    }

    // Optional: a progress widget class and whether to keep rendering the player viewport.
    RenderSubsystem->SetConfiguration({}, /*bRenderPlayerViewport=*/ false);

    // AllocateJob resets the queue, so exactly one job is rendered.
    UMoviePipelineExecutorJob* Job = RenderSubsystem->AllocateJob(ShotSequence);
    Job->JobName = TEXT("Shot_0010");
    Job->Map     = FSoftObjectPath(GetWorld());
    Job->SetGraphPreset(RenderGraphAsset);

    RenderSubsystem->OnRenderFinished.AddDynamic(this, &AMyRenderDirector::HandleRenderFinished);
    RenderSubsystem->RenderJob(Job);
}

void AMyRenderDirector::HandleRenderFinished(FMoviePipelineOutputData Results)
{
    UE_LOG(LogMyGame, Log, TEXT("Render finished, success: %s"),
           Results.bSuccess ? TEXT("true") : TEXT("false"));
}
```

`AMyRenderDirector` declares `UFUNCTION() void HandleRenderFinished(FMoviePipelineOutputData Results);`
in its header — `AddDynamic` requires it. `OnRenderFinished` is an `FMoviePipelineWorkFinished`
dynamic multicast delegate (`MoviePipelineBase.h:9`) and only fires for the `RenderJob` convenience
path. For a batch, build the queue yourself:

```cpp
#include "MoviePipelineInProcessExecutor.h"

UMoviePipelineQueue* Queue = RenderSubsystem->GetQueue();
UMoviePipelineExecutorJob* Job =
    Queue->AllocateNewJob(UMoviePipelineExecutorJob::StaticClass());
Job->SetGraphPreset(RenderGraphAsset);
RenderSubsystem->RenderQueueWithExecutor(UMoviePipelineInProcessExecutor::StaticClass());
```

### Render layers

`UMovieGraphRenderLayerSubsystem` is a `UWorldSubsystem`; the modifier functions live on
`UMovieGraphRenderLayer`. Its header instantiates Slate row widgets, so add `"Slate"` and `"SlateCore"` to
Build.cs or the include fails to link (`Graph/MovieGraphRenderLayerSubsystem.h:250`).

```cpp
#include "Graph/MovieGraphRenderLayerSubsystem.h"

UMovieGraphRenderLayerSubsystem* LayerSubsystem =
    UMovieGraphRenderLayerSubsystem::GetFromWorld(GetWorld());
if (LayerSubsystem)
{
    LayerSubsystem->AddRenderLayer(CharacterLayer);   // UMovieGraphRenderLayer*
    CharacterLayer->AddLayerModifier(HideEnvironmentModifier);   // UMovieGraphModifierBase*
    LayerSubsystem->SetActiveRenderLayerByName(FName("Characters"));
}
```

`CharacterLayer->GetLayerModifiers()` returns the current modifiers (Blueprint only from a game module:
it is not exported from the `MinimalAPI` class, `Graph/MovieGraphRenderLayerSubsystem.h:1216`) and
`RemoveLayerModifier(Modifier)` drops one. The pre-5.7 `AddModifier` / `GetModifiers` /
`RemoveModifier` spellings are deprecated.

### Legacy Movie Render Queue configuration

Use this shape only for jobs that are not graph-configured.

Requires `MovieRenderPipelineRenderPasses` in Build.cs alongside `MovieRenderPipelineCore`, plus `Imath` and
`UEOpenExr` — `MoviePipelineEXROutput.h` includes their headers (`MoviePipelineEXROutput.h:11-18`).

```cpp
#include "MoviePipelineDeferredPasses.h"     // MovieRenderPipelineRenderPasses
#include "MoviePipelineEXROutput.h"          // MovieRenderPipelineRenderPasses
#include "MoviePipelineOutputSetting.h"
#include "MoviePipelinePrimaryConfig.h"
#include "MoviePipelineQueue.h"

UMoviePipelinePrimaryConfig* Config = NewObject<UMoviePipelinePrimaryConfig>(Job);
Job->SetConfiguration(Config);

UMoviePipelineOutputSetting* Output = Cast<UMoviePipelineOutputSetting>(
    Config->FindOrAddSettingByClass(UMoviePipelineOutputSetting::StaticClass()));
Output->OutputDirectory.Path  = TEXT("{project_dir}/Saved/MovieRenders/");
Output->FileNameFormat        = TEXT("{sequence_name}.{frame_number}");
Output->OutputResolution      = FIntPoint(1920, 1080);
Output->bUseCustomFrameRate   = true;
Output->OutputFrameRate       = FFrameRate(24, 1);
Output->ZeroPadFrameNumbers   = 4;

// Render pass and container:
Config->FindOrAddSettingByClass(UMoviePipelineDeferredPassBase::StaticClass());
Config->FindOrAddSettingByClass(UMoviePipelineImageSequenceOutput_EXR::StaticClass());
```

`UMoviePipelineImageSequenceOutput_PNG`, `_JPG` and `_BMP` are the other containers, and
`UMoviePipelineDeferredPass_Unlit`, `_LightingOnly`, `_ReflectionsOnly`, `_DetailLighting` and
`_PathTracer` are the extra deferred passes. A job is either graph-configured or legacy-configured;
check `Job->IsUsingGraphConfiguration()` before calling `Job->GetConfiguration()`.

---

## API Quick Reference

### ULevelSequencePlayer

```
static ULevelSequencePlayer* CreateLevelSequencePlayer(
    UObject* WorldContextObject, ULevelSequence* LevelSequence,
    FMovieSceneSequencePlaybackSettings Settings, ALevelSequenceActor*& OutActor)

UCameraComponent* GetActiveCameraComponent() const
void TakeFrameSnapshot(FLevelSequencePlayerSnapshot& OutSnapshot) const
void EnableCinematicMode(bool bEnable)
FOnLevelSequencePlayerCameraCutEvent OnCameraCut
```

### UMovieSceneSequencePlayer

```
void Play()
void PlayReverse()
void PlayLooping(int32 NumLoops = -1)
void ChangePlaybackDirection()
void Pause()
void Scrub()
void Stop()
void StopAtCurrentTime()
void GoToEndAndStop()
void SetPlayRate(float PlayRate)
float GetPlayRate() const
void SetFrameRange(int32 StartFrame, int32 Duration, float SubFrames = 0.f)
void SetTimeRange(float StartTime, float Duration)
void SetFrameRate(FFrameRate FrameRate)
void SetPlaybackPosition(FMovieSceneSequencePlaybackParams PlaybackParams)
void PlayTo(FMovieSceneSequencePlaybackParams PlaybackParams,
            FMovieSceneSequencePlayToParams PlayToParams)
void RestoreState()
void SetCompletionModeOverride(EMovieSceneCompletionModeOverride CompletionModeOverride)
void SetWeight(double InWeight)          // recommended with PlaybackSettings.bDynamicWeighting = true
void RemoveWeight()
void SetHideHud(bool HideHud)
void SetDisableCameraCuts(bool bInDisableCameraCuts)
bool IsPlaying() const
bool IsPaused() const
bool IsReversed() const
FQualifiedFrameTime GetCurrentTime() const
FQualifiedFrameTime GetDuration() const
FQualifiedFrameTime GetStartTime() const
FQualifiedFrameTime GetEndTime() const
int32 GetFrameDuration() const
FFrameRate GetFrameRate() const
UMovieSceneSequence* GetSequence() const
FString GetSequenceName(bool bAddClientInfo = false) const
TArray<UObject*> GetBoundObjects(FMovieSceneObjectBindingID ObjectBinding)
TArray<FMovieSceneObjectBindingID> GetObjectBindings(UObject* InObject)
void RequestInvalidateBinding(FMovieSceneObjectBindingID ObjectBinding)
```

Delegates: `OnPlay`, `OnPlayReverse`, `OnStop`, `OnPause`, `OnFinished`
(`FOnMovieSceneSequencePlayerEvent`, dynamic, no parameters) and `OnNativeFinished`
(`FOnMovieSceneSequencePlayerNativeEvent`, native).

### FMovieSceneSequencePlaybackSettings

```
uint32 bAutoPlay : 1
FMovieSceneSequenceLoopCount LoopCount      // .Value: 0 = once, -1 = infinite
FMovieSceneSequenceTickInterval TickInterval
float PlayRate
float StartTime                             // seconds offset, "Start Offset" in the UI
uint32 bRandomStartTime : 1
uint32 bDisableMovementInput : 1
uint32 bDisableLookAtInput : 1
uint32 bHidePlayer : 1
uint32 bHideHud : 1
uint32 bDisableCameraCuts : 1
EMovieSceneCompletionModeOverride FinishCompletionStateOverride
uint32 bPauseAtEnd : 1
uint32 bInheritTickIntervalFromOwner : 1
uint32 bDynamicWeighting : 1
```

`EMovieSceneCompletionModeOverride`: `None`, `ForceKeepState`, `ForceRestoreState`.

### ALevelSequenceActor

```
FMovieSceneSequencePlaybackSettings PlaybackSettings
TObjectPtr<ULevelSequence> LevelSequenceAsset
TObjectPtr<UMovieSceneBindingOverrides> BindingOverrides
TObjectPtr<ULevelSequenceBurnInOptions> BurnInOptions
TObjectPtr<UObject> DefaultInstanceData
uint8 bOverrideInstanceData : 1
uint8 bReplicatePlayback : 1
FLevelSequenceCameraSettings CameraSettings

ULevelSequence* GetSequence() const
void SetSequence(ULevelSequence* InSequence)
ULevelSequencePlayer* GetSequencePlayer() const
void SetReplicatePlayback(bool ReplicatePlayback)
void HideBurnin()
void ShowBurnin()

void SetBinding(FMovieSceneObjectBindingID Binding, const TArray<AActor*>& Actors,
                bool bAllowBindingsFromAsset = false)
void SetBindingByTag(FName BindingTag, const TArray<AActor*>& Actors,
                     bool bAllowBindingsFromAsset = false)
void AddBinding(FMovieSceneObjectBindingID Binding, AActor* Actor,
                bool bAllowBindingsFromAsset = false)
void AddBindingByTag(FName BindingTag, AActor* Actor, bool bAllowBindingsFromAsset = false)
void RemoveBinding(FMovieSceneObjectBindingID Binding, AActor* Actor)
void RemoveBindingByTag(FName Tag, AActor* Actor)
void ResetBinding(FMovieSceneObjectBindingID Binding)
void ResetBindings()
FMovieSceneObjectBindingID FindNamedBinding(FName Tag) const
const TArray<FMovieSceneObjectBindingID>& FindNamedBindings(FName Tag) const
```

`AReplicatedLevelSequenceActor` subclasses it and is always net-relevant.

### ULevelSequenceDirector

```
UMovieSceneSequence* GetSequence()
FQualifiedFrameTime GetCurrentTime() const
FQualifiedFrameTime GetRootSequenceTime() const
UMovieSceneClock* GetSequenceCustomClock() const
UMovieSceneClock* GetRootSequenceCustomClock() const
TArray<UObject*> GetBoundObjects(FMovieSceneObjectBindingID ObjectBinding)
UObject* GetBoundObject(FMovieSceneObjectBindingID ObjectBinding)
TArray<AActor*> GetBoundActors(FMovieSceneObjectBindingID ObjectBinding)
AActor* GetBoundActor(FMovieSceneObjectBindingID ObjectBinding)
void OnCreated()                          // BlueprintImplementableEvent
TObjectPtr<ULevelSequencePlayer> Player   // UPROPERTY(BlueprintReadOnly)
```

### UCineCameraComponent

```
FCameraFilmbackSettings Filmback       // SensorWidth, SensorHeight (mm); SensorAspectRatio read-only
FCameraLensSettings LensSettings       // MinFocalLength, MaxFocalLength, MinFStop, MaxFStop,
                                       // MinimumFocusDistance, SqueezeFactor, DiaphragmBladeCount
FCameraFocusSettings FocusSettings     // FocusMethod, ManualFocusDistance (cm),
                                       // TrackingFocusSettings, bSmoothFocusChanges,
                                       // FocusSmoothingInterpSpeed
float CurrentFocalLength               // UPROPERTY(Interp)
float CurrentAperture                  // UPROPERTY(Interp)
float CurrentFocusDistance             // read-only
float CurrentHorizontalFOV             // read-only

void SetFilmback(const FCameraFilmbackSettings& NewFilmback)
void SetLensSettings(const FCameraLensSettings& NewLensSettings)
void SetFocusSettings(const FCameraFocusSettings& NewFocusSettings)
void SetCropSettings(const FPlateCropSettings& NewCropSettings)
void SetCurrentFocalLength(float InFocalLength)
void SetCurrentAperture(const float NewCurrentAperture)
void SetCustomNearClippingPlane(const float NewCustomNearClippingPlane)
float GetHorizontalFieldOfView() const
float GetVerticalFieldOfView() const
FString GetFilmbackPresetName() const
void SetFilmbackPresetByName(const FString& InPresetName)
void SetLensPresetByName(const FString& InPresetName)
```

`ECameraFocusMethod`: `DoNotOverride`, `Manual`, `Tracking`, `Disable`.

### Camera rigs

```
// ACameraRig_Rail
float CurrentPositionOnRail        // 0..1, UPROPERTY(Interp)
bool bLockOrientationToRail        // UPROPERTY(Interp)
USplineComponent* GetRailSplineComponent()
USceneComponent* GetDefaultAttachComponent() const

// ACameraRig_Crane
float CranePitch                   // degrees, UPROPERTY(Interp)
float CraneYaw                     // degrees, UPROPERTY(Interp)
float CraneArmLength               // cm, UPROPERTY(Interp)
bool bLockMountPitch
bool bLockMountYaw
USceneComponent* GetDefaultAttachComponent() const
```
