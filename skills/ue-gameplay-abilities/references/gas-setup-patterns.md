# GAS Setup Patterns

Ownership, initialization order and the boilerplate around `UAbilitySystemComponent`.
Target engine: UE 5.8.

---

## Pattern 1: ASC on PlayerState (networked players)

`APlayerState` outlives the pawn, so ability grants, active effects and cooldowns survive death and
respawn. The character is the *avatar*; the PlayerState is the *owner*.

### PlayerState

```cpp
// MyPlayerState.h
#pragma once

#include "GameFramework/PlayerState.h"
#include "AbilitySystemInterface.h"
#include "MyPlayerState.generated.h"

class UAbilitySystemComponent;
class UMyHealthSet;

UCLASS()
class MYGAME_API AMyPlayerState : public APlayerState, public IAbilitySystemInterface
{
    GENERATED_BODY()

public:
    AMyPlayerState();

    virtual UAbilitySystemComponent* GetAbilitySystemComponent() const override;

    UMyHealthSet* GetHealthSet() const { return HealthSet; }

protected:
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "GAS")
    TObjectPtr<UAbilitySystemComponent> AbilitySystemComponent;

    UPROPERTY()
    TObjectPtr<UMyHealthSet> HealthSet;
};
```

```cpp
// MyPlayerState.cpp
#include "MyPlayerState.h"
#include "MyHealthSet.h"
#include "AbilitySystemComponent.h"

AMyPlayerState::AMyPlayerState()
{
    AbilitySystemComponent = CreateDefaultSubobject<UAbilitySystemComponent>(TEXT("AbilitySystemComponent"));
    AbilitySystemComponent->SetIsReplicated(true);
    AbilitySystemComponent->SetReplicationMode(EGameplayEffectReplicationMode::Mixed);

    // Subobject attribute sets are discovered by the ASC automatically.
    HealthSet = CreateDefaultSubobject<UMyHealthSet>(TEXT("HealthSet"));

    // PlayerState replicates slowly by default; GAS wants it responsive.
    SetNetUpdateFrequency(100.f);
}

UAbilitySystemComponent* AMyPlayerState::GetAbilitySystemComponent() const
{
    return AbilitySystemComponent;
}
```

### Character (avatar)

```cpp
// MyPlayerCharacter.h
#pragma once

#include "GameFramework/Character.h"
#include "AbilitySystemInterface.h"
#include "MyPlayerCharacter.generated.h"

class UAbilitySystemComponent;
class UGameplayAbility;

UCLASS()
class MYGAME_API AMyPlayerCharacter : public ACharacter, public IAbilitySystemInterface
{
    GENERATED_BODY()

public:
    virtual UAbilitySystemComponent* GetAbilitySystemComponent() const override;

    virtual void PossessedBy(AController* NewController) override;
    virtual void OnRep_PlayerState() override;

protected:
    UPROPERTY(EditDefaultsOnly, Category = "GAS")
    TArray<TSubclassOf<UGameplayAbility>> DefaultAbilities;

private:
    void InitGAS();
    void GiveDefaultAbilities();
};
```

```cpp
// MyPlayerCharacter.cpp
#include "MyPlayerCharacter.h"
#include "MyPlayerState.h"
#include "AbilitySystemComponent.h"
#include "Abilities/GameplayAbility.h"

UAbilitySystemComponent* AMyPlayerCharacter::GetAbilitySystemComponent() const
{
    const AMyPlayerState* PS = GetPlayerState<AMyPlayerState>();
    return PS ? PS->GetAbilitySystemComponent() : nullptr;
}

void AMyPlayerCharacter::PossessedBy(AController* NewController)
{
    Super::PossessedBy(NewController);
    InitGAS();
    GiveDefaultAbilities();
}

void AMyPlayerCharacter::OnRep_PlayerState()
{
    Super::OnRep_PlayerState();
    InitGAS();
}

void AMyPlayerCharacter::InitGAS()
{
    AMyPlayerState* PS = GetPlayerState<AMyPlayerState>();
    if (!PS)
    {
        return;
    }

    if (UAbilitySystemComponent* ASC = PS->GetAbilitySystemComponent())
    {
        // OwnerActor = logical owner (PlayerState), AvatarActor = physical actor (Character).
        ASC->InitAbilityActorInfo(PS, this);
    }
}

void AMyPlayerCharacter::GiveDefaultAbilities()
{
    if (!HasAuthority())
    {
        return;
    }

    UAbilitySystemComponent* ASC = GetAbilitySystemComponent();
    if (!ASC)
    {
        return;
    }

    for (const TSubclassOf<UGameplayAbility>& AbilityClass : DefaultAbilities)
    {
        ASC->GiveAbility(FGameplayAbilitySpec(AbilityClass, 1));
    }
}
```

### Respawn

The PlayerState-owned ASC is untouched by pawn destruction. The new pawn only has to re-run
`InitAbilityActorInfo`, which `PossessedBy` already does. Do **not** re-grant startup abilities on
respawn unless you also cleared them, or the character ends up with duplicate specs.

---

## Pattern 2: ASC on the Pawn (AI, single-player)

```cpp
// MyAICharacter.h
#pragma once

#include "GameFramework/Character.h"
#include "AbilitySystemInterface.h"
#include "MyAICharacter.generated.h"

class UAbilitySystemComponent;
class UGameplayAbility;
class UMyHealthSet;

UCLASS()
class MYGAME_API AMyAICharacter : public ACharacter, public IAbilitySystemInterface
{
    GENERATED_BODY()

public:
    AMyAICharacter();

    virtual UAbilitySystemComponent* GetAbilitySystemComponent() const override;
    virtual void BeginPlay() override;

protected:
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "GAS")
    TObjectPtr<UAbilitySystemComponent> AbilitySystemComponent;

    UPROPERTY()
    TObjectPtr<UMyHealthSet> HealthSet;

    UPROPERTY(EditDefaultsOnly, Category = "GAS")
    TArray<TSubclassOf<UGameplayAbility>> DefaultAbilities;
};
```

```cpp
// MyAICharacter.cpp
#include "MyAICharacter.h"
#include "MyHealthSet.h"
#include "AbilitySystemComponent.h"
#include "Abilities/GameplayAbility.h"

AMyAICharacter::AMyAICharacter()
{
    AbilitySystemComponent = CreateDefaultSubobject<UAbilitySystemComponent>(TEXT("AbilitySystemComponent"));
    AbilitySystemComponent->SetIsReplicated(true);
    AbilitySystemComponent->SetReplicationMode(EGameplayEffectReplicationMode::Minimal);

    HealthSet = CreateDefaultSubobject<UMyHealthSet>(TEXT("HealthSet"));
}

UAbilitySystemComponent* AMyAICharacter::GetAbilitySystemComponent() const
{
    return AbilitySystemComponent;
}

void AMyAICharacter::BeginPlay()
{
    Super::BeginPlay();

    // Owner and avatar are the same actor when the ASC lives on the pawn.
    AbilitySystemComponent->InitAbilityActorInfo(this, this);

    if (HasAuthority())
    {
        for (const TSubclassOf<UGameplayAbility>& AbilityClass : DefaultAbilities)
        {
            AbilitySystemComponent->GiveAbility(FGameplayAbilitySpec(AbilityClass, 1));
        }
    }
}
```

---

## AttributeSet implementation

The header form is in the skill's Attribute Sets section. This is the matching `.cpp`.

```cpp
// MyHealthSet.cpp
#include "MyHealthSet.h"
#include "GameplayEffectExtension.h" // FGameplayEffectModCallbackData
#include "Net/UnrealNetwork.h"

UMyHealthSet::UMyHealthSet()
{
    InitMaxHealth(100.f);
    InitHealth(100.f);
}

void UMyHealthSet::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    DOREPLIFETIME_CONDITION_NOTIFY(UMyHealthSet, Health, COND_None, REPNOTIFY_Always);
    DOREPLIFETIME_CONDITION_NOTIFY(UMyHealthSet, MaxHealth, COND_None, REPNOTIFY_Always);
}

void UMyHealthSet::OnRep_Health(const FGameplayAttributeData& OldHealth)
{
    GAMEPLAYATTRIBUTE_REPNOTIFY(UMyHealthSet, Health, OldHealth);
}

void UMyHealthSet::OnRep_MaxHealth(const FGameplayAttributeData& OldMaxHealth)
{
    GAMEPLAYATTRIBUTE_REPNOTIFY(UMyHealthSet, MaxHealth, OldMaxHealth);
}

// Clamp the CURRENT value. This runs on every aggregator recompute, so do nothing else here.
void UMyHealthSet::PreAttributeChange(const FGameplayAttribute& Attribute, float& NewValue)
{
    Super::PreAttributeChange(Attribute, NewValue);

    if (Attribute == GetMaxHealthAttribute())
    {
        NewValue = FMath::Max(NewValue, 1.f);
    }
}

// React AFTER an instant or periodic GE wrote the base value: this is where death belongs.
void UMyHealthSet::PostGameplayEffectExecute(const FGameplayEffectModCallbackData& Data)
{
    Super::PostGameplayEffectExecute(Data);

    if (Data.EvaluatedData.Attribute == GetHealthAttribute())
    {
        SetHealth(FMath::Clamp(GetHealth(), 0.f, GetMaxHealth()));
    }
}
```

`Data.EvaluatedData` is an `FGameplayModifierEvaluatedData` (attribute, `ModifierOp`, `Magnitude`);
`Data.Target` is the target ASC and `Data.EffectSpec` the spec that caused the change.

---

## Attribute initialization from a CurveTable

`FAttributeSetInitter::InitAttributeSetDefaults` fills every attribute from a `UCurveTable` whose
row names follow `GroupName.AttributeSetName.AttributeName`, one column per level:

```
Default.MyHealthSet.Health
Default.MyHealthSet.MaxHealth
Hero1.MyHealthSet.Health
```

```cpp
#include "AbilitySystemGlobals.h"
#include "AttributeSet.h"

// Server only, after InitAbilityActorInfo:
UAbilitySystemGlobals::Get().GetAttributeSetInitter()->InitAttributeSetDefaults(
    AbilitySystemComponent, FName("Default"), PlayerLevel, true);
```

The table asset itself is registered in Project Settings (see `UGameplayAbilitiesDeveloperSettings`).
Authoring the `UCurveTable` is covered by `ue-data-assets-tables`.

---

## Tracking granted ability handles

```cpp
// In the granting class:
UPROPERTY()
TArray<FGameplayAbilitySpecHandle> GrantedAbilityHandles;
```

```cpp
const FGameplayAbilitySpecHandle Handle =
    AbilitySystemComponent->GiveAbility(FGameplayAbilitySpec(UMyFireballAbility::StaticClass(), 1));
GrantedAbilityHandles.Add(Handle);

// Remove one, or all:
AbilitySystemComponent->ClearAbility(Handle);

for (const FGameplayAbilitySpecHandle& H : GrantedAbilityHandles)
{
    AbilitySystemComponent->ClearAbility(H);
}
GrantedAbilityHandles.Empty();
```

`GiveAbility` and `ClearAbility` are authority-only. `FGameplayAbilitySpec(Class, Level, InputID, SourceObject)`
lets you stamp an input id and a source object onto the spec at grant time.

---

## Listening to tags and attributes

Tag events and attribute-change delegates are the two hooks a HUD or a controller normally needs.

```cpp
// MyCharacter.cpp — bind after InitAbilityActorInfo has run.
void AMyCharacter::BindGASDelegates(UAbilitySystemComponent* ASC)
{
    StunnedTagDelegateHandle = ASC->RegisterGameplayTagEvent(
        FGameplayTag::RequestGameplayTag(TEXT("State.Stunned")),
        EGameplayTagEventType::NewOrRemoved)
        .AddUObject(this, &AMyCharacter::HandleStunnedTagChanged);

    HealthChangedDelegateHandle = ASC->GetGameplayAttributeValueChangeDelegate(
        UMyHealthSet::GetHealthAttribute())
        .AddUObject(this, &AMyCharacter::HandleHealthChanged);
}

void AMyCharacter::HandleStunnedTagChanged(const FGameplayTag Tag, int32 NewCount)
{
    // NewCount > 0 means the tag is present.
}

void AMyCharacter::HandleHealthChanged(const FOnAttributeChangeData& Data)
{
    // Data.NewValue, Data.OldValue, Data.Attribute, Data.GEModData
}
```

Both delegates are plain multicast delegates, so store the `FDelegateHandle` and call
`.Remove(Handle)` on the same delegate when the listener goes away:

```cpp
ASC->GetGameplayAttributeValueChangeDelegate(UMyHealthSet::GetHealthAttribute())
    .Remove(HealthChangedDelegateHandle);
```

Declare the handles and the callbacks in the owning class:

```cpp
FDelegateHandle StunnedTagDelegateHandle;
FDelegateHandle HealthChangedDelegateHandle;

void BindGASDelegates(UAbilitySystemComponent* ASC);
void HandleStunnedTagChanged(const FGameplayTag Tag, int32 NewCount);
void HandleHealthChanged(const FOnAttributeChangeData& Data);
```

---

## Replication mode summary

| Mode | GEs to simulated proxies | Typical owner |
|---|---|---|
| `Minimal` | None; minimal tag and cue data only | AI, non-player actors |
| `Mixed` | Full to the owning connection, minimal to the rest | Player character with a PlayerState-owned ASC |
| `Full` | Full to everyone | Single-player, or small sessions where fidelity beats bandwidth |

`Mixed` finds the owning connection through the ASC's owner actor (`GameplayEffect.cpp:5240`).
A possessed pawn is owned by its PlayerController, so a pawn-owned player ASC works with `Mixed` too;
use `Full` only when simulated proxies need full GE data. AI ASCs use `Minimal`.
