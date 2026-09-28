# Specifier Reference

Specifiers accepted by `UCLASS()`, `UPROPERTY()`, `UFUNCTION()`, `USTRUCT()`, `UENUM()` and `UINTERFACE()` in UE 5.8, taken from the `UC::`, `UP::`, `UF::`, `US::`, `UI::` and `UM::` enumerations in `Engine/Source/Runtime/CoreUObject/Public/UObject/ObjectMacros.h`. Parsing is case-insensitive. Keys that only exist inside `meta=(...)` are listed as meta keys and never work as bare specifiers.

---

## UPROPERTY() Specifiers

### Editor Visibility and Editability

| Specifier | Editable | Where it shows |
|-----------|----------|----------------|
| `EditAnywhere` | Yes | Archetype (Blueprint defaults) and instance Details panels |
| `EditDefaultsOnly` | Yes | Archetype only |
| `EditInstanceOnly` | Yes | Placed instances only |
| `VisibleAnywhere` | No | Archetype and instance, read-only |
| `VisibleDefaultsOnly` | No | Archetype only, read-only |
| `VisibleInstanceOnly` | No | Instances only, read-only |
| `AdvancedDisplay` | — | Collapsed under the category's Advanced arrow |
| `SimpleDisplay` | — | Shown even when the category is in simple view |
| `EditFixedSize` | — | Array length cannot change in the editor |
| `NoClear` | — | Object reference cannot be set to null in the editor |
| `Interp` | — | Exposed to Sequencer tracks |

A property with none of these does not appear in the Details panel.

### Blueprint Access

| Specifier | Read | Write |
|-----------|------|-------|
| `BlueprintReadWrite` | Yes | Yes |
| `BlueprintReadOnly` | Yes | No |
| `BlueprintGetter=FuncName` | Through the named `UFUNCTION(BlueprintGetter)` | — |
| `BlueprintSetter=FuncName` | — | Through the named `UFUNCTION(BlueprintSetter)` |
| `BlueprintAssignable` | Blueprint binds events to this multicast delegate | — |
| `BlueprintCallable` | Blueprint can call `Broadcast` on this delegate | — |
| `BlueprintAuthorityOnly` | Blueprint may bind only `BlueprintAuthorityOnly` events to this delegate | — |
| `FieldNotify` | Property participates in MVVM field notification | — |

Private members exposed to Blueprint need `meta=(AllowPrivateAccess="true")`.

### Replication

| Specifier | Effect |
|-----------|--------|
| `Replicated` | Replicated to clients; register it in `GetLifetimeReplicatedProps` |
| `ReplicatedUsing=FuncName` | Replicated; the named `UFUNCTION()` runs on clients when the value arrives |
| `NotReplicated` | Excludes a member of a replicated struct |

The registration macros, `COND_*` conditions, push model and RPC rules are covered in `ue-networking-replication`.

### Serialization and Config

| Specifier | Effect |
|-----------|--------|
| `Transient` | Not saved; default-initialized on load |
| `DuplicateTransient` | Reset when the object is duplicated |
| `NonPIEDuplicateTransient` | Reset on duplication except for Play In Editor |
| `TextExportTransient` | Skipped by text export (copy/paste, T3D) |
| `NonTransactional` | Changes are not recorded for undo/redo |
| `SaveGame` | Included by save-game archives that filter on this flag |
| `SkipSerialization` | Never serialized, but still reflected |
| `Config` | Loaded from the class's `Config=` ini section |
| `GlobalConfig` | Like `Config`, but only the base class section is used |
| `Instanced` | The referenced object is a sub-object owned by this object |
| `Export` | Referenced object is exported with this object |
| `AssetRegistrySearchable` | Value is written to the Asset Registry tags |

### Property Meta Keys

```cpp
UPROPERTY(EditAnywhere, BlueprintReadWrite, Category="Stats",
    meta=(ClampMin="0.0", ClampMax="100.0", UIMin="0.0", UIMax="100.0",
          ToolTip="Character health, clamped to [0, 100]", DisplayName="Health Points",
          EditCondition="bHealthEnabled", EditConditionHides, AllowPrivateAccess="true"))
float Health = 100.f;
```

| Meta key | Effect |
|----------|--------|
| `ClampMin` / `ClampMax` | Hard clamp on numeric input |
| `UIMin` / `UIMax` | Slider range without clamping typed values |
| `Delta`, `SliderExponent`, `LinearDeltaSensitivity` | Slider behaviour |
| `ArrayClamp` | Clamps an integer to the valid indices of the named array property |
| `ToolTip`, `ShortTooltip` | Tooltip text |
| `DisplayName` | Label override |
| `DisplayPriority`, `DisplayAfter` | Ordering inside the category |
| `EditCondition` | Boolean expression that enables editing |
| `EditConditionHides` | Hide instead of grey out when the condition is false |
| `InlineEditConditionToggle` | Show this bool as a checkbox next to the property that uses it |
| `AllowPrivateAccess` | Blueprint access to a private or protected member |
| `Units`, `ForceUnits` | Unit suffix (`cm`, `deg`, `s`, ...); `ForceUnits` disables conversion |
| `MakeStructureDefaultValue` | Default for struct members in user-defined structs |
| `MustImplement` | `TSubclassOf`/class pickers limited to classes implementing the interface |
| `MetaClass` | Base class for `FSoftClassPath` pickers |
| `AllowedClasses` / `DisallowedClasses` | Filter object and class pickers |
| `AllowAbstract`, `ExactClass`, `ShowTreeView`, `HideViewOptions` | Class picker behaviour |
| `RequiredAssetDataTags` / `DisallowedAssetDataTags` | Filter the asset picker by Asset Registry tags |
| `AssetBundles` | Bundle names for soft references in primary assets |
| `GetOptions` | Function returning the allowed `FName`/`FString` values |
| `TitleProperty` | Member shown as the array element title |
| `NoElementDuplicate`, `EditFixedOrder` | Array editing restrictions |
| `NoResetToDefault` | Hides the reset arrow |
| `NoEditInline` | Do not expand an `Instanced` object inline |
| `ShowOnlyInnerProperties` | Flatten a struct's members into the parent category |
| `HideInDetailPanel` | Hidden in Details, still Blueprint-visible |
| `HideAlphaChannel` | Colour pickers without alpha |
| `MultiLine`, `PasswordField`, `MaxLength` | Text field behaviour |
| `ContentDir`, `RelativePath`, `RelativeToGameContentDir`, `LongPackageName`, `FilePathFilter` | Path pickers |
| `ExposeOnSpawn` | Pin on SpawnActor / Construct Object nodes |
| `Bitmask`, `BitmaskEnum` | Integer edited as flags of the named `UENUM(meta=(Bitflags))` |
| `Categories` | Gameplay tag filter root for `FGameplayTag`/`FGameplayTagContainer` properties (read by the GameplayTags property customization, not `UM::`) |
| `DeprecatedProperty`, `DeprecationMessage` | Mark as deprecated with a message |

---

## UFUNCTION() Specifiers

| Specifier | Effect |
|-----------|--------|
| `BlueprintCallable` | Blueprint node with execution pins |
| `BlueprintPure` | Node without execution pins; implies no side effects |
| `BlueprintNativeEvent` | Blueprint may override; C++ body is `Name_Implementation` |
| `BlueprintImplementableEvent` | Blueprint implements; no C++ body |
| `BlueprintAuthorityOnly` | Blueprint calls run only when the owner has network authority |
| `BlueprintCosmetic` | Blueprint calls are skipped on dedicated servers |
| `BlueprintGetter` / `BlueprintSetter` | Accessor referenced by a property's `BlueprintGetter=`/`BlueprintSetter=` |
| `BlueprintInternalUseOnly` | Hidden from the palette (used by generated proxies) |
| `Server` / `Client` / `NetMulticast` | RPC executes on the server / owning client / everyone |
| `Reliable` / `Unreliable` | Delivery guarantee for an RPC |
| `WithValidation` | Requires `bool Name_Validate(...)`; returning false disconnects the caller |
| `Exec` | Console command on classes the console routes to (PlayerController, Pawn, HUD, GameMode, GameState, CheatManager, PlayerCameraManager, PlayerInput, GameInstance) |
| `CallInEditor` | Button in the Details panel |
| `Category="..."` | Palette grouping |
| `SealedEvent` | Event cannot be overridden in Blueprint |
| `CustomThunk` | Hand-written `execName` thunk (wildcard parameters) |
| `FieldNotify` | Function participates in MVVM field notification |

**`BlueprintNativeEvent` pattern:**
```cpp
// MyActor.h (inside the UCLASS body)
UFUNCTION(BlueprintNativeEvent, Category="Spawning")
FVector GetSpawnLocation() const;
virtual FVector GetSpawnLocation_Implementation() const;

// MyActor.cpp
FVector AMyActor::GetSpawnLocation_Implementation() const
{
    return GetActorLocation() + FVector(0.f, 0.f, 200.f);
}
```

### Function Meta Keys

| Meta key | Effect |
|----------|--------|
| `DisplayName` | Node title override |
| `ToolTip`, `ShortTooltip` | Tooltip text |
| `Keywords` | Extra search terms in the palette |
| `CompactNodeTitle` | Renders as a compact node with this title |
| `AdvancedDisplay` | Parameters (by name or count) hidden behind the Advanced arrow |
| `DefaultToSelf` | Parameter pin defaults to Self |
| `HidePin` | Hides a parameter pin |
| `WorldContext` | Parameter that receives the world context object |
| `CallableWithoutWorldContext` | Callable from graphs with no world context |
| `AutoCreateRefTerm` | Reference parameters get an inline default value |
| `DeterminesOutputType` / `DynamicOutputParam` | Output pin type follows a class input |
| `ExpandEnumAsExecs` / `ExpandBoolAsExecs` | One execution pin per enum value / per bool |
| `Latent`, `LatentInfo` | Latent action node with an `FLatentActionInfo` parameter |
| `BlueprintThreadSafe` / `NotBlueprintThreadSafe` | Allowed / disallowed in thread-safe animation graphs |
| `BlueprintProtected` | Only callable from the owning Blueprint class |
| `DevelopmentOnly` | Node compiled out of Shipping |
| `UnsafeDuringActorConstruction` | Not callable from construction scripts |
| `BlueprintAutocast` | Used as an implicit conversion node |
| `DeprecatedFunction`, `DeprecationMessage` | Mark as deprecated with a message |

Function libraries, latent actions, async action proxies (`ExposedAsyncProxy`) and script bindings (`ScriptMethod`, `ScriptName`) are covered in `ue-blueprint-cpp-interop`.

---

## UCLASS() Specifiers

| Specifier | Effect |
|-----------|--------|
| `Blueprintable` / `NotBlueprintable` | Allow / block Blueprint subclasses |
| `BlueprintType` / `NotBlueprintType` | Allow / block use as a Blueprint variable type |
| `Abstract` | Cannot be instantiated directly |
| `Config=<Name>` | `Config` properties load from `Default<Name>.ini` and the user's `<Name>.ini` |
| `DefaultConfig` | Saves to `Default<Name>.ini` instead of the user ini |
| `PerObjectConfig` | Each instance gets its own ini section, keyed by object name |
| `ConfigDoNotCheckDefaults` | Skip default-value comparison when saving config |
| `Transient` / `NonTransient` | Instances are never saved / undo a parent's `Transient` |
| `Deprecated` | Class is deprecated; its objects are not saved (inherited by subclasses) |
| `Within=<ClassName>` | Outer must be an instance of the class |
| `ClassGroup=<Name>` | Group in the Add Component menu (with `meta=(BlueprintSpawnableComponent)`) |
| `Placeable` / `NotPlaceable` | Can / cannot be placed in a level |
| `Const` | All properties and functions are const |
| `CollapseCategories` / `DontCollapseCategories` | Flatten categories in Details / undo a parent's flattening |
| `AutoExpandCategories=(...)` / `AutoCollapseCategories=(...)` / `DontAutoCollapseCategories=(...)` | Category expansion defaults |
| `HideCategories=(...)` / `ShowCategories=(...)` | Hide / re-show categories |
| `PrioritizeCategories=(...)` | Categories listed first |
| `HideFunctions=(...)` / `ShowFunctions=(...)` | Hide / re-show functions in Blueprint |
| `HideDropdown` | Not offered in class pickers |
| `MinimalAPI` | Export only type information |
| `CustomConstructor` | Skip the generated constructor declaration |
| `EditInlineNew` | Can be created inline from an `Instanced` property |
| `DefaultToInstanced` | Object properties of this class default to `Instanced` |
| `SparseClassDataType=<Struct>` | Store rarely-changing defaults in a sparse struct |
| `ComponentWrapperClass` | Actor exists only to wrap one component |
| `Experimental` / `EarlyAccessPreview` | Editor badge and warning |
| `EditorConfig=<Name>` | Editor-only JSON config |
| `Intrinsic` / `NoExport` | Class defined without generated code / header parsed for metadata only |
| `CustomFieldNotify` | Hand-written field-notify implementation |

Class meta keys: `meta=(BlueprintSpawnableComponent)`, `meta=(ChildCanTick)` / `meta=(ChildCannotTick)`, `meta=(IsBlueprintBase="true")`, `meta=(DisplayName="...")`, `meta=(ToolTip="...")`, `meta=(ShortTooltip="...")`, `meta=(IgnoreCategoryKeywordsInSubclasses)`, `meta=(DontUseGenericSpawnObject)`, `meta=(LoadBehavior="LazyOnDemand")`.

---

## USTRUCT() Specifiers

| Specifier | Effect |
|-----------|--------|
| `BlueprintType` | Usable as a Blueprint variable, with Make/Break nodes |
| `Atomic` | Always serialized as a whole (no per-member deltas) |
| `Immutable` | Only legal in `UObject/Object.h` and being phased out; do not use on new structs |
| `NoExport` | No generated code; header parsed for metadata only |
| `BlueprintInternalUseOnly` | Hidden from Blueprint (generated helper structs) |
| `BlueprintInternalUseOnlyHierarchical` | As above, including all derived structs |

Struct meta keys: `meta=(HasNativeMake="/Script/Module.Class:Function")`, `meta=(HasNativeBreak="...")`, `meta=(HiddenByDefault)`, `meta=(DisableSplitPin)`, `meta=(ShowOnlyInnerProperties)`.

Serialization hooks and `TStructOpsTypeTraits` (`WithSerializer`, `WithNetSerializer`, `WithIdenticalViaEquality`, ...) are covered in `ue-serialization-savegames` and `ue-networking-replication`.

---

## UENUM() Specifiers

| Specifier / meta | Effect |
|------------------|--------|
| `BlueprintType` | Usable in Blueprint (requires `enum class : uint8`) |
| `meta=(Bitflags)` | Values are bit flags; pair with `UPROPERTY(meta=(Bitmask, BitmaskEnum="/Script/MyGame.EMyFlags"))` |
| `meta=(UseEnumValuesAsMaskValuesInEditor="true")` | Enumerators already hold mask values instead of bit indices |
| `meta=(ScriptName="...")` | Name exposed to scripting |

**`UMETA` per-value keys:**

| Key | Effect |
|-----|--------|
| `DisplayName="Label"` | Name shown in Blueprint and the editor |
| `Hidden` | Removed from pickers (sentinels such as `MAX`) |
| `ToolTip="text"` | Tooltip for the value |

```cpp
UENUM(BlueprintType)
enum class EMyGamePhase : uint8
{
    PreGame    UMETA(DisplayName="Pre-Game"),
    InGame     UMETA(DisplayName="In Game"),
    PostGame   UMETA(DisplayName="Post-Game"),
    MAX        UMETA(Hidden)
};
```

Replicated enum properties must fit in a byte: declare `enum class : uint8`. `TEnumAsByte<T>` exists only for legacy namespaced enums (`namespace EMyOld { enum Type { ... }; }`).

---

## UINTERFACE() Specifiers

| Specifier | Effect |
|-----------|--------|
| `Blueprintable` / `NotBlueprintable` | Blueprint classes may / may not implement the interface |
| `MinimalAPI` | Export only the type information of the `U` stub |
| `ConversionRoot` | Root of a Blueprint conversion hierarchy |
| `meta=(CannotImplementInterfaceInBlueprint)` | C++-only interface; functions may then be plain virtuals or `BlueprintCallable` without `BlueprintNativeEvent` |

Native-only interface (no Blueprint implementation):

```cpp
// MyDamageable.h
#pragma once
#include "CoreMinimal.h"
#include "UObject/Interface.h"
#include "MyDamageable.generated.h"

UINTERFACE(MinimalAPI, NotBlueprintable, meta=(CannotImplementInterfaceInBlueprint))
class UMyDamageable : public UInterface
{
    GENERATED_BODY()
};

class MYGAME_API IMyDamageable
{
    GENERATED_BODY()
public:
    virtual void ApplyMyDamage(float Amount, AActor* DamageInstigator) = 0;
};
```

```cpp
// MyTarget.h
#pragma once
#include "CoreMinimal.h"
#include "GameFramework/Actor.h"
#include "MyDamageable.h"
#include "MyTarget.generated.h"

UCLASS()
class MYGAME_API AMyTarget : public AActor, public IMyDamageable
{
    GENERATED_BODY()
public:
    virtual void ApplyMyDamage(float Amount, AActor* DamageInstigator) override;
};
```

```cpp
// Calling a native interface
void DamageIfPossible(AActor* Target, float Amount, AActor* Source)
{
    if (IMyDamageable* Damageable = Cast<IMyDamageable>(Target))
    {
        Damageable->ApplyMyDamage(Amount, Source);
    }
    bool bImplements = IsValid(Target) && Target->Implements<UMyDamageable>();   // type check without casting
}
```

Interfaces that Blueprint may implement use `BlueprintNativeEvent`/`BlueprintImplementableEvent` functions and are invoked through the generated `Execute_FunctionName` statics; that pattern, plus `TScriptInterface<T>` properties, is covered in `ue-blueprint-cpp-interop`.
