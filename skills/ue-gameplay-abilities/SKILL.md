---
name: ue-gameplay-abilities
description: "Use when implementing or debugging the Gameplay Ability System in C++ — abilities, effects, attributes, cues and the AbilitySystemComponent. Also use when the user mentions 'GAS', 'UGameplayAbility', 'ActivateAbility', 'CommitAbility', 'UGameplayEffect', 'UAttributeSet', 'FGameplayAttributeData', 'AbilitySystemComponent', 'InitAbilityActorInfo', 'GiveAbility', 'TryActivateAbility', 'ability task', 'GameplayCue', 'cooldown', 'cost', 'buff', 'debuff', 'damage execution' or 'ASC on PlayerState'. For montage playback, see ue-animation-system; for replication modes and prediction, see ue-networking-replication."
metadata:
  version: "2.0.0"
  engine: "5.8"
---

# UE Gameplay Ability System

Target engine: **UE 5.8**. APIs below are verified against the 5.8 headers; older forms are listed under "Deprecated — do not use".

GAS ships as the `GameplayAbilities` plugin (`Engine/Plugins/Runtime/GameplayAbilities`), which is production-ready, not Experimental. Enable it in the `.uproject`, then add `"GameplayAbilities"`, `"GameplayTags"` and `"GameplayTasks"` to `PublicDependencyModuleNames` in your module's `Build.cs`. This skill covers `UAbilitySystemComponent`, `UGameplayAbility`, `UGameplayEffect`, `UAttributeSet`, `UAbilityTask` and Gameplay Cues.

## Context

Read `.agents/ue-project-context.md` if it exists (module names, conventions, enabled plugins, GAS/networking setup). Do not stop if it is missing.

Identify the area from the request and the codebase. Ask only when two plausible readings would produce different code.

| Request is about… | Go to |
|---|---|
| Where the ASC lives, init order, respawn, replication mode | [ASC Ownership and Initialization](#asc-ownership-and-initialization) |
| Writing an ability, activation, cost, cooldown, policies | [Gameplay Abilities](#gameplay-abilities) |
| Waiting on montages, events, targeting, timers, tag queries | [Ability Tasks](#ability-tasks) |
| Damage, buffs, stacking, executions, immunity, GE components | [Gameplay Effects](#gameplay-effects) |
| Health/mana floats, clamping, attribute replication | [Attribute Sets](#attribute-sets) |
| Tags that block, grant or cancel abilities; loose tags | [Tags in GAS](#tags-in-gas) |
| Particles, sounds, hit impacts, cue notify classes | [Gameplay Cues](#gameplay-cues) |
| Cue scan paths, global curve tables, project config | [Project Settings](#project-settings) |

## GAS Architecture

| Pillar | Class | Header | Purpose |
|---|---|---|---|
| Component | `UAbilitySystemComponent` | `AbilitySystemComponent.h` | Owns abilities, effects, attributes, tags |
| Abilities | `UGameplayAbility` | `Abilities/GameplayAbility.h` | What happens when activated |
| Effects | `UGameplayEffect` | `GameplayEffect.h` | Data-driven attribute mutation |
| Attributes | `UAttributeSet` | `AttributeSet.h` | Replicated `FGameplayAttributeData` stats |
| Tasks | `UAbilityTask` | `Abilities/Tasks/AbilityTask.h` | Latent steps inside an ability |

Gameplay tags thread through all of them as requirements, grants and blockers.

## ASC Ownership and Initialization

| Owner | When | Notes |
|---|---|---|
| `APlayerState` | Networked player characters | ASC survives pawn death; use `Mixed` replication |
| `APawn` / `ACharacter` | AI, single-player | Simplest; use `Minimal` for AI, `Full` if a pawn-owned ASC needs full data on all clients |

Every actor that owns or exposes an ASC implements `IAbilitySystemInterface`:

```cpp
// MyCharacter.h
#pragma once

#include "GameFramework/Character.h"
#include "AbilitySystemInterface.h"
#include "MyCharacter.generated.h"

class UAbilitySystemComponent;

UCLASS()
class MYGAME_API AMyCharacter : public ACharacter, public IAbilitySystemInterface
{
    GENERATED_BODY()

public:
    AMyCharacter();

    virtual UAbilitySystemComponent* GetAbilitySystemComponent() const override;

protected:
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "GAS")
    TObjectPtr<UAbilitySystemComponent> AbilitySystemComponent;
};
```

```cpp
// MyCharacter.cpp
#include "MyCharacter.h"
#include "AbilitySystemComponent.h"

AMyCharacter::AMyCharacter()
{
    AbilitySystemComponent = CreateDefaultSubobject<UAbilitySystemComponent>(TEXT("AbilitySystemComponent"));
    AbilitySystemComponent->SetIsReplicated(true);
    AbilitySystemComponent->SetReplicationMode(EGameplayEffectReplicationMode::Mixed);
}

UAbilitySystemComponent* AMyCharacter::GetAbilitySystemComponent() const
{
    return AbilitySystemComponent;
}
```

`InitAbilityActorInfo(OwnerActor, AvatarActor)` must run on **both** server and client. For a pawn-owned ASC that is `BeginPlay` with `(this, this)`. For a PlayerState-owned ASC it is `PossessedBy` on the server and `OnRep_PlayerState` on the client, with `(PlayerState, Character)`. Full dual-path code, respawn handling and the AI variant are in [gas-setup-patterns.md](references/gas-setup-patterns.md).

| `EGameplayEffectReplicationMode` | Effect on simulated proxies |
|---|---|
| `Minimal` | No GE replication; minimal tag/cue data only. AI and non-player actors |
| `Mixed` | Full GEs to the owning connection, minimal to everyone else. Player characters |
| `Full` | Every GE replicates to every client |

`Mixed` sends full GE data to the connection that owns the ASC's owner actor (`GameplayEffect.cpp:5240`), so it works for a PlayerState-owned ASC and for a pawn-owned ASC once a PlayerController possesses the pawn. Choosing `Full` over `Mixed` is a bandwidth decision, not a correctness one. `Minimal` does not work for player-owned ASCs (`AbilitySystemComponent.h:83`). See `ue-networking-replication` for what each mode costs on the wire.

## Gameplay Abilities

```cpp
// MyFireballAbility.h
#pragma once

#include "Abilities/GameplayAbility.h"
#include "MyFireballAbility.generated.h"

class UGameplayEffect;

UCLASS()
class MYGAME_API UMyFireballAbility : public UGameplayAbility
{
    GENERATED_BODY()

public:
    UMyFireballAbility();

    virtual bool CanActivateAbility(const FGameplayAbilitySpecHandle Handle,
        const FGameplayAbilityActorInfo* ActorInfo,
        const FGameplayTagContainer* SourceTags = nullptr,
        const FGameplayTagContainer* TargetTags = nullptr,
        OUT FGameplayTagContainer* OptionalRelevantTags = nullptr) const override;

    virtual void ActivateAbility(const FGameplayAbilitySpecHandle Handle,
        const FGameplayAbilityActorInfo* ActorInfo,
        const FGameplayAbilityActivationInfo ActivationInfo,
        const FGameplayEventData* TriggerEventData) override;

    virtual void EndAbility(const FGameplayAbilitySpecHandle Handle,
        const FGameplayAbilityActorInfo* ActorInfo,
        const FGameplayAbilityActivationInfo ActivationInfo,
        bool bReplicateEndAbility, bool bWasCancelled) override;

    virtual void CancelAbility(const FGameplayAbilitySpecHandle Handle,
        const FGameplayAbilityActorInfo* ActorInfo,
        const FGameplayAbilityActivationInfo ActivationInfo,
        bool bReplicateCancelAbility) override;

protected:
    UPROPERTY(EditDefaultsOnly, Category = "GAS")
    TSubclassOf<UGameplayEffect> DamageEffect;
};
```

Constructor sets the policies and the ability's own tags:

```cpp
// MyFireballAbility.cpp
#include "MyFireballAbility.h"
#include "AbilitySystemBlueprintLibrary.h"
#include "AbilitySystemComponent.h"

UMyFireballAbility::UMyFireballAbility()
{
    InstancingPolicy = EGameplayAbilityInstancingPolicy::InstancedPerActor;
    NetExecutionPolicy = EGameplayAbilityNetExecutionPolicy::LocalPredicted;
    NetSecurityPolicy = EGameplayAbilityNetSecurityPolicy::ClientOrServer;

    FGameplayTagContainer Tags;
    Tags.AddTag(FGameplayTag::RequestGameplayTag(TEXT("Ability.Skill.Fireball")));
    SetAssetTags(Tags);

    ActivationOwnedTags.AddTag(FGameplayTag::RequestGameplayTag(TEXT("Ability.Active.Casting")));
    ActivationBlockedTags.AddTag(FGameplayTag::RequestGameplayTag(TEXT("State.Stunned")));
    CancelAbilitiesWithTag.AddTag(FGameplayTag::RequestGameplayTag(TEXT("Ability.Active.Melee")));
}
```

`SetAssetTags` is constructor-only; read them back with `GetAssetTags()`.

| Enum | Values | Use |
|---|---|---|
| `EGameplayAbilityInstancingPolicy` | `InstancedPerActor`, `InstancedPerExecution` | `InstancedPerActor` is the default choice; per-execution when several copies may run at once |
| `EGameplayAbilityNetExecutionPolicy` | `LocalPredicted`, `LocalOnly`, `ServerInitiated`, `ServerOnly` | `LocalPredicted` for responsive player abilities |
| `EGameplayAbilityNetSecurityPolicy` | `ClientOrServer`, `ServerOnlyExecution`, `ServerOnlyTermination`, `ServerOnly` | Rejects client requests the server should own |

```cpp
void UMyFireballAbility::ActivateAbility(const FGameplayAbilitySpecHandle Handle,
    const FGameplayAbilityActorInfo* ActorInfo,
    const FGameplayAbilityActivationInfo ActivationInfo,
    const FGameplayEventData* TriggerEventData)
{
    Super::ActivateAbility(Handle, ActorInfo, ActivationInfo, TriggerEventData);

    // CommitAbility applies cost and cooldown; it fails if either cannot be paid.
    if (!CommitAbility(Handle, ActorInfo, ActivationInfo))
    {
        EndAbility(Handle, ActorInfo, ActivationInfo, true, true);
        return;
    }

    const FGameplayEffectSpecHandle Spec =
        MakeOutgoingGameplayEffectSpec(DamageEffect, GetAbilityLevel());
    const FGameplayAbilityTargetDataHandle TargetData =
        UAbilitySystemBlueprintLibrary::AbilityTargetDataFromActor(ActorInfo->AvatarActor.Get());

    ApplyGameplayEffectSpecToTarget(Handle, ActorInfo, ActivationInfo, Spec, TargetData);
    EndAbility(Handle, ActorInfo, ActivationInfo, true, false);
}
```

Split the commit when the two halves must happen at different times: `CommitAbilityCost(Handle, ActorInfo, ActivationInfo)` and `CommitAbilityCooldown(Handle, ActorInfo, ActivationInfo, ForceCooldown)` — note the cooldown overload takes the extra `const bool ForceCooldown`.

Inside an instanced ability, `GetAbilitySystemComponentFromActorInfo_Ensured()` returns the ASC and logs an ensure instead of crashing when actor info is not set up.

Granting and activating, from the server:

```cpp
const FGameplayAbilitySpecHandle Handle =
    ASC->GiveAbility(FGameplayAbilitySpec(UMyFireballAbility::StaticClass(), 1));

ASC->TryActivateAbility(Handle);
ASC->TryActivateAbilityByClass(UMyFireballAbility::StaticClass());
ASC->TryActivateAbilitiesByTag(
    FGameplayTagContainer(FGameplayTag::RequestGameplayTag(TEXT("Ability.Skill.Fireball"))));
ASC->ClearAbility(Handle);
```

## Ability Tasks

Ability tasks make an ability span frames. The pattern is always: static factory → bind delegates → `ReadyForActivation()` (inherited from `UGameplayTask`). Task delegates are dynamic, so each handler needs `UFUNCTION()`:

```cpp
// In the ability's header:
UFUNCTION()
void HandleMontageCompleted();

UFUNCTION()
void HandleMontageInterrupted();
```

```cpp
#include "Abilities/Tasks/AbilityTask_PlayMontageAndWait.h"

UAbilityTask_PlayMontageAndWait* Task =
    UAbilityTask_PlayMontageAndWait::CreatePlayMontageAndWaitProxy(
        this, NAME_None, AttackMontage, 1.f);
Task->OnCompleted.AddDynamic(this, &UMyFireballAbility::HandleMontageCompleted);
Task->OnInterrupted.AddDynamic(this, &UMyFireballAbility::HandleMontageInterrupted);
Task->ReadyForActivation();
```

| Task | Waits for |
|---|---|
| `UAbilityTask_PlayMontageAndWait` | An `UAnimMontage` to finish, blend out, interrupt or cancel |
| `UAbilityTask_PlayAnimAndWait` | A raw `UAnimSequence` played into a slot, for linked anim layers |
| `UAbilityTask_WaitGameplayEvent` | A gameplay event tag sent to the ASC (anim notify, hit registration) |
| `UAbilityTask_WaitDelay` | A fixed number of seconds |
| `UAbilityTask_WaitTargetData` | An `AGameplayAbilityTargetActor` to confirm targets |
| `UAbilityTask_WaitAttributeChange` / `...ChangeThreshold` | An attribute to change or cross a comparison |
| `UAbilityTask_WaitGameplayTagQuery` | An `FGameplayTagQuery` over the target's tags to become true or false |
| `UAbilityTask_WaitGameplayTagCountChanged` | The count of one tag on the ASC to change |
| `UAbilityTask_WaitGameplayEffectApplied_Self` / `_Target` | A GE matching tag requirements (or an `FGameplayTagQuery` via the `_Query` factories) |

Full factory signatures, delegate names, the custom-task template and the task pitfalls are in [ability-task-reference.md](references/ability-task-reference.md).

## Gameplay Effects

| `EGameplayEffectDurationType` | Behaviour |
|---|---|
| `Instant` | Executes once against the **base** value; fires `PostGameplayEffectExecute` |
| `HasDuration` | Aggregates onto the **current** value for `DurationMagnitude` seconds |
| `Infinite` | Aggregates until removed explicitly |

```cpp
const FGameplayEffectContextHandle Ctx = ASC->MakeEffectContext();
const FGameplayEffectSpecHandle Spec =
    ASC->MakeOutgoingSpec(UMyDamageEffect::StaticClass(), Level, Ctx);

Spec.Data->SetSetByCallerMagnitude(
    FGameplayTag::RequestGameplayTag(TEXT("SetByCaller.Damage")), 75.f);

ASC->ApplyGameplayEffectSpecToSelf(*Spec.Data.Get());
ASC->ApplyGameplayEffectSpecToTarget(*Spec.Data.Get(), TargetASC);

ASC->RemoveActiveGameplayEffect(ActiveHandle);
ASC->RemoveActiveGameplayEffectBySourceEffect(UMyDamageEffect::StaticClass(), nullptr);
```

Modifier ops live in `EGameplayModOp` (a namespaced enum, so values are `TEnumAsByte<EGameplayModOp::Type>`). The aggregation equation is
`((BaseValue + AddBase) * MultiplyAdditive / DivideAdditive * MultiplyCompound) + AddFinal`.

| Value | Meaning |
|---|---|
| `AddBase` | Added to the base before anything else |
| `MultiplyAdditive` | Multipliers summed, then multiplied against the running result |
| `DivideAdditive` | Divisors summed, then divided against the running result |
| `MultiplyCompound` | Multiplied into the running result, compounding with other compounds |
| `AddFinal` | Added after all multiplication |
| `Override` | Replaces the computed value |

Behaviour that used to be monolithic properties on `UGameplayEffect` now lives in `UGameplayEffectComponent` subclasses under `GameplayEffectComponents/`. Stacking, magnitude calculations, `UGameplayEffectExecutionCalculation`, periodic effects, immunity, queries, lifecycle delegates and the full component list are in [gameplay-effect-reference.md](references/gameplay-effect-reference.md).

## Attribute Sets

```cpp
// MyHealthSet.h
#pragma once

#include "AttributeSet.h"
#include "AbilitySystemComponent.h"
#include "MyHealthSet.generated.h"

UCLASS()
class MYGAME_API UMyHealthSet : public UAttributeSet
{
    GENERATED_BODY()

public:
    UMyHealthSet();

    virtual void PreAttributeChange(const FGameplayAttribute& Attribute, float& NewValue) override;
    virtual void PostGameplayEffectExecute(const struct FGameplayEffectModCallbackData& Data) override;
    virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override;

    UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_Health, Category = "Health")
    FGameplayAttributeData Health;
    ATTRIBUTE_ACCESSORS_BASIC(UMyHealthSet, Health)

    UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_MaxHealth, Category = "Health")
    FGameplayAttributeData MaxHealth;
    ATTRIBUTE_ACCESSORS_BASIC(UMyHealthSet, MaxHealth)

protected:
    UFUNCTION()
    void OnRep_Health(const FGameplayAttributeData& OldHealth);

    UFUNCTION()
    void OnRep_MaxHealth(const FGameplayAttributeData& OldMaxHealth);
};
```

`ATTRIBUTE_ACCESSORS_BASIC` generates `GetHealthAttribute()`, `GetHealth()`, `SetHealth()` and `InitHealth()`. Projects that want extra accessors define their own macro chaining `GAMEPLAYATTRIBUTE_PROPERTY_GETTER`, `GAMEPLAYATTRIBUTE_VALUE_GETTER`, `GAMEPLAYATTRIBUTE_VALUE_SETTER` and `GAMEPLAYATTRIBUTE_VALUE_INITTER`.

Each `OnRep_` body is one `GAMEPLAYATTRIBUTE_REPNOTIFY(UMyHealthSet, Health, OldHealth)` call, and every attribute needs a `DOREPLIFETIME_CONDITION_NOTIFY(UMyHealthSet, Health, COND_None, REPNOTIFY_Always)` line. The full `.cpp` with clamping is in [gas-setup-patterns.md](references/gas-setup-patterns.md#attributeset-implementation).

| Hook | Fires | Use for |
|---|---|---|
| `PreAttributeChange` | Every current-value recompute | Clamping the current value only |
| `PreAttributeBaseChange` | Before a base-value write (`const`) | Clamping the base value |
| `PostGameplayEffectExecute` | After an instant GE, or one period of a periodic GE, writes the base value | Death, hit reactions, game events |
| `PostAttributeChange` | After a current value changed | Reacting to derived state |

Registering and reading:

```cpp
// Constructor of the ASC owner — the ASC finds subobject attribute sets automatically:
HealthSet = CreateDefaultSubobject<UMyHealthSet>(TEXT("HealthSet"));

// Runtime, authority only:
UMyHealthSet* NewSet = NewObject<UMyHealthSet>(this);
ASC->AddSpawnedAttribute(NewSet);

const float CurrentHealth = ASC->GetNumericAttribute(UMyHealthSet::GetHealthAttribute());
```

One ASC can host several `UAttributeSet` subclasses, but only one instance per class.

## Tags in GAS

Tag definition (native tag macros, `.ini` tag sources), container matching and `FGameplayTagQuery` semantics belong to `ue-gameplay-tags-messaging`. What is GAS-specific:

```cpp
// On the ability CDO (constructor): tags granted while the ability runs, and the gates.
ActivationOwnedTags.AddTag(CastingTag);
ActivationRequiredTags.AddTag(WeaponEquippedTag);
ActivationBlockedTags.AddTag(StunnedTag);
BlockAbilitiesWithTag.AddTag(MeleeTag);
CancelAbilitiesWithTag.AddTag(ChannelTag);

// On the ASC: query and react.
ASC->HasMatchingGameplayTag(StunnedTag);
const FGameplayTagContainer& Owned = ASC->GetOwnedGameplayTags();
ASC->RegisterGameplayTagEvent(StunnedTag, EGameplayTagEventType::NewOrRemoved)
    .AddUObject(this, &AMyCharacter::HandleStunnedTagChanged);
```

The callback is declared as `void HandleStunnedTagChanged(const FGameplayTag Tag, int32 NewCount);` and needs no `UFUNCTION()` — tag events are plain multicast delegates.

Loose tags are set directly instead of through a GE. The third argument chooses how the tag replicates:

```cpp
ASC->AddLooseGameplayTag(StunnedTag);                                              // local only
ASC->AddLooseGameplayTag(BuffedTag, 1, EGameplayTagReplicationState::TagOnly);     // tag to all
ASC->AddLooseGameplayTag(BuffedTag, 1, EGameplayTagReplicationState::CountToOwner);// + count to owner
ASC->RemoveLooseGameplayTag(BuffedTag, 1, EGameplayTagReplicationState::TagOnly);
ASC->SetLooseGameplayTagCount(BuffedTag, 3, EGameplayTagReplicationState::TagAndCountToAll);
```

`EGameplayTagReplicationState` values: `None`, `SimulatedTagOnly`, `TagOnly`, `CountToOwner`, `TagAndCountToAll`. Anything other than `None` must be added on the server.

## Gameplay Cues

Cues are cosmetic only. Tag prefix `GameplayCue.`; a GE triggers them through its `GameplayCues` array, or code fires them directly:

```cpp
ASC->ExecuteGameplayCue(HitCueTag, ASC->MakeEffectContext());   // one-shot
ASC->AddGameplayCue(BuffCueTag);                                // persistent
ASC->RemoveGameplayCue(BuffCueTag);
const bool bActive = ASC->IsGameplayCueActive(BuffCueTag);
```

| Class | Base | Override | Use |
|---|---|---|---|
| `UGameplayCueNotify_Static` | `UObject` | `OnExecute_Implementation` | Burst cue with no spawned actor |
| `UGameplayCueNotify_Burst` | `UGameplayCueNotify_Static` | data-driven burst effects | One-shot particles/sounds, no code |
| `UGameplayCueNotify_HitImpact` | `UGameplayCueNotify_Static` | data-driven impact effects | Surface-typed impacts |
| `AGameplayCueNotify_Actor` | `AActor` | `OnActive_`/`WhileActive_`/`OnRemove_Implementation` | Persistent cue needing an actor |
| `AGameplayCueNotify_BurstLatent` | `AGameplayCueNotify_Actor` | data-driven burst effects | Burst that must spawn an actor |
| `AGameplayCueNotify_Looping` | `AGameplayCueNotify_Actor` | data-driven looping effects | Looping auras and channels |

```cpp
// MyHitCue.h
#pragma once

#include "GameplayCueNotify_Static.h"
#include "MyHitCue.generated.h"

UCLASS()
class MYGAME_API UMyHitCue : public UGameplayCueNotify_Static
{
    GENERATED_BODY()

public:
    virtual bool OnExecute_Implementation(AActor* MyTarget,
        const FGameplayCueParameters& Parameters) const override;
};
```

`UGameplayCueManager` discovers cue assets under the scan paths configured in project settings.

## Project Settings

GAS configuration is a `UDeveloperSettings` class: `UGameplayAbilitiesDeveloperSettings` (Project Settings → Gameplay Abilities). Read it with `GetDefault<UGameplayAbilitiesDeveloperSettings>()`. Notable fields: `GameplayCueNotifyPaths`, `ReplicateActivationOwnedTags`, `bAllowGameplayModEvaluationChannels`, plus the global curve/attribute tables. Runtime helpers stay on `UAbilitySystemGlobals` (`GetGameplayCueNotifyPaths()`, `AddGameplayCueNotifyPath()`, `GetAttributeSetInitter()`).

## Deprecated — do not use

| Do not emit | Use in 5.8 | Source |
|---|---|---|
| `AGameplayCueNotify_Static` | `UGameplayCueNotify_Static` (a `UObject`) | Class never existed; `GameplayCueNotify_Static.h:19` |
| `EGameplayModOp::Multiplicative` | `EGameplayModOp::MultiplyAdditive` (legacy alias is spelled `Multiplicitive`) | `GameplayEffectTypes.h:112-148` |
| `EGameplayModOp::Additive` / `Division` | `AddBase` / `DivideAdditive` | Hidden back-compat aliases, `GameplayEffectTypes.h:141-143` |
| `EGameplayAbilityInstancingPolicy::NonInstanced` | `InstancedPerActor` | `UE_DEPRECATED_FORGAME(5.5)` in `Abilities/GameplayAbilityTypes.h:46` |
| `UGameplayAbility::AbilityTags` | `GetAssetTags()` / `SetAssetTags()` | `UE_DEPRECATED_FORGAME(5.5)` in `Abilities/GameplayAbility.h:474` |
| `GetAbilitySystemComponentFromActorInfo_Checked()` | `GetAbilitySystemComponentFromActorInfo_Ensured()` | `UE_DEPRECATED(5.5)` in `Abilities/GameplayAbility.h:168` |
| `NetUpdateFrequency` assignment on an actor | `SetNetUpdateFrequency(100.f)` | `UE_DEPRECATED(5.5)` in `GameFramework/Actor.h:903` |
| `UGameplayEffect::StackingType` (public read) | `GetStackingType()` | `UE_DEPRECATED(5.7)` in `GameplayEffect.h:2407` |
| `UGameplayEffect::GrantedApplicationImmunityTags` | `UImmunityGameplayEffectComponent` | `UE_DEPRECATED(5.3)` in `GameplayEffect.h:2379` |
| `AddReplicatedLooseGameplayTag` | `AddLooseGameplayTag(Tag, Count, EGameplayTagReplicationState::CountToOwner)` (the deprecation message's replacement; `TagOnly` if no client needs the count) | Function removed (no declaration in 5.8); the backing `ReplicatedLooseTags` member is `UE_DEPRECATED(5.7)` at `AbilitySystemComponent.h:1941` |
| `FGameplayAbilitySpec::DynamicAbilityTags` | `GetDynamicSpecSourceTags()` | `UE_DEPRECATED(5.5)` in `GameplayAbilitySpec.h:242` |
| `UAbilitySystemBlueprintLibrary::MakeSpecHandle` | `MakeSpecHandleByClass` | `UE_DEPRECATED(5.5)` in `AbilitySystemBlueprintLibrary.h:249` |
| `UAbilitySystemGlobals` config members | `GetDefault<UGameplayAbilitiesDeveloperSettings>()` | `UE_DEPRECATED(5.5)` throughout `AbilitySystemGlobals.h` |
| `FGameplayEffectSpec::GetModifierMagnitude(ModifierIdx, bFactorInStackCount)` | `GetModifierMagnitude(int32 ModifierIdx)` | `UE_DEPRECATED(5.6)` in `GameplayEffect.h:1153` |
| `FActiveGameplayEffectHandle(int32)`, `ResetGlobalHandleMap()`, `RemoveFromGlobalMap()` | `FActiveGameplayEffectHandle::GenerateNewHandle()` or `GetInstantExecutedHandle()`; the global map is gone | `UE_DEPRECATED(5.8)` in `ActiveGameplayEffectHandle.h:30,66,69` |

## Common Mistakes

**Implementing `IAbilitySystemInterface` only on the pawn:** when the ASC lives on the PlayerState, the PlayerState must implement the interface too, otherwise `UAbilitySystemBlueprintLibrary::GetAbilitySystemComponent` fails on the owner.

**Calling `InitAbilityActorInfo` only on the server:** clients need it as well. Server path is `PossessedBy`, client path is `OnRep_PlayerState`. Without the client call, attribute change delegates never fire on the owning client.

**Applying GEs before init:** granting abilities or applying effects before `InitAbilityActorInfo` runs leaves the ASC without actor info and the attribute sets unregistered.

**Clamping in the wrong hook:** `PreAttributeChange` fires on every aggregator recompute, so it may only clamp `NewValue`. Death, hit reactions and gameplay events belong in `PostGameplayEffectExecute`, which only runs for executions (instant GEs and periodic ticks).

**Forgetting `CommitAbility`:** the ability runs, but no cost is paid and no cooldown starts.

**Assuming loose tags replicate:** `AddLooseGameplayTag(Tag)` defaults to `EGameplayTagReplicationState::None`. Pass `TagOnly` or `CountToOwner` from the server, or grant the tag through a GE.

**Stacks silently capped:** applications beyond `GetStackLimitCount()` are rejected without an error. Check `GetCurrentStackCount(Handle)` before assuming a stack landed.

**Startup effects in `BeginPlay` without an authority check:** late joiners re-apply them. Gate every `GiveAbility` and startup GE behind `HasAuthority()`.

**AI with no PlayerState:** put the ASC on the AI character, call `InitAbilityActorInfo(this, this)` and set `Minimal` replication.

## Related Skills

- `ue-gameplay-tags-messaging` — tag definition (native tag macros, ini tag sources), containers, `FGameplayTagQuery` and gameplay messaging
- `ue-networking-replication` — `DOREPLIFETIME` variants, RPCs, prediction keys, push model
- `ue-animation-system` — montage assets, notifies and slots that the montage ability tasks drive
- `ue-character-movement` — movement modes that root-motion ability tasks manipulate
- `ue-cpp-foundations` — `UPROPERTY`/`UFUNCTION` specifiers, delegates, `TSubclassOf`, subsystems
- `ue-data-assets-tables` — `UCurveTable` and `UDataTable` assets behind attribute initialisation and `FScalableFloat`
- `ue-state-trees` — running abilities from State Tree tasks
- `ue-actor-component-architecture` — component creation, subobject registration, replication of subobjects
- `ue-gameplay-framework` — PlayerState and Pawn lifecycle, and damage without GAS (`TakeDamage`, `UDamageType`)
- `ue-input-system` — Enhanced Input actions bound to ability input ids
- `ue-game-features` — granting abilities and attribute sets from a game feature plugin
- `ue-niagara-effects` — Niagara systems, user parameters, data interfaces and data channels
- `ue-ui-umg-slate` — UMG widgets, Slate, Common UI and MVVM
