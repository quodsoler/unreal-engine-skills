# Blueprint-Facing Specifiers and `meta=` Keys

Every key below is declared in `Engine/Source/Runtime/CoreUObject/Public/UObject/ObjectMacros.h` in UE 5.8, in the `UC` (UCLASS), `UI` (UINTERFACE), `UF` (UFUNCTION/UDELEGATE), `UP` (UPROPERTY), `US` (USTRUCT) or `UM` (metadata) enumerations; the line number after each name is its declaration. Parsing is case-insensitive, so engine headers write both `Config=Game` and `config=Game`. A key that is not in one of those enumerations is not a specifier — UHT ignores it silently or errors.

`meta=` values are strings: `meta=(ClampMin="0.0")`, `meta=(BlueprintInternalUseOnly="true")`. Boolean-style keys may be written bare (`meta=(Latent)`, `meta=(EditConditionHides)`).

---

## UFUNCTION Specifiers

| Specifier | Line | Effect |
|---|---|---|
| `BlueprintCallable` | 1029 | Callable from Blueprint; node has exec pins |
| `BlueprintPure` | 1026 | No exec pins by default; implies `BlueprintCallable` |
| `BlueprintImplementableEvent` | 992 | Blueprint provides the body; C++ must not define the function |
| `BlueprintNativeEvent` | 997 | Blueprint may override; C++ provides `[FunctionName]_Implementation` |
| `BlueprintGetter` | 1032 | Accessor for a Blueprint-exposed property; implies `BlueprintPure` + `BlueprintCallable` |
| `BlueprintSetter` | 1035 | Mutator for a Blueprint-exposed property; implies `BlueprintCallable` |
| `BlueprintAuthorityOnly` | 1038 | Does not execute from Blueprint without network authority |
| `BlueprintCosmetic` | 1041 | Does not run on dedicated servers |
| `BlueprintInternalUseOnly` | 1044 | Hidden from the palette; only generated nodes call it |
| `CallInEditor` | 1047 | Button in the Details panel for selected instances |
| `SealedEvent` | 1001 | Event cannot be overridden in subclasses |
| `Exec` | 1004 | Console command |
| `CustomThunk` | 1050 | UHT emits no `execFoo` thunk; you write it |
| `Category` | 1054 | Palette category; `Category="Major,Sub"` or `Category="A\|B"` |
| `Variadic` | 1070 | Extra wildcard terms after the declared parameters |
| `ReturnDisplayName` | 1073 | Display name of the return value pin |
| `FieldNotify` | 1057 | Emits a NotifyFieldValueChanged entry (MVVM) |

RPC specifiers (`Server`, `Client`, `NetMulticast`, `Reliable`, `Unreliable`, `WithValidation`) live in the same enumeration but belong to `ue-networking-replication`.

## UFUNCTION `meta=` Keys

| Key | Line | Effect |
|---|---|---|
| `DisplayName` | 1282 | Node title instead of the auto-generated one |
| `ToolTip` | 1248 | Overrides the doc-comment tooltip |
| `Keywords` | 1761 | Extra palette search terms |
| `CompactNodeTitle` | 1691 | Draws the node in compact mode with this title |
| `AdvancedDisplay` | 1661 | Comma-separated pin names to hide behind the expander, or a count of leading pins to keep visible (`AdvancedDisplay="2"`) |
| `HidePin` | 1755 | Hides a named parameter pin entirely |
| `WorldContext` | 1782 | Names the parameter used to resolve the `UWorld` |
| `CallableWithoutWorldContext` | 1685 | Allows the call when the calling class has no `GetWorld()` |
| `DefaultToSelf` | 1697 | Named object parameter defaults to the node's self context |
| `AutoCreateRefTerm` | 1671 | Creates a value for an unconnected reference input; inputs only |
| `ExpandEnumAsExecs` | 1707 | One exec pin per enumerator; `ReturnValue` expands the return value |
| `ExpandBoolAsExecs` | 1710 | Synonym of `ExpandEnumAsExecs` for bools |
| `DeterminesOutputType` | 1791 | Names the class/object parameter that types the return pin |
| `DynamicOutputParam` | 1794 | Names the output parameter typed by `DeterminesOutputType` |
| `ArrayParm` | 1664 | Comma-separated array parameters treated as wildcards (Call Array Function node) |
| `ArrayTypeDependentParams` | 1667 | Parameters whose type follows the `ArrayParm` element type |
| `BlueprintInternalUseOnly` | 1679 | Implementation detail; never shown in a graph |
| `BlueprintProtected` | 1682 | Callable only from the class and its subclasses |
| `Latent` | 1764 | Marks the function latent |
| `LatentInfo` | 1767 | Names the `FLatentActionInfo` parameter |
| `DevelopmentOnly` | 1566 | Compiled out of Shipping |
| `UnsafeDuringActorConstruction` | 1779 | Disallowed in a Construction Script |
| `NotBlueprintThreadSafe` | 1788 | Exception to a library's class-level `BlueprintThreadSafe` |
| `DeprecatedFunction` | 1700 | Blueprint references raise a compile warning |
| `DeprecationMessage` | 1279 | Text shown with `DeprecatedFunction` |
| `HidePinAssetPicker` | 1676 | Hides the asset picker on the named pins |
| `HideSpawnParms` | 1758 | Parameters to skip when an async node exposes pins |
| `BlueprintAutocast` | 1785 | Static `BlueprintPure` library function used as an implicit cast |
| `NativeMakeFunc` / `NativeBreakFunc` | 1776 / 1773 | Display as the implicit Make/Break Struct node |
| `CustomStructureParam` | 1694 | With `CustomThunk`, marks a polymorphic parameter |
| `MapKeyParam` / `MapValueParam` / `SetParam` | 1806 / 1809 / 1800 | Wildcard container parameters resolved at Blueprint compile time |

`BlueprintThreadSafe` (1313) is **class** metadata for a `UBlueprintFunctionLibrary`: it marks every function in the library callable off the game thread in an Animation Blueprint. Use `NotBlueprintThreadSafe` on individual exceptions.

## UPROPERTY Specifiers That Matter to Blueprint

| Specifier | Line | Effect |
|---|---|---|
| `BlueprintReadWrite` | 1182 | Get and Set nodes |
| `BlueprintReadOnly` | 1176 | Get node only |
| `BlueprintGetter=Fn` | 1179 | Reads route through the named `UFUNCTION`; implies read-only unless a setter is given |
| `BlueprintSetter=Fn` | 1185 | Writes route through the named `UFUNCTION`; implies `BlueprintReadWrite` |
| `BlueprintAssignable` | 1146 | Multicast dynamic delegates only: Blueprint can bind events |
| `BlueprintCallable` | 1197 | Multicast dynamic delegates only: Blueprint can broadcast |
| `BlueprintAuthorityOnly` | 1200 | Delegate accepts only `BlueprintAuthorityOnly` events |
| `Instanced` | 1143 | Sub-object reference edited inline; implies EditInline and Export |
| `Category` | 1149 | Details-panel and variable category |
| `AdvancedDisplay` | 1155 | Behind the category's Advanced expander |
| `EditAnywhere` / `EditDefaultsOnly` / `EditInstanceOnly` | 1158 / 1164 / 1161 | Editability |
| `VisibleAnywhere` / `VisibleDefaultsOnly` / `VisibleInstanceOnly` | 1167 / 1173 / 1170 | Read-only visibility |
| `SaveGame` | 1194 | Serialized into save archives |
| `HideSelfPin` | 1209 | Suppresses the self pin on the accessor nodes |
| `FieldNotify` | 1212 | Emits a NotifyFieldValueChanged entry (MVVM) |

UHT errors, not warnings:

| Message | Cause |
|---|---|
| `BlueprintReadWrite should not be used on private members` | `private:` member without `meta=(AllowPrivateAccess="true")` |
| `BlueprintReadOnly should not be used on private members` | same, for the read-only specifier |
| `Cannot specify a property as being both BlueprintReadOnly and BlueprintReadWrite.` | both specifiers on one property |
| `Blueprint exposed struct members cannot be editor only` | `WITH_EDITORONLY_DATA` member of a `BlueprintType` struct |

## UPROPERTY `meta=` Keys

| Key | Line | Effect |
|---|---|---|
| `AllowPrivateAccess` | 1351 | Lets a private member carry `BlueprintReadOnly`/`BlueprintReadWrite` |
| `ExposeOnSpawn` | 1434 | Adds a pin to Spawn Actor from Class / Create Widget |
| `ClampMin` / `ClampMax` | 1366 / 1369 | Hard limits on typed values |
| `UIMin` / `UIMax` | 1549 / 1552 | Slider range only |
| `EditCondition` | 1419 | Boolean expression controlling editability |
| `EditConditionHides` | 1422 | Hides the row instead of greying it out |
| `InlineEditConditionToggle` | 1467 | The bool renders as the other property's inline checkbox |
| `TitleProperty` | 1546 | Member of a struct used as the collapsed array-entry title |
| `ShowOnlyInnerProperties` | 1537 | Promotes a struct's members up a level |
| `MultiLine` | 1504 | Multi-line text box for `FString`/`FText` |
| `MaxLength` | 1501 | Maximum editable length for `FString`/`FText` |
| `GetOptions` | 1579 | `UFUNCTION` returning `TArray<FName>`/`TArray<FString>` that fills a dropdown |
| `AllowedClasses` | 1345 | Comma-delimited class filter for asset, component and class pickers |
| `MustImplement` | 1492 | Class picker shows only classes implementing this interface |
| `BlueprintBaseOnly` | 1360 | Class picker shows only Blueprint-able bases |
| `Bitmask` | 1812 | Integer property edited as flags |
| `BitmaskEnum` | 1815 | Associates the bitmask with a flag enum |
| `Units` / `ForceUnits` | 1557 / 1560 | Unit display and conversion |
| `AssetBundles` | 1357 | Bundle names for soft references inside a primary data asset |
| `DisplayName` / `ToolTip` | 1282 / 1248 | Row label and tooltip |
| `HidePinAssetPicker` | 1676 | Hides the asset picker on the generated pin |

`Categories` for gameplay-tag filtering is declared in the same header inside `namespace GameplayTagsManager` (line 2470), not in `UM`.

## UPARAM

`UPARAM(...)` (ObjectMacros.h:783) annotates one parameter:

| Form | Effect |
|---|---|
| `UPARAM(ref) FVector& InOut` | Adds `OutParm | ReferenceParm`: the pin stays an input and is written back |
| `UPARAM(Const) FVector& In` | Adds `ConstParm` |
| `UPARAM(DisplayName="Event") FMyDelegate D` | Renames the pin |
| `UPARAM(NotReplicated)` | Skips the parameter in a replicated function |

Without `UPARAM(ref)`, a bare `T&` parameter is given `OutParm` only and becomes an **output** pin. A `const T&` parameter stays an input.

## UCLASS Specifiers That Matter to Blueprint

| Specifier | Line | Effect |
|---|---|---|
| `BlueprintType` | 844 | Usable as a Blueprint variable type |
| `NotBlueprintType` | 847 | Explicitly not a variable type |
| `Blueprintable` | 850 | Blueprint subclasses allowed |
| `NotBlueprintable` | 853 | Blueprint subclasses blocked |
| `Abstract` | 886 | Not directly instantiable |
| `config` / `defaultconfig` | 902 / 909 | `UPROPERTY(Config)` members load from / save to the Default INI |
| `MinimalAPI` | 857 | Export only what `Cast<>` needs |
| `Within=Outer` | 841 | Instances must be created inside that outer class |

Useful class `meta=` keys: `DisplayName` (1282), `BlueprintSpawnableComponent` (1261, puts a component in the Add Component menu), `ExposedAsyncProxy` (1310, the async node returns the proxy object), `IsBlueprintBase` (1288), `DontUseGenericSpawnObject` (1307), `BlueprintThreadSafe` (1313), `RestrictedToClasses` (1300, limits which graphs a function library appears in).

## UINTERFACE Specifiers

The `UI::` enum documents these four (ObjectMacros.h:965-984). UHT also accepts UCLASS specifiers such as `BlueprintType` on a `UINTERFACE`; the engine uses `UINTERFACE(BlueprintType, MinimalAPI)` (`ActorSoundParameterInterface.h:28`):

| Specifier | Line | Effect |
|---|---|---|
| `MinimalAPI` | 972 | Exports only the autogenerated `Cast<>` support |
| `Blueprintable` | 975 | Blueprint classes can implement it (implied when it has Blueprint events) |
| `NotBlueprintable` | 978 | Equivalent to `meta=(CannotImplementInterfaceInBlueprint)` |
| `ConversionRoot` | 981 | Sets the `IsConversionRoot` metadata flag |

Interface `meta=` keys: `CannotImplementInterfaceInBlueprint` (1834) and `CannotGenerateMessageNodes` (1847, suppresses the type-agnostic Message nodes). Engine interfaces also carry `BlueprintType` so the interface is usable as a variable type — `UActorSoundParameterInterface` is declared `UINTERFACE(BlueprintType, MinimalAPI)`.

## USTRUCT and UENUM

| Specifier | Line | Effect |
|---|---|---|
| `USTRUCT(BlueprintType)` | 1231 | Struct usable as a Blueprint variable; gets Make/Break nodes |
| `USTRUCT(BlueprintInternalUseOnly)` | 1234 | Blueprint type hidden from the user (`FTableRowBase` uses this) |
| `USTRUCT(BlueprintInternalUseOnlyHierarchical)` | 1237 | Same, applied to derived structs too |
| `USTRUCT(Atomic)` | 1225 | Always serialized as a unit |

`UENUM(BlueprintType)` requires `enum class E… : uint8`; any other underlying type is a UHT error ("Invalid BlueprintType enum base - currently only uint8 supported"). Per-enumerator `UMETA(...)` (ObjectMacros.h:782) takes `DisplayName="…"` and `Hidden` — `ESpawnActorScaleMethod::SelectDefaultAtRuntime` in `GameFramework/Actor.h:93` is an engine example of `UMETA(Hidden)`.

Containers: `TArray`, `TSet` and `TMap` are Blueprint-exposable only when their element type — and, for `TMap`, their key type — is itself Blueprint-supported, and they cannot nest, because each clears its own `CanBeContainerValue`/`CanBeContainerKey` capability in UnrealHeaderTool.

---

## A Worked Property Example

```cpp
// MyStatsComponent.h
#pragma once
#include "CoreMinimal.h"
#include "Components/ActorComponent.h"
#include "StructUtils/InstancedStruct.h"
#include "Templates/SubclassOf.h"
#include "MyStatsComponent.generated.h"

class AActor;
class UStaticMesh;

DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FMyHealthChangedSignature, double, NewHealth);

UCLASS(ClassGroup=(Custom), meta=(BlueprintSpawnableComponent))
class MYGAME_API UMyStatsComponent : public UActorComponent
{
    GENERATED_BODY()
public:
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category="Stats", meta=(ClampMin="0.0", UIMax="500.0"))
    double MaxHealth = 100.0;                       // Real pin, slider capped at 500

    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category="Stats", meta=(ExposeOnSpawn="true"))
    FName StatsRowName;                             // extra pin on Spawn Actor from Class

    UPROPERTY(EditAnywhere, Category="Stats", meta=(InlineEditConditionToggle))
    bool bOverrideRegen = false;                    // renders as the checkbox on the row below

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category="Stats", meta=(EditCondition="bOverrideRegen", EditConditionHides))
    double RegenPerSecond = 1.0;                    // row disappears unless bOverrideRegen is set

    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category="Stats")
    TObjectPtr<UStaticMesh> Mesh;                   // object pin

    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category="Stats", meta=(MustImplement="/Script/MyGame.MyInteractable"))
    TSubclassOf<AActor> SpawnClass;                 // class pin, picker filtered by the interface

    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category="Stats")
    TSoftObjectPtr<UStaticMesh> LazyMesh;           // soft object pin

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category="Stats")
    FInstancedStruct Effect;                        // struct pin, concrete type chosen in the editor

    UPROPERTY(BlueprintAssignable, BlueprintCallable, Category="Stats")
    FMyHealthChangedSignature OnHealthChanged;      // Bind Event and Broadcast nodes

    UPROPERTY(BlueprintGetter=GetShield, BlueprintSetter=SetShield, Category="Stats")
    double Shield = 0.0;

    UFUNCTION(BlueprintGetter)
    double GetShield() const { return Shield; }

    UFUNCTION(BlueprintSetter)
    void SetShield(double NewShield);

    UFUNCTION(BlueprintCallable, Category="Stats")
    void ApplyDamage(double Amount);

private:
    UPROPERTY(EditAnywhere, Category="Stats", meta=(AllowPrivateAccess="true"))
    double CurrentHealth = 100.0;
};
```

```cpp
// MyStatsComponent.cpp
#include "MyStatsComponent.h"

void UMyStatsComponent::SetShield(double NewShield)
{
    Shield = FMath::Clamp(NewShield, 0.0, MaxHealth);
}

void UMyStatsComponent::ApplyDamage(double Amount)
{
    CurrentHealth = FMath::Max(0.0, CurrentHealth - Amount);
    OnHealthChanged.Broadcast(CurrentHealth);
}
```

A `BlueprintSetter` is the only way to run validation on a Blueprint write; a plain `BlueprintReadWrite` property is written directly by the VM with no hook.
