# Audio Setup Patterns

Worked audio architectures for UE 5.8. Every API here is verified against `Components/AudioComponent.h`,
`Kismet/GameplayStatics.h`, `Sound/SoundSubmix.h`, `Sound/SoundSubmixSend.h`, `Sound/SoundConcurrency.h`,
`Sound/SoundAttenuation.h`, `Quartz/QuartzSubsystem.h`, `MetasoundBuilderSubsystem.h`,
`AudioModulationStatics.h` and the Audio Synesthesia NRT classes.

Example types use the `AMy*` / `UMy*` / `FMy*` convention and the `MYGAME_API` export macro.

---

## Pattern 1: Music System with Crossfading

Two `UAudioComponent` slots — one active, one incoming — crossfaded with `FadeIn`/`FadeOut`.

### MyMusicManager.h

```cpp
#pragma once
#include "CoreMinimal.h"
#include "GameFramework/Actor.h"
#include "Components/AudioComponent.h"
#include "MyMusicManager.generated.h"

class USoundSubmix;

UCLASS(BlueprintType)
class MYGAME_API AMyMusicManager : public AActor
{
    GENERATED_BODY()

public:
    AMyMusicManager();

    UFUNCTION(BlueprintCallable, Category = "Audio|Music")
    void PlayMusic(USoundBase* NewTrack, float CrossfadeDuration = 1.0f);

    UFUNCTION(BlueprintCallable, Category = "Audio|Music")
    void StopMusic(float FadeOutDuration = 1.0f);

    UFUNCTION(BlueprintCallable, Category = "Audio|Music")
    void SetMusicVolume(float Volume);

protected:
    virtual void BeginPlay() override;
    virtual void EndPlay(const EEndPlayReason::Type EndPlayReason) override;

    UFUNCTION()
    void OnTrackFinished();

private:
    // Slot A: currently playing track. Slot B: incoming track during a crossfade.
    UPROPERTY()
    TObjectPtr<UAudioComponent> TrackA;

    UPROPERTY()
    TObjectPtr<UAudioComponent> TrackB;

    bool bSlotAIsActive = true;

    // Volume scalar applied to both tracks, independent of the crossfade envelope
    float MyMusicVolume = 1.0f;

    UPROPERTY(EditDefaultsOnly, Category = "Audio|Music")
    TObjectPtr<USoundSubmix> MusicSubmix;
};
```

### MyMusicManager.cpp

```cpp
#include "MyMusicManager.h"
#include "Sound/SoundSubmix.h"

AMyMusicManager::AMyMusicManager()
{
    PrimaryActorTick.bCanEverTick = false;

    TrackA = CreateDefaultSubobject<UAudioComponent>(TEXT("TrackA"));
    TrackA->SetupAttachment(RootComponent);
    TrackA->bAutoActivate = false;
    TrackA->bIsUISound = true;   // keeps playing while the game is paused

    TrackB = CreateDefaultSubobject<UAudioComponent>(TEXT("TrackB"));
    TrackB->SetupAttachment(RootComponent);
    TrackB->bAutoActivate = false;
    TrackB->bIsUISound = true;
}

void AMyMusicManager::BeginPlay()
{
    Super::BeginPlay();
    TrackA->OnAudioFinished.AddDynamic(this, &AMyMusicManager::OnTrackFinished);
    TrackB->OnAudioFinished.AddDynamic(this, &AMyMusicManager::OnTrackFinished);
}

void AMyMusicManager::EndPlay(const EEndPlayReason::Type EndPlayReason)
{
    TrackA->OnAudioFinished.RemoveAll(this);
    TrackB->OnAudioFinished.RemoveAll(this);
    Super::EndPlay(EndPlayReason);
}

void AMyMusicManager::PlayMusic(USoundBase* NewTrack, float CrossfadeDuration)
{
    if (!NewTrack || GetNetMode() == NM_DedicatedServer)
    {
        return;
    }

    UAudioComponent* ActiveSlot   = bSlotAIsActive ? TrackA : TrackB;
    UAudioComponent* IncomingSlot = bSlotAIsActive ? TrackB : TrackA;

    const EAudioComponentPlayState ActiveState = ActiveSlot->GetPlayState();
    if (ActiveState == EAudioComponentPlayState::Playing ||
        ActiveState == EAudioComponentPlayState::FadingIn)
    {
        ActiveSlot->FadeOut(CrossfadeDuration, 0.0f, EAudioFaderCurve::Linear);
    }

    IncomingSlot->SetSound(NewTrack);
    IncomingSlot->FadeIn(CrossfadeDuration, MyMusicVolume, 0.0f, EAudioFaderCurve::Linear);

    bSlotAIsActive = !bSlotAIsActive;
}

void AMyMusicManager::StopMusic(float FadeOutDuration)
{
    TrackA->FadeOut(FadeOutDuration, 0.0f, EAudioFaderCurve::Linear);
    TrackB->FadeOut(FadeOutDuration, 0.0f, EAudioFaderCurve::Linear);
}

void AMyMusicManager::SetMusicVolume(float Volume)
{
    MyMusicVolume = FMath::Clamp(Volume, 0.0f, 1.0f);
    if (MusicSubmix)
    {
        MusicSubmix->SetSubmixOutputVolume(this, MyMusicVolume);
    }
}

void AMyMusicManager::OnTrackFinished()
{
    // Fires only for one-shot music. Looping tracks should loop inside the SoundCue or SoundWave.
}
```

Use `EAudioFaderCurve::Sin` (equal power) instead of `Linear` when both tracks are full-bandwidth
music — a linear crossfade dips in perceived loudness at the midpoint.

---

## Pattern 2: Ambient Soundscape System

Looping ambient layers whose volumes are blended from gameplay state.

### MyAmbientSoundscape.h

```cpp
#pragma once
#include "CoreMinimal.h"
#include "GameFramework/Actor.h"
#include "Components/AudioComponent.h"
#include "MyAmbientSoundscape.generated.h"

class USoundAttenuation;

USTRUCT(BlueprintType)
struct FMyAmbientLayer
{
    GENERATED_BODY()

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    TObjectPtr<USoundBase> Sound;

    UPROPERTY(EditAnywhere, BlueprintReadWrite, meta = (ClampMin = "0.0", ClampMax = "1.0"))
    float MaxVolume = 1.0f;

    UPROPERTY(Transient)
    TObjectPtr<UAudioComponent> Component;
};

UCLASS(BlueprintType)
class MYGAME_API AMyAmbientSoundscape : public AActor
{
    GENERATED_BODY()

public:
    AMyAmbientSoundscape();

    UFUNCTION(BlueprintCallable, Category = "Audio|Ambient")
    void SetLayerVolume(int32 LayerIndex, float NormalizedVolume, float BlendTime = 0.5f);

    UFUNCTION(BlueprintCallable, Category = "Audio|Ambient")
    void FadeOutAll(float FadeDuration = 1.0f);

    UFUNCTION(BlueprintCallable, Category = "Audio|Ambient")
    void FadeInAll(float FadeDuration = 1.0f);

protected:
    virtual void BeginPlay() override;

    UPROPERTY(EditAnywhere, Category = "Audio|Ambient")
    TArray<FMyAmbientLayer> AmbientLayers;

    UPROPERTY(EditDefaultsOnly, Category = "Audio|Ambient")
    TObjectPtr<USoundAttenuation> AmbientAttenuation;
};
```

### MyAmbientSoundscape.cpp

```cpp
#include "MyAmbientSoundscape.h"
#include "Kismet/GameplayStatics.h"

AMyAmbientSoundscape::AMyAmbientSoundscape()
{
    PrimaryActorTick.bCanEverTick = false;
}

void AMyAmbientSoundscape::BeginPlay()
{
    Super::BeginPlay();

    if (GetNetMode() == NM_DedicatedServer)
    {
        return;
    }

    for (FMyAmbientLayer& Layer : AmbientLayers)
    {
        if (!Layer.Sound)
        {
            continue;
        }

        UAudioComponent* Comp = UGameplayStatics::SpawnSoundAttached(
            Layer.Sound,
            GetRootComponent(),
            NAME_None,
            FVector::ZeroVector,
            FRotator::ZeroRotator,
            EAttachLocation::SnapToTarget,
            /*bStopWhenAttachedToDestroyed=*/true,
            0.0f,   // start silent, fade in below
            1.0f,   // pitch
            0.0f,   // start time
            AmbientAttenuation,
            nullptr,
            /*bAutoDestroy=*/false);   // must be false: these loop forever

        if (Comp)
        {
            Layer.Component = Comp;
            Comp->FadeIn(2.0f, Layer.MaxVolume, 0.0f, EAudioFaderCurve::Linear);
        }
    }
}

void AMyAmbientSoundscape::SetLayerVolume(int32 LayerIndex, float NormalizedVolume, float BlendTime)
{
    if (!AmbientLayers.IsValidIndex(LayerIndex))
    {
        return;
    }

    FMyAmbientLayer& Layer = AmbientLayers[LayerIndex];
    if (!Layer.Component)
    {
        return;
    }

    const float TargetVolume = Layer.MaxVolume * FMath::Clamp(NormalizedVolume, 0.0f, 1.0f);
    if (BlendTime <= 0.0f)
    {
        Layer.Component->SetVolumeMultiplier(TargetVolume);
    }
    else
    {
        Layer.Component->AdjustVolume(BlendTime, TargetVolume, EAudioFaderCurve::Linear);
    }
}

void AMyAmbientSoundscape::FadeOutAll(float FadeDuration)
{
    for (FMyAmbientLayer& Layer : AmbientLayers)
    {
        if (Layer.Component)
        {
            Layer.Component->FadeOut(FadeDuration, 0.0f, EAudioFaderCurve::Linear);
        }
    }
}

void AMyAmbientSoundscape::FadeInAll(float FadeDuration)
{
    for (FMyAmbientLayer& Layer : AmbientLayers)
    {
        if (Layer.Component)
        {
            Layer.Component->FadeIn(FadeDuration, Layer.MaxVolume, 0.0f, EAudioFaderCurve::Linear);
        }
    }
}
```

---

## Pattern 3: Gameplay Audio Component (Weapon SFX)

One-shots go through `PlaySoundAtLocation` with concurrency; the mechanical loop keeps a handle.

### MyWeaponAudioComponent.h

```cpp
#pragma once
#include "CoreMinimal.h"
#include "Components/ActorComponent.h"
#include "MyWeaponAudioComponent.generated.h"

class UAudioComponent;
class USoundAttenuation;
class USoundConcurrency;

UCLASS(ClassGroup = Audio, meta = (BlueprintSpawnableComponent))
class MYGAME_API UMyWeaponAudioComponent : public UActorComponent
{
    GENERATED_BODY()

public:
    UMyWeaponAudioComponent();

    UFUNCTION(BlueprintCallable, Category = "Audio|Weapon")
    void PlayFireSound();

    UFUNCTION(BlueprintCallable, Category = "Audio|Weapon")
    void PlayReloadSound();

    UFUNCTION(BlueprintCallable, Category = "Audio|Weapon")
    void StartMechanicalLoop();

    UFUNCTION(BlueprintCallable, Category = "Audio|Weapon")
    void StopMechanicalLoop(float FadeTime = 0.15f);

protected:
    virtual void BeginPlay() override;
    virtual void EndPlay(const EEndPlayReason::Type EndPlayReason) override;

    UPROPERTY(EditDefaultsOnly, Category = "Audio")
    TObjectPtr<USoundBase> FireSound;

    UPROPERTY(EditDefaultsOnly, Category = "Audio")
    TObjectPtr<USoundBase> ReloadSound;

    UPROPERTY(EditDefaultsOnly, Category = "Audio")
    TObjectPtr<USoundBase> MechanicalLoopSound;

    UPROPERTY(EditDefaultsOnly, Category = "Audio")
    TObjectPtr<USoundAttenuation> WeaponAttenuation;

    // MaxCount = 4, ResolutionRule = StopFarthestThenOldest, VoiceStealReleaseTime = 0.08
    UPROPERTY(EditDefaultsOnly, Category = "Audio")
    TObjectPtr<USoundConcurrency> FireConcurrency;

private:
    UPROPERTY()
    TObjectPtr<UAudioComponent> MechanicalLoopComp;

    bool IsDedicatedServer() const;
    FVector GetOwnerLocation() const;
};
```

### MyWeaponAudioComponent.cpp

```cpp
#include "MyWeaponAudioComponent.h"
#include "GameFramework/Actor.h"
#include "Kismet/GameplayStatics.h"
#include "Components/AudioComponent.h"

UMyWeaponAudioComponent::UMyWeaponAudioComponent()
{
    PrimaryComponentTick.bCanEverTick = false;
}

void UMyWeaponAudioComponent::BeginPlay()
{
    Super::BeginPlay();

    if (IsDedicatedServer() || !MechanicalLoopSound || !GetOwner())
    {
        return;
    }

    MechanicalLoopComp = UGameplayStatics::SpawnSoundAttached(
        MechanicalLoopSound,
        GetOwner()->GetRootComponent(),
        NAME_None,
        FVector::ZeroVector,
        FRotator::ZeroRotator,
        EAttachLocation::SnapToTarget,
        /*bStopWhenAttachedToDestroyed=*/true,
        1.0f, 1.0f, 0.0f,
        WeaponAttenuation,
        nullptr,
        /*bAutoDestroy=*/false);

    if (MechanicalLoopComp)
    {
        MechanicalLoopComp->Stop();   // spawned components start playing; silence until asked
    }
}

void UMyWeaponAudioComponent::EndPlay(const EEndPlayReason::Type EndPlayReason)
{
    if (MechanicalLoopComp)
    {
        MechanicalLoopComp->Stop();
    }
    Super::EndPlay(EndPlayReason);
}

void UMyWeaponAudioComponent::PlayFireSound()
{
    if (IsDedicatedServer() || !FireSound)
    {
        return;
    }

    UGameplayStatics::PlaySoundAtLocation(
        this,
        FireSound,
        GetOwnerLocation(),
        FRotator::ZeroRotator,
        1.0f, 1.0f, 0.0f,
        WeaponAttenuation,
        FireConcurrency,
        GetOwner());   // OwningActor — required for bLimitToOwner concurrency
}

void UMyWeaponAudioComponent::PlayReloadSound()
{
    if (IsDedicatedServer() || !ReloadSound)
    {
        return;
    }

    UGameplayStatics::PlaySoundAtLocation(
        this, ReloadSound, GetOwnerLocation(), FRotator::ZeroRotator,
        1.0f, 1.0f, 0.0f, WeaponAttenuation, nullptr, GetOwner());
}

void UMyWeaponAudioComponent::StartMechanicalLoop()
{
    if (IsDedicatedServer() || !MechanicalLoopComp)
    {
        return;
    }
    MechanicalLoopComp->FadeIn(0.1f, 1.0f, 0.0f, EAudioFaderCurve::Linear);
}

void UMyWeaponAudioComponent::StopMechanicalLoop(float FadeTime)
{
    if (MechanicalLoopComp)
    {
        MechanicalLoopComp->FadeOut(FadeTime, 0.0f, EAudioFaderCurve::Linear);
    }
}

bool UMyWeaponAudioComponent::IsDedicatedServer() const
{
    const AActor* OwnerActor = GetOwner();
    return OwnerActor && OwnerActor->GetNetMode() == NM_DedicatedServer;
}

FVector UMyWeaponAudioComponent::GetOwnerLocation() const
{
    const AActor* OwnerActor = GetOwner();
    return OwnerActor ? OwnerActor->GetActorLocation() : FVector::ZeroVector;
}
```

---

## Pattern 4: MetaSound Parameter-Driven Engine Audio

A `UMetaSoundSource` with `RPM` (float), `Engaged` (bool), `GearIndex` (int32) and an
`OnGearShift` trigger input, driven from the vehicle.

```cpp
// MyVehicle.h — inside the class body
UPROPERTY(EditDefaultsOnly, Category = "Audio")
TObjectPtr<UMetaSoundSource> EngineSoundAsset;

UPROPERTY(EditDefaultsOnly, Category = "Audio")
TObjectPtr<USoundAttenuation> EngineAttenuation;

UPROPERTY(Transient)
TObjectPtr<UAudioComponent> EngineAudioComp;
```

```cpp
// MyVehicle.cpp
#include "Kismet/GameplayStatics.h"
#include "Components/SkeletalMeshComponent.h"   // GetMesh() -> USceneComponent* needs the full type
#include "Components/AudioComponent.h"
#include "MetasoundSource.h"

void AMyVehicle::BeginPlay()
{
    Super::BeginPlay();
    if (GetNetMode() == NM_DedicatedServer)
    {
        return;
    }

    EngineAudioComp = UGameplayStatics::SpawnSoundAttached(
        EngineSoundAsset,
        GetMesh(),
        TEXT("AudioSocket"),
        FVector::ZeroVector,
        FRotator::ZeroRotator,
        EAttachLocation::SnapToTarget,
        /*bStopWhenAttachedToDestroyed=*/true,
        1.0f, 1.0f, 0.0f,
        EngineAttenuation,
        nullptr,
        /*bAutoDestroy=*/false);
}

void AMyVehicle::Tick(float DeltaTime)
{
    Super::Tick(DeltaTime);
    if (!EngineAudioComp)
    {
        return;
    }

    // Batch the per-frame values: one transmitter hop instead of two
    TArray<FAudioParameter> Params;
    Params.Emplace(FAudioParameter(FName("RPM"), GetCurrentRPM()));
    Params.Emplace(FAudioParameter(FName("Engaged"), IsTransmissionEngaged()));
    EngineAudioComp->SetParameters(MoveTemp(Params));
}

void AMyVehicle::OnGearShift(int32 NewGear)
{
    if (EngineAudioComp)
    {
        EngineAudioComp->SetIntParameter(FName("GearIndex"), NewGear);
        EngineAudioComp->SetTriggerParameter(FName("OnGearShift"));
    }
}
```

Parameter names are matched by `FName` against the MetaSound graph inputs — a typo fails silently.

---

## Pattern 5: Submix-Based Volume Control (Settings Menu)

Player volume sliders belong on submixes, not sound mixes: one linear gain, no stack to unwind.

```cpp
// MyGameUserSettings.h — inside the class body
UPROPERTY(config)
float MasterVolume = 1.0f;

UPROPERTY(config)
float MusicVolume = 1.0f;

UPROPERTY(config)
float SFXVolume = 1.0f;

UPROPERTY(EditDefaultsOnly, Category = "Audio")
TObjectPtr<USoundSubmix> MasterSubmix;

UPROPERTY(EditDefaultsOnly, Category = "Audio")
TObjectPtr<USoundSubmix> MusicSubmix;

UPROPERTY(EditDefaultsOnly, Category = "Audio")
TObjectPtr<USoundSubmix> SFXSubmix;
```

```cpp
// MyGameUserSettings.cpp
#include "Sound/SoundSubmix.h"
#include "Engine/Engine.h"

// UGameUserSettings is outered to the transient package (UnrealEngine.cpp:17941), so its own
// GetWorld() is always null — take the world context from the caller.
void UMyGameUserSettings::ApplyAudioSettings(const UObject* WorldContextObject)
{
    UWorld* World = GEngine ? GEngine->GetWorldFromContextObject(WorldContextObject, EGetWorldErrorMode::ReturnNull) : nullptr;
    if (!World)
    {
        return;
    }

    if (MasterSubmix)
    {
        MasterSubmix->SetSubmixOutputVolume(World, MasterVolume);
    }
    if (MusicSubmix)
    {
        MusicSubmix->SetSubmixOutputVolume(World, MusicVolume);
    }
    if (SFXSubmix)
    {
        SFXSubmix->SetSubmixOutputVolume(World, SFXVolume);
    }
}

void UMyGameUserSettings::SetMasterVolume(const UObject* WorldContextObject, float NewVolume)
{
    MasterVolume = FMath::Clamp(NewVolume, 0.0f, 1.0f);
    ApplyAudioSettings(WorldContextObject);
    SaveConfig();
}
```

Slider positions are perceptual, so map the UI 0..1 to gain with a curve (for example
`FMath::Pow(SliderValue, 3.0f)`) rather than feeding the raw value to `SetSubmixOutputVolume`.

---

## Pattern 6: Beat-Reactive Visual (Spectral Bands)

Drive a material parameter and a light radius from submix spectral analysis.

```cpp
// MyBeaconLight.h — inside the class body
UPROPERTY(EditDefaultsOnly, Category = "Audio")
TObjectPtr<USoundSubmix> MusicSubmix;

UPROPERTY(Transient)
TObjectPtr<UMaterialInstanceDynamic> LightMaterial;

UPROPERTY(EditDefaultsOnly, Category = "Audio")
TObjectPtr<UPointLightComponent> PointLight;

UFUNCTION()
void OnSpectralData(const TArray<float>& Magnitudes);
```

```cpp
// MyBeaconLight.cpp
#include "Sound/SoundSubmix.h"
#include "Components/PointLightComponent.h"
#include "Materials/MaterialInstanceDynamic.h"

void AMyBeaconLight::BeginPlay()
{
    Super::BeginPlay();
    if (GetNetMode() == NM_DedicatedServer || !MusicSubmix)
    {
        return;
    }

    MusicSubmix->StartSpectralAnalysis(
        this,
        EFFTSize::Medium,
        EFFTPeakInterpolationMethod::Linear,
        EFFTWindowType::Hann,
        /*HopSize=*/0.0f,
        EAudioSpectrumType::MagnitudeSpectrum);

    TArray<FSoundSubmixSpectralAnalysisBandSettings> Bands;

    FSoundSubmixSpectralAnalysisBandSettings Bass;
    Bass.BandFrequency   = 80.0f;
    Bass.AttackTimeMsec  = 5;
    Bass.ReleaseTimeMsec = 80;
    Bands.Add(Bass);

    FSoundSubmixSpectralAnalysisBandSettings Mid;
    Mid.BandFrequency   = 800.0f;
    Mid.AttackTimeMsec  = 10;
    Mid.ReleaseTimeMsec = 120;
    Bands.Add(Mid);

    FOnSubmixSpectralAnalysisBP SpectralDelegate;
    SpectralDelegate.BindDynamic(this, &AMyBeaconLight::OnSpectralData);

    MusicSubmix->AddSpectralAnalysisDelegate(
        this, Bands, SpectralDelegate,
        /*UpdateRate=*/30.0f,
        /*DecibelNoiseFloor=*/-40.0f,
        /*bDoNormalize=*/true,
        /*bDoAutoRange=*/false);
}

void AMyBeaconLight::OnSpectralData(const TArray<float>& Magnitudes)
{
    const float BassMag = Magnitudes.IsValidIndex(0) ? Magnitudes[0] : 0.0f;
    const float MidMag  = Magnitudes.IsValidIndex(1) ? Magnitudes[1] : 0.0f;

    if (LightMaterial)
    {
        LightMaterial->SetScalarParameterValue(TEXT("BassIntensity"), BassMag);
        LightMaterial->SetScalarParameterValue(TEXT("MidIntensity"), MidMag);
    }

    if (PointLight)
    {
        PointLight->SetAttenuationRadius(200.0f + BassMag * 800.0f);
    }
}

void AMyBeaconLight::EndPlay(const EEndPlayReason::Type EndPlayReason)
{
    if (MusicSubmix)
    {
        MusicSubmix->StopSpectralAnalysis(this);
    }
    Super::EndPlay(EndPlayReason);
}
```

`AttackTimeMsec` and `ReleaseTimeMsec` are `int32` on `FSoundSubmixSpectralAnalysisBandSettings`, not floats.

---

## Pattern 7: Quartz-Synced Stingers

A Quartz clock keeps gameplay stingers on the musical grid regardless of when the event fires.

```cpp
// MyMusicDirector.h — inside the class body
UPROPERTY(Transient)
TObjectPtr<UQuartzClockHandle> ClockHandle;

UPROPERTY(EditDefaultsOnly, Category = "Audio")
TObjectPtr<USoundBase> StingerSound;

UPROPERTY(Transient)
TObjectPtr<UAudioComponent> StingerComponent;

UFUNCTION()
void OnBeat(FName ClockName, EQuartzCommandQuantization QuantizationType,
            int32 NumBars, int32 Beat, float BeatFraction);
```

```cpp
// MyMusicDirector.cpp
#include "Quartz/QuartzSubsystem.h"
#include "Quartz/AudioMixerClockHandle.h"
#include "Kismet/GameplayStatics.h"

void AMyMusicDirector::BeginPlay()
{
    Super::BeginPlay();
    if (GetNetMode() == NM_DedicatedServer)
    {
        return;
    }

    UQuartzSubsystem* Quartz = GetWorld()->GetSubsystem<UQuartzSubsystem>();
    if (!Quartz)
    {
        return;
    }

    FQuartzClockSettings ClockSettings;                 // FQuartzTimeSignature defaults to 4/4
    UQuartzClockHandle* Handle = Quartz->CreateNewClock(this, TEXT("MusicClock"), ClockSettings,
        /*bOverrideSettingsIfClockExists=*/false, /*bUseAudioEngineClockManager=*/true);
    ClockHandle = Handle;
    if (!Handle)
    {
        return;
    }

    Handle->StartClock(this, Handle);

    FOnQuartzMetronomeEventBP BeatDelegate;
    BeatDelegate.BindDynamic(this, &AMyMusicDirector::OnBeat);
    Handle->SubscribeToQuantizationEvent(this, EQuartzCommandQuantization::Beat,
        BeatDelegate, Handle);

    StingerComponent = UGameplayStatics::CreateSound2D(this, StingerSound, 1.0f, 1.0f, 0.0f,
        nullptr, /*bPersistAcrossLevelTransition=*/false, /*bAutoDestroy=*/false);
}

void AMyMusicDirector::TriggerStinger()
{
    if (!ClockHandle || !StingerComponent)
    {
        return;
    }

    FQuartzQuantizationBoundary Boundary;
    Boundary.Quantization           = EQuartzCommandQuantization::Bar;
    Boundary.Multiplier             = 1.0f;
    Boundary.CountingReferencePoint = EQuarztQuantizationReference::BarRelative;
    Boundary.bFireOnClockStart      = false;
    Boundary.bCancelCommandIfClockIsNotRunning = true;

    FOnQuartzCommandEventBP OnCommandEvent;
    UQuartzClockHandle* Handle = ClockHandle;
    StingerComponent->PlayQuantized(this, Handle, Boundary, OnCommandEvent,
        /*InStartTime=*/0.0f, /*InFadeInDuration=*/0.0f,
        /*InFadeVolumeLevel=*/1.0f, EAudioFaderCurve::Linear);
}

void AMyMusicDirector::OnBeat(FName ClockName, EQuartzCommandQuantization QuantizationType,
                              int32 NumBars, int32 Beat, float BeatFraction)
{
    // Runs on the game thread, already marshalled off the audio render thread.
}

void AMyMusicDirector::EndPlay(const EEndPlayReason::Type EndPlayReason)
{
    if (UQuartzClockHandle* Handle = ClockHandle)
    {
        Handle->StopClock(this, /*CancelPendingEvents=*/true, Handle);
    }
    Super::EndPlay(EndPlayReason);
}
```

`StartClock`, `StopClock`, `SubscribeToQuantizationEvent` and `PlayQuantized` take the handle as
`UQuartzClockHandle*&` (`Quartz/AudioMixerClockHandle.h:53,56,99`; `Components/AudioComponent.h:523`).
A `TObjectPtr` member cannot bind to that (its `T*&` conversion is `explicit` and deprecated,
`UObject/ObjectPtr.h:745-746`), so keep the `UPROPERTY` for GC and pass a raw local copy.

---

## Pattern 8: Building a MetaSound at Runtime

`UMetaSoundBuilderSubsystem` is a `UEngineSubsystem`. `Build(const FMetaSoundBuilderOptions&)` is
editor-only (`WITH_EDITORONLY_DATA`); at runtime either `Audition` the builder directly or create a
transient asset with `BuildNewMetaSound(FName NameBase)` (`MetasoundBuilderBase.h:586`).

```cpp
#include "MetasoundBuilderSubsystem.h"
#include "MetasoundBuilderBase.h"
#include "Components/AudioComponent.h"

void AMyProceduralEmitter::BuildAndAudition()
{
    UMetaSoundBuilderSubsystem* BuilderSubsystem = UMetaSoundBuilderSubsystem::Get();
    if (!BuilderSubsystem || !AudioComponent)
    {
        return;
    }

    FMetaSoundBuilderNodeOutputHandle OnPlayOutput;
    FMetaSoundBuilderNodeInputHandle OnFinishedInput;
    TArray<FMetaSoundBuilderNodeInputHandle> AudioOutInputs;
    EMetaSoundBuilderResult Result = EMetaSoundBuilderResult::Failed;

    UMetaSoundSourceBuilder* Builder = BuilderSubsystem->CreateSourceBuilder(
        TEXT("MyToneBuilder"), OnPlayOutput, OnFinishedInput, AudioOutInputs, Result,
        EMetaSoundOutputAudioFormat::Mono, /*bIsOneShot=*/false);

    if (Result != EMetaSoundBuilderResult::Succeeded || !Builder)
    {
        return;
    }

    // Expose a float graph input the game can drive through the parameter interface
    FName FloatType = TEXT("float");
    const FMetasoundFrontendLiteral DefaultFrequency =
        BuilderSubsystem->CreateFloatMetaSoundLiteral(220.0f, FloatType);
    Builder->AddGraphInputNode(TEXT("Frequency"), FloatType, DefaultFrequency, Result,
        /*bIsConstructorInput=*/false);

    FOnCreateAuditionGeneratorHandleDelegate OnCreateGenerator;
    OnCreateGenerator.BindDynamic(this, &AMyProceduralEmitter::OnGeneratorCreated);
    Builder->Audition(this, AudioComponent, OnCreateGenerator, /*bLiveUpdatesEnabled=*/true);

    AudioComponent->SetFloatParameter(FName("Frequency"), 440.0f);
}

// Declared in MyProceduralEmitter.h:
//   UFUNCTION()
//   void OnGeneratorCreated(UMetasoundGeneratorHandle* GeneratorHandle);
//   void HandleGeneratorDetached();
void AMyProceduralEmitter::HandleGeneratorDetached()
{
    // The generator went away (sound stopped or virtualized) — drop cached state here.
}

void AMyProceduralEmitter::OnGeneratorCreated(UMetasoundGeneratorHandle* GeneratorHandle)
{
    if (GeneratorHandle && GeneratorHandle->IsValid())
    {
        GeneratorHandle->OnGeneratorHandleDetached.AddUObject(
            this, &AMyProceduralEmitter::HandleGeneratorDetached);
    }
}
```

Node wiring uses `AddNodeByClassName(ClassName, Result, MajorVersion)`, `ConnectNodes`,
`ConnectNodeInputToGraphInput` and `SetNodeInputDefault`. The underlying document API is
`FMetaSoundFrontendDocumentBuilder` in `MetasoundFrontend/Public/MetasoundFrontendDocumentBuilder.h`.
Register builders you want to find again with `RegisterSourceBuilder` / `FindSourceBuilder`.

---

## Pattern 9: Modulation Control Bus for Gameplay Tension

A single control bus drives volume and filter cutoff across many sounds at once, without any
per-component bookkeeping.

```cpp
#include "AudioModulationStatics.h"
#include "SoundControlBus.h"
#include "SoundModulationParameter.h"
#include "Components/AudioComponent.h"

void AMyTensionDirector::BeginPlay()
{
    Super::BeginPlay();
    if (GetNetMode() == NM_DedicatedServer)
    {
        return;
    }

    // VolumeParameter is a USoundModulationParameterVolume asset. Leave Activate false: manual
    // activation is deprecated (AudioModulationStatics.h:81) — routing onto a playing sound activates it.
    TensionBus = UAudioModulationStatics::CreateBus(this, TEXT("TensionBus"),
        VolumeParameter, /*Activate=*/false);

    if (TensionBus && AmbientLoopComponent)
    {
        TSet<USoundModulatorBase*> Modulators;
        Modulators.Add(TensionBus);
        AmbientLoopComponent->SetModulationRouting(Modulators,
            EModulationDestination::Volume, EModulationRouting::Override);
    }
}

void AMyTensionDirector::SetTension(float Normalized)
{
    if (TensionBus)
    {
        UAudioModulationStatics::SetGlobalBusMixValue(this, TensionBus,
            FMath::Clamp(Normalized, 0.0f, 1.0f), /*FadeTime=*/0.5f);
    }
}

void AMyTensionDirector::EndPlay(const EEndPlayReason::Type EndPlayReason)
{
    if (TensionBus)
    {
        // DeactivateBus is UE_DEPRECATED(5.6) (AudioModulationStatics.h:227)
        UAudioModulationStatics::ClearGlobalBusMixValue(this, TensionBus);
    }
    Super::EndPlay(EndPlayReason);
}
```

`AddModulationRouting` unions a modulator into whatever is already routed; `SetModulationRouting`
replaces it. On assets and in native `Audio::FModulationDestination` code the setter is `SetModulators`.

---

## Pattern 10: Offline Beat Map with Audio Synesthesia NRT

`AnalyzeAudio()` is `WITH_EDITOR` only. Bake the data in an editor utility, store it in a data asset,
and read it at runtime — never run FFT per frame to find beats you already know about.

`AnalyzeAudio()` only starts a background task (`AudioAnalyzerNRT.cpp:180`); the result is set later
on the game thread, which then broadcasts `OnAnalysisComplete` (`WITH_EDITORONLY_DATA`,
`AudioAnalyzerNRT.h:174`). Read the analyzer from that callback, and hold it in a `UPROPERTY` until
then — the task only keeps a weak pointer to it.

```cpp
// MyBeatMapFactory.h — editor-only utility. Build.cs: "AudioSynesthesia", "AudioAnalyzer"
#pragma once
#include "CoreMinimal.h"
#include "UObject/Object.h"
#include "MyBeatMapFactory.generated.h"

class ULoudnessNRT;
class UOnsetNRT;
class USoundWave;

UCLASS()
class MYGAME_API UMyBeatMapFactory : public UObject
{
    GENERATED_BODY()

public:
#if WITH_EDITOR
    void BakeBeatMap(USoundWave* MusicWave);
#endif

    UPROPERTY()
    TArray<float> BeatTimes;

protected:
#if WITH_EDITOR
    UFUNCTION()
    void HandleOnsetsReady();

    UFUNCTION()
    void HandleLoudnessReady();
#endif

    UPROPERTY(Transient)
    TObjectPtr<UOnsetNRT> OnsetAnalyzer;

    UPROPERTY(Transient)
    TObjectPtr<ULoudnessNRT> LoudnessAnalyzer;
};
```

```cpp
// MyBeatMapFactory.cpp
#include "MyBeatMapFactory.h"
#include "LoudnessNRT.h"
#include "OnsetNRT.h"
#include "Sound/SoundWave.h"

#if WITH_EDITOR
void UMyBeatMapFactory::BakeBeatMap(USoundWave* MusicWave)
{
    if (!MusicWave)
    {
        return;
    }
    BeatTimes.Reset();

    OnsetAnalyzer = NewObject<UOnsetNRT>(this);
    OnsetAnalyzer->OnAnalysisComplete.AddDynamic(this, &UMyBeatMapFactory::HandleOnsetsReady);
    OnsetAnalyzer->Sound = MusicWave;
    OnsetAnalyzer->AnalyzeAudio();   // returns immediately; results arrive in HandleOnsetsReady

    LoudnessAnalyzer = NewObject<ULoudnessNRT>(this);
    ULoudnessNRTSettings* LoudnessSettings = NewObject<ULoudnessNRTSettings>(this);
    LoudnessSettings->AnalysisPeriod = 0.01f;   // editor range 0.01..0.25
    LoudnessAnalyzer->Settings = LoudnessSettings;
    LoudnessAnalyzer->OnAnalysisComplete.AddDynamic(this, &UMyBeatMapFactory::HandleLoudnessReady);
    LoudnessAnalyzer->Sound = MusicWave;
    LoudnessAnalyzer->AnalyzeAudio();
}

void UMyBeatMapFactory::HandleOnsetsReady()
{
    TArray<float> OnsetTimestamps;
    TArray<float> OnsetStrengths;
    OnsetAnalyzer->GetNormalizedChannelOnsetsBetweenTimes(
        0.0f, OnsetAnalyzer->DurationInSeconds, /*InChannel=*/0,
        OnsetTimestamps, OnsetStrengths);

    for (int32 Index = 0; Index < OnsetTimestamps.Num(); ++Index)
    {
        if (OnsetStrengths.IsValidIndex(Index) && OnsetStrengths[Index] > 0.5f)
        {
            BeatTimes.Add(OnsetTimestamps[Index]);
        }
    }
}

void UMyBeatMapFactory::HandleLoudnessReady()
{
    float Loudness = 0.0f;
    LoudnessAnalyzer->GetLoudnessAtTime(1.5f, Loudness);
}
#endif // WITH_EDITOR
```

`UAudioAnalyzerNRT::Sound` is a `USoundWave` whose editor picker disallows `UMetaSoundSource` and `USoundSourceBus` (`DisallowedClasses`, `AudioAnalyzerNRT.h:83`) — it needs a real wave.
For live reactions use submix spectral analysis (Pattern 6) instead.

---

## Asset Setup Checklist

For every 3D sound:

- [ ] `AttenuationSettings` assigned, with `FalloffDistance` tuned to gameplay scale
- [ ] `bAttenuate` and `bSpatialize` both enabled in the attenuation asset
- [ ] `DistanceAlgorithm` chosen deliberately (`NaturalSound` for most gameplay sources)
- [ ] `SoundClassObject` set — SFX, Music, Voice or Ambient
- [ ] `SoundSubmixObject` set, and `SoundSubmixSends` configured for reverb/analysis buses
- [ ] `ConcurrencySet` populated for anything that can fire more than twice a second
- [ ] `Priority` raised above 1.0 for sounds that must survive voice culling
- [ ] `bEnableOcclusion` considered for interior/exterior transitions
- [ ] `LoadingBehavior` set: `ForceInline` for short SFX, `LoadOnDemand` for long music
- [ ] `VirtualizationMode` chosen for loops that must resume in sync after going inaudible

For submix sends:

- [ ] `SendStage` matches intent (`PostDistanceAttenuation` for world reverb)
- [ ] `bEnableLPFCutoff` / `LPFCutoff` and `bEnableHPFCutoff` / `HPFCutoff` set explicitly

For MetaSound sources:

- [ ] Every gameplay-driven input declared with a name matching the C++ `FName` exactly
- [ ] Sensible defaults on each graph input, so a missed `SetFloatParameter` is not silence
- [ ] `USoundConcurrency` assigned on the `UMetaSoundSource` asset
- [ ] Attenuation assigned if the source plays in world space
