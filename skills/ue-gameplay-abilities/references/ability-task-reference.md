# Ability Task Reference

Built-in `UAbilityTask` subclasses with their 5.8 factory signatures, plus the custom-task template.
Target engine: UE 5.8. Headers: `Abilities/Tasks/*.h`.

---

## The task pattern

1. Call the static factory (never `NewObject` and never `new`).
2. Bind every delegate you care about.
3. Call `ReadyForActivation()` — inherited from `UGameplayTask`, not declared on `UAbilityTask`.
4. The ability stays active; the task calls back when its event fires.
5. In the callback, run the follow-up logic and call `EndAbility` when the ability is finished.

```cpp
// Declarations inside the ability class:
UFUNCTION()
void HandleMontageCompleted();

UFUNCTION()
void HandleMontageInterrupted();
```

```cpp
void UMyAbility::ActivateAbility(const FGameplayAbilitySpecHandle Handle,
    const FGameplayAbilityActorInfo* ActorInfo,
    const FGameplayAbilityActivationInfo ActivationInfo,
    const FGameplayEventData* TriggerEventData)
{
    Super::ActivateAbility(Handle, ActorInfo, ActivationInfo, TriggerEventData);

    if (!CommitAbility(Handle, ActorInfo, ActivationInfo))
    {
        EndAbility(Handle, ActorInfo, ActivationInfo, true, true);
        return;
    }

    UAbilityTask_PlayMontageAndWait* Task =
        UAbilityTask_PlayMontageAndWait::CreatePlayMontageAndWaitProxy(
            this, NAME_None, AttackMontage, 1.f);

    Task->OnCompleted.AddDynamic(this, &UMyAbility::HandleMontageCompleted);
    Task->OnInterrupted.AddDynamic(this, &UMyAbility::HandleMontageInterrupted);
    Task->ReadyForActivation();
}

void UMyAbility::HandleMontageCompleted()
{
    EndAbility(CurrentSpecHandle, CurrentActorInfo, CurrentActivationInfo, true, false);
}

void UMyAbility::HandleMontageInterrupted()
{
    EndAbility(CurrentSpecHandle, CurrentActorInfo, CurrentActivationInfo, true, true);
}
```

Delegate bindings use `AddDynamic` because ability-task delegates are `DECLARE_DYNAMIC_MULTICAST_DELEGATE`
types, so every bound function needs `UFUNCTION()`. The handlers used by the snippets below are
declared in the ability's header like this:

```cpp
UFUNCTION()
void HandleAttackHitEvent(FGameplayEventData Payload);

UFUNCTION()
void HandleDelayFinished();

UFUNCTION()
void HandleValidTargetData(const FGameplayAbilityTargetDataHandle& Data);

UFUNCTION()
void HandleTargetingCancelled(const FGameplayAbilityTargetDataHandle& Data);

UFUNCTION()
void HandleHealthChanged();

UFUNCTION()
void HandleHealthBelowThreshold(bool bMatchesComparison, float CurrentValue);

UFUNCTION()
void HandleEffectApplied(AActor* Source, FGameplayEffectSpecHandle SpecHandle,
    FActiveGameplayEffectHandle ActiveHandle);
```

---

## UAbilityTask_PlayMontageAndWait

```cpp
#include "Abilities/Tasks/AbilityTask_PlayMontageAndWait.h"

UAbilityTask_PlayMontageAndWait* Task =
    UAbilityTask_PlayMontageAndWait::CreatePlayMontageAndWaitProxy(
        this,          // OwningAbility
        NAME_None,     // TaskInstanceName
        AttackMontage, // UAnimMontage*
        1.f,           // Rate
        NAME_None,     // StartSection
        true,          // bStopWhenAbilityEnds
        1.f,           // AnimRootMotionTranslationScale
        0.f,           // StartTimeSeconds
        false);        // bAllowInterruptAfterBlendOut
Task->ReadyForActivation();
```

Delegates: `OnCompleted`, `OnBlendedIn`, `OnBlendOut`, `OnInterrupted`, `OnCancelled`, all
`FMontageWaitSimpleDelegate` (no parameters).

`UAbilityTask_PlayAnimAndWait` is the equivalent for a raw `UAnimSequence` played into a slot:
`CreatePlayAnimAndWaitProxy(OwningAbility, TaskInstanceName, AnimSequence, SlotName, BlendInTime, BlendOutTime, InPlayRate, StartTimeSeconds, bStopWhenAbilityEnds, AnimRootMotionTranslationScale, InPlayCount)`,
with the same delegate set plus `OnBlendIn`. Montage authoring itself is `ue-animation-system`.

---

## UAbilityTask_WaitGameplayEvent

```cpp
#include "Abilities/Tasks/AbilityTask_WaitGameplayEvent.h"

UAbilityTask_WaitGameplayEvent* Task =
    UAbilityTask_WaitGameplayEvent::WaitGameplayEvent(
        this,      // OwningAbility
        EventTag,  // FGameplayTag
        nullptr,   // OptionalExternalTarget
        true,      // OnlyTriggerOnce
        true);     // OnlyMatchExact
Task->EventReceived.AddDynamic(this, &UMyAbility::HandleAttackHitEvent);
Task->ReadyForActivation();
```

Delegate: `EventReceived(FGameplayEventData Payload)`. Send the event from an anim notify or from
gameplay code:

```cpp
#include "AbilitySystemBlueprintLibrary.h"

FGameplayEventData EventData;
EventData.Instigator = GetOwner();
UAbilitySystemBlueprintLibrary::SendGameplayEventToActor(GetOwner(), EventTag, EventData);
```

---

## UAbilityTask_WaitDelay

```cpp
#include "Abilities/Tasks/AbilityTask_WaitDelay.h"

UAbilityTask_WaitDelay* Task = UAbilityTask_WaitDelay::WaitDelay(this, 2.f);
Task->OnFinish.AddDynamic(this, &UMyAbility::HandleDelayFinished);
Task->ReadyForActivation();
```

Delegate: `OnFinish()`.

---

## UAbilityTask_WaitTargetData

Spawns an `AGameplayAbilityTargetActor` and returns an `FGameplayAbilityTargetDataHandle`.

```cpp
#include "Abilities/Tasks/AbilityTask_WaitTargetData.h"
#include "Abilities/GameplayAbilityTargetActor_SingleLineTrace.h"

UAbilityTask_WaitTargetData* Task =
    UAbilityTask_WaitTargetData::WaitTargetData(
        this,
        NAME_None,
        EGameplayTargetingConfirmation::Instant,
        AGameplayAbilityTargetActor_SingleLineTrace::StaticClass());
Task->ValidData.AddDynamic(this, &UMyAbility::HandleValidTargetData);
Task->Cancelled.AddDynamic(this, &UMyAbility::HandleTargetingCancelled);
Task->ReadyForActivation();
```

| `EGameplayTargetingConfirmation` | Behaviour |
|---|---|
| `Instant` | Confirms as soon as the target actor produces data |
| `UserConfirmed` | Waits for the player's confirm input |
| `Custom` | The ability decides when data is ready |
| `CustomMulti` | As `Custom`, but the target actor is not destroyed after producing data |

```cpp
void UMyAbility::HandleValidTargetData(const FGameplayAbilityTargetDataHandle& Data)
{
    const FGameplayEffectSpecHandle SpecHandle =
        MakeOutgoingGameplayEffectSpec(DamageEffect, GetAbilityLevel());
    ApplyGameplayEffectSpecToTarget(CurrentSpecHandle, CurrentActorInfo,
        CurrentActivationInfo, SpecHandle, Data);
    EndAbility(CurrentSpecHandle, CurrentActorInfo, CurrentActivationInfo, true, false);
}
```

Other target actors: `AGameplayAbilityTargetActor_GroundTrace`, `..._Radius`,
`..._ActorPlacement`, `..._Trace`.

---

## Attribute tasks

```cpp
#include "Abilities/Tasks/AbilityTask_WaitAttributeChange.h"

// Engine parameter names are WithSrcTag / WithoutSrcTag.
UAbilityTask_WaitAttributeChange* Task =
    UAbilityTask_WaitAttributeChange::WaitForAttributeChange(
        this,
        UMyHealthSet::GetHealthAttribute(),
        FGameplayTag(),
        FGameplayTag(),
        true);
Task->OnChange.AddDynamic(this, &UMyAbility::HandleHealthChanged);
Task->ReadyForActivation();
```

`WaitForAttributeChangeWithComparison(OwningAbility, InAttribute, InWithTag, InWithoutTag, InComparisonType, InComparisonValue, TriggerOnce, OptionalExternalOwner)`
adds a comparison to the same task.

```cpp
#include "Abilities/Tasks/AbilityTask_WaitAttributeChangeThreshold.h"

UAbilityTask_WaitAttributeChangeThreshold* Task =
    UAbilityTask_WaitAttributeChangeThreshold::WaitForAttributeChangeThreshold(
        this,
        UMyHealthSet::GetHealthAttribute(),
        EWaitAttributeChangeComparison::LessThan,
        30.f,
        false,
        nullptr);
Task->OnChange.AddDynamic(this, &UMyAbility::HandleHealthBelowThreshold);
Task->ReadyForActivation();
```

`EWaitAttributeChangeComparison`: `None`, `GreaterThan`, `LessThan`, `GreaterThanOrEqualTo`,
`LessThanOrEqualTo`, `NotEqualTo`, `ExactlyEqualTo`. The threshold delegate is
`OnChange(bool bMatchesComparison, float CurrentValue)`.
`UAbilityTask_WaitAttributeChangeRatioThreshold` does the same for the ratio of two attributes.

---

## Effect and tag tasks

```cpp
#include "Abilities/Tasks/AbilityTask_WaitGameplayEffectApplied_Self.h"

FGameplayTargetDataFilterHandle SourceFilter;
FGameplayTagRequirements SourceTagRequirements;
FGameplayTagRequirements TargetTagRequirements;
FGameplayTagRequirements AssetTagRequirements;
FGameplayTagRequirements GrantedTagRequirements;

UAbilityTask_WaitGameplayEffectApplied_Self* Task =
    UAbilityTask_WaitGameplayEffectApplied_Self::WaitGameplayEffectAppliedToSelf(
        this,
        SourceFilter,
        SourceTagRequirements,
        TargetTagRequirements,
        AssetTagRequirements,
        GrantedTagRequirements,
        false,    // TriggerOnce
        nullptr,  // OptionalExternalOwner
        false);   // ListenForPeriodicEffect
Task->OnApplied.AddDynamic(this, &UMyAbility::HandleEffectApplied);
Task->ReadyForActivation();
```

`WaitGameplayEffectAppliedToSelf_Query` takes four `FGameplayTagQuery` values instead of the
`FGameplayTagRequirements` — more expressive, slightly slower. `UAbilityTask_WaitGameplayEffectApplied_Target`
mirrors both factories for effects this ASC applies to others.

```cpp
#include "Abilities/Tasks/AbilityTask_WaitGameplayTagQuery.h"

UAbilityTask_WaitGameplayTagQuery* Task =
    UAbilityTask_WaitGameplayTagQuery::WaitGameplayTagQuery(
        this,
        TagQuery,     // const FGameplayTagQuery
        nullptr,      // InOptionalExternalTarget
        EWaitGameplayTagQueryTriggerCondition::WhenTrue,
        false);       // bOnlyTriggerOnce
Task->ReadyForActivation();
```

`EWaitGameplayTagQueryTriggerCondition` is `WhenTrue` or `WhenFalse`. If the query already matches
when the task activates, it broadcasts immediately. Its `Triggered` delegate is a **protected**
member, as is `TagCountChanged` on `UAbilityTask_WaitGameplayTagCountChanged`
(`WaitGameplayTagCountChange(OwningAbility, Tag, InOptionalExternalTarget)`): bind them from
Blueprint, or from a subclass of the task. Pure C++ that only needs the callback is usually better
served by `ASC->RegisterGameplayTagEvent(Tag, EGameplayTagEventType::NewOrRemoved)`.

`UAbilityTask_WaitGameplayTagAdded` and `UAbilityTask_WaitGameplayTagRemoved`, in
`AbilityTask_WaitGameplayTag.h`, cover the simple add/remove cases.

Other useful tasks: `UAbilityTask_WaitGameplayEffectRemoved`, `UAbilityTask_WaitGameplayEffectStackChange`,
`UAbilityTask_WaitGameplayEffectBlockedImmunity`, `UAbilityTask_WaitInputPress`,
`UAbilityTask_WaitInputRelease`, `UAbilityTask_WaitConfirmCancel`, `UAbilityTask_WaitMovementModeChange`,
`UAbilityTask_WaitOverlap`, `UAbilityTask_NetworkSyncPoint`, `UAbilityTask_Repeat`,
`UAbilityTask_SpawnActor`, and the root-motion family (`UAbilityTask_ApplyRootMotionConstantForce`,
`UAbilityTask_ApplyRootMotionJumpForce`, `UAbilityTask_ApplyRootMotionMoveToForce`,
`UAbilityTask_ApplyRootMotionMoveToActorForce`, `UAbilityTask_ApplyRootMotionRadialForce`).

---

## Custom ability task

```cpp
// MyAbilityTask_WaitInputRelease.h
#pragma once

#include "Abilities/Tasks/AbilityTask.h"
#include "MyAbilityTask_WaitInputRelease.generated.h"

DECLARE_DYNAMIC_MULTICAST_DELEGATE(FMyInputReleasedDelegate);

UCLASS()
class MYGAME_API UMyAbilityTask_WaitInputRelease : public UAbilityTask
{
    GENERATED_BODY()

public:
    UFUNCTION(BlueprintCallable, Category = "Ability|Tasks",
        meta = (HidePin = "OwningAbility", DefaultToSelf = "OwningAbility",
                BlueprintInternalUseOnly = "TRUE"))
    static UMyAbilityTask_WaitInputRelease* WaitInputRelease(UGameplayAbility* OwningAbility);

    UPROPERTY(BlueprintAssignable)
    FMyInputReleasedDelegate OnReleased;

    virtual void Activate() override;
    virtual void TickTask(float DeltaTime) override;
    virtual void OnDestroy(bool bInOwnerFinished) override;
};
```

```cpp
// MyAbilityTask_WaitInputRelease.cpp
#include "MyAbilityTask_WaitInputRelease.h"
#include "AbilitySystemComponent.h"

UMyAbilityTask_WaitInputRelease* UMyAbilityTask_WaitInputRelease::WaitInputRelease(
    UGameplayAbility* OwningAbility)
{
    return NewAbilityTask<UMyAbilityTask_WaitInputRelease>(OwningAbility);
}

void UMyAbilityTask_WaitInputRelease::Activate()
{
    Super::Activate();
    bTickingTask = true;
}

void UMyAbilityTask_WaitInputRelease::TickTask(float DeltaTime)
{
    Super::TickTask(DeltaTime);

    if (!AbilitySystemComponent.IsValid())
    {
        return;
    }

    const FGameplayAbilitySpec* Spec =
        AbilitySystemComponent->FindAbilitySpecFromHandle(GetAbilitySpecHandle());
    if (Spec && !Spec->InputPressed)
    {
        if (ShouldBroadcastAbilityTaskDelegates())
        {
            OnReleased.Broadcast();
        }
        EndTask();
    }
}

void UMyAbilityTask_WaitInputRelease::OnDestroy(bool bInOwnerFinished)
{
    Super::OnDestroy(bInOwnerFinished);
}
```

`AbilitySystemComponent` on `UAbilityTask` is a `TWeakObjectPtr<UAbilitySystemComponent>`, so always
check `IsValid()` before dereferencing. `UAbilityTask::NewTask` is blocked by a `static_assert`;
`NewAbilityTask<T>(OwningAbility, InstanceName)` is the only correct factory.

---

## Task lifecycle

```
ActivateAbility()
  └─ static factory              → task object created, not running
  └─ bind delegates
  └─ ReadyForActivation()        → UGameplayTask starts the task
        └─ Activate()            → task registers its listeners
              └─ event fires     → delegate broadcast
                    └─ EndAbility() or EndTask()
                          └─ OnDestroy() cleanup
```

---

## Common task mistakes

**No `ReadyForActivation()`:** the task exists but never runs, and the ability hangs until something
else ends it.

**Binding after `ReadyForActivation()`:** some tasks broadcast inside `Activate()` (a zero-length
`WaitDelay`, a tag query that is already true). Bind first.

**Broadcasting after the ability ended:** guard every broadcast with
`ShouldBroadcastAbilityTaskDelegates()`.

**Ticking without `bTickingTask`:** `TickTask` is never called unless `Activate()` sets
`bTickingTask = true`.

**Binding a non-`UFUNCTION` handler:** `AddDynamic` on a dynamic multicast delegate fails at
runtime if the callback is not marked `UFUNCTION()`.

**Deleting tasks:** never `delete` a task. Call `EndTask()`, or let `EndAbility` tear it down.

**Ticking tasks on an ability that runs on the CDO:** use `InstancedPerActor` or
`InstancedPerExecution` so the task has per-instance state to work with.
