---
name: ue-audio-system
description: "Use when playing, mixing or debugging sound in Unreal Engine — one-shot SFX, looping ambience, music, dialogue or procedural audio. Also use when the user mentions 'UAudioComponent', 'PlaySoundAtLocation', 'SpawnSoundAttached', 'SoundCue', 'MetaSound', 'UMetaSoundSource', 'SetFloatParameter', 'sound attenuation', 'spatialization', 'occlusion', 'sound concurrency', 'submix', 'audio bus', 'sound mix', 'SetSubmixOutputVolume', 'reverb', 'Quartz', 'audio modulation' or 'subtitles'. For VFX timing, see ue-niagara-effects; for anim-notify sounds, see ue-animation-system; for component lifetime and attachment, see ue-actor-component-architecture."
metadata:
  version: "2.0.0"
  engine: "5.8"
---

# UE Audio System

Target engine: **UE 5.8**. APIs below are verified against the 5.8 headers; older forms are listed under "Deprecated — do not use".

Covers sound assets, playback from C++, attenuation, concurrency, sound classes and mixes, submix/bus routing, MetaSounds, modulation, Quartz and analysis. Engine-side code lives in `Engine/Classes/Sound` and `Components/AudioComponent.h` (module `Engine`), the mixer in `AudioMixer`, and the parameter interface in `AudioExtensions`. Plugin modules: `MetasoundEngine` / `MetasoundFrontend` (MetaSounds), `AudioModulation`, `AudioSynesthesia` (Beta in 5.8), `SubtitlesAndClosedCaptions` (Beta in 5.8).

```csharp
// MyGame.Build.cs
PublicDependencyModuleNames.AddRange(new string[] { "Engine", "AudioMixer", "AudioExtensions" });
PrivateDependencyModuleNames.AddRange(new string[] { "MetasoundEngine", "MetasoundFrontend", "AudioModulation" });
```

## Context

Read `.agents/ue-project-context.md` if it exists (module names, conventions, enabled plugins, GAS/networking setup). Do not stop if it is missing.

Identify the area from the request and the codebase. Ask only when two plausible readings would produce different code.

| Request is about… | Go to |
|---|---|
| Which asset type to author (wave, cue, MetaSound), streaming | [Sound Assets](#sound-assets) |
| Firing a sound from gameplay code | [Playing Sounds](#playing-sounds) |
| Fades, pause, volume/pitch, finish callbacks, per-instance sends | [UAudioComponent Control](#uaudiocomponent-control) |
| Falloff, 3D panning, HRTF, occlusion, reverb send, focus | [Attenuation and Spatialization](#attenuation-and-spatialization) |
| Too many voices, footsteps/gunshots stacking, voice stealing | [Concurrency](#concurrency) |
| Volume sliders by category, ducking, context mixes | [Sound Classes and Sound Mixes](#sound-classes-and-sound-mixes) |
| Routing groups, master volume, audio buses, sidechain sources | [Submixes, Buses and Sends](#submixes-buses-and-sends) |
| Reverb, EQ, compression on a group of sounds | [Submix Effects](#submix-effects) |
| Procedural/adaptive audio, runtime parameters, graph building | [MetaSounds](#metasounds) |
| Continuous parameter control from gameplay curves/LFOs | [Modulation](#modulation) |
| Beat-synced stingers, tempo, musical quantization | [Quartz Music Sync](#quartz-music-sync) |
| Reacting to audio (FFT, envelope, offline beat maps) | [Analysis](#analysis) |
| Captions, dialogue text | [Subtitles](#subtitles) |
| Mobile voice budgets, dedicated servers, backgrounding, VR | [Platform Notes](#platform-notes) |
| Full worked systems (music manager, ambience, weapon audio) | [audio setup patterns](references/audio-setup-patterns.md) |

## Sound Assets

```
USoundBase                    // abstract base (Sound/SoundBase.h)
  ├── USoundWave              // imported PCM/compressed asset (Sound/SoundWave.h)
  │     └── USoundSourceBus   // sonifies a UAudioBus (Sound/SoundSourceBus.h)
  ├── USoundCue               // node graph (Sound/SoundCue.h)
  └── UMetaSoundSource        // procedural graph, derives USoundWaveProcedural
```

Key `USoundBase` fields set on every asset: `SoundClassObject`, `AttenuationSettings`, `ConcurrencySet` (or `ConcurrencyOverrides` with `bOverrideConcurrency`), `Priority` (higher survives voice culling), `SoundSubmixObject`, `SoundSubmixSends`, `BusSends` / `PreEffectBusSends` / `PostAttenuationBusSends`, `VirtualizationMode`.

`USoundCue` nodes: `USoundNodeRandom`, `USoundNodeModulator`, `USoundNodeMixer`, `USoundNodeAttenuation`, `USoundNodeLooping`, `USoundNodeDelay`, `USoundNodeDistanceCrossFade`, `USoundNodeConcatenator`, `USoundNodeSwitch`, `USoundNodeWavePlayer`. `USoundWave::LoadingBehavior` (`ESoundWaveLoadingBehavior`, `Sound/SoundWaveLoadingBehavior.h`): `Inherited`, `RetainOnLoad`, `PrimeOnLoad`, `LoadOnDemand`, `ForceInline`. Use `ForceInline` for short SFX, `LoadOnDemand` for long music so minutes of audio never sit resident. Read it with `GetLoadingBehavior(bCheckSoundClasses)`; override at runtime with `OverrideLoadingBehavior(ESoundWaveLoadingBehavior::LoadOnDemand)`.

| Criterion | `USoundCue` | `UMetaSoundSource` |
|---|---|---|
| Runtime parameters | Wave params only | Typed inputs (float, bool, int32, string, object, trigger) |
| Procedural DSP | No | Yes (oscillators, noise, filters) |
| Built at runtime from C++ | No | Yes — `UMetaSoundBuilderSubsystem` |
| Use for | Randomized pre-authored one-shots | Adaptive music, state-driven and synthesized SFX |

## Playing Sounds

All functions are `UGameplayStatics` statics in `Kismet/GameplayStatics.h`. Never call them on a dedicated server.

```cpp
#include "Kismet/GameplayStatics.h"   // + Components/SkeletalMeshComponent.h for GetMesh() below
// PlaySound2D(WorldContext, Sound, Volume=1, Pitch=1, StartTime=0,
//             Concurrency=nullptr, OwningActor=nullptr, bIsUISound=true) — UI, music; no handle
UGameplayStatics::PlaySound2D(this, MenuSelectSound, 1.0f, 1.0f, 0.0f, nullptr, nullptr, true);

// PlaySoundAtLocation(WorldContext, Sound, Location, Rotation, Volume=1, Pitch=1, StartTime=0,
//                     Attenuation=nullptr, Concurrency=nullptr, OwningActor=nullptr, InitialParams=nullptr)
UGameplayStatics::PlaySoundAtLocation(this, GunShotSound, GetActorLocation(), FRotator::ZeroRotator,
    1.0f, 1.0f, 0.0f, WeaponAttenuation, FireConcurrency, this);

// SpawnSoundAtLocation(WorldContext, Sound, Location, Rotation=ZeroRotator, Volume=1, Pitch=1,
//                      StartTime=0, Attenuation=nullptr, Concurrency=nullptr, bAutoDestroy=true)
UAudioComponent* Boom = UGameplayStatics::SpawnSoundAtLocation(this, ExplosionSound, Location,
    FRotator::ZeroRotator, 1.0f, 1.0f, 0.0f, ExplosionAttenuation, nullptr, true);

// SpawnSoundAttached(Sound, AttachToComponent, AttachPointName=NAME_None, Location=FVector(ForceInit),
//                    Rotation=ZeroRotator, LocationType=EAttachLocation::KeepRelativeOffset,
//                    bStopWhenAttachedToDestroyed=false, Volume=1, Pitch=1, StartTime=0,
//                    Attenuation=nullptr, Concurrency=nullptr, bAutoDestroy=true)
UAudioComponent* EngineAudio = UGameplayStatics::SpawnSoundAttached(EngineLoopSound, GetMesh(),
    TEXT("AudioSocket"), FVector::ZeroVector, FRotator::ZeroRotator,
    EAttachLocation::SnapToTarget, /*bStopWhenAttachedToDestroyed=*/true,
    1.0f, 1.0f, 0.0f, EngineAttenuation, nullptr, /*bAutoDestroy=*/false);
```

`OwningActor` is what per-owner concurrency (`bLimitToOwner`) keys on — pass it whenever the sound belongs to an actor. `SpawnSound2D` has the same leading arguments as `PlaySound2D` but ends in `bPersistAcrossLevelTransition, bAutoDestroy` (no `OwningActor`/`bIsUISound`) and returns a component; `CreateSound2D` takes the same arguments as `SpawnSound2D` but returns a component that is **not** playing — call `Play()` yourself.

Choose: fire-and-forget for one-shots, a spawned handle for anything you must fade, stop or re-parameterise, and a `CreateDefaultSubobject<UAudioComponent>` member for a permanent per-actor loop.

## UAudioComponent Control

`UAudioComponent` (`Components/AudioComponent.h`) derives `USceneComponent` and implements `ISoundParameterControllerInterface`.

```cpp
// Constructor
AudioComponent = CreateDefaultSubobject<UAudioComponent>(TEXT("AudioComponent"));
AudioComponent->SetupAttachment(RootComponent);
AudioComponent->bAutoActivate = false;
AudioComponent->bStopWhenOwnerDestroyed = true;
AudioComponent->bIsUISound = false;   // true = keeps playing while the game is paused

// Playback — EAudioFaderCurve: Linear, Logarithmic, SCurve, Sin
AudioComponent->SetSound(EngineLoopSound);
AudioComponent->Play(/*StartTime=*/0.0f);
AudioComponent->FadeIn(0.5f, /*FadeVolumeLevel=*/1.0f, /*StartTime=*/0.0f, EAudioFaderCurve::Linear);
AudioComponent->AdjustVolume(/*Duration=*/0.25f, /*Level=*/0.3f, EAudioFaderCurve::Linear);
AudioComponent->FadeOut(1.0f, /*FadeVolumeLevel=*/0.0f, EAudioFaderCurve::Linear);
AudioComponent->SetVolumeMultiplier(0.5f);
AudioComponent->SetPitchMultiplier(1.2f);
AudioComponent->SetPaused(true);
AudioComponent->Stop();

const bool bPlaying = AudioComponent->IsPlaying();
// EAudioComponentPlayState: Playing, Stopped, Paused, FadingIn, FadingOut
const EAudioComponentPlayState State = AudioComponent->GetPlayState();
```

Per-instance routing and filtering, without touching the asset: `SetSubmixSend(USoundSubmixBase*, float)`, `SetSourceBusSendPreEffect` / `PostEffect` / `PostAttenuation`, `SetAudioBusSendPostEffect`, `SetLowPassFilterEnabled` / `SetLowPassFilterFrequency`, the matching high-pass pair, and `SetOutputToBusOnly`. For inline attenuation, set `bOverrideAttenuation = true` and fill `AttenuationOverrides` (an `FSoundAttenuationSettings`).

Each delegate comes in two forms (`Components/AudioComponent.h:441-481`). The dynamic, Blueprint-visible form binds a `UFUNCTION` with `AddDynamic`. The `...Native` twin binds with `AddUObject` and also passes the `UAudioComponent*`. The pairs are `OnAudioFinished`, `OnAudioPlayStateChanged`, `OnAudioVirtualizationChanged`, `OnAudioPlaybackPercent`, `OnAudioSingleEnvelopeValue` and `OnAudioMultiEnvelopeValue`. Unbind with `RemoveAll(this)` in `EndPlay`. The code for both lists is in [references/audio-component-reference.md](references/audio-component-reference.md).

## Attenuation and Spatialization

`USoundAttenuation` assets wrap `FSoundAttenuationSettings` (`Sound/SoundAttenuation.h`), which derives `FBaseAttenuationSettings` (`Engine/Attenuation.h`).

Shape (`EAttenuationShape::Type`): `Sphere`, `Capsule`, `Box`, `Cone`. Curve (`EAttenuationDistanceModel`): `Linear`, `Logarithmic`, `Inverse`, `LogReverse`, `NaturalSound`, `Custom`. `NaturalSound` matches perceived loudness best for gameplay sources.

```cpp
FSoundAttenuationSettings Settings;
Settings.DistanceAlgorithm = EAttenuationDistanceModel::NaturalSound;  // FBaseAttenuationSettings
Settings.AttenuationShape  = EAttenuationShape::Sphere;
Settings.FalloffDistance   = 3000.0f;

Settings.bAttenuate  = true;
Settings.bSpatialize = true;
// ESoundSpatializationAlgorithm: SPATIALIZATION_Default (panning), SPATIALIZATION_HRTF (plugin)
Settings.SpatializationAlgorithm = SPATIALIZATION_Default;
// Air absorption
Settings.bAttenuateWithLPF  = true;
Settings.LPFRadiusMin       = 1000.0f;
Settings.LPFRadiusMax       = 6000.0f;
Settings.LPFFrequencyAtMin  = 20000.0f;
Settings.LPFFrequencyAtMax  = 800.0f;
// Occlusion
Settings.bEnableOcclusion                = true;
Settings.OcclusionTraceChannel           = ECC_Visibility;
Settings.OcclusionLowPassFilterFrequency = 300.0f;
Settings.OcclusionVolumeAttenuation      = 0.5f;
Settings.OcclusionInterpolationTime      = 0.1f;
// Reverb send — EReverbSendMethod: Linear | CustomCurve | Manual
Settings.bEnableReverbSend = true;
Settings.ReverbSendMethod  = EReverbSendMethod::Linear;
Settings.ReverbWetLevelMin = 0.3f;
Settings.ReverbWetLevelMax = 0.95f;
Settings.ReverbDistanceMin = 400.0f;
Settings.ReverbDistanceMax = 4000.0f;
// Distant sounds lose the voice-budget fight first
Settings.bEnablePriorityAttenuation = true;
Settings.PriorityAttenuationMin     = 1.0f;
Settings.PriorityAttenuationMax     = 0.0f;
```

Listener focus (`bEnableListenerFocus`, `FocusAzimuth`, `NonFocusAzimuth`, `FocusDistanceScale`, `NonFocusDistanceScale`, `NonFocusVolumeAttenuation`) lets sounds the player is looking at cut through. `bEnableSendToAudioLink` controls external AudioLink routing — leave it off for background ambience.

## Concurrency

`USoundConcurrency` (`Sound/SoundConcurrency.h`) wraps `FSoundConcurrencySettings`. Assign through `USoundBase::ConcurrencySet`, or set `bOverrideConcurrency` and fill `ConcurrencyOverrides`.

```cpp
FSoundConcurrencySettings& Rules = FireConcurrency->Concurrency;   // struct ctor is not ENGINE_API — edit an asset's copy
Rules.MaxCount      = 4;
Rules.bLimitToOwner = false;
// EMaxConcurrentResolutionRule::Type — PreventNew, StopOldest, StopFarthestThenPreventNew,
// StopFarthestThenOldest, StopLowestPriority, StopQuietest, StopLowestPriorityThenPreventNew
Rules.ResolutionRule         = EMaxConcurrentResolutionRule::StopFarthestThenOldest;
Rules.RetriggerTime          = 0.05f;   // minimum seconds between plays in this group
Rules.VoiceStealReleaseTime  = 0.08f;   // fade for evicted voices; 0 clicks
Rules.VolumeScaleMode        = EConcurrencyVolumeScaleMode::Distance;  // Default | Distance | Priority
Rules.bVolumeScaleCanRelease = true;
Rules.VolumeScaleAttackTime  = 0.01f;
Rules.VolumeScaleReleaseTime = 0.5f;
```

One shared concurrency asset per sound family (footsteps, impacts, bullet whizzes). `USoundBase::Priority` decides who survives once `MaxCount` is reached.

## Sound Classes and Sound Mixes

`USoundClass` gives every sound a category with its own `Volume`/`Pitch` in `FSoundClassProperties`, and a parent/child hierarchy. `USoundMix` holds `TArray<FSoundClassAdjuster> SoundClassEffects` (`SoundClassObject`, `VolumeAdjuster`, `PitchAdjuster`, `bApplyToChildren`) plus `FadeInTime`.

```cpp
UGameplayStatics::SetBaseSoundMix(this, DefaultMix);          // usually once, or via AudioSettings
UGameplayStatics::PushSoundMixModifier(this, CombatMix);      // layers on top
UGameplayStatics::PopSoundMixModifier(this, CombatMix);
UGameplayStatics::ClearSoundMixModifiers(this);

// Duck one class inside an already-pushed mix
UGameplayStatics::SetSoundMixClassOverride(this, CombatMix, MusicClass,
    /*Volume=*/0.3f, /*Pitch=*/1.0f, /*FadeInTime=*/0.5f, /*bApplyToChildren=*/true);
UGameplayStatics::ClearSoundMixClassOverride(this, CombatMix, MusicClass, /*FadeOutTime=*/1.0f);
```

Sound classes are for *gameplay-driven* ducking. For player volume sliders use submix output volume instead — it is a single linear gain with no stack to unwind.

## Submixes, Buses and Sends

```
USoundSubmixBase (Sound/SoundSubmix.h)
  ├── USoundSubmixWithParentBase
  │     ├── USoundSubmix          // effect chain, analysis, recording
  │     └── USoundfieldSubmix     // ambisonics / soundfield
  ├── UEndpointSubmix
  └── USoundfieldEndpointSubmix
```

```cpp
MusicSubmix->SetSubmixOutputVolume(this, 0.5f);   // linear gain
MusicSubmix->SetSubmixWetLevel(this, 1.0f);
MusicSubmix->SetSubmixDryLevel(this, 0.0f);

MusicSubmix->DynamicConnect(this, MasterSubmix);  // re-parent at runtime
MusicSubmix->DynamicDisconnect(this);
```

Set `bMuteWhenBackgrounded = true` on music/SFX submixes so they stop when the app loses focus.

**Sends.** `FSoundSubmixSendInfo` (`Sound/SoundSubmixSend.h`) and `FSoundSourceBusSendInfo` (`Sound/SoundSourceBusSend.h`) share the same shape: a control method (`ESendLevelControlMethod` / `ESourceBusSendLevelControlMethod` — `Linear`, `CustomCurve`, `Manual`), `SendLevel` or `MinSendLevel`/`MaxSendLevel` + `MinSendDistance`/`MaxSendDistance`, `CustomSendLevelCurve`, and in 5.8 an explicit filter pair:

```cpp
FSoundSubmixSendInfo Send;
Send.SoundSubmix           = ReverbSubmix;
Send.SendLevelControlMethod = ESendLevelControlMethod::Manual;
Send.SendLevel             = 0.35f;
Send.SendStage             = ESubmixSendStage::PostDistanceAttenuation;  // or PreDistanceAttenuation
Send.bEnableLPFCutoff      = true;
Send.LPFCutoff             = 3000.0f;   // Hz, 20..20000
Send.bEnableHPFCutoff      = false;
Send.HPFCutoff             = 20.0f;
```

**Audio buses.** `UAudioBus` (`Sound/AudioBus.h`, `AudioBusChannels` of `EAudioBusChannels`) is a runtime patch point: sounds send into it, a `USoundSourceBus` sonifies it back, and submixes can register it. Buses are how you feed a sidechain compressor or an analyser without an extra playback path.

```cpp
UAudioMixerBlueprintLibrary::StartAudioBus(this, DuckingBus);
UAudioMixerBlueprintLibrary::RegisterAudioBusToSubmix(this, AnalysisSubmix, DuckingBus);
const bool bActive = UAudioMixerBlueprintLibrary::IsAudioBusActive(this, DuckingBus);
UAudioMixerBlueprintLibrary::UnregisterAudioBusFromSubmix(this, AnalysisSubmix, DuckingBus);
UAudioMixerBlueprintLibrary::StopAudioBus(this, DuckingBus);
```

## Submix Effects

`USoundSubmix::SubmixEffectChain` is a `TArray<TObjectPtr<USoundEffectSubmixPreset>>`. Shipping presets in `AudioMixer/Classes/SubmixEffects`: `USubmixEffectReverbPreset`, `USubmixEffectSubmixEQPreset`, `USubmixEffectDynamicsProcessorPreset`.

```cpp
#include "AudioMixerBlueprintLibrary.h"
#include "SubmixEffects/AudioMixerSubmixEffectReverb.h"

USubmixEffectReverbPreset* Preset = NewObject<USubmixEffectReverbPreset>(this);
FSubmixEffectReverbSettings ReverbSettings;
ReverbSettings.DecayTime  = 2.5f;
ReverbSettings.Density    = 0.85f;
ReverbSettings.Diffusion  = 0.8f;
ReverbSettings.WetLevel   = 0.4f;
Preset->SetSettings(ReverbSettings);   // thread-safe; assigning Settings directly is editor-time only

const int32 ChainIndex = UAudioMixerBlueprintLibrary::AddSubmixEffect(this, ReverbSubmix, Preset);
UAudioMixerBlueprintLibrary::RemoveSubmixEffect(this, ReverbSubmix, Preset);
UAudioMixerBlueprintLibrary::RemoveSubmixEffectAtIndex(this, ReverbSubmix, ChainIndex);
UAudioMixerBlueprintLibrary::ClearSubmixEffects(this, ReverbSubmix);
```

Swap whole chains for an environment with `SetSubmixEffectChainOverride(WorldContext, Submix, Chain, FadeTimeSec)` and `ClearSubmixEffectChainOverride(WorldContext, Submix, FadeTimeSec)`. `AddMasterSubmixEffect` / `RemoveMasterSubmixEffect` / `ClearMasterSubmixEffects` act on the master submix.

Source-side effects use `USoundEffectSourcePreset` inside a `USoundEffectSourcePresetChain` (`Sound/SoundEffectSource.h`). `AAudioVolume` applies `FReverbSettings`, `FInteriorSettings` and submix send/override settings by volume, ordered by `Priority`.

## MetaSounds

`UMetaSoundSource` derives `USoundWaveProcedural` → `USoundWave` → `USoundBase`, so it plays through every function above. Drive its typed inputs through `ISoundParameterControllerInterface`, which `UAudioComponent` implements (base interface `IAudioParameterControllerInterface` in `AudioExtensions/Public/AudioParameterControllerInterface.h`).

```cpp
EngineAudioComp->SetFloatParameter(FName("RPM"), CurrentRPM);
EngineAudioComp->SetBoolParameter(FName("Engaged"), bEngaged);
EngineAudioComp->SetIntParameter(FName("GearIndex"), Gear);
EngineAudioComp->SetStringParameter(FName("SurfaceName"), TEXT("Gravel"));
EngineAudioComp->SetObjectParameter(FName("ImpactWave"), ImpactWave);
EngineAudioComp->SetTriggerParameter(FName("OnGearShift"));
EngineAudioComp->ResetParameters();

// Batch — cheaper than one call per value; SetParameters takes an rvalue array
TArray<FAudioParameter> Batch;
Batch.Emplace(FAudioParameter(FName("RPM"), CurrentRPM));
Batch.Emplace(FAudioParameter(FName("Engaged"), bEngaged));
EngineAudioComp->SetParameters(MoveTemp(Batch));
```

Parameter names must match the MetaSound graph inputs exactly. Setting a parameter on a component whose sound is not a MetaSound is a no-op, not an error.

`UMetasoundGeneratorHandle::CreateMetaSoundGeneratorHandle(AudioComponent)` (`MetasoundGeneratorHandle.h`) gives access to the running generator: `IsValid()`, `GetAudioComponentId()`, `ApplyParameterPack(UMetasoundParameterPack*)`, `GetGenerator()`, plus `OnGeneratorHandleAttached` / `OnGeneratorHandleDetached`. Read graph outputs with `UMetaSoundOutputSubsystem::WatchOutput`.

Build graphs from code with `UMetaSoundBuilderSubsystem::Get()` → `CreateSourceBuilder` / `CreatePatchBuilder` / `CreateSourcePresetBuilder`, then `AddGraphInputNode`, `AddNodeByClassName`, `ConnectNodes`, `SetNodeInputDefault` and `Audition`. `Build(const FMetaSoundBuilderOptions&)` is `WITH_EDITORONLY_DATA`; at runtime use `Audition` or `BuildNewMetaSound(FName)` (`MetasoundBuilderBase.h:586`). The underlying document API is `FMetaSoundFrontendDocumentBuilder` (`MetasoundFrontend/Public/MetasoundFrontendDocumentBuilder.h`). Worked example: [audio setup patterns](references/audio-setup-patterns.md).

## Modulation

The `AudioModulation` plugin routes continuous control signals into volume, pitch and filter cutoffs. `USoundControlBus` carries a value, `USoundModulationGenerator` produces one (LFO, envelope follower), `USoundControlBusMix` sets bus targets, `USoundModulationPatch` maps between parameters. Bus, generator and patch derive `USoundModulatorBase`; `USoundControlBusMix` is a plain `UObject` (`SoundControlBusMix.h:37`).

```cpp
// EModulationDestination: Volume, Pitch, Lowpass, Highpass
// EModulationRouting: Disable, Inherit, Override, Union
TSet<USoundModulatorBase*> Modulators;
Modulators.Add(TensionControlBus);
AudioComponent->SetModulationRouting(Modulators, EModulationDestination::Volume, EModulationRouting::Override);
AudioComponent->AddModulationRouting(Modulators, EModulationDestination::Lowpass);
AudioComponent->RemoveModulationRouting(Modulators, EModulationDestination::Lowpass);
const TSet<USoundModulatorBase*> Current = AudioComponent->GetModulators(EModulationDestination::Volume);
```

On assets, modulation lives in `FSoundModulationDestinationSettings` (`Sound/SoundModulationDestination.h`): a base `Value` plus a `TSet<TObjectPtr<USoundModulatorBase>> Modulators`. In native `Audio::FModulationDestination` code the setter is `SetModulators`.

## Quartz Music Sync

`UQuartzSubsystem` is a `UTickableWorldSubsystem` (`AudioMixer/Public/Quartz/QuartzSubsystem.h`) that runs sample-accurate musical clocks.

```cpp
#include "Quartz/QuartzSubsystem.h"
#include "Quartz/AudioMixerClockHandle.h"

UQuartzSubsystem* Quartz = GetWorld()->GetSubsystem<UQuartzSubsystem>();
FQuartzClockSettings ClockSettings;                    // TimeSignature defaults to 4/4
// Handle-taking calls use UQuartzClockHandle*& — pass a raw local, not a TObjectPtr member
UQuartzClockHandle* ClockHandle = Quartz->CreateNewClock(this, TEXT("MusicClock"), ClockSettings,
    /*bOverrideSettingsIfClockExists=*/false, /*bUseAudioEngineClockManager=*/true);
ClockHandle->StartClock(this, ClockHandle);

// EQuartzCommandQuantization: Bar, Beat, ThirtySecondNote and the other note values
FQuartzQuantizationBoundary Boundary;
Boundary.Quantization           = EQuartzCommandQuantization::Bar;
Boundary.Multiplier             = 1.0f;
Boundary.CountingReferencePoint = EQuarztQuantizationReference::BarRelative;
Boundary.bFireOnClockStart      = true;
FOnQuartzCommandEventBP OnQueued;
StingerComponent->PlayQuantized(this, ClockHandle, Boundary, OnQueued,
    /*InStartTime=*/0.0f, /*InFadeInDuration=*/0.0f, /*InFadeVolumeLevel=*/1.0f, EAudioFaderCurve::Linear);
```

Subscribe to the metronome with `ClockHandle->SubscribeToQuantizationEvent(this, EQuartzCommandQuantization::Beat, OnBeat, ClockHandle)` (or `SubscribeToAllQuantizationEvents`); the `FOnQuartzMetronomeEventBP` callback receives `(FName ClockName, EQuartzCommandQuantization, int32 NumBars, int32 Beat, float BeatFraction)`. Tempo changes go through `SetBeatsPerMinute`, which is itself quantized.

## Analysis

Real time, on a `USoundSubmix`:

```cpp
MySFXSubmix->StartEnvelopeFollowing(this);
FOnSubmixEnvelopeBP EnvelopeDelegate;
EnvelopeDelegate.BindDynamic(this, &AMyActor::OnSubmixEnvelope);
MySFXSubmix->AddEnvelopeFollowerDelegate(this, EnvelopeDelegate);
MySFXSubmix->StartSpectralAnalysis(this, EFFTSize::Medium, EFFTPeakInterpolationMethod::Linear,
    EFFTWindowType::Hann, /*HopSize=*/0.0f, EAudioSpectrumType::MagnitudeSpectrum);
MySFXSubmix->StopSpectralAnalysis(this);
MySFXSubmix->StopEnvelopeFollowing(this);
```

Band-based FFT callbacks go through `AddSpectralAnalysisDelegate(WorldContext, BandSettings, Delegate, UpdateRate, DecibelNoiseFloor, bDoNormalize, bDoAutoRange, AutoRangeAttackTime, AutoRangeReleaseTime)` with an array of `FSoundSubmixSpectralAnalysisBandSettings` — see [audio setup patterns](references/audio-setup-patterns.md). `UAudioMixerBlueprintLibrary::MakeMusicalSpectralAnalysisBandSettings`, `MakeFullSpectrumSpectralAnalysisBandSettings` and `MakePresetSpectralAnalysisBandSettings` build those arrays for you; `StartAnalyzingOutput` / `StopAnalyzingOutput` plus `GetMagnitudeForFrequencies` / `GetPhaseForFrequencies` give a polling API. Recording uses `StartRecordingOutput` / `StopRecordingOutput` (`EAudioRecordingExportType::SoundWave` or `WavFile`).

Offline, with the `AudioSynesthesia` plugin (Beta in 5.8): `ULoudnessNRT` and `UOnsetNRT` derive `UAudioSynesthesiaNRT` → `UAudioAnalyzerNRT`. `AnalyzeAudio()` is `WITH_EDITOR` only — bake beat maps at cook time, then read them at runtime with `GetLoudnessAtTime` / `GetNormalizedChannelOnsetsBetweenTimes`. Full example in [audio setup patterns](references/audio-setup-patterns.md).

## Subtitles

The `SubtitlesAndClosedCaptions` plugin (Beta in 5.8) drives captions from `USubtitlesBlueprintFunctionLibrary`:

```cpp
#include "SubtitlesBlueprintFunctionLibrary.h"

USubtitlesBlueprintFunctionLibrary::QueueSingleSubtitle(
    FText::FromString(TEXT("Contact, north ridge.")), /*Duration=*/2.5f, /*StartOffset=*/0.0f,
    /*Priority=*/1.0f, ESubtitleType::Subtitle, ESubtitleTiming::InternallyTimed);
USubtitlesBlueprintFunctionLibrary::StopAllSubtitles();
```

`QueueSubtitlesFromAsset` / `StopSubtitlesInAsset` take a `USubtitleAssetUserData` attached to the sound. The engine-side `FSubtitleManager` (`Engine/Public/SubtitleManager.h`) still exists for `USoundWave::Subtitles` cues.

## Platform Notes

- **Windows backend** — `AudioMixerModuleName=AudioMixerWasapi` in `Config/Windows/BaseWindowsEngine.ini`. XAudio2 (`AudioMixerXAudio2`) is still selectable per project.
- **Dedicated servers** — no audio device. Guard every play call with `GetNetMode() != NM_DedicatedServer`.
- **Mobile** — tight voice budgets. Lean on concurrency and `Priority`, avoid `SPATIALIZATION_HRTF`, prefer `ForceInline` for short SFX.
- **Backgrounding** — `bMuteWhenBackgrounded` on submixes, or hook `FCoreDelegates::ApplicationWillDeactivateDelegate` and call `FAudioDevice::SetTransientPrimaryVolume(0.0f)` on `GEngine->GetMainAudioDevice()`.
- **VR** — `bSpatialize = true` with `SPATIALIZATION_HRTF` on 3D attenuation assets, backed by a spatialization plugin.
- **Pitch range** — clamped by `UAudioSettings::GlobalMinPitchScale` / `GlobalMaxPitchScale`.

## Deprecated — do not use

| Do not emit | Use in 5.8 | Source |
|---|---|---|
| `USubmixEffectEQPreset` | `USubmixEffectSubmixEQPreset` | never existed; real class in `SubmixEffects/AudioMixerSubmixEffectEQ.h:110` |
| `FSoundSubmixSendInfo::LPFCutoffFrequency` | `bEnableLPFCutoff` + `LPFCutoff` | `UE_DEPRECATED(5.8)` in `Sound/SoundSubmixSend.h:157` |
| `FSoundSubmixSendInfo::HPFCutoffFrequency` | `bEnableHPFCutoff` + `HPFCutoff` | `UE_DEPRECATED(5.8)` in `Sound/SoundSubmixSend.h:161` |
| `FSoundSourceBusSendInfo::LPFCutoffFrequency` | `bEnableLPFCutoff` + `LPFCutoff` | `UE_DEPRECATED(5.8)` in `Sound/SoundSourceBusSend.h:86` |
| `Audio::FModulationDestination::UpdateModulators` | `SetModulators` | `UE_DEPRECATED(5.8)` in `Sound/SoundModulationDestination.h:198` |
| `Audio::FModulationDestination::UpdateModulator` | `SetModulators` | `UE_DEPRECATED(5.1)` in `Sound/SoundModulationDestination.h:195` |
| `USoundWave::CreateSoundWaveProxy` | `GetSoundWaveProxy()` | `UE_DEPRECATED(5.8)` in `Sound/SoundWave.h:1204` |
| `FSoundWaveProxy::GetZerothChunk` and the other proxy accessors | hold the result of `GetSoundWaveDataRef()` | `UE_DEPRECATED(5.8)` in `Sound/SoundWave.h:1813` |
| `FSoundWaveProxyReader` | `FSoundWaveProxyPlayer` | `UE_DEPRECATED(5.8)` in `Sound/SoundWaveProxyReader.h:19` |
| `USubtitlesBlueprintFunctionLibrary::QueueSubtitle` | `QueueSingleSubtitle` | `UE_DEPRECATED(5.8)` in `SubtitlesBlueprintFunctionLibrary.h:32` |
| `UAudioBusSubsystem::StartAudioBus(Key, Channels, bAutomatic)` | overload taking `InAudioBusName` | `UE_DEPRECATED(5.6)` in `AudioMixer/Public/AudioBusSubsystem.h:83` |
| `UAudioMixerBlueprintLibrary::RemoveSubmixEffectPreset` | `RemoveSubmixEffect` | `UE_DEPRECATED(4.27)` in `AudioMixerBlueprintLibrary.h:242` |
| `UAudioMixerBlueprintLibrary::ReplaceSoundEffectSubmix` | `ReplaceSubmixEffect` | `UE_DEPRECATED(4.27)` in `AudioMixerBlueprintLibrary.h:258` |
| `FAudioDevice::SetTransientMasterVolume` | `SetTransientPrimaryVolume` | `UE_DEPRECATED(5.1)` in `Engine/Public/AudioDevice.h:1892` |
| `FAudioComponentParam` | `FAudioParameter` | `UE_DEPRECATED(5.0)` in `Components/AudioComponent.h:133` |
| `USoundBase::SoundConcurrencySettings_DEPRECATED` | `ConcurrencySet` / `ConcurrencyOverrides` | `_DEPRECATED` property in `Sound/SoundBase.h:183` |

## Common Mistakes

**Spawning a component per one-shot:** `NewObject<UAudioComponent>` leaves an unattached, unregistered component behind on every call.
```cpp
// WRONG
UAudioComponent* Comp = NewObject<UAudioComponent>(this);
Comp->SetSound(FireSound);
Comp->Play();
// RIGHT — one-shot
UGameplayStatics::PlaySoundAtLocation(this, FireSound, GetActorLocation());
```

**Playing audio on a dedicated server:** there is no audio device, so the call is wasted work and floods the log.
```cpp
if (GetNetMode() != NM_DedicatedServer)
{
    UGameplayStatics::PlaySoundAtLocation(this, FireSound, GetActorLocation());
}
```

**`PlaySound2D` for a world sound:** 2D playback skips attenuation and spatialization entirely, so a gunshot across the map is full volume and centred. Use `PlaySoundAtLocation` with a `USoundAttenuation`.

**A looping sound with `bAutoDestroy = true`:** the handle dies the first time the sound reports finished, and later `FadeOut` calls hit a dangling component. Pass `bAutoDestroy = false` for anything you keep a pointer to.

**Forgetting `OwningActor` on `PlaySoundAtLocation`:** `bLimitToOwner` concurrency then falls back to group-wide limiting (`FConcurrencyHandle::GetMode`, `SoundConcurrency.cpp:179`), so per-actor limits silently become global.

**Leaving delegates bound past `EndPlay`:** call `OnAudioFinished.RemoveAll(this)` and `Stop()` in `EndPlay`, otherwise a pooled or auto-destroyed component fires into a stale object.

**Volume sliders via sound mixes:** push/pop mixes are a stack, and a missed pop leaves the game permanently ducked. Use `USoundSubmix::SetSubmixOutputVolume` for user settings; reserve mixes for transient gameplay ducking.

**Assigning `Preset->Settings` at runtime:** the audio render thread reads its own copy. Call `SetSettings` on the preset instead.

## Related Skills

- `ue-niagara-effects` — particle systems and VFX timing that audio events are synchronized with
- `ue-animation-system` — anim notifies that trigger footsteps, foley and dialogue
- `ue-actor-component-architecture` — component lifetime, attachment and replication for audio components
- `ue-gameplay-framework` — where audio managers live (GameInstance, subsystems, player controller)
- `ue-data-assets-tables` — data-driven sound banks and lookup tables for surface/impact audio
- `ue-sequencer-cinematics` — audio tracks, dialogue and music in Level Sequences
- `ue-cpp-foundations` — delegate binding, `UPROPERTY`/`UFUNCTION` and subsystem access patterns
