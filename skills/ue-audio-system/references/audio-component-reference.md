# UAudioComponent Routing and Delegates

Code for the per-instance routing calls and the delegate bindings summarised in [UAudioComponent Control](../SKILL.md#uaudiocomponent-control). Everything here is declared in `Components/AudioComponent.h`.

## Per-instance routing and filtering

These override the asset's routing for one playing instance:

```cpp
AudioComponent->SetSubmixSend(ReverbSubmix, 0.4f);                 // USoundSubmixBase*, level
AudioComponent->SetSourceBusSendPreEffect(DuckingSourceBus, 1.0f); // also PostEffect / PostAttenuation
AudioComponent->SetAudioBusSendPostEffect(AnalysisAudioBus, 1.0f);
AudioComponent->SetLowPassFilterEnabled(true);
AudioComponent->SetLowPassFilterFrequency(800.0f);
AudioComponent->SetHighPassFilterEnabled(true);
AudioComponent->SetHighPassFilterFrequency(120.0f);
AudioComponent->SetOutputToBusOnly(false);

// Inline attenuation instead of an asset
AudioComponent->bOverrideAttenuation = true;
AudioComponent->AttenuationOverrides.bAttenuate = true;
AudioComponent->AttenuationOverrides.bSpatialize = true;
AudioComponent->AttenuationOverrides.FalloffDistance = 3000.0f;
```

## Delegates

Binding the finish and playback-percent delegates, and unbinding them:

```cpp
// In the class body:
UFUNCTION()
void OnSoundFinished();

UFUNCTION()
void OnPlaybackPercent(const USoundWave* PlayingSoundWave, const float PlaybackPercent);

void OnSoundFinishedNative(UAudioComponent* Component);

// In BeginPlay:
AudioComponent->OnAudioFinished.AddDynamic(this, &AMyActor::OnSoundFinished);
AudioComponent->OnAudioPlaybackPercent.AddDynamic(this, &AMyActor::OnPlaybackPercent);
AudioComponent->OnAudioFinishedNative.AddUObject(this, &AMyActor::OnSoundFinishedNative);

// In EndPlay:
AudioComponent->OnAudioFinished.RemoveAll(this);
AudioComponent->OnAudioPlaybackPercent.RemoveAll(this);
AudioComponent->OnAudioFinishedNative.RemoveAll(this);
```

The other delegates with a `...Native` twin (`AudioComponent.h:441-481`) are `OnAudioPlayStateChanged`, `OnAudioVirtualizationChanged`, `OnAudioSingleEnvelopeValue` and `OnAudioMultiEnvelopeValue`. `OnQueueSubtitles` (`:485`) has no twin: it is a single-cast dynamic delegate (`DECLARE_DYNAMIC_DELEGATE_TwoParams`, `:67`), bound with `BindDynamic`.
