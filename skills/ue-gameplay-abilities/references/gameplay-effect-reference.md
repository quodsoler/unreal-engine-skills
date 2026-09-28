# GameplayEffect Reference

Duration, modifiers, magnitude calculations, executions, stacking, immunity, queries and the
`UGameplayEffectComponent` list. Target engine: UE 5.8. Header: `GameplayEffect.h`.

---

## Duration policies (`EGameplayEffectDurationType`)

| Value | Behaviour |
|---|---|
| `Instant` | Executes once against the base value; never enters the active container |
| `Infinite` | Aggregates onto the current value until removed |
| `HasDuration` | Aggregates for `DurationMagnitude` seconds, then is removed |

Only executions reach `UAttributeSet::PostGameplayEffectExecute`: `Instant` effects and each period
of a periodic effect (see [Periodic effects](#periodic-effects)). Non-periodic duration and infinite
effects work through aggregators on the current value, so they surface in `PreAttributeChange`.

---

## Modifier operations (`EGameplayModOp`)

`EGameplayModOp` is a namespaced enum: the type is `EGameplayModOp::Type` and a stored value is a
`TEnumAsByte<EGameplayModOp::Type>`. Aggregated values are applied as

```
((BaseValue + AddBase) * MultiplyAdditive / DivideAdditive * MultiplyCompound) + AddFinal
```

| Value | Effect |
|---|---|
| `EGameplayModOp::AddBase` | Adds to the base value first |
| `EGameplayModOp::MultiplyAdditive` | Multipliers are summed, then multiplied against the running result |
| `EGameplayModOp::DivideAdditive` | Divisors are summed, then divided against the running result |
| `EGameplayModOp::MultiplyCompound` | Multiplied into the running result, compounding |
| `EGameplayModOp::AddFinal` | Added after all multiplication |
| `EGameplayModOp::Override` | Replaces the computed value |

`Additive`, `Multiplicitive` (engine spelling) and `Division` still exist as hidden back-compat
aliases for `AddBase`, `MultiplyAdditive` and `DivideAdditive`. Write the modern names.

---

## Magnitude calculations (`EGameplayEffectMagnitudeCalculation`)

`ScalableFloat`, `AttributeBased`, `CustomCalculationClass`, `SetByCaller`.

### ScalableFloat

An `FScalableFloat`: a constant, or a `UCurveTable` row evaluated at the effect level.

### AttributeBased

Derives the magnitude from a captured attribute:
`(Coefficient * (PreMultiplyAdditiveValue + AttributeValue)) + PostMultiplyAdditiveValue`.

`EAttributeBasedFloatCalculationType` picks which value is read:

| Value | Reads |
|---|---|
| `AttributeMagnitude` | Final evaluated value, including active modifiers |
| `AttributeBaseValue` | Base value only |
| `AttributeBonusMagnitude` | Final minus base |
| `AttributeMagnitudeEvaluatedUpToChannel` | Evaluation stopped at a chosen channel |

### CustomCalculationClass

The examples below capture two attributes from a combat attribute set declared the same way as
`UMyHealthSet`: `ATTRIBUTE_ACCESSORS_BASIC(UMyCombatSet, AttackPower)` generates
`static FGameplayAttribute GetAttackPowerAttribute();` and `ATTRIBUTE_ACCESSORS_BASIC(UMyCombatSet, Armor)`
generates `static FGameplayAttribute GetArmorAttribute();`.

```cpp
// MyDamageMagnitudeCalc.h
#pragma once

#include "GameplayModMagnitudeCalculation.h"
#include "MyDamageMagnitudeCalc.generated.h"

UCLASS()
class MYGAME_API UMyDamageMagnitudeCalc : public UGameplayModMagnitudeCalculation
{
    GENERATED_BODY()

public:
    UMyDamageMagnitudeCalc();

    virtual float CalculateBaseMagnitude_Implementation(const FGameplayEffectSpec& Spec) const override;

private:
    FGameplayEffectAttributeCaptureDefinition AttackPowerDef;
};
```

```cpp
// MyDamageMagnitudeCalc.cpp
#include "MyDamageMagnitudeCalc.h"
#include "MyCombatSet.h"

UMyDamageMagnitudeCalc::UMyDamageMagnitudeCalc()
{
    // Capture AttackPower from the source, evaluated at execution time rather than snapshotted.
    AttackPowerDef = FGameplayEffectAttributeCaptureDefinition(
        UMyCombatSet::GetAttackPowerAttribute(),
        EGameplayEffectAttributeCaptureSource::Source,
        false);

    RelevantAttributesToCapture.Add(AttackPowerDef);
}

float UMyDamageMagnitudeCalc::CalculateBaseMagnitude_Implementation(const FGameplayEffectSpec& Spec) const
{
    FAggregatorEvaluateParameters EvalParams;
    EvalParams.SourceTags = Spec.CapturedSourceTags.GetAggregatedTags();
    EvalParams.TargetTags = Spec.CapturedTargetTags.GetAggregatedTags();

    float AttackPower = 0.f;
    GetCapturedAttributeMagnitude(AttackPowerDef, Spec, EvalParams, AttackPower);

    return AttackPower * 1.5f;
}
```

`RelevantAttributesToCapture` comes from `UGameplayEffectCalculation` and must be filled in the
constructor: captures are resolved when the spec is created, not when the calculation runs.

### SetByCaller

The GE asset declares a `DataTag`; the caller supplies the number:

```cpp
const FGameplayEffectSpecHandle SpecHandle =
    MakeOutgoingGameplayEffectSpec(DamageEffectClass, GetAbilityLevel());
SpecHandle.Data->SetSetByCallerMagnitude(
    FGameplayTag::RequestGameplayTag(TEXT("SetByCaller.Damage")), 75.f);
```

There is also an `FName` overload, `SetSetByCallerMagnitude(FName DataName, float Magnitude)`.

---

## Execution calculations (`UGameplayEffectExecutionCalculation`)

Executions run whenever the effect executes (instant effects and each period of a periodic effect) and can write several attributes from one formula.

```cpp
// MyDamageExecution.h
#pragma once

#include "GameplayEffectExecutionCalculation.h"
#include "MyDamageExecution.generated.h"

UCLASS()
class MYGAME_API UMyDamageExecution : public UGameplayEffectExecutionCalculation
{
    GENERATED_BODY()

public:
    UMyDamageExecution();

    virtual void Execute_Implementation(
        const FGameplayEffectCustomExecutionParameters& ExecutionParams,
        FGameplayEffectCustomExecutionOutput& OutExecutionOutput) const override;

private:
    FGameplayEffectAttributeCaptureDefinition AttackPowerDef;
    FGameplayEffectAttributeCaptureDefinition ArmorDef;
};
```

```cpp
// MyDamageExecution.cpp
#include "MyDamageExecution.h"
#include "MyCombatSet.h"
#include "MyHealthSet.h"

UMyDamageExecution::UMyDamageExecution()
{
    AttackPowerDef = FGameplayEffectAttributeCaptureDefinition(
        UMyCombatSet::GetAttackPowerAttribute(),
        EGameplayEffectAttributeCaptureSource::Source,
        true);

    ArmorDef = FGameplayEffectAttributeCaptureDefinition(
        UMyCombatSet::GetArmorAttribute(),
        EGameplayEffectAttributeCaptureSource::Target,
        false);

    RelevantAttributesToCapture.Add(AttackPowerDef);
    RelevantAttributesToCapture.Add(ArmorDef);
}

void UMyDamageExecution::Execute_Implementation(
    const FGameplayEffectCustomExecutionParameters& ExecutionParams,
    FGameplayEffectCustomExecutionOutput& OutExecutionOutput) const
{
    const FGameplayEffectSpec& Spec = ExecutionParams.GetOwningSpec();

    FAggregatorEvaluateParameters EvalParams;
    EvalParams.SourceTags = Spec.CapturedSourceTags.GetAggregatedTags();
    EvalParams.TargetTags = Spec.CapturedTargetTags.GetAggregatedTags();

    float AttackPower = 0.f;
    ExecutionParams.AttemptCalculateCapturedAttributeMagnitude(AttackPowerDef, EvalParams, AttackPower);

    float Armor = 0.f;
    ExecutionParams.AttemptCalculateCapturedAttributeMagnitude(ArmorDef, EvalParams, Armor);

    const float Damage = FMath::Max(1.f, (AttackPower * 2.f) - Armor);

    OutExecutionOutput.AddOutputModifier(FGameplayModifierEvaluatedData(
        UMyHealthSet::GetHealthAttribute(),
        EGameplayModOp::AddBase,
        -Damage));
}
```

`FGameplayModifierEvaluatedData(Attribute, ModOp, Magnitude)` takes an optional fourth
`FActiveGameplayEffectHandle`. Add the execution class to the GE asset's `Executions` array; it
replaces per-modifier setup for anything that needs more than one input.

---

## Stacking

| `EGameplayEffectStackingType` | Behaviour |
|---|---|
| `None` | Every application is an independent active effect |
| `AggregateBySource` | One stack group per applying ASC |
| `AggregateByTarget` | One stack group on the target, shared by all sources |

Read it with `UGameplayEffect::GetStackingType()`; the public `StackingType` member is going
private. The stack cap is `GetStackLimitCount()` (0 means unlimited).

| `EGameplayEffectStackingDurationPolicy` | Behaviour on a new stack |
|---|---|
| `RefreshOnSuccessfulApplication` | Duration restarts |
| `NeverRefresh` | Duration stays as first applied |
| `ExtendDuration` | The new spec's duration is added to the remaining time |

| `EGameplayEffectStackingExpirationPolicy` | Behaviour when the duration runs out |
|---|---|
| `ClearEntireStack` | All stacks removed |
| `RemoveSingleStackAndRefreshDuration` | One stack removed, duration restarts |
| `RefreshDuration` | Duration restarts; removal must be explicit |

| `EGameplayEffectStackingPeriodPolicy` | Behaviour of the period timer on a new stack |
|---|---|
| `ResetOnSuccessfulApplication` | Timer restarts at the full period |
| `NeverReset` | Timer keeps running |

```cpp
const int32 StackCount = ASC->GetCurrentStackCount(ActiveHandle);

const FGameplayEffectQuery Query = FGameplayEffectQuery::MakeQuery_MatchAllOwningTags(
    FGameplayTagContainer(PoisonTag));
const int32 TotalStacks = ASC->GetAggregatedStackCount(Query);

ASC->UpdateActiveGameplayEffectSetByCallerMagnitude(
    ActiveHandle,
    FGameplayTag::RequestGameplayTag(TEXT("SetByCaller.StackDamage")),
    NewDamageValue);
```

---

## Periodic effects

Set on the GE asset: `DurationPolicy = HasDuration`, `DurationMagnitude` for the total time,
`Period` for the interval and `bExecutePeriodicEffectOnApplication` for whether the first tick
fires immediately. Each period runs the effect as if it were instant, so it hits the base value and
calls `PostGameplayEffectExecute`.

```cpp
// Declaration in the listening class:
void HandlePeriodicEffectExecuted(UAbilitySystemComponent* Source,
    const FGameplayEffectSpec& Spec, FActiveGameplayEffectHandle Handle);

// Binding (server side):
ASC->OnPeriodicGameplayEffectExecuteDelegateOnSelf.AddUObject(
    this, &AMyCharacter::HandlePeriodicEffectExecuted);
```

---

## GameplayEffect components

`UGameplayEffect` behaviour lives in `UGameplayEffectComponent` subclasses under
`GameplayEffectComponents/`, added to the asset's `GEComponents` array:

| Component | Owns |
|---|---|
| `UAbilitiesGameplayEffectComponent` | Abilities granted while the effect is active |
| `UAdditionalEffectsGameplayEffectComponent` | Chained effects: `OnApplicationGameplayEffects` (conditional) and the `OnComplete*` arrays |
| `UAssetTagsGameplayEffectComponent` | The tags identifying the effect asset itself |
| `UBlockAbilityTagsGameplayEffectComponent` | Ability tags blocked while the effect is active |
| `UCancelAbilityTagsGameplayEffectComponent` | Abilities cancelled when the effect is applied |
| `UChanceToApplyGameplayEffectComponent` | Probability gate on application |
| `UCustomCanApplyGameplayEffectComponent` | Custom `UGameplayEffectCustomApplicationRequirement` classes |
| `UImmunityGameplayEffectComponent` | Queries that block incoming effects |
| `URemoveOtherGameplayEffectComponent` | Other effects removed on application |
| `UTargetTagRequirementsGameplayEffectComponent` | Tags required on the target to apply, keep or remove |
| `UTargetTagsGameplayEffectComponent` | Tags granted to the target while active |

When reading older assets, the legacy monolithic properties map onto these: granted tags to
`UTargetTagsGameplayEffectComponent`, application tag requirements to
`UTargetTagRequirementsGameplayEffectComponent`, immunity tags to
`UImmunityGameplayEffectComponent`.

---

## Conditional effects (`FConditionalGameplayEffect`)

`FConditionalGameplayEffect` has `EffectClass` and `RequiredSourceTags`; it is applied to the same
target as the parent effect when the source tags match. Configure it through
`UAdditionalEffectsGameplayEffectComponent` on the GE asset.

---

## Immunity

Add `UImmunityGameplayEffectComponent` to the GE and configure its tag queries. While that effect is
active on an ASC, incoming effects matching the query are rejected and the ASC broadcasts:

```cpp
// Declaration:
void HandleImmunityBlock(const FGameplayEffectSpec& BlockedSpec,
    const FActiveGameplayEffect* ImmunityEffect);

// Binding:
ASC->OnImmunityBlockGameplayEffectDelegate.AddUObject(this, &AMyCharacter::HandleImmunityBlock);
```

---

## Querying and removing active effects

```cpp
FGameplayEffectQuery Query;
Query.EffectTagQuery = FGameplayTagQuery::MakeQuery_MatchAnyTags(
    FGameplayTagContainer(FGameplayTag::RequestGameplayTag(TEXT("Effect.Type.Buff"))));

const TArray<FActiveGameplayEffectHandle> Handles = ASC->GetActiveEffects(Query);
const TArray<float> Remaining = ASC->GetActiveEffectsTimeRemaining(Query);

ASC->RemoveActiveEffects(Query);
ASC->RemoveActiveEffectsWithGrantedTags(FGameplayTagContainer(SpeedBuffTag));
ASC->RemoveActiveEffectsWithSourceTags(FGameplayTagContainer(PoisonTag));
```

`FGameplayEffectQuery::MakeQuery_MatchAnyOwningTags` and `MakeQuery_MatchAllOwningTags` are the
convenience constructors for tag-only queries.

---

## Effect lifecycle delegates

| Delegate on the ASC | Fires |
|---|---|
| `OnGameplayEffectAppliedDelegateToSelf` | Any effect applied to this ASC |
| `OnGameplayEffectAppliedDelegateToTarget` | Any effect this ASC applied to someone else |
| `OnActiveGameplayEffectAddedDelegateToSelf` | A duration or infinite effect entered the active container |
| `OnPeriodicGameplayEffectExecuteDelegateOnSelf` | A periodic effect ticked on this ASC |
| `OnImmunityBlockGameplayEffectDelegate` | An incoming effect was blocked by immunity |

The first four are `FOnGameplayEffectAppliedDelegate`, a three-parameter multicast delegate:
`(UAbilitySystemComponent* Source, const FGameplayEffectSpec& Spec, FActiveGameplayEffectHandle Handle)`.

Removal is per-handle rather than global:

```cpp
// Declaration:
void HandleEffectRemoved(const FGameplayEffectRemovalInfo& RemovalInfo);

// Binding, after the effect was applied:
if (FOnActiveGameplayEffectRemoved_Info* RemovalDelegate =
        ASC->OnGameplayEffectRemoved_InfoDelegate(ActiveHandle))
{
    RemovalDelegate->AddUObject(this, &AMyCharacter::HandleEffectRemoved);
}
```

`FGameplayEffectRemovalInfo` carries `bPrematureRemoval`, `bPredictionRejected`, `StackCount`,
`EffectContext`, `OwningASC` and a `const FActiveGameplayEffect* ActiveEffect`.
