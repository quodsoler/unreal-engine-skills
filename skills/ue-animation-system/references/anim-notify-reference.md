# Anim Notify Reference (UE 5.8)

Source headers:
- `Engine/Source/Runtime/Engine/Classes/Animation/AnimNotifies/AnimNotify.h`
- `Engine/Source/Runtime/Engine/Classes/Animation/AnimNotifies/AnimNotifyState.h`
- `Engine/Source/Runtime/Engine/Public/Animation/AnimTypes.h` (`FAnimNotifyEvent`)
- `Engine/Source/Runtime/Engine/Public/Animation/AnimNotifyQueue.h` (`FAnimNotifyEventReference`, `FAnimNotifyQueue`)

---

## Class Hierarchy

```
UObject
  ├── UAnimNotify          point-in-time event      (AnimNotify.h:51)
  └── UAnimNotifyState     duration event           (AnimNotifyState.h:34)
```

Both are `UCLASS(abstract, editinlinenew, Blueprintable, const, ...)`, so a notify
object is shared by every instance playing the animation. Keep no per-character
state on the notify itself; read what you need from `MeshComp`.

### UAnimNotify — virtuals

```cpp
// AnimNotify.h:85-86
virtual void Notify(USkeletalMeshComponent* MeshComp, UAnimSequenceBase* Animation,
    const FAnimNotifyEventReference& EventReference);
virtual void BranchingPointNotify(FBranchingPointNotifyPayload& BranchingPointPayload);
```

```cpp
// BlueprintNativeEvent declarations (AnimNotify.h:59, :95)
FString GetNotifyName() const;                 // override GetNotifyName_Implementation
float GetDefaultTriggerWeightThreshold() const; // override ..._Implementation
```

`Received_Notify(USkeletalMeshComponent*, UAnimSequenceBase*, const FAnimNotifyEventReference&) const`
(`AnimNotify.h:62`) is a `BlueprintImplementableEvent`. It is the Blueprint entry
point only — implement it in a Blueprint notify, never override it in C++.

### UAnimNotifyState — virtuals

```cpp
// AnimNotifyState.h:74-80
virtual void NotifyBegin(USkeletalMeshComponent* MeshComp, UAnimSequenceBase* Animation,
    float TotalDuration, const FAnimNotifyEventReference& EventReference);
virtual void NotifyTick(USkeletalMeshComponent* MeshComp, UAnimSequenceBase* Animation,
    float FrameDeltaTime, const FAnimNotifyEventReference& EventReference);
virtual void NotifyEnd(USkeletalMeshComponent* MeshComp, UAnimSequenceBase* Animation,
    const FAnimNotifyEventReference& EventReference);

virtual void BranchingPointNotifyBegin(FBranchingPointNotifyPayload& BranchingPointPayload);
virtual void BranchingPointNotifyTick(FBranchingPointNotifyPayload& BranchingPointPayload,
    float FrameDeltaTime);
virtual void BranchingPointNotifyEnd(FBranchingPointNotifyPayload& BranchingPointPayload);
```

Blueprint entry points (`AnimNotifyState.h:45-51`): `Received_NotifyBegin`,
`Received_NotifyTick`, `Received_NotifyEnd` — same rule, Blueprint only.

`GetNotifyName() const` is a `BlueprintNativeEvent` here too (`AnimNotifyState.h:42`).

---

## FAnimNotifyEvent and FAnimNotifyEventReference

`FAnimNotifyEvent : public FAnimLinkableElement` (`Public/Animation/AnimTypes.h:276`)
is the placed event on the timeline. Fields you will actually read:

| Field | Type | Meaning |
|---|---|---|
| `NotifyName` | `FName` | Name shown on the track; what `OnPlayMontageNotifyBegin` passes you |
| `Notify` | `TObjectPtr<UAnimNotify>` | Instanced point-in-time notify object, or null |
| `NotifyStateClass` | `TObjectPtr<UAnimNotifyState>` | Instanced duration notify object, or null |
| `Duration` | `float` | Length of a notify state; 0 for a point notify |
| `TriggerWeightThreshold` | `float` | Minimum blend weight required to fire |
| `NotifyTriggerChance` | `float` | 0–1 probability the notify fires |
| `MontageTickType` | `TEnumAsByte<EMontageNotifyTickType::Type>` | `Queued` or `BranchingPoint` |

Accessors (`AnimTypes.h:408-418`): `GetTriggerTime()`, `GetEndTriggerTime()`,
`GetDuration()`, `SetDuration(float)`, `IsBranchingPoint()`.

`FAnimNotifyEventReference` (`Public/Animation/AnimNotifyQueue.h:21`) wraps the event
plus its source object and optional mirror table. Use `GetNotify()` (`:40`) to reach
the `const FAnimNotifyEvent*`; it can be null, so check it.

---

## Timing: queued vs branching point

`EMontageNotifyTickType::Type` (`Public/Animation/AnimTypes.h:86-95`):

| Value | Fires | Header note |
|---|---|---|
| `Queued` | at the end of the evaluation phase | faster; "not suitable for changing sections or montage position" |
| `BranchingPoint` | as encountered during montage advance | slower; "suitable for changing sections or montage position" |

Per-event this is the `MontageTickType` field, set in the montage editor. A native
notify can force the branching-point path for every placement by setting
`bIsNativeBranchingPoint = true` in its constructor (`AnimNotify.h:123`,
`AnimNotifyState.h:108`) and overriding the `BranchingPointNotify*` functions.

Queued notifies are collected in `FAnimNotifyQueue` (`AnimNotifyQueue.h:169`):
`AddAnimNotify(const FAnimNotifyEvent* Notify, const UObject* NotifySource)` (`:186`)
appends to `TArray<FAnimNotifyEventReference> AnimNotifies` (`:215`), which is drained
after evaluation. That is why a `Montage_JumpToSection` issued from a queued notify
takes effect one frame late.

---

## Built-in notifies

| Class | Header | Key properties |
|---|---|---|
| `UAnimNotify_PlaySound` | `AnimNotifies/AnimNotify_PlaySound.h:16` | `Sound` (`TObjectPtr<USoundBase>`), `VolumeMultiplier`, `PitchMultiplier`, `bFollow`, `AttachName` |
| `UAnimNotify_PlayParticleEffect` | `AnimNotifies/AnimNotify_PlayParticleEffect.h:17` | `PSTemplate`, `LocationOffset`, `RotationOffset`, `Scale`, `SocketName`, `Attached` |
| `UAnimNotifyState_TimedParticleEffect` | `AnimNotifies/AnimNotifyState_TimedParticleEffect.h:17` | `PSTemplate`, `SocketName`, `LocationOffset`, `RotationOffset`, `bDestroyAtEnd` |
| `UAnimNotifyState_Trail` | `AnimNotifies/AnimNotifyState_Trail.h:19` | `PSTemplate`, `FirstSocketName`, `SecondSocketName`, `WidthScaleMode`, `WidthScaleCurve` |
| `UAnimNotifyState_DisableRootMotion` | `AnimNotifies/AnimNotifyState_DisableRootMotion.h:8` | no properties; suppresses root motion for its duration |
| `UAnimNotify_PlayNiagaraEffect` | `Plugins/FX/Niagara/.../AnimNotify_PlayNiagaraEffect.h:19` | `Template` (`TObjectPtr<UNiagaraSystem>`), `LocationOffset`, `SocketName` |
| `UAnimNotifyState_TimedNiagaraEffect` | `Plugins/FX/Niagara/.../AnimNotifyState_TimedNiagaraEffect.h:31` | `Template`, `SocketName`, `LocationOffset`, `bDestroyAtEnd` |

The two Niagara notifies live in the `NiagaraAnimNotifies` module — see `ue-niagara-effects`.

---

## Pattern 1 — point-in-time notify with a surface trace

```cpp
// MyFootstepNotify.h
#pragma once

#include "CoreMinimal.h"
#include "Animation/AnimNotifies/AnimNotify.h"
#include "MyFootstepNotify.generated.h"

class UAnimSequenceBase;
class USkeletalMeshComponent;
class USoundBase;

UCLASS(meta = (DisplayName = "My Footstep"))
class MYGAME_API UMyFootstepNotify : public UAnimNotify
{
    GENERATED_BODY()

public:
    UMyFootstepNotify()
    {
        bIsNativeBranchingPoint = false; // queued: fine for SFX/VFX
    }

    virtual void Notify(USkeletalMeshComponent* MeshComp, UAnimSequenceBase* Animation,
        const FAnimNotifyEventReference& EventReference) override;

    virtual FString GetNotifyName_Implementation() const override;

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Footstep")
    FName FootSocket = FName("foot_l");

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Footstep")
    float TraceDistance = 75.f;

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Footstep")
    TMap<TEnumAsByte<EPhysicalSurface>, TObjectPtr<USoundBase>> SurfaceSounds;
};
```

```cpp
// MyFootstepNotify.cpp
#include "MyFootstepNotify.h"
#include "Components/SkeletalMeshComponent.h"
#include "Engine/World.h"
#include "Kismet/GameplayStatics.h"
#include "PhysicalMaterials/PhysicalMaterial.h"
#include "Sound/SoundBase.h"

FString UMyFootstepNotify::GetNotifyName_Implementation() const
{
    return FString::Printf(TEXT("Footstep_%s"), *FootSocket.ToString());
}

void UMyFootstepNotify::Notify(USkeletalMeshComponent* MeshComp, UAnimSequenceBase* Animation,
    const FAnimNotifyEventReference& EventReference)
{
    Super::Notify(MeshComp, Animation, EventReference);

    UWorld* World = MeshComp ? MeshComp->GetWorld() : nullptr;
    if (!World)
    {
        return;
    }

    const FVector SocketLoc = MeshComp->GetSocketLocation(FootSocket);
    const FVector TraceStart = SocketLoc + FVector(0.f, 0.f, 20.f);
    const FVector TraceEnd = SocketLoc - FVector(0.f, 0.f, TraceDistance);

    FCollisionQueryParams Params(SCENE_QUERY_STAT(MyFootstepTrace), true);
    Params.AddIgnoredActor(MeshComp->GetOwner());
    Params.bReturnPhysicalMaterial = true;

    FHitResult Hit;
    if (World->LineTraceSingleByChannel(Hit, TraceStart, TraceEnd, ECC_Visibility, Params))
    {
        const EPhysicalSurface Surface = UGameplayStatics::GetSurfaceType(Hit);
        if (USoundBase* Sound = SurfaceSounds.FindRef(Surface))
        {
            UGameplayStatics::PlaySoundAtLocation(World, Sound, Hit.ImpactPoint);
        }
    }
}
```

`SurfaceSounds` keeps the notify itself stateless: the notify object is shared across
every character playing the animation, so the only per-hit data lives in local variables.
Build.cs: add `PhysicsCore` — `EPhysicalSurface` in a `UPROPERTY` is reflected from that
module (`PhysicsCore/Public/Chaos/ChaosEngineInterface.h:20`), and the link fails without it.

---

## Pattern 2 — notify state as a collision window

```cpp
// MyWeaponWindowNotifyState.h
#pragma once

#include "CoreMinimal.h"
#include "Animation/AnimNotifies/AnimNotifyState.h"
#include "MyWeaponWindowNotifyState.generated.h"

class UAnimSequenceBase;
class USkeletalMeshComponent;

UCLASS(meta = (DisplayName = "My Weapon Collision Window"))
class MYGAME_API UMyWeaponWindowNotifyState : public UAnimNotifyState
{
    GENERATED_BODY()

public:
    virtual void NotifyBegin(USkeletalMeshComponent* MeshComp, UAnimSequenceBase* Animation,
        float TotalDuration, const FAnimNotifyEventReference& EventReference) override;

    virtual void NotifyEnd(USkeletalMeshComponent* MeshComp, UAnimSequenceBase* Animation,
        const FAnimNotifyEventReference& EventReference) override;

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Weapon")
    FName WeaponComponentTag = FName("PrimaryWeapon");
};
```

```cpp
// MyWeaponWindowNotifyState.cpp
#include "MyWeaponWindowNotifyState.h"
#include "Components/PrimitiveComponent.h"
#include "Components/SkeletalMeshComponent.h"
#include "GameFramework/Actor.h"

namespace
{
    UPrimitiveComponent* FindTaggedWeaponComponent(USkeletalMeshComponent* MeshComp, FName Tag)
    {
        AActor* Owner = MeshComp ? MeshComp->GetOwner() : nullptr;
        if (!Owner)
        {
            return nullptr;
        }

        TArray<UActorComponent*> Found = Owner->GetComponentsByTag(UPrimitiveComponent::StaticClass(), Tag);
        return Found.Num() > 0 ? Cast<UPrimitiveComponent>(Found[0]) : nullptr;
    }
}

void UMyWeaponWindowNotifyState::NotifyBegin(USkeletalMeshComponent* MeshComp,
    UAnimSequenceBase* Animation, float TotalDuration,
    const FAnimNotifyEventReference& EventReference)
{
    Super::NotifyBegin(MeshComp, Animation, TotalDuration, EventReference);

    if (UPrimitiveComponent* Weapon = FindTaggedWeaponComponent(MeshComp, WeaponComponentTag))
    {
        Weapon->SetCollisionEnabled(ECollisionEnabled::QueryOnly);
    }
}

void UMyWeaponWindowNotifyState::NotifyEnd(USkeletalMeshComponent* MeshComp,
    UAnimSequenceBase* Animation, const FAnimNotifyEventReference& EventReference)
{
    Super::NotifyEnd(MeshComp, Animation, EventReference);

    if (UPrimitiveComponent* Weapon = FindTaggedWeaponComponent(MeshComp, WeaponComponentTag))
    {
        Weapon->SetCollisionEnabled(ECollisionEnabled::NoCollision);
    }
}
```

`NotifyEnd` also fires when the montage is interrupted or blended out early, so it is
the right place to undo whatever `NotifyBegin` enabled.

---

## Pattern 3 — branching point notify that redirects a montage

```cpp
// MyComboBranchNotify.h
#pragma once

#include "CoreMinimal.h"
#include "Animation/AnimNotifies/AnimNotify.h"
#include "MyComboBranchNotify.generated.h"

UCLASS(meta = (DisplayName = "My Combo Branch Point"))
class MYGAME_API UMyComboBranchNotify : public UAnimNotify
{
    GENERATED_BODY()

public:
    UMyComboBranchNotify()
    {
        bIsNativeBranchingPoint = true; // synchronous during montage advance
    }

    virtual void BranchingPointNotify(FBranchingPointNotifyPayload& BranchingPointPayload) override;

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Combo")
    FName NextSection = FName("ComboEnd");
};
```

```cpp
// MyComboBranchNotify.cpp
#include "MyComboBranchNotify.h"
#include "Animation/AnimInstance.h"
#include "Components/SkeletalMeshComponent.h"

void UMyComboBranchNotify::BranchingPointNotify(FBranchingPointNotifyPayload& BranchingPointPayload)
{
    Super::BranchingPointNotify(BranchingPointPayload);

    if (!BranchingPointPayload.SkelMeshComponent)
    {
        return;
    }

    if (UAnimInstance* AnimInst = BranchingPointPayload.SkelMeshComponent->GetAnimInstance())
    {
        AnimInst->Montage_JumpToSection(NextSection);
    }
}
```

`FBranchingPointNotifyPayload` (`AnimNotify.h:23-41`) carries `SkelMeshComponent`,
`SequenceAsset`, `NotifyEvent`, `MontageInstanceID` and `bReachedEnd`.

---

## Pattern 4 — named montage notifies from outside the AnimInstance

`UAnimInstance` exposes two dynamic multicast delegates (`AnimInstance.h:1820-1823`):

```cpp
// DECLARE_DYNAMIC_MULTICAST_DELEGATE_TwoParams(FPlayMontageAnimNotifyDelegate,
//     FName, NotifyName, const FBranchingPointNotifyPayload&, BranchingPointPayload);
FPlayMontageAnimNotifyDelegate OnPlayMontageNotifyBegin;
FPlayMontageAnimNotifyDelegate OnPlayMontageNotifyEnd;
```

```cpp
// MyCombatComponent.h
#pragma once

#include "CoreMinimal.h"
#include "Components/ActorComponent.h"
#include "MyCombatComponent.generated.h"

class UAnimInstance;
struct FBranchingPointNotifyPayload;

UCLASS(ClassGroup = (Custom), meta = (BlueprintSpawnableComponent))
class MYGAME_API UMyCombatComponent : public UActorComponent
{
    GENERATED_BODY()

public:
    void BindMontageNotifies(UAnimInstance* AnimInst);

protected:
    UFUNCTION()
    void HandleMontageNotifyBegin(FName NotifyName, const FBranchingPointNotifyPayload& Payload);

    UFUNCTION()
    void HandleMontageNotifyEnd(FName NotifyName, const FBranchingPointNotifyPayload& Payload);

    void ActivateHitDetection();
    void DeactivateHitDetection();
};
```

```cpp
// MyCombatComponent.cpp
#include "MyCombatComponent.h"
#include "Animation/AnimInstance.h"

void UMyCombatComponent::BindMontageNotifies(UAnimInstance* AnimInst)
{
    if (!AnimInst)
    {
        return;
    }

    AnimInst->OnPlayMontageNotifyBegin.AddDynamic(this, &UMyCombatComponent::HandleMontageNotifyBegin);
    AnimInst->OnPlayMontageNotifyEnd.AddDynamic(this, &UMyCombatComponent::HandleMontageNotifyEnd);
}

void UMyCombatComponent::HandleMontageNotifyBegin(FName NotifyName,
    const FBranchingPointNotifyPayload& Payload)
{
    if (NotifyName == FName("EnableHitbox"))
    {
        ActivateHitDetection();
    }
}

void UMyCombatComponent::HandleMontageNotifyEnd(FName NotifyName,
    const FBranchingPointNotifyPayload& Payload)
{
    if (NotifyName == FName("EnableHitbox"))
    {
        DeactivateHitDetection();
    }
}
```

These fire for every named notify on any montage on that instance, so always filter by
`NotifyName`, and call `RemoveDynamic` when the listener is destroyed. The callbacks
must be `UFUNCTION()` because the delegate is dynamic.

---

## Notifies across linked instances

By default a notify fires on the instance that owns the animation. To cross the boundary:

```cpp
// AnimInstance.h:523-531
AnimInst->SetReceiveNotifiesFromLinkedInstances(true);  // main instance receives from layers
AnimInst->SetPropagateNotifiesToLinkedInstances(true);  // source instance sends named notifies to all instances on the mesh
```

The same switches exist per node on `FAnimNode_LinkedAnimGraph`
(`Animation/AnimNode_LinkedAnimGraph.h:76-80`):

```cpp
uint8 bReceiveNotifiesFromLinkedInstances : 1;
uint8 bPropagateNotifiesToLinkedInstances : 1;
```

Turning both on for a deep layer stack makes each notify fire once per instance in the
chain; enable only the direction you need.
