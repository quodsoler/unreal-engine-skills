---
name: ue-cpp-foundations
description: "Use when writing Unreal Engine C++ that touches the reflection system or core types: UCLASS, UPROPERTY, UFUNCTION, USTRUCT, UENUM, GENERATED_BODY, TObjectPtr, TWeakObjectPtr, TSharedPtr, FGCObject, TArray, TMap, TSet, TOptional, TVariant, DECLARE_DYNAMIC_MULTICAST_DELEGATE, AddDynamic, FName, FString, FText, UGameInstanceSubsystem, UWorldSubsystem, GetDefault, TNotNull. Also use when the user mentions 'UE C++', 'garbage collection', 'CDO', 'smart pointers' or 'specifiers'. For Actor and component lifecycle, see ue-actor-component-architecture; for Build.cs and modules, see ue-module-build-system; for Blueprint-facing functions and latent actions, see ue-blueprint-cpp-interop."
metadata:
  version: "2.0.0"
  engine: "5.8"
---

# UE C++ Foundations

Target engine: **UE 5.8**. APIs below are verified against the 5.8 headers; older forms are listed under "Deprecated — do not use".

This skill covers the C++ layer every Unreal module is built on: reflection macros (`UCLASS`, `UPROPERTY`, `UFUNCTION`, `USTRUCT`, `UENUM`, `UINTERFACE`), containers, delegates, string types, object lifetime and garbage collection, CDO access, and the subsystem family. All of it lives in the `Core`, `CoreUObject` and `Engine` modules (`PublicDependencyModuleNames.AddRange(new string[] { "Core", "CoreUObject", "Engine" });`). Editor subsystems additionally need `EditorSubsystem` and `UnrealEd` in an editor-only module.

## Context

Read `.agents/ue-project-context.md` if it exists (module names, conventions, enabled plugins, GAS/networking setup). Do not stop if it is missing.

Identify the area from the request and the codebase. Ask only when two plausible readings would produce different code.

| Request is about… | Go to |
|---|---|
| `UCLASS`, `UPROPERTY`, `UFUNCTION`, `USTRUCT`, `UENUM` specifiers | [Reflection Macros](#reflection-macros) |
| `TArray`, `TMap`, `TSet`, `TOptional`, `TVariant` | [Containers](#containers) |
| Delegates, events, `AddDynamic`, `Broadcast` | [Delegates](#delegates) |
| `FName`, `FString`, `FText`, `LOCTEXT` | [String Types](#string-types) |
| `TObjectPtr`, `TWeakObjectPtr`, `TSharedPtr`, GC roots, `FGCObject` | [Object Lifetime and Pointers](#object-lifetime-and-pointers) |
| Class default objects, `GetDefault`, `TNotNull` parameters | [CDO Access and Non-Null Parameters](#cdo-access-and-non-null-parameters) |
| Game instance / world / local player / engine / editor subsystems | [Subsystems](#subsystems) |
| `Replicated`, `ReplicatedUsing`, `GetLifetimeReplicatedProps` | [Replicated Properties](#replicated-properties) |
| `UE_LOG`, log categories | [Logging](#logging) |
| `WITH_EDITOR`, `UE_BUILD_SHIPPING` guards | [Conditional Compilation Guards](#conditional-compilation-guards) |

## Reflection Macros

Every reflected type has `GENERATED_BODY()` as the first line of its body and `#include "TypeName.generated.h"` as the **last** include of its header. Every `UCLASS` in a game module carries the module API macro (`MYGAME_API`). Specifier parsing is case-insensitive (engine headers use both `config=Game` and `Config=Game`). Complete specifier and `meta=` tables: [references/property-specifiers.md](references/property-specifiers.md).

### UCLASS

| Specifier | Effect |
|---|---|
| `Blueprintable` / `BlueprintType` | Allow Blueprint subclasses / use as a Blueprint variable type (`Not…` forms block them) |
| `Abstract` | Cannot be instantiated; subclasses can |
| `Config=Game`, `DefaultConfig` | `UPROPERTY(Config)` members load from `DefaultGame.ini`; `DefaultConfig` also saves there |
| `Transient` | Instances are never saved |
| `ClassGroup=Name` + `meta=(BlueprintSpawnableComponent)` | Component appears in the Add Component menu |

### UPROPERTY

| Group | Specifiers |
|---|---|
| Editor visibility | `EditAnywhere`, `EditDefaultsOnly`, `EditInstanceOnly`, `VisibleAnywhere`, `VisibleDefaultsOnly`, `VisibleInstanceOnly`, `AdvancedDisplay` |
| Blueprint access | `BlueprintReadWrite`, `BlueprintReadOnly`, `BlueprintGetter=Fn`, `BlueprintSetter=Fn`, `BlueprintAssignable` (multicast delegates only) |
| Serialization | `Transient`, `DuplicateTransient`, `SaveGame`, `Config`, `GlobalConfig`, `Instanced`, `SkipSerialization`, `NonTransactional` |
| Replication | `Replicated`, `ReplicatedUsing=OnRep_Fn`, `NotReplicated` |
| Frequent `meta=` keys | `ClampMin`/`ClampMax`, `UIMin`/`UIMax`, `EditCondition`, `EditConditionHides`, `AllowPrivateAccess`, `DisplayName`, `ToolTip`, `Units`, `AllowedClasses`, `MustImplement`, `ExposeOnSpawn`, `TitleProperty` |

```cpp
// MyCharacter.h
#pragma once
#include "CoreMinimal.h"
#include "GameFramework/Character.h"
#include "MyCharacter.generated.h"

class UStaticMeshComponent;
class UStaticMesh;

UCLASS(Blueprintable)
class MYGAME_API AMyCharacter : public ACharacter
{
    GENERATED_BODY()
public:
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category="Stats", meta=(ClampMin="1.0"))
    float MaxHealth = 100.f;

    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category="Stats")
    float CurrentHealth = 100.f;

    UPROPERTY(Transient)
    TObjectPtr<UStaticMeshComponent> CachedMesh;     // not serialized, still GC-tracked

    UPROPERTY(EditDefaultsOnly, Category="Config")
    TSubclassOf<AActor> ProjectileClass;             // class reference restricted to AActor subclasses
    UPROPERTY(EditDefaultsOnly, Category="Config")
    TSoftObjectPtr<UStaticMesh> LazyMesh;            // path reference; load on demand

    UFUNCTION(BlueprintNativeEvent, Category="Combat")
    void OnDamageTaken(float Amount);
    virtual void OnDamageTaken_Implementation(float Amount);
};
```

### UFUNCTION

| Specifier | Effect |
|---|---|
| `BlueprintCallable` / `BlueprintPure` | Node with / without execution pins; `BlueprintPure` must not mutate state |
| `BlueprintNativeEvent` | Blueprint may override; the C++ default body is `Name_Implementation` |
| `BlueprintImplementableEvent` | Blueprint implements; no C++ body at all |
| `BlueprintAuthorityOnly` / `BlueprintCosmetic` | Blueprint calls run only with network authority / never on a dedicated server (direct C++ calls are not filtered) |
| `Server`, `Client`, `NetMulticast` with `Reliable` or `Unreliable`, `WithValidation` | RPCs — rules and examples in `ue-networking-replication` |
| `Exec` | Console command; the console routes to PlayerInput, PlayerController, Pawn, HUD, GameMode, CheatManager, GameState, PlayerCameraManager and GameInstance |
| `CallInEditor` | Adds a button to the Details panel |

```cpp
// MyCharacter.cpp — the only body a BlueprintNativeEvent gets in C++
void AMyCharacter::OnDamageTaken_Implementation(float Amount) { CurrentHealth = FMath::Max(0.f, CurrentHealth - Amount); }
```

Function libraries, latent and async nodes, `WorldContext`/`DefaultToSelf`/`ExpandEnumAsExecs` and the rest of the Blueprint-facing `meta=` keys: `ue-blueprint-cpp-interop`.

### USTRUCT, UENUM, UINTERFACE

```cpp
// MyTypes.h
#pragma once
#include "CoreMinimal.h"
#include "MyTypes.generated.h"

USTRUCT(BlueprintType)
struct MYGAME_API FMyWeaponStats
{
    GENERATED_BODY()

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category="Weapon")
    float BaseDamage = 10.f;

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category="Weapon")
    float FireRate = 0.5f;
};

UENUM(BlueprintType)
enum class EMyWeaponState : uint8      // Blueprint-visible enums are enum class : uint8
{
    Idle,
    Firing    UMETA(DisplayName="Firing Weapon"),
    MAX       UMETA(Hidden)
};
```

- A `UDataTable` row type is a `USTRUCT` deriving from `FTableRowBase` (`Engine/DataTable.h`); see `ue-data-assets-tables`.
- Initialize every `USTRUCT` member in-class; reflected structs are value types copied freely by the engine.
- A polymorphic struct value (any `USTRUCT` derived from a base) is stored as `FInstancedStruct` or typed `TInstancedStruct<FMyBase>` (`StructUtils/InstancedStruct.h`, module `CoreUObject`).
- `UINTERFACE` pairs a `UMyThing : UInterface` stub with an `IMyThing` class; specifier table and a native example are in [references/property-specifiers.md](references/property-specifiers.md#uinterface-specifiers). Blueprint-implementable interfaces belong to `ue-blueprint-cpp-interop`.

## Containers

Full API, iteration and performance notes: [references/container-patterns.md](references/container-patterns.md). Containers of UObject pointers are GC-tracked only as `UPROPERTY() TArray<TObjectPtr<T>>` (likewise `TMap`/`TSet`).

```cpp
TArray<FString> Names;
Names.Add(TEXT("Alpha"));
Names.Emplace(TEXT("Beta"));                       // construct in place
Names.Reserve(100);                                // pre-size before a known number of Adds
int32 Idx = Names.Find(TEXT("Beta"));              // INDEX_NONE if absent
FString* Ptr = Names.FindByPredicate([](const FString& S) { return S.StartsWith(TEXT("A")); });
Names.Sort([](const FString& A, const FString& B) { return A.Len() < B.Len(); });
Names.Remove(TEXT("Alpha"));                       // order-preserving, O(n)
Names.RemoveAtSwap(0);                             // O(1), reorders
for (int32 i = Names.Num() - 1; i >= 0; --i) { if (Names[i].IsEmpty()) { Names.RemoveAt(i); } }  // never mutate inside ranged-for

TMap<FName, int32> ItemCounts;
int32& Count = ItemCounts.FindOrAdd(FName("Sword"));  // inserts default if absent
int32* Found = ItemCounts.Find(FName("Axe"));         // nullptr if absent
for (const TPair<FName, int32>& Pair : ItemCounts) { UE_LOG(LogMyGame, Log, TEXT("%s x%d"), *Pair.Key.ToString(), Pair.Value); }

TSet<FName> Tags;
Tags.Add(FName("Flying"));
bool bFlying = Tags.Contains(FName("Flying"));     // hashed, no duplicates

TOptional<float> MaybeHP;
float Safe = MaybeHP.Get(0.f);                     // default when unset
```

### TVariant

```cpp
#include "Misc/TVariant.h"
#include <type_traits>

TVariant<int32, float, FString> Value;             // first type must be default-constructible (or use FEmptyVariantState)
Value.Set<FString>(TEXT("Hello"));
if (const FString* Str = Value.TryGet<FString>()) { UE_LOG(LogMyGame, Log, TEXT("%s"), **Str); }

Visit([](auto& Held)                               // one generic lambda; branch with if constexpr
{
    using HeldType = std::decay_t<decltype(Held)>;
    if constexpr (std::is_same_v<HeldType, FString>) { UE_LOG(LogMyGame, Log, TEXT("%s"), *Held); }
}, Value);
```

## Delegates

All declaration macros, binding methods and payload rules: [references/delegate-patterns.md](references/delegate-patterns.md).

| Macro family | Bindings | Blueprint | Use for |
|---|---|---|---|
| `DECLARE_DELEGATE` (`_RetVal`, `_OneParam` …) | 1 | No | Single-owner callbacks, may return a value |
| `DECLARE_MULTICAST_DELEGATE` | N | No | C++ events (`DECLARE_TS_…` for other threads) |
| `DECLARE_DYNAMIC_DELEGATE` | 1 | Yes | Blueprint-assignable callback (UFUNCTION targets only) |
| `DECLARE_DYNAMIC_MULTICAST_DELEGATE` | N | Yes | `UPROPERTY(BlueprintAssignable)` events |

```cpp
// MyHealthComponent.h — delegate types are declared at file scope, above the UCLASS
DECLARE_MULTICAST_DELEGATE_TwoParams(FOnMyHealthChangedNative, float /*Current*/, float /*Max*/);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_TwoParams(FOnMyHealthChanged, float, CurrentHealth, float, MaxHealth);

// in UMyHealthComponent : public UActorComponent
UPROPERTY(BlueprintAssignable, Category="Health")
FOnMyHealthChanged OnHealthChanged;               // dynamic: needs UPROPERTY, params need names
FOnMyHealthChangedNative OnHealthChangedNative;   // native: no reflection, cheaper to broadcast

// in AMyListenerActor : public AActor
UFUNCTION()                                       // AddDynamic targets must be UFUNCTIONs
void HandleHealthChanged(float CurrentHealth, float MaxHealth);
void HandleHealthChangedNative(float Current, float Max);
UPROPERTY() TObjectPtr<UMyHealthComponent> Health;
FDelegateHandle NativeHandle;
```

```cpp
// AMyListenerActor.cpp — bind in BeginPlay (after Super::), unbind in EndPlay (before Super::)
Health->OnHealthChanged.AddDynamic(this, &AMyListenerActor::HandleHealthChanged);
NativeHandle = Health->OnHealthChangedNative.AddUObject(this, &AMyListenerActor::HandleHealthChangedNative);
// EndPlay: guard with IsValid(Health) first
Health->OnHealthChanged.RemoveDynamic(this, &AMyListenerActor::HandleHealthChanged);
Health->OnHealthChangedNative.Remove(NativeHandle);
```

The owner broadcasts with `OnHealthChanged.Broadcast(CurrentHealth, MaxHealth);`. Both classes in full: [references/delegate-patterns.md](references/delegate-patterns.md).

Single delegates bind with `BindUObject`, `BindSP`, `BindRaw`, `BindStatic`, `BindLambda`, `BindWeakLambda(this, ...)` and fire with `ExecuteIfBound(...)`. For a lambda that captures `this`, prefer `AddWeakLambda`/`BindWeakLambda` so the binding dies with the object.

## String Types

| Type | Use for | Compare | Notes |
|---|---|---|---|
| `FName` | Identifiers, asset and bone names, tags | O(1) integer, case-insensitive | Interned in a global table; immutable |
| `FString` | Paths, parsing, runtime-built text | O(n) | Mutable heap string; `*Str` yields `const TCHAR*` |
| `FText` | Anything a player reads | — | Localizable; build with `LOCTEXT`/`NSLOCTEXT`/`FText::Format` |

```cpp
float Health = 42.f; int32 Current = 3; int32 Max = 5;

FName   Tag(TEXT("Weapon.Rifle"));
FString S    = Tag.ToString();
FName   Back = FName(*S);
FString Msg  = FString::Printf(TEXT("HP: %.1f (%d/%d)"), Health, Current, Max);
int32 Parsed = FCString::Atoi(TEXT("42"));

#define LOCTEXT_NAMESPACE "MyGame"
FText Label = LOCTEXT("RifleName", "Assault Rifle");
FText Fmt   = FText::Format(LOCTEXT("HPFmt", "HP: {0}/{1}"), FText::AsNumber(Current), FText::AsNumber(Max));
#undef LOCTEXT_NAMESPACE
FText Other = NSLOCTEXT("MyGame", "Key", "Text");   // no #define needed
FText Raw   = FText::FromString(S);                 // not localized; runtime-generated text only
```

## Object Lifetime and Pointers

The garbage collector keeps alive every `UObject` reachable from a root through reflected (`UPROPERTY`) references, `FGCObject` referencers, `TStrongObjectPtr` and rooted objects. A pointer the GC cannot see may dangle.

```cpp
// Reflected members: TObjectPtr — GC-tracked, access-tracked in the editor, used like a raw pointer
UPROPERTY() TObjectPtr<UStaticMeshComponent> MeshComp;

// Legacy APIs that want raw pointers
void FillObjects(TArray<UObject*>& Out);
TArray<TObjectPtr<UObject>> Objects;
FillObjects(MutableView(Objects));                       // temporary TArray<UObject*>& view; ToRawPtr(Ptr) for a single read-only copy

// Non-owning reference: TWeakObjectPtr becomes null once the object is destroyed or collected
TWeakObjectPtr<AMyCharacter> WeakTarget;
if (AMyCharacter* Target = WeakTarget.Get()) { Target->OnDamageTaken(5.f); }

// Keep-alive from non-reflected code: scoped root instead of AddToRoot()/RemoveFromRoot()
TStrongObjectPtr<UMyDataObject> Pinned(NewObject<UMyDataObject>());

// Plain C++ types only: never wrap a UObject in TSharedPtr/TUniquePtr
struct FMyData { int32 Score = 0; };
TSharedPtr<FMyData> Data = MakeShared<FMyData>();
TWeakPtr<FMyData>   WeakData = Data;                     // Pin() returns a TSharedPtr if still alive
```

Non-UObject classes that hold UObjects derive from `FGCObject` and report them through `TObjectPtr` members; the raw `UObject*` overloads of `AddReferencedObject(s)` are deprecated and unsafe with incremental GC.

```cpp
#include "UObject/GCObject.h"

class FMyManager : public FGCObject
{
public:
    virtual void AddReferencedObjects(FReferenceCollector& Collector) override { Collector.AddReferencedObject(ManagedObject); }
    virtual FString GetReferencerName() const override { return TEXT("FMyManager"); }
private:
    TObjectPtr<UMyDataObject> ManagedObject;   // TObjectPtr<T>& overload; raw UObject* is deprecated
};
```

## CDO Access and Non-Null Parameters

```cpp
#include "Misc/NotNull.h"

const UMyDataObject* Defaults = GetDefault<UMyDataObject>();        // class default object, read-only
UMyDataObject*       Settings = GetMutableDefault<UMyDataObject>(); // settings-style objects
UClass* SomeClass = AMyCharacter::StaticClass();
const AMyCharacter* ClassCDO = SomeClass->GetDefaultObject<AMyCharacter>();  // CDO of a runtime UClass

// TNotNull<T*>: caller cannot pass null; checked at the call site in Debug/Development, a plain pointer in Test and Shipping
void RegisterOwner(TNotNull<AActor*> Owner);
void RegisterOwner(TNotNull<AActor*> Owner)
{
    Owner->SetActorHiddenInGame(false);   // no null check needed inside
}
```

`UClass::ClassDefaultObject` is deprecated for direct access. `TNotNull<T*>` converts implicitly to `T*`; keep `T*` where null is a meaningful value.

## Subsystems

Auto-instanced singletons owned by an engine object; no manual rooting. Override `Initialize(FSubsystemCollectionBase&)`/`Deinitialize()`; call `Collection.InitializeDependency<UMyOtherSubsystem>()` inside `Initialize` when ordering matters; return `false` from `ShouldCreateSubsystem(UObject* Outer) const` to opt out.

| Class | Header | Owner / lifetime | Accessor |
|---|---|---|---|
| `UEngineSubsystem` | `Subsystems/EngineSubsystem.h` | `UEngine`; whole process | `GEngine->GetEngineSubsystem<T>()` |
| `UGameInstanceSubsystem` | `Subsystems/GameInstanceSubsystem.h` | `UGameInstance`; survives map changes | `GetGameInstance()->GetSubsystem<T>()` |
| `UWorldSubsystem` | `Subsystems/WorldSubsystem.h` | `UWorld`; recreated per world; filter with `DoesSupportWorldType` | `GetWorld()->GetSubsystem<T>()` |
| `UTickableWorldSubsystem` | `Subsystems/WorldSubsystem.h` | As above, plus `Tick()`; must implement `GetStatId()` | `GetWorld()->GetSubsystem<T>()` |
| `ULocalPlayerSubsystem` | `Subsystems/LocalPlayerSubsystem.h` | `ULocalPlayer`; one per local player | `LocalPlayer->GetSubsystem<T>()`, `ULocalPlayer::GetSubsystemFromController<T>(PC)` |
| `UEditorSubsystem` | `EditorSubsystem.h` (module `EditorSubsystem`) | `UEditorEngine`; editor session only | `GEditor->GetEditorSubsystem<T>()` |

```cpp
// MyInventorySubsystem.h
#pragma once
#include "CoreMinimal.h"
#include "Subsystems/GameInstanceSubsystem.h"
#include "MyInventorySubsystem.generated.h"

UCLASS()
class MYGAME_API UMyInventorySubsystem : public UGameInstanceSubsystem
{
    GENERATED_BODY()
public:
    virtual void Initialize(FSubsystemCollectionBase& Collection) override;
    virtual void Deinitialize() override;

    UFUNCTION(BlueprintCallable, Category="Inventory")
    void AddItem(FName ItemID, int32 Count);
private:
    TMap<FName, int32> Inventory;
};

// A ticking world subsystem adds these; Initialize/Deinitialize must call Super:: (the base registers the ticker)
UCLASS()
class MYGAME_API UMyTickableSubsystem : public UTickableWorldSubsystem
{
    GENERATED_BODY()
public:
    virtual void Tick(float DeltaTime) override;
    virtual TStatId GetStatId() const override { RETURN_QUICK_DECLARE_CYCLE_STAT(UMyTickableSubsystem, STATGROUP_Tickables); }
    virtual bool DoesSupportWorldType(const EWorldType::Type WorldType) const override;  // e.g. Game and PIE only
};
```

```cpp
// Access from an AActor member function
UMyInventorySubsystem* Inv = GetGameInstance()->GetSubsystem<UMyInventorySubsystem>();
UMyTickableSubsystem*  Tck = GetWorld()->GetSubsystem<UMyTickableSubsystem>();
APlayerController*     PC  = GetWorld()->GetFirstPlayerController();
UMyUISubsystem*        UI  = ULocalPlayer::GetSubsystemFromController<UMyUISubsystem>(PC);   // null-safe
UMyEngineSubsystem*    Eng = GEngine->GetEngineSubsystem<UMyEngineSubsystem>();
```

`UWorldSubsystem` also offers `PostInitialize()`, `OnWorldBeginPlay(UWorld& InWorld)`, `OnWorldEndPlay(UWorld& InWorld)` and `OnWorldComponentsUpdated(UWorld& World)` overrides; `ULocalPlayerSubsystem::GetLocalPlayer<T>()` and `UGameInstanceSubsystem::GetGameInstance()` return the owner.

## Replicated Properties

Both the specifier and the `GetLifetimeReplicatedProps` entry are required. Conditions (`COND_*`), RPC rules, push model and Iris: `ue-networking-replication`.

```cpp
// MyReplicatedActor.h
#pragma once
#include "CoreMinimal.h"
#include "GameFramework/Actor.h"
#include "MyReplicatedActor.generated.h"

UCLASS()
class MYGAME_API AMyReplicatedActor : public AActor
{
    GENERATED_BODY()
public:
    AMyReplicatedActor() { bReplicates = true; }
    virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override;
protected:
    UPROPERTY(ReplicatedUsing=OnRep_Health)
    float Health = 100.f;

    UFUNCTION()
    void OnRep_Health();                 // runs on clients after Health changes
};
```

```cpp
// MyReplicatedActor.cpp
#include "MyReplicatedActor.h"
#include "Net/UnrealNetwork.h"          // required for DOREPLIFETIME

void AMyReplicatedActor::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);
    DOREPLIFETIME(AMyReplicatedActor, Health);
}

void AMyReplicatedActor::OnRep_Health() {}
```

## Logging

```cpp
DECLARE_LOG_CATEGORY_EXTERN(LogMyGame, Log, All);   // MyGame.h (module header)
DEFINE_LOG_CATEGORY(LogMyGame);                     // MyGame.cpp
UE_LOG(LogMyGame, Warning, TEXT("HP low: %.1f"), 12.5f);
```

Verbosities (`Logging/LogVerbosity.h`): `Fatal`, `Error`, `Warning`, `Display`, `Log`, `Verbose`, `VeryVerbose`. `Display` prints to the console and the log file; `Log` goes to the log file only. Shipping builds compile `UE_LOG` out unless the target sets `bUseLoggingInShipping = true`. Categories, structured logging (`UE_LOGFMT`), runtime verbosity control and profiling: `ue-testing-debugging`.

## Conditional Compilation Guards

```cpp
#if WITH_EDITORONLY_DATA
    UPROPERTY(EditAnywhere, Category="Debug")
    bool bShowDebugSpheres = false;                 // reflected editor-only data
#endif
#if WITH_EDITOR
    virtual void PostEditChangeProperty(FPropertyChangedEvent& PropertyChangedEvent) override;   // editor-only code
#endif
#if !UE_BUILD_SHIPPING
    void DrawDebugInfo();                           // Debug, Development and Test only
#endif
```

## Deprecated — do not use

| Do not emit | Use in 5.8 | Source |
|---|---|---|
| `SomeClass->ClassDefaultObject` | `GetDefault<T>()`, `GetMutableDefault<T>()`, `SomeClass->GetDefaultObject<T>()` | `UE_DEPRECATED(5.6)` in `UObject/Class.h:4045` |
| `Collector.AddReferencedObject(RawUObjectPtr)` with `UObject*` members | `TObjectPtr<T>` members passed to `AddReferencedObject(s)` | `UE_DEPRECATED(5.3)` via `UE_REFERENCE_COLLECTOR_REQUIRE_OBJECTPTR_DEPRECATED` in `UObject/UObjectGlobals.h:109` |
| `ToRawPtrArrayUnsafe(Array)`, `ToRawPtrTArrayUnsafe(Array)` on mutable arrays | `MutableView(Array)` or keep `TArray<TObjectPtr<T>>` | `UE_DEPRECATED(5.6)` in `UObject/ObjectPtr.h:1112,1156` |
| `Visit(<overload-set helper struct>{...}, Variant)` | One generic lambda with `if constexpr (std::is_same_v<...>)` | Helper is Lyra sample code; absent from 5.8 headers (`Misc/TVariant.h`) |
| `GENERATED_USTRUCT_BODY()`, `GENERATED_UCLASS_BODY()`, `GENERATED_IINTERFACE_BODY()` | `GENERATED_BODY()` | Legacy aliases in `UObject/ObjectMacros.h:802-805` |
| `Obj->AddToRoot()` / `RemoveFromRoot()` for temporary keep-alive | `TStrongObjectPtr<T>` (`UObject/StrongObjectPtr.h`) | Rooting is permanent until removed; scoped pointer cannot leak |
| `TMap::TIterator(Map, /*bRequiresRehashOnRemoval*/ true)` (auto-`Relax` after removal) | `Map.CreateIterator()`; call `Compact()`/`Shrink()` yourself if needed | `UE_DEPRECATED(5.8)` in `Containers/Map.h.inl:916`; `TSet::Relax()` itself is not yet marked (`Containers/SparseSet.h.inl:319`) |
| `TIsConst<T>`, `TIsMemberPointer<T>` | `std::is_const_v<T>`, `std::is_member_pointer_v<T>` | `UE_DEPRECATED(5.7)` `Templates/IsConst.h:15`; `UE_DEPRECATED(5.8)` `Templates/IsMemberPointer.h:14` |
| `FThreadSafeRefCountedObject`, `FRefCountBase` | `FRefCountedObject` (now thread-safe) | `UE_DEPRECATED(5.8)` in `Templates/RefCounting.h:151-153` |
| `FCoreDelegates::OnPostEngineInit.AddRaw(...)` | `FCoreDelegates::GetOnPostEngineInit().AddRaw(...)` | `UE_DEPRECATED(5.8)` in `Misc/CoreDelegates.h:240` |
| `Delegate.ProcessMulticastDelegate<UObject>(Params)` | `Delegate.ProcessDelegate<UObject>(Params)` | `UE_DEPRECATED(5.8)` in `UObject/ScriptDelegates.h:1462` |
| `meta=(HideAssetPicker)` | `meta=(HidePinAssetPicker)` | `UE_DEPRECATED(5.6)` in `UObject/ObjectMacros.h:1673` |
| `ObjPtr.IsRemote()` | `ObjPtr.GetResidence()` | `UE_DEPRECATED(5.8)` in `UObject/ObjectPtr.h:206` |

## Common Mistakes

**Raw `UObject*` member without `UPROPERTY` — dangling after GC:**
```cpp
UMyDataObject* Obj;                        // WRONG: invisible to GC
UPROPERTY() TObjectPtr<UMyDataObject> Obj; // RIGHT
```

**Mutating a `TArray` inside ranged-for — iterator invalidation:**
```cpp
TArray<AActor*> Actors;
for (AActor* A : Actors) { Actors.Remove(A); }                                                      // WRONG
for (int32 i = Actors.Num() - 1; i >= 0; --i) { if (!IsValid(Actors[i])) { Actors.RemoveAt(i); } }  // RIGHT
```

**`TSharedPtr` around a `UObject` — two owners, one crash:** UObjects are owned by the GC; use `TObjectPtr`, `TWeakObjectPtr` or `TStrongObjectPtr`.

**`.generated.h` not the last include, or `MYGAME_API` missing:** UnrealHeaderTool rejects the header; the missing API macro surfaces later as `LNK2019` from other modules.

**`AddDynamic` bound to a non-`UFUNCTION`:** the reflection lookup fails at runtime and the handler never runs; mark it `UFUNCTION()`.

**`UPROPERTY`/`UFUNCTION` written above a `.cpp` definition:** UnrealHeaderTool only parses headers; the macro is silently ignored. Put it on the declaration inside the class body.

**`BlueprintNativeEvent` without `Name_Implementation`:** unresolved external at link time. Declare `virtual void Name_Implementation(...)` and define it.

**`UTickableWorldSubsystem::Initialize`/`Deinitialize` override without `Super::`:** the tickable registration lives in the base implementation; the subsystem never ticks.

**Reading `ClassDefaultObject` directly:** deprecated and about to become private; use `GetDefault<T>()`.

**`Log` verbosity "not showing up":** `Log` is file-only by design; use `Display` for console output.

## Related Skills

- `ue-actor-component-architecture` — AActor/UActorComponent lifecycle, spawning, tick groups, component setup
- `ue-module-build-system` — Build.cs, module dependencies, API export macros, include paths, PCH
- `ue-testing-debugging` — log categories and verbosity, structured logging, automation tests, profiling
- `ue-networking-replication` — DOREPLIFETIME conditions, RPC rules, push model, Iris
- `ue-blueprint-cpp-interop` — UBlueprintFunctionLibrary, latent and async actions, Blueprint-implementable
- Also relevant: `ue-data-assets-tables`, `ue-animation-system`, `ue-async-threading`, `ue-audio-system`, `ue-editor-tools`, `ue-gameplay-abilities`, `ue-gameplay-framework`, `ue-gameplay-tags-messaging`, `ue-input-system`, `ue-mass-entity`, `ue-materials-rendering`, `ue-serialization-savegames`, `ue-state-trees`, `ue-ui-umg-slate`, `ue-world-level-streaming`
