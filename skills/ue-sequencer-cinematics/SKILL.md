---
name: ue-sequencer-cinematics
description: "Use when playing Level Sequences from C++, building cutscenes or in-game camera moments, binding actors to sequence tracks at runtime, wiring Sequencer event tracks, or rendering with Movie Render Graph. Also use when the user mentions 'Sequencer', 'LevelSequence', 'ALevelSequenceActor', 'ULevelSequencePlayer', 'CreateLevelSequencePlayer', 'cutscene', 'camera cut', 'event track', 'ULevelSequenceDirector', 'SetBindingByTag', 'spawnable', 'possessable', 'CineCamera', 'Movie Render Queue', or 'MRQ'. For animation tracks, see ue-animation-system; for gameplay cameras and shakes, see ue-gameplay-cameras; for sequence audio, see ue-audio-system."
metadata:
  version: "2.0.0"
  engine: "5.8"
---

# UE Sequencer and Cinematics

Target engine: **UE 5.8**. APIs below are verified against the 5.8 headers; older forms are listed under "Deprecated — do not use".

Sequencer spans three runtime modules plus a plugin. `MovieScene` holds the data model and the player base class, `MovieSceneTracks` holds the concrete track and section types, `LevelSequence` holds the asset, actor, player and director, and `CinematicCamera` holds the cine camera and camera rigs. Offline rendering lives in the **Movie Render Queue** plugin (`MovieRenderPipelineCore` for runtime, `MovieRenderPipelineEditor` for the editor queue UI); it is disabled by default and must be enabled in the `.uproject`.

## Context

Read `.agents/ue-project-context.md` if it exists (module names, conventions, enabled plugins, GAS/networking setup). Do not stop if it is missing.

Identify the area from the request and the codebase. Ask only when two plausible readings would produce different code.

| Request is about… | Go to |
|---|---|
| Which modules to add to Build.cs | [Build.cs Modules](#buildcs-modules) |
| What C++ can do at runtime vs editor-only | [Runtime vs Editor](#runtime-vs-editor) |
| Starting, stopping, seeking, looping a sequence | [Playing a Level Sequence](#playing-a-level-sequence) |
| Pointing sequence tracks at gameplay actors | [Binding Actors at Runtime](#binding-actors-at-runtime) |
| Calling C++ from a key on the timeline | [Sequencer Event Tracks](#sequencer-event-tracks) |
| Tracks, sections, sub-sequences, frame math | [MovieScene Data Model](#moviescene-data-model) |
| Cine camera setup, focus, rails and cranes | [Cine Cameras and Camera Rigs](#cine-cameras-and-camera-rigs) |
| Rendering frames or video to disk | [Movie Render Graph](#movie-render-graph) |
| Full worked cutscene, multiplayer and rig examples | [Sequencer runtime patterns](references/sequencer-patterns.md) |

## Build.cs Modules

```csharp
PublicDependencyModuleNames.AddRange(new string[]
{
    "LevelSequence",    // ULevelSequence, ALevelSequenceActor, ULevelSequencePlayer, ULevelSequenceDirector
    "MovieScene",       // UMovieScene, UMovieSceneSequencePlayer, FMovieSceneObjectBindingID
    "MovieSceneTracks", // UMovieSceneEventTrack, UMovieSceneCameraCutTrack, property tracks
    "CinematicCamera",  // ACineCameraActor, UCineCameraComponent, ACameraRig_Rail, ACameraRig_Crane
});

// Offline rendering only. Requires the MovieRenderPipeline plugin enabled in the .uproject.
PrivateDependencyModuleNames.Add("MovieRenderPipelineCore");
```

## Runtime vs Editor

Authoring a sequence is an editor operation. Shipping game code plays and re-binds sequences; it does not build them.

| Task | API | Availability |
|---|---|---|
| Play, stop, seek, set play rate | `UMovieSceneSequencePlayer` | Runtime |
| Override which actors a binding resolves to | `ALevelSequenceActor::SetBinding*` | Runtime |
| Read tracks, bindings, frame rates | `UMovieScene` getters, `UMovieSceneSequence::GetMovieScene` | Runtime |
| Call C++ from the timeline | `ULevelSequenceDirector` subclass | Runtime |
| Queue and run a render | `UMoviePipelineQueueEngineSubsystem` (`UEngineSubsystem`) | Runtime |
| Add actors, cameras, spawnables to a sequence | `ULevelSequenceEditorSubsystem` (`UEditorSubsystem`) | Editor only |
| Scripted track/section creation and keying | `UMovieSceneSequenceExtensions` (Runtime module of the Beta SequencerScripting plugin) | Editor scripting — mutates assets |
| Register a custom track editor | `ISequencerModule::RegisterTrackEditor` | Editor only |
| Drive the Movie Render Queue window | `UMoviePipelineQueueSubsystem` (`UEditorSubsystem`) | Editor only |

`UMovieScene::AddTrack` and `UMovieSceneSubTrack::AddSequence` do compile into runtime builds, but they mutate the sequence asset. Call them only from editor tooling or commandlets — a cooked `ULevelSequence` must not be edited in a shipped game.

## Playing a Level Sequence

### From a placed ALevelSequenceActor

```cpp
#include "LevelSequenceActor.h"

// LevelSequenceActor.h:153 — UFUNCTION(BlueprintGetter) ULevelSequencePlayer* GetSequencePlayer() const;
if (ULevelSequencePlayer* Player = SeqActor->GetSequencePlayer())
{
    Player->Play();
}
```

Useful `ALevelSequenceActor` members: `PlaybackSettings`, `LevelSequenceAsset`, `BindingOverrides`, `DefaultInstanceData`, `bOverrideInstanceData`, `bReplicatePlayback`, `GetSequence()`, `SetSequence()`, `SetReplicatePlayback()`. Auto-play is `PlaybackSettings.bAutoPlay`; the actor's own `bAutoPlay` field no longer exists. `AReplicatedLevelSequenceActor` is a subclass that is always net-relevant.

### Spawning a player at runtime

```cpp
#include "LevelSequencePlayer.h"
#include "MovieSceneSequencePlaybackSettings.h"

// LevelSequencePlayer.h:106
// static ULevelSequencePlayer* CreateLevelSequencePlayer(
//     UObject* WorldContextObject, ULevelSequence* LevelSequence,
//     FMovieSceneSequencePlaybackSettings Settings, ALevelSequenceActor*& OutActor);

FMovieSceneSequencePlaybackSettings Settings;
Settings.bAutoPlay             = false;
Settings.PlayRate              = 1.0f;
Settings.StartTime             = 0.0f;   // seconds into the playback range
Settings.LoopCount.Value       = 0;      // 0 = once, -1 = infinite, N = N extra loops
Settings.bDisableMovementInput = true;
Settings.bDisableLookAtInput   = true;
Settings.bHidePlayer           = false;
Settings.bHideHud              = true;
Settings.bDisableCameraCuts    = false;
Settings.bPauseAtEnd           = false;
Settings.bDynamicWeighting     = false;  // set true (or enable it on the asset) before using SetWeight() — MovieSceneSequencePlayer.h:278
Settings.FinishCompletionStateOverride = EMovieSceneCompletionModeOverride::ForceRestoreState;

ALevelSequenceActor* OutActor = nullptr;
ULevelSequencePlayer* Player = ULevelSequencePlayer::CreateLevelSequencePlayer(
    this, CutsceneAsset, Settings, OutActor);

ActiveSequenceActor = OutActor;   // UPROPERTY member — otherwise both are collected
ActivePlayer        = Player;     // UPROPERTY member
// Remaining fields: bRandomStartTime, TickInterval, bInheritTickIntervalFromOwner
```

### Playback control

```cpp
Player->Play();
Player->PlayReverse();
Player->PlayLooping(-1);          // -1 = infinite
Player->ChangePlaybackDirection();
Player->Pause();
Player->Scrub();
Player->Stop();                   // resets the cursor and fires OnStop
Player->StopAtCurrentTime();
Player->GoToEndAndStop();
Player->SetPlayRate(0.5f);
Player->SetFrameRange(0, 60);     // (StartFrame, Duration) in display-rate frames
Player->SetTimeRange(0.0f, 2.5f); // (StartTime, Duration) in seconds

// Seek without firing the events in between
Player->SetPlaybackPosition(
    FMovieSceneSequencePlaybackParams(FFrameTime(120), EUpdatePositionMethod::Jump));

// Advance to a frame, firing every event along the way
Player->PlayTo(
    FMovieSceneSequencePlaybackParams(FFrameTime(240), EUpdatePositionMethod::Play),
    FMovieSceneSequencePlayToParams());

// Seconds instead of frames: FMovieSceneSequencePlaybackParams(2.5f, EUpdatePositionMethod::Jump)

const bool bPlaying           = Player->IsPlaying();
const FQualifiedFrameTime Now = Player->GetCurrentTime();
const FQualifiedFrameTime Dur = Player->GetDuration();
const FFrameRate Rate         = Player->GetFrameRate();
```

`EUpdatePositionMethod` is `Play`, `Jump` or `Scrub`. `FMovieSceneSequencePlaybackParams` has constructors taking `FFrameTime`, `float` seconds, an `FString` marked-frame name, or an `FTimecode`.

### Delegates

```cpp
// UPROPERTY(BlueprintAssignable) FOnMovieSceneSequencePlayerEvent — dynamic, no parameters
Player->OnPlay.AddDynamic(this, &AMyCutsceneManager::HandleSequencePlay);
Player->OnStop.AddDynamic(this, &AMyCutsceneManager::HandleSequenceStop);
Player->OnFinished.AddDynamic(this, &AMyCutsceneManager::HandleSequenceFinished);

Player->OnPause.AddDynamic(this, &AMyCutsceneManager::HandleSequencePause);
Player->OnNativeFinished.BindUObject(this, &AMyCutsceneManager::HandleNativeFinished); // native

// LevelSequencePlayer.h:27 — DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(
//     FOnLevelSequencePlayerCameraCutEvent, UCameraComponent*, CameraComponent)
Player->OnCameraCut.AddDynamic(this, &AMyCutsceneManager::HandleCameraCut);
```

`OnFinished` fires only when playback ends naturally (a natural end fires `OnStop` too); an explicit `Stop()` fires only `OnStop`. Bind before `Play()`.

## Binding Actors at Runtime

A **possessable** resolves to an actor that already exists in the level. A **spawnable** is created and destroyed by the sequence itself. Possessables outlive the sequence; spawnables do not. Runtime overrides apply to possessables.

Tag the object binding in the Sequencer editor (right-click the binding, then Tags), and override by tag from C++ — tags survive re-binding, GUIDs do not.

```cpp
#include "LevelSequenceActor.h"
#include "MovieSceneObjectBindingID.h"

// LevelSequenceActor.h:178-253
SeqActor->SetBindingByTag(FName("Hero"), TArray<AActor*>{ HeroActor },
                          /*bAllowBindingsFromAsset=*/ false);
SeqActor->AddBindingByTag(FName("Crowd"), ExtraActor, /*bAllowBindingsFromAsset=*/ true);
SeqActor->RemoveBindingByTag(FName("Crowd"), ExtraActor);

// GUID form — expose the ID so a designer can pick the binding in the details panel:
// UPROPERTY(EditAnywhere, Category = "Cinematics") FMovieSceneObjectBindingID HeroBindingID;
SeqActor->SetBinding(HeroBindingID, TArray<AActor*>{ HeroActor }, false);
SeqActor->AddBinding(HeroBindingID, HeroActor, false);
SeqActor->RemoveBinding(HeroBindingID, HeroActor);
SeqActor->ResetBinding(HeroBindingID);   // back to whatever the asset says
SeqActor->ResetBindings();               // clear every override

const FMovieSceneObjectBindingID Found = SeqActor->FindNamedBinding(FName("Hero"));
const TArray<FMovieSceneObjectBindingID>& All = SeqActor->FindNamedBindings(FName("Crowd"));

// What is actually bound right now
TArray<UObject*> Bound = Player->GetBoundObjects(HeroBindingID);
```

`FMovieSceneObjectBindingID` keeps its guid and sequence id private. Read them with `GetGuid()`, `GetRelativeSequenceID()` and `IsFixedBinding()`; move an ID across a sub-sequence hierarchy with `ResolveToFixed()`, then `ConvertToRelative()` on the resulting `UE::MovieScene::FFixedObjectBindingID` (`ConvertToRelative` is not a member of `FMovieSceneObjectBindingID`, `MovieSceneObjectBindingID.h:113-160,341`). Never assign the fields directly.

### Relocating a whole sequence

`ALevelSequenceActor` implements `IMovieScenePlaybackClient::GetInstanceData`. Set `bOverrideInstanceData` and supply a `UDefaultLevelSequenceInstanceData` to offset every absolute transform section, so one authored cutscene can play anywhere in the world.

```cpp
#include "DefaultLevelSequenceInstanceData.h"

UDefaultLevelSequenceInstanceData* Data =
    NewObject<UDefaultLevelSequenceInstanceData>(SeqActor);
Data->TransformOriginActor = StageMarkerActor;   // or leave null and set TransformOrigin
SeqActor->DefaultInstanceData  = Data;
SeqActor->bOverrideInstanceData = true;
```

## Sequencer Event Tracks

An event key does **not** call a loose global function. `FMovieSceneEvent` stores a compiled `UFunction` pointer, and the evaluation system calls it on the sequence's **director instance** — an object of the class in `ULevelSequence::DirectorClass`, which derives from `ULevelSequenceDirector`. The `UFUNCTION` must therefore be a member of a `ULevelSequenceDirector` subclass or of its Director Blueprint.

Rules the engine enforces:

- `MovieSceneEvent.h:70` — the endpoint takes **no parameters, or exactly one pass-by-value object/interface parameter**, and returns nothing.
- When the event track sits under an object binding, the bound object arrives as that single parameter.
- On a root track, a parameterless endpoint runs once on the director; an endpoint with the object parameter runs once per global event context (`MovieSceneEventSystems.cpp:206-213`). `ULevelSequencePlayer::GetEventContexts` returns the persistent level's `ALevelScriptActor` plus one per streaming level — not the `UWorld`.
- In a non-game world (the Sequencer preview), the event is skipped unless the `UFUNCTION` carries `CallInEditor`.

```cpp
// MyCinematicDirector.h
#pragma once

#include "LevelSequenceDirector.h"
#include "MovieSceneObjectBindingID.h"
#include "MyCinematicDirector.generated.h"

class AActor;

UCLASS(Blueprintable)
class MYGAME_API UMyCinematicDirector : public ULevelSequenceDirector
{
    GENERATED_BODY()

public:
    /** Event endpoint with no parameters. */
    UFUNCTION(BlueprintCallable, CallInEditor, Category = "Cinematics")
    void OnExplosionCue();

    /** Event endpoint on an object-binding track: receives the bound actor. */
    UFUNCTION(BlueprintCallable, CallInEditor, Category = "Cinematics")
    void OnActorCue(AActor* BoundActor);

    /** Assign in the Director Blueprint's class defaults using the Get Sequence Binding node. */
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Cinematics")
    FMovieSceneObjectBindingID HeroBinding;
};
```

```cpp
// MyCinematicDirector.cpp
#include "MyCinematicDirector.h"
#include "LevelSequencePlayer.h"
#include "GameFramework/Actor.h"

void UMyCinematicDirector::OnExplosionCue()
{
    // GetBoundObjects and Player are both declared on ULevelSequenceDirector.
    for (UObject* Object : GetBoundObjects(HeroBinding))
    {
        if (AActor* Hero = Cast<AActor>(Object))
        {
            Hero->SetActorHiddenInGame(true);
        }
    }

    if (Player)
    {
        Player->SetPlayRate(0.25f);
    }
}

void UMyCinematicDirector::OnActorCue(AActor* BoundActor)
{
    if (BoundActor)
    {
        BoundActor->SetActorTickEnabled(false);
    }
}
```

To attach it: open the Level Sequence, open its Director Blueprint from the Sequencer toolbar, and reparent that Blueprint to `UMyCinematicDirector` (Class Settings, Parent Class). Sequencer compiles the Blueprint into `ULevelSequence::DirectorClass`; one director instance is created per sequence during evaluation, and `ULevelSequenceDirector::Player` points at the owning player. `ULevelSequenceDirector` also exposes `GetBoundObject`, `GetBoundActor`, `GetBoundActors`, `GetSequence`, `GetCurrentTime`, `GetRootSequenceTime` and a `BlueprintImplementableEvent` `OnCreated`.

The track classes involved are `UMovieSceneEventTrack` (with `bFireEventsWhenForwards`), `UMovieSceneEventTriggerSection` for one-shot keys and `UMovieSceneEventRepeaterSection` for a key that fires on every pass. Seeking with `EUpdatePositionMethod::Jump` skips every event in between; use `PlayTo` when the events must fire.

## MovieScene Data Model

```cpp
#include "MovieScene.h"
#include "MovieSceneSpawnable.h"
#include "MovieScenePossessable.h"

UMovieScene* MS = SeqActor->GetSequence()->GetMovieScene();

const FFrameRate DisplayRate     = MS->GetDisplayRate();
const FFrameRate TickResolution  = MS->GetTickResolution();
const TRange<FFrameNumber> Range = MS->GetPlaybackRange();

for (int32 Index = 0; Index < MS->GetPossessableCount(); ++Index)
{
    const FMovieScenePossessable& Possessable = MS->GetPossessable(Index);
    const FGuid Guid = Possessable.GetGuid();       // also GetName, GetPossessedObjectClass
}
for (int32 Index = 0; Index < MS->GetSpawnableCount(); ++Index)
{
    const FMovieSceneSpawnable& Spawnable = MS->GetSpawnable(Index);   // GetGuid, GetName
}

const TArray<UMovieSceneTrack*>& RootTracks = MS->GetTracks();
if (const FMovieSceneBinding* Binding = MS->FindBinding(SomeGuid))
{
    for (UMovieSceneTrack* Track : Binding->GetTracks()) { /* Cast to a concrete type */ }
}
```

| Track class (`MovieSceneTracks`) | Animates |
|---|---|
| `UMovieScene3DTransformTrack` | Actor and component transforms |
| `UMovieSceneSkeletalAnimationTrack` | Skeletal mesh animation |
| `UMovieSceneCameraCutTrack` | Which camera the viewport uses |
| `UMovieSceneEventTrack` | Director function calls |
| `UMovieSceneAudioTrack` | Sound assets |
| `UMovieSceneFadeTrack` | Screen fade |
| `UMovieScenePropertyTrack` | Base class for property animation |

`UMovieScenePropertyTrack` subclasses drive a named `UPROPERTY` marked `Interp`: `UMovieSceneFloatTrack` produces `UMovieSceneFloatSection`, which stores an `FMovieSceneFloatChannel`; `UMovieSceneBoolTrack` and `UMovieSceneColorTrack` follow the same shape. `UMovieSceneSubTrack` and `UMovieSceneSubSection` live in `MovieScene`, not `MovieSceneTracks`.

```cpp
#include "Tracks/MovieSceneSubTrack.h"
#include "Sections/MovieSceneSubSection.h"

UMovieSceneSubTrack* SubTrack = RootMovieScene->AddTrack<UMovieSceneSubTrack>();
const FFrameNumber Start = (2.0 * RootMovieScene->GetTickResolution()).FloorToFrame();
const int32 Duration = (5.0 * RootMovieScene->GetTickResolution()).FloorToFrame().Value;

UMovieSceneSubSection* SubSection = SubTrack->AddSequence(ChildSequence, Start, Duration);
SubSection->Parameters.TimeScale.Set(1.0);   // FMovieSceneTimeWarpVariant — Set(), not assignment
SubSection->Parameters.bCanLoop = false;
SubSection->Parameters.StartFrameOffset = FFrameNumber(0);
```

Sub-section offsets and durations are in tick-resolution frames, not display-rate frames. `UE::MovieScene::DiscreteSize`, `DiscreteInclusiveLower` and `DiscreteExclusiveUpper` in `MovieSceneTimeHelpers.h` turn a `TRange<FFrameNumber>` into frame counts.

## Cine Cameras and Camera Rigs

This section covers cameras a sequence drives. Gameplay cameras, camera modifiers, camera shakes and view-target blending belong to `ue-gameplay-cameras`.

```cpp
#include "CineCameraActor.h"
#include "CineCameraComponent.h"
#include "CineCameraSettings.h"

ACineCameraActor* Cam = GetWorld()->SpawnActor<ACineCameraActor>(SpawnLocation, SpawnRotation);
UCineCameraComponent* CineComp = Cam->GetCineCameraComponent();

FCameraFilmbackSettings Filmback;          // SensorWidth / SensorHeight in mm
Filmback.SensorWidth  = 36.0f;
Filmback.SensorHeight = 24.0f;
CineComp->SetFilmback(Filmback);           // SensorAspectRatio is derived, not settable

FCameraLensSettings Lens;                  // mm and f-stops
Lens.MinFocalLength       = 35.0f;
Lens.MaxFocalLength       = 35.0f;
Lens.MinFStop             = 1.4f;
Lens.MaxFStop             = 16.0f;
Lens.MinimumFocusDistance = 30.0f;
Lens.DiaphragmBladeCount  = 7;
CineComp->SetLensSettings(Lens);

FCameraFocusSettings Focus;
Focus.FocusMethod               = ECameraFocusMethod::Manual;  // DoNotOverride|Manual|Tracking|Disable
Focus.ManualFocusDistance       = 500.0f;                      // cm
Focus.bSmoothFocusChanges       = true;
Focus.FocusSmoothingInterpSpeed = 8.0f;
CineComp->SetFocusSettings(Focus);

CineComp->SetCurrentFocalLength(35.0f);    // mm; UPROPERTY(Interp) so Sequencer can key it
CineComp->SetCurrentAperture(2.0f);        // f-stop; also Interp

Cam->LookatTrackingSettings.bEnableLookAtTracking     = true;
Cam->LookatTrackingSettings.ActorToTrack              = TargetActor;  // TSoftObjectPtr<AActor>
Cam->LookatTrackingSettings.RelativeOffset            = FVector(0.f, 0.f, 80.f);
Cam->LookatTrackingSettings.LookAtTrackingInterpSpeed = 4.0f;
```

`CurrentFocusDistance` and `CurrentHorizontalFOV` are read-only; query the field of view with `GetHorizontalFieldOfView()` and `GetVerticalFieldOfView()`. Presets go through `SetFilmbackPresetByName`, `SetLensPresetByName` and `SetCropPresetByName`.

Attach a camera to a rig and key the rig's `Interp` properties in Sequencer instead of hand-animating the camera transform:

```cpp
#include "CameraRig_Rail.h"
#include "CameraRig_Crane.h"

Rail->CurrentPositionOnRail  = 0.35f;   // 0..1 along the spline
Rail->bLockOrientationToRail = true;
USplineComponent* RailSpline = Rail->GetRailSplineComponent();

Crane->CraneArmLength  = 400.0f;        // cm
Crane->CranePitch      = 12.0f;         // degrees
Crane->CraneYaw        = -45.0f;
Crane->bLockMountPitch = true;          // also bLockMountYaw
```

Both rigs override `GetDefaultAttachComponent()`, so attaching the camera as a child puts it on the correct mount.

## Movie Render Graph

Movie Render Graph is production-ready in 5.8 and is the path to prefer for new work. A `UMovieGraphConfig` is a node graph of `UMovieGraphNode` subclasses (`UMovieGraphSettingNode` for settings, `UMovieGraphRenderPassNode` for passes, `UMovieGraphGlobalOutputSettingNode` for resolution and output paths); `UMovieGraphPipeline` executes it through `Initialize(UMoviePipelineExecutorJob*, const FMovieGraphInitConfig&)`.

Runtime rendering goes through `UMoviePipelineQueueEngineSubsystem`, a `UEngineSubsystem` in `MovieRenderPipelineCore`. The editor-facing `UMoviePipelineQueueSubsystem` is a `UEditorSubsystem` in `MovieRenderPipelineEditor` and **cannot be referenced from runtime code** — doing so breaks a packaged build.

```cpp
#include "MoviePipelineQueueEngineSubsystem.h"
#include "MoviePipelineQueue.h"
#include "Graph/MovieGraphConfig.h"

UMoviePipelineQueueEngineSubsystem* RenderSubsystem =
    GEngine->GetEngineSubsystem<UMoviePipelineQueueEngineSubsystem>();
if (!RenderSubsystem || RenderSubsystem->IsRendering())
{
    return;
}

UMoviePipelineExecutorJob* Job = RenderSubsystem->AllocateJob(ShotSequence);
Job->JobName = TEXT("Shot_0010");
Job->Map     = FSoftObjectPath(GetWorld());
Job->SetGraphPreset(RenderGraphAsset);          // UMovieGraphConfig*

RenderSubsystem->OnRenderFinished.AddDynamic(this, &AMyRenderDirector::HandleRenderFinished);
RenderSubsystem->RenderJob(Job);
```

`AllocateJob` clears the queue, so it renders exactly one job; for a batch call `GetQueue()->AllocateNewJob(UMoviePipelineExecutorJob::StaticClass())` per job and then `RenderQueueWithExecutor(UMoviePipelineInProcessExecutor::StaticClass())`. Custom executors subclass `UMoviePipelineExecutorBase` (override `Execute_Implementation` and `IsRendering_Implementation`) or `UMoviePipelineLinearExecutorBase`, which already walks the queue and leaves `Start(const UMoviePipelineExecutorJob*)` to you.

Render layers are a world feature: `UMovieGraphRenderLayerSubsystem` is a `UWorldSubsystem` reached with `UMovieGraphRenderLayerSubsystem::GetFromWorld(World)`, holding `UMovieGraphRenderLayer` objects. Add modifiers with `UMovieGraphRenderLayer::AddLayerModifier(UMovieGraphModifierBase*)`, read them with `GetLayerModifiers()` (Blueprint only — not exported from the `MinimalAPI` class, `Graph/MovieGraphRenderLayerSubsystem.h:1216`), drop them with `RemoveLayerModifier()`, and switch layers with `SetActiveRenderLayerByName` or `ClearActiveRenderLayer`.

**Legacy Movie Render Queue** — still shipped and supported, but not the default for new graphs. `UMoviePipeline` is the executing object, configured by a `UMoviePipelinePrimaryConfig` holding `UMoviePipelineSetting` objects, reached with `FindOrAddSettingByClass` / `FindSettingByClass`. `UMoviePipelineOutputSetting` lives in `MovieRenderPipelineCore` (`OutputDirectory`, `FileNameFormat`, `OutputResolution`, `bUseCustomFrameRate`, `OutputFrameRate`, `ZeroPadFrameNumbers`); the passes and containers (`UMoviePipelineDeferredPassBase`, `UMoviePipelineImageSequenceOutput_PNG`, `UMoviePipelineImageSequenceOutput_EXR`) live in `MovieRenderPipelineRenderPasses`, which must be added to Build.cs separately. A job is either graph-configured (`SetGraphPreset`, `IsUsingGraphConfiguration()`) or legacy-configured (`SetConfiguration`, `IsUsingBasicConfiguration()`) — never both.

`DaySequence` (Experimental in 5.8) builds on MovieScene for time-of-day; it is a separate asset type and the APIs above do not transfer to it unchanged.

## Deprecated — do not use

| Do not emit | Use in 5.8 | Source |
|---|---|---|
| `SeqActor->SequencePlayer` | `SeqActor->GetSequencePlayer()` | `UE_DEPRECATED(5.4)` in `LevelSequenceActor.h:86` |
| `SeqActor->bAutoPlay` | `SeqActor->PlaybackSettings.bAutoPlay` | `bAutoPlay_DEPRECATED` in `LevelSequenceActor.h:109` |
| non-const `MovieScene->GetBindings()` | const `GetBindings()` | `UE_DEPRECATED(5.7)` in `MovieScene.h:782` |
| `FMovieSceneBinding::GetName` / `SetName` | `FMovieScenePossessable::GetName`, `FMovieSceneSpawnable::GetName` | `UE_DEPRECATED(5.7)` in `MovieSceneBinding.h:89` |
| `MovieScene->GetSoloNodes()` / `GetMuteNodes()` | `FindDecoration` | `UE_DEPRECATED(5.7)` in `MovieScene.h:978` |
| `MovieScene->OnChannelChanged()` | `OnChannelChangedWithTime()` | `UE_DEPRECATED(5.8)` in `MovieScene.h:993` |
| `RenderLayer->AddModifier()` / `GetModifiers()` / `RemoveModifier()` | `AddLayerModifier()` / `GetLayerModifiers()` / `RemoveLayerModifier()` | `UE_DEPRECATED(5.7)` in `Graph/MovieGraphRenderLayerSubsystem.h:1204-1218` |
| `ConditionGroup->Evaluate()` | `EvaluateActorsAndComponents()` | `UE_DEPRECATED(5.6)` in `Graph/MovieGraphRenderLayerSubsystem.h:817` |
| `GetEffectiveOutputResolution()` | `GetOverscannedResolution()` | `UE_DEPRECATED(5.6)` in `MoviePipelineBlueprintLibrary.h:154` |
| `GameModeOverride` | `SoftGameModeOverride` | `UE_DEPRECATED(5.5)` in `MoviePipelineGameOverrideSetting.h:77` |
| `RendererName` member | `GetRendererName()` | `UE_DEPRECATED(5.7)` in `Graph/Nodes/MovieGraphBurnInNode.h:43` |
| `Section->Parameters.TimeScale = 1.0` | `Parameters.TimeScale.Set(1.0)` | explicit ctor, `Variants/MovieSceneTimeWarpVariant.h:66` |

## Common Mistakes

**`UFUNCTION` on a free function for an event track:** a `UFUNCTION` is only legal inside a `UCLASS` or `USTRUCT` body, and Sequencer calls event endpoints on the director instance. Put the endpoint on a `ULevelSequenceDirector` subclass, with no parameters or one object parameter.

**Endpoint with the wrong signature:** `void Foo(FName Tag, int32 Index)` will never bind. The engine accepts no parameters, or one pass-by-value object/interface parameter, and no return value.

**Nothing fires in the Sequencer preview:** non-game worlds skip endpoints whose `UFUNCTION` lacks `CallInEditor`.

**Binding after `Play()`:** frame 0 has already evaluated against the asset binding. Call `SetBindingByTag` or `SetBinding` on the `ALevelSequenceActor` before `Play()`.

**Losing the spawned actor:** `CreateLevelSequencePlayer` hands back a raw `ALevelSequenceActor*`. Store it and the player in `UPROPERTY()` members or they are garbage collected mid-cutscene.

**Camera never returns to the player:** the camera cut track restores the previous view target only when its section resolves to Restore State (Level Sequences default to it, `DefaultCompletionMode=RestoreState` in `BaseEngine.ini`; restore in `MovieSceneCameraCutGameHandler.cpp:115-124`), and cinematic mode is undone on stop (`LevelSequencePlayer.cpp:177`). With a Keep State section, `ForceKeepState` or `bPauseAtEnd`, handle `OnFinished` and call `PC->SetViewTargetWithBlend(PC->GetPawn(), 0.5f, VTBlend_Cubic)` yourself.

**Skipping leaves actors mid-pose:** set `Settings.FinishCompletionStateOverride = EMovieSceneCompletionModeOverride::ForceRestoreState`, or call `SetCompletionModeOverride`, before `Stop()`.

**Touching `UMoviePipelineQueueSubsystem` from game code:** it is a `UEditorSubsystem` in `MovieRenderPipelineEditor`. Runtime renders use `UMoviePipelineQueueEngineSubsystem` from `MovieRenderPipelineCore`.

**Mixing graph and legacy config on one job:** `SetGraphPreset` and `SetConfiguration` are mutually exclusive. Check `IsUsingGraphConfiguration()` before assuming `GetConfiguration()` is valid.

**Frame-rate confusion:** `SetFrameRange` uses display-rate frames, while `UMovieSceneSubSection` offsets and `UMovieScene::GetPlaybackRange` use tick-resolution frames. Convert with `GetDisplayRate()` and `GetTickResolution()`.

**Assuming camera cuts replicate:** cuts are evaluated per client. Use `SetReplicatePlayback(true)`, or `AReplicatedLevelSequenceActor`, so the server drives playback time and each client applies its own cut.

## Related Skills

- `ue-animation-system` — skeletal animation assets, montages and anim notifies that sequence animation tracks play
- `ue-gameplay-cameras` — gameplay cameras, camera modifiers, camera shakes and view-target blending outside Sequencer
- `ue-audio-system` — the sound assets, submixes and attenuation behind sequence audio tracks
- `ue-niagara-effects` — particle systems a cinematic triggers or that a sequence keys
- `ue-materials-rendering` — post process, Lumen and material setup that render passes capture
- `ue-actor-component-architecture` — spawning and owning the actors a sequence possesses
- `ue-world-level-streaming` — streaming levels in before a cutscene and Data Layer state during a render
