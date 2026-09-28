# Delegate Patterns Reference

Declaration macros, binding methods and invocation patterns for the delegate system in UE 5.8. Source: `Engine/Source/Runtime/Core/Public/Delegates/DelegateCombinations.h`, `Delegate.h`, `DelegateSignatureImpl.inl` and `Engine/Source/Runtime/Core/Public/UObject/ScriptDelegates.h` (module `Core`).

---

## Delegate Type Overview

| Type | Macro family | Bindings | Serializable | Blueprint | Use case |
|------|-------------|----------|--------------|-----------|----------|
| Single | `DECLARE_DELEGATE` | 1 | No | No | Single-owner callbacks |
| Single with return value | `DECLARE_DELEGATE_RetVal` | 1 | No | No | Query callbacks |
| Multicast | `DECLARE_MULTICAST_DELEGATE` | N | No | No | C++ events |
| Thread-safe multicast | `DECLARE_TS_MULTICAST_DELEGATE` | N | No | No | Broadcast from worker threads |
| Dynamic single | `DECLARE_DYNAMIC_DELEGATE` | 1 | Yes | Yes | Blueprint-assignable callback |
| Dynamic single with return value | `DECLARE_DYNAMIC_DELEGATE_RetVal` | 1 | Yes | Yes | Blueprint-assignable query |
| Dynamic multicast | `DECLARE_DYNAMIC_MULTICAST_DELEGATE` | N | Yes | Yes | `UPROPERTY(BlueprintAssignable)` events |

Every family has `_OneParam` through `_NineParams` variants. Dynamic macros take `Type, Name` pairs because the names become Blueprint pin labels.

---

## Declaration Macros

Declare delegate types at file scope, before the `UCLASS` that uses them.

```cpp
// Single delegates
DECLARE_DELEGATE(FOnMyGamePaused);
DECLARE_DELEGATE_OneParam(FOnMyItemCollected, AActor* /*Item*/);
DECLARE_DELEGATE_TwoParams(FOnMyDamageDealt, AActor* /*Target*/, float /*Amount*/);
DECLARE_DELEGATE_ThreeParams(FOnMyAbilityUsed, FName /*AbilityID*/, int32 /*Level*/, float /*Cooldown*/);

// Single delegates with a return value
DECLARE_DELEGATE_RetVal(bool, FMyShouldSpawnDelegate);
DECLARE_DELEGATE_RetVal_OneParam(float, FMyCalculateDamageDelegate, float /*BaseDamage*/);
DECLARE_DELEGATE_RetVal_TwoParams(AActor*, FMySelectTargetDelegate, FVector /*Origin*/, float /*Range*/);

// Multicast
DECLARE_MULTICAST_DELEGATE(FOnMyLevelLoaded);
DECLARE_MULTICAST_DELEGATE_OneParam(FOnMyActorSpawned, AActor* /*SpawnedActor*/);
DECLARE_MULTICAST_DELEGATE_TwoParams(FOnMyHealthChangedNative, float /*Current*/, float /*Max*/);

// Thread-safe multicast: safe to Add/Remove/Broadcast from any thread
DECLARE_TS_MULTICAST_DELEGATE_OneParam(FOnMyAsyncDataReady, const TArray<uint8>& /*Data*/);

// Dynamic single (parameter names required)
DECLARE_DYNAMIC_DELEGATE(FOnMySimpleEvent);
DECLARE_DYNAMIC_DELEGATE_OneParam(FOnMyActorDestroyed, AActor*, DestroyedActor);
DECLARE_DYNAMIC_DELEGATE_TwoParams(FOnMyTransformChanged, FVector, NewLocation, FRotator, NewRotation);
DECLARE_DYNAMIC_DELEGATE_RetVal(bool, FMyValidationCheckDelegate);
DECLARE_DYNAMIC_DELEGATE_RetVal_OneParam(float, FMyGetModifiedDamageDelegate, float, BaseDamage);

// Dynamic multicast
DECLARE_DYNAMIC_MULTICAST_DELEGATE(FOnMyGameStarted);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnMyScoreChanged, int32, NewScore);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_TwoParams(FOnMyPlayerDied, APlayerController*, PlayerController, AActor*, KillerActor);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_FourParams(FOnMyHitResult, FVector, HitLocation, FVector, HitNormal, float, Damage, AActor*, HitActor);
```

---

## Declaring Delegates in a UCLASS

```cpp
// MyCharacter.h
#pragma once
#include "CoreMinimal.h"
#include "GameFramework/Character.h"
#include "MyCharacter.generated.h"

DECLARE_DYNAMIC_MULTICAST_DELEGATE_TwoParams(FOnMyHealthChanged, float, CurrentHealth, float, MaxHealth);
DECLARE_MULTICAST_DELEGATE_OneParam(FOnMyDeathNative, AActor* /*DeadActor*/);

UCLASS(Blueprintable)
class MYGAME_API AMyCharacter : public ACharacter
{
    GENERATED_BODY()
public:
    // Blueprint can bind in its event graph
    UPROPERTY(BlueprintAssignable, Category="Events")
    FOnMyHealthChanged OnHealthChanged;

    // BlueprintCallable on a delegate property lets Blueprint call Broadcast
    UPROPERTY(BlueprintAssignable, BlueprintCallable, Category="Events")
    FOnMyHealthChanged OnHealthChangedCallable;

    // Native-only multicast: no UPROPERTY, no reflection cost, no Blueprint access
    FOnMyDeathNative OnDeathNative;

    UFUNCTION()
    void HandleActorDestroyed(AActor* DestroyedActor);

    void HandlePickupWithExtras(AActor* Item, float Bonus, FName Sound);
};
```

A dynamic single delegate declared as a `UFUNCTION` parameter produces a Blueprint event pin; that pattern belongs to `ue-blueprint-cpp-interop`.

---

## Binding Methods

### Single Delegate

```cpp
// MyPickupHandler.h
#pragma once
#include "CoreMinimal.h"
#include "GameFramework/Actor.h"
#include "MyPickupHandler.generated.h"

DECLARE_DELEGATE_OneParam(FOnMyItemCollected, AActor* /*Item*/);

class FMyHelper
{
public:
    void HandleItem(AActor* Item) {}
    void OnHealthChanged(float Current, float Max) {}
};

UCLASS()
class MYGAME_API AMyPickupHandler : public AActor
{
    GENERATED_BODY()
public:
    void BindAll(AActor* SomeActor);
    void HandleItemCollected(AActor* Item);
    void HandleItemWithBonus(AActor* Item, float Bonus);
    static void StaticHandleItem(AActor* Item);

    UFUNCTION()
    void HandleItemUFunction(AActor* Item);
private:
    FOnMyItemCollected Delegate;
    TSharedPtr<FMyHelper> SharedHelper;
    FMyHelper RawHelper;
};
```

```cpp
// MyPickupHandler.cpp
void AMyPickupHandler::BindAll(AActor* SomeActor)
{
    Delegate.BindUObject(this, &AMyPickupHandler::HandleItemCollected);   // weak to the UObject: auto-unbinds when destroyed
    Delegate.BindSP(SharedHelper.ToSharedRef(), &FMyHelper::HandleItem);   // weak to the shared owner
    Delegate.BindRaw(&RawHelper, &FMyHelper::HandleItem);                  // no lifetime tracking: Unbind before RawHelper dies
    Delegate.BindStatic(&AMyPickupHandler::StaticHandleItem);
    Delegate.BindUFunction(this, FName("HandleItemUFunction"));             // by name; target must be a UFUNCTION
    Delegate.BindLambda([](AActor* Item) { UE_LOG(LogMyGame, Log, TEXT("Collected %s"), *Item->GetName()); });
    Delegate.BindWeakLambda(this, [this](AActor* Item) { HandleItemCollected(Item); });   // skipped once this is invalid

    // Payload values are appended after the declared parameters; any number is allowed
    Delegate.BindUObject(this, &AMyPickupHandler::HandleItemWithBonus, 1.5f);

    if (Delegate.IsBound())
    {
        Delegate.Execute(SomeActor);        // asserts when unbound
    }
    Delegate.ExecuteIfBound(SomeActor);     // silent no-op when unbound
    Delegate.Unbind();

    // Factory form when a function wants a delegate argument
    FOnMyItemCollected Created = FOnMyItemCollected::CreateUObject(this, &AMyPickupHandler::HandleItemCollected);
}
```

### Multicast Delegate

```cpp
// MyHealthListener.h
#pragma once
#include "CoreMinimal.h"
#include "GameFramework/Actor.h"
#include "MyPickupHandler.h"
#include "MyHealthListener.generated.h"

DECLARE_MULTICAST_DELEGATE_TwoParams(FOnMyHealthChangedNative, float /*Current*/, float /*Max*/);

UCLASS()
class MYGAME_API AMyHealthListener : public AActor
{
    GENERATED_BODY()
public:
    FOnMyHealthChangedNative HealthDelegate;

    void BindAll();
    void OnHealthChangedNative(float Current, float Max);
    static void StaticOnHealthChanged(float Current, float Max);
private:
    TSharedPtr<FMyHelper> SharedHelper;
    FMyHelper RawHelper;
};
```

```cpp
// MyHealthListener.cpp
void AMyHealthListener::BindAll()
{
    FDelegateHandle Handle       = HealthDelegate.AddUObject(this, &AMyHealthListener::OnHealthChangedNative);
    FDelegateHandle LambdaHandle = HealthDelegate.AddLambda([](float Current, float Max) {});
    FDelegateHandle WeakHandle   = HealthDelegate.AddWeakLambda(this, [this](float Current, float Max) { OnHealthChangedNative(Current, Max); });
    FDelegateHandle RawHandle    = HealthDelegate.AddRaw(&RawHelper, &FMyHelper::OnHealthChanged);
    FDelegateHandle SpHandle     = HealthDelegate.AddSP(SharedHelper.ToSharedRef(), &FMyHelper::OnHealthChanged);
    FDelegateHandle StaticHandle = HealthDelegate.AddStatic(&AMyHealthListener::StaticOnHealthChanged);

    HealthDelegate.Broadcast(75.f, 100.f);

    HealthDelegate.Remove(Handle);              // one binding
    HealthDelegate.RemoveAll(this);             // every binding owned by this object (UObject, SP or raw owner)
    bool bHasListeners = HealthDelegate.IsBound();
    bool bMine         = HealthDelegate.IsBoundToObject(this);
    HealthDelegate.Clear();
}
```

### Dynamic Single Delegate

```cpp
// Inside AMyCharacter, which declares UFUNCTION() void HandleActorDestroyed(AActor* DestroyedActor)
FOnMyActorDestroyed Dynamic;
AActor* SomeActor = nullptr;
Dynamic.BindDynamic(this, &AMyCharacter::HandleActorDestroyed);   // target must be a UFUNCTION
Dynamic.BindUFunction(this, FName("HandleActorDestroyed"));
bool bBound = Dynamic.IsBound();
Dynamic.ExecuteIfBound(SomeActor);
Dynamic.Unbind();
```

### Dynamic Multicast Delegate

```cpp
// Inside AMyHUD, which declares UFUNCTION() void HandleHealthChanged(float CurrentHealth, float MaxHealth); the first argument must be an object of the class that owns the handler
AMyCharacter* Character = Cast<AMyCharacter>(GetOwningPawn());   // owner of OnHealthChanged
float NewHealth = 50.f;
float MaxHealth = 100.f;
Character->OnHealthChanged.AddDynamic(this, &AMyHUD::HandleHealthChanged);        // duplicate binding trips an ensure and is still added (fires twice)
Character->OnHealthChanged.AddUniqueDynamic(this, &AMyHUD::HandleHealthChanged);  // no-op if already bound
bool bAlready = Character->OnHealthChanged.IsAlreadyBound(this, &AMyHUD::HandleHealthChanged);
Character->OnHealthChanged.RemoveDynamic(this, &AMyHUD::HandleHealthChanged);
Character->OnHealthChanged.RemoveAll(this);
Character->OnHealthChanged.Broadcast(NewHealth, MaxHealth);
bool bHasBindings = Character->OnHealthChanged.IsBound();
Character->OnHealthChanged.Clear();
```

---

## Complete Example: Health Component with Native and Dynamic Events

```cpp
// MyHealthComponent.h
#pragma once
#include "CoreMinimal.h"
#include "Components/ActorComponent.h"
#include "MyHealthComponent.generated.h"

DECLARE_DYNAMIC_MULTICAST_DELEGATE_TwoParams(FOnMyHealthChangedSignature, float, CurrentHealth, float, MaxHealth);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnMyDeathSignature, AActor*, KillerActor);
DECLARE_MULTICAST_DELEGATE_OneParam(FOnMyHealthFractionChanged, float /*NewHealthFraction*/);

UCLASS(ClassGroup=(Custom), meta=(BlueprintSpawnableComponent))
class MYGAME_API UMyHealthComponent : public UActorComponent
{
    GENERATED_BODY()
public:
    UPROPERTY(BlueprintAssignable, Category="Health")
    FOnMyHealthChangedSignature OnHealthChanged;

    UPROPERTY(BlueprintAssignable, Category="Health")
    FOnMyDeathSignature OnDeath;

    FOnMyHealthFractionChanged OnHealthFractionChanged;   // C++ listeners only

    UFUNCTION(BlueprintCallable, Category="Health")
    void ApplyHealthDamage(float DamageAmount, AActor* DamageInstigator);

    UFUNCTION(BlueprintPure, Category="Health")
    float GetHealthFraction() const { return MaxHealth > 0.f ? CurrentHealth / MaxHealth : 0.f; }

protected:
    virtual void BeginPlay() override;

private:
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category="Health", meta=(AllowPrivateAccess="true", ClampMin="1.0"))
    float MaxHealth = 100.f;

    UPROPERTY(Transient)
    float CurrentHealth = 0.f;
};
```

```cpp
// MyHealthComponent.cpp
#include "MyHealthComponent.h"

void UMyHealthComponent::BeginPlay()
{
    Super::BeginPlay();
    CurrentHealth = MaxHealth;
}

void UMyHealthComponent::ApplyHealthDamage(float DamageAmount, AActor* DamageInstigator)
{
    if (CurrentHealth <= 0.f) { return; }

    CurrentHealth = FMath::Clamp(CurrentHealth - DamageAmount, 0.f, MaxHealth);
    OnHealthChanged.Broadcast(CurrentHealth, MaxHealth);
    OnHealthFractionChanged.Broadcast(GetHealthFraction());

    if (CurrentHealth <= 0.f)
    {
        OnDeath.Broadcast(DamageInstigator);
    }
}
```

```cpp
// MyListenerCharacter.h
#pragma once
#include "CoreMinimal.h"
#include "GameFramework/Character.h"
#include "MyHealthComponent.h"
#include "MyListenerCharacter.generated.h"

UCLASS()
class MYGAME_API AMyListenerCharacter : public ACharacter
{
    GENERATED_BODY()
protected:
    virtual void BeginPlay() override;
    virtual void EndPlay(const EEndPlayReason::Type EndPlayReason) override;

    UFUNCTION()
    void HandleHealthChanged(float CurrentHealth, float MaxHealth);

    UFUNCTION()
    void HandleDeath(AActor* KillerActor);

    void HandleHealthFraction(float HealthFraction);

private:
    UPROPERTY() TObjectPtr<UMyHealthComponent> HealthComponent;
    FDelegateHandle FractionHandle;
};
```

```cpp
// MyListenerCharacter.cpp
#include "MyListenerCharacter.h"

void AMyListenerCharacter::BeginPlay()
{
    Super::BeginPlay();

    HealthComponent = FindComponentByClass<UMyHealthComponent>();
    if (!HealthComponent) { return; }

    HealthComponent->OnHealthChanged.AddDynamic(this, &AMyListenerCharacter::HandleHealthChanged);
    HealthComponent->OnDeath.AddDynamic(this, &AMyListenerCharacter::HandleDeath);
    FractionHandle = HealthComponent->OnHealthFractionChanged.AddUObject(this, &AMyListenerCharacter::HandleHealthFraction);
}

void AMyListenerCharacter::EndPlay(const EEndPlayReason::Type EndPlayReason)
{
    if (IsValid(HealthComponent))
    {
        HealthComponent->OnHealthChanged.RemoveDynamic(this, &AMyListenerCharacter::HandleHealthChanged);
        HealthComponent->OnDeath.RemoveDynamic(this, &AMyListenerCharacter::HandleDeath);
        HealthComponent->OnHealthFractionChanged.Remove(FractionHandle);
    }
    Super::EndPlay(EndPlayReason);
}

void AMyListenerCharacter::HandleHealthChanged(float CurrentHealth, float MaxHealth)
{
    UE_LOG(LogMyGame, Log, TEXT("[%s] Health: %.1f / %.1f"), *GetName(), CurrentHealth, MaxHealth);
}

void AMyListenerCharacter::HandleDeath(AActor* KillerActor)
{
    UE_LOG(LogMyGame, Warning, TEXT("[%s] Died. Killer: %s"), *GetName(), KillerActor ? *KillerActor->GetName() : TEXT("Unknown"));
}

void AMyListenerCharacter::HandleHealthFraction(float HealthFraction)
{
    UE_LOG(LogMyGame, Verbose, TEXT("Fraction %.2f"), HealthFraction);
}
```

---

## Binding Method Quick Reference

| Method | Owner tracking | Auto-unbind | Use with |
|--------|----------------|-------------|----------|
| `BindUObject` / `AddUObject` | Weak object pointer | Yes, when the UObject is destroyed | UObject-derived classes |
| `BindSP` / `AddSP`, `BindThreadSafeSP` / `AddThreadSafeSP` | Weak shared pointer | Yes, when the shared object dies | Non-UObject types owned by `TSharedPtr` |
| `BindRaw` / `AddRaw` | None | Never | Raw pointers; unbind manually before the owner dies |
| `BindStatic` / `AddStatic` | None | Never | Free and static functions |
| `BindLambda` / `AddLambda` | None | Never | Lambdas that do not capture an object lifetime |
| `BindWeakLambda` / `AddWeakLambda` | Weak object pointer to the given UObject | Yes | Lambdas capturing `this` |
| `BindUFunction` / `AddUFunction` | Weak object pointer | Yes | Bind by function name |
| `BindDynamic` / `AddDynamic` / `AddUniqueDynamic` | UObject + function name | Yes | Dynamic delegates only; target must be `UFUNCTION` |

Lambdas that capture `this` use `AddWeakLambda(this, ...)`; a plain `AddLambda` keeps running after the object is gone. Store the returned `FDelegateHandle` for anything you must remove later.

---

## Return-Value Delegates

Only single delegates return values; multicast delegates cannot (which listener's value would win?).

```cpp
DECLARE_DELEGATE_RetVal_OneParam(bool, FMyValidatePickupDelegate, AActor* /*Item*/);

FMyValidatePickupDelegate ValidationDelegate;
ValidationDelegate.BindLambda([](AActor* Item) -> bool
{
    return IsValid(Item) && Item->ActorHasTag(FName("Pickup"));
});

AActor* PotentialPickup = nullptr;
if (ValidationDelegate.IsBound() && ValidationDelegate.Execute(PotentialPickup))
{
    // collect
}
```

---

## Payload Parameters

Bind methods accept extra payload values after the function pointer. They are stored at bind time and appended after the delegate's declared parameters when the bound function is called.

```cpp
DECLARE_DELEGATE_OneParam(FOnMyItemPickedUp, AActor* /*Item*/);

// AMyCharacter declares: void HandlePickupWithExtras(AActor* Item, float Bonus, FName Sound);
float BonusMultiplier = 1.5f;
FName PickupSound = FName("Pickup");
AActor* SomeActor = nullptr;

FOnMyItemPickedUp PickedUp;
PickedUp.BindUObject(this, &AMyCharacter::HandlePickupWithExtras, BonusMultiplier, PickupSound);
PickedUp.ExecuteIfBound(SomeActor);   // calls HandlePickupWithExtras(SomeActor, 1.5f, "Pickup")
```

---

## Common Mistakes

**`Execute()` on an unbound delegate asserts:**
```cpp
PickupDelegate.Execute(SomeActor);        // WRONG when nothing is bound
PickupDelegate.ExecuteIfBound(SomeActor); // RIGHT
```

**Lambda bound to a multicast without keeping the handle:**
```cpp
OnHealthChanged.AddLambda([](float C, float M) {});                          // WRONG: cannot be removed
FDelegateHandle Handle = OnHealthChanged.AddLambda([](float C, float M) {}); // RIGHT
OnHealthChanged.Remove(Handle);
```

**`AddLambda` capturing `this`:** the lambda outlives the object. Use `AddWeakLambda(this, ...)`.

**`AddDynamic` with a non-`UFUNCTION`:** the reflection lookup fails at runtime. Add `UFUNCTION()` to the handler declaration.

**Never removing dynamic bindings:** bindings to a destroyed object are skipped, but bindings to a live object that has logically finished keep firing. Pair every `AddDynamic` in `BeginPlay` with `RemoveDynamic` in `EndPlay`.

**Dynamic multicast delegate without `UPROPERTY`:** Blueprint cannot see it and bindings are not serialized. Declare it `UPROPERTY(BlueprintAssignable)`.
