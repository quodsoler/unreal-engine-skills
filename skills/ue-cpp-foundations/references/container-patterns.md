# Container Patterns Reference

Detailed patterns, performance notes and advanced usage for `TArray`, `TMap`, `TSet`, `TOptional`, `TVariant` and related containers in UE 5.8. Headers live under `Engine/Source/Runtime/Core/Public/Containers/` (module `Core`); the `TMap`/`TSet` bodies are in `Map.h.inl` and `SparseSet.h.inl`.

---

## TArray

`TArray<T, AllocatorType>` (`Containers/Array.h`) stores elements contiguously and is the default ordered container.

### Core Operations

```cpp
TArray<int32> Numbers;

// Construction
TArray<int32> FromInit = { 1, 2, 3, 4, 5 };
TArray<int32> Copy     = FromInit;
TArray<int32> Moved    = MoveTemp(FromInit);  // FromInit is empty afterwards

// Capacity management
Numbers.Reserve(64);             // pre-allocate without changing Num()
Numbers.SetNum(10);              // resize, default-constructs new elements
Numbers.SetNumZeroed(10);        // resize, zero-fills new elements
Numbers.SetNumUninitialized(10); // resize without construction (trivial types only)
Numbers.Shrink();                // release slack
Numbers.Empty();                 // clear and free the allocation
Numbers.Reset();                 // clear, keep the allocation
Numbers.Empty(64);               // clear and reserve 64 slots

// Counts
int32 Count  = Numbers.Num();
bool  bEmpty = Numbers.IsEmpty();
int32 MaxCap = Numbers.Max();    // current capacity
```

### Adding Elements

```cpp
TArray<FString> Names;

Names.Add(TEXT("Alpha"));
Names.Add(FString(TEXT("Beta")));       // moves from the rvalue
Names.Emplace(TEXT("Gamma"));           // constructs in place
FString& Ref = Names.Emplace_GetRef(TEXT("Delta"));   // in place and returns the new element
Names.AddUnique(TEXT("Alpha"));         // O(n) search first; prefer TSet when uniqueness is the goal
Names.Insert(TEXT("First"), 0);         // shifts everything after index 0

TArray<FString> Extra = { TEXT("X"), TEXT("Y") };
Names.Append(Extra);
Names.Append({ TEXT("P"), TEXT("Q") });

Names.Push(TEXT("Top"));                // stack semantics
FString Top = Names.Pop();              // removes and returns the last element
const FString& Peek = Names.Last();     // last element without removing
```

### Removal

```cpp
TArray<int32> V = { 1, 2, 3, 2, 4, 2 };

int32 Removed = V.Remove(2);    // all occurrences, order-preserving, O(n); Removed == 3
V.RemoveSwap(3);                // all occurrences, swaps with last, reorders
V.RemoveAt(0);                  // by index, order-preserving
V.RemoveAtSwap(0);              // by index, O(1), reorders
V.RemoveSingle(4);              // first occurrence only
V.RemoveAll([](int32 N) { return N % 2 == 0; });      // predicate, order-preserving
V.RemoveAllSwap([](int32 N) { return N < 0; });       // predicate, reorders

if (V.Num() > 0)
{
    int32 Last = V.Pop();
}
```

### Search and Query

```cpp
TArray<FString> Names = { TEXT("Alpha"), TEXT("Beta"), TEXT("Gamma") };

int32 Idx  = Names.Find(TEXT("Beta"));       // 1, INDEX_NONE if absent
int32 Rev  = Names.FindLast(TEXT("Beta"));   // searches from the end
bool  bHas = Names.Contains(TEXT("Delta"));  // false

FString* Match = Names.FindByPredicate([](const FString& S) { return S.StartsWith(TEXT("G")); });  // pointer or nullptr
int32 GIdx     = Names.IndexOfByPredicate([](const FString& S) { return S.Contains(TEXT("amm")); });
bool  bAnyLong = Names.ContainsByPredicate([](const FString& S) { return S.Len() > 4; });
TArray<FString> Long = Names.FilterByPredicate([](const FString& S) { return S.Len() > 4; });   // new array
bool bValid = Names.IsValidIndex(5);         // false
```

### Sorting

```cpp
TArray<int32> Numbers = { 5, 3, 1, 4, 2 };

Numbers.Sort();                                            // operator<
Numbers.Sort([](int32 A, int32 B) { return A > B; });      // descending
Numbers.StableSort([](int32 A, int32 B) { return A < B; }); // keeps relative order of equal elements
Numbers.HeapSort();                                        // heap sort, not stable

struct FMyWeaponStats { float BaseDamage = 0.f; };
TArray<FMyWeaponStats*> Weapons;
Weapons.Sort([](const FMyWeaponStats& A, const FMyWeaponStats& B) { return A.BaseDamage > B.BaseDamage; });  // pointer arrays: the predicate receives dereferenced elements
```

### Iteration Patterns

```cpp
TArray<AActor*> Actors;

// Ranged-for: never add or remove inside the loop
for (AActor* Actor : Actors)
{
    if (IsValid(Actor)) { Actor->Destroy(); }
}

// Reverse index loop: safe to remove
for (int32 i = Actors.Num() - 1; i >= 0; --i)
{
    if (!IsValid(Actors[i])) { Actors.RemoveAtSwap(i); }
}

// Iterator with inline removal
for (auto It = Actors.CreateIterator(); It; ++It)
{
    if (!IsValid(*It)) { It.RemoveCurrent(); }
}

// Const iterator
for (auto It = Actors.CreateConstIterator(); It; ++It)
{
    UE_LOG(LogMyGame, Log, TEXT("%s"), *(*It)->GetName());
}
```

### Memory and Performance Notes

- `Add`/`Emplace` are amortized O(1); a reallocation is O(n).
- `Remove`/`RemoveAt` shift elements (O(n)); `RemoveSwap`/`RemoveAtSwap` are O(1) when order does not matter.
- `Reserve` before bulk fills to avoid reallocation churn.
- Prefer `Emplace` over `Add` for non-trivial element types.
- `SetNumUninitialized` is the fastest resize for trivially-constructible bulk data.
- `Contains`/`Find` are O(n); switch to `TSet`/`TMap` for hot-path membership tests.

### Inline and Fixed Allocators

```cpp
#include "Containers/ContainerAllocationPolicies.h"

TArray<int32, TInlineAllocator<8>> SmallList;   // first 8 elements live inline; heap only beyond that
SmallList.Add(1);

TArray<int32, TFixedAllocator<16>> FixedList;   // never allocates; asserts past 16 elements
```

---

## TMap

`TMap<K, V>` (`Containers/Map.h`) is a hash map on a sparse array: average O(1) lookup, insertion and removal. Keys need `GetTypeHash(Key)` and `operator==`. `TMultiMap<K, V>` allows duplicate keys (`MultiFind`); `TSortedMap<K, V>` (`Containers/SortedMap.h`) is a sorted array-backed map for small key sets.

### Core Operations

```cpp
TMap<FName, float> WeaponDamage;

WeaponDamage.Add(FName("Rifle"), 35.f);        // overwrites an existing key
WeaponDamage.Emplace(FName("Pistol"), 20.f);   // constructs the value in place

float& RifleRef  = WeaponDamage.FindOrAdd(FName("Rifle"));   // existing value
float& SniperRef = WeaponDamage.FindOrAdd(FName("Sniper"));  // inserts 0.f and returns it
SniperRef = 120.f;

float* DamagePtr = WeaponDamage.Find(FName("Rifle"));        // nullptr if absent
if (DamagePtr) { *DamagePtr *= 1.5f; }

float GrenadeDmg = WeaponDamage.FindRef(FName("Grenade"));   // copy or default-constructed value; cannot tell "absent" from "zero"
float& Checked   = WeaponDamage.FindChecked(FName("Rifle")); // asserts if absent
const FName* KeyOf = WeaponDamage.FindKey(120.f);            // reverse lookup, O(n)

bool  bHasRifle  = WeaponDamage.Contains(FName("Rifle"));
int32 NumRemoved = WeaponDamage.Remove(FName("Pistol"));
int32 Count      = WeaponDamage.Num();
bool  bEmpty     = WeaponDamage.IsEmpty();
```

### Iteration

```cpp
TMap<FName, int32> ItemCounts;

for (const TPair<FName, int32>& Pair : ItemCounts)
{
    UE_LOG(LogMyGame, Log, TEXT("%s = %d"), *Pair.Key.ToString(), Pair.Value);
}

for (auto& [Key, Value] : ItemCounts)   // structured bindings work on TPair
{
    Value += 10;
}

TArray<FName> Keys;
ItemCounts.GetKeys(Keys);
TArray<FName> OutKeys;
TArray<int32> OutValues;
ItemCounts.GenerateKeyArray(OutKeys);
ItemCounts.GenerateValueArray(OutValues);

for (auto It = ItemCounts.CreateIterator(); It; ++It)   // removal while iterating
{
    if (It.Value() <= 0) { It.RemoveCurrent(); }
}

ItemCounts.KeySort([](FName A, FName B) { return A.LexicalLess(B); });          // reorders the map's internal storage
ItemCounts.ValueSort([](int32 A, int32 B) { return A > B; });
```

### Memory and Performance Notes

- Iteration skips holes left by removals; call `Compact()` or `Shrink()` after heavy removal churn if memory matters.
- `Reserve(N)` before bulk adds; `Empty(Slack)` clears and reserves.
- Read-heavy maps that rarely change: build once, then only query.

### Custom Key Hashing

```cpp
#include "Templates/TypeHash.h"

struct FMyItemKey
{
    FName Category;
    int32 Tier = 0;

    bool operator==(const FMyItemKey& Other) const
    {
        return Category == Other.Category && Tier == Other.Tier;
    }
};

inline uint32 GetTypeHash(const FMyItemKey& Key)
{
    return HashCombineFast(GetTypeHash(Key.Category), GetTypeHash(Key.Tier));
}

TMap<FMyItemKey, float> ItemValues;
```

---

## TSet

`TSet<T>` (`Containers/Set.h`) is a hash set with unique elements and average O(1) operations, built on the same sparse storage as `TMap`.

### Core Operations

```cpp
TSet<FName> Tags;

Tags.Add(FName("Flying"));
Tags.Add(FName("Aquatic"));
Tags.Add(FName("Flying"));                    // no-op, already present
bool  bFlying = Tags.Contains(FName("Flying"));
Tags.Remove(FName("Aquatic"));
int32 Count   = Tags.Num();
Tags.Reserve(32);
TArray<FName> AsArray = Tags.Array();         // copy out
```

### Set Operations

```cpp
TSet<FName> A = { FName("Fire"), FName("Ice"), FName("Wind") };
TSet<FName> B = { FName("Ice"), FName("Wind"), FName("Earth") };

TSet<FName> Intersection = A.Intersect(B);   // Ice, Wind
TSet<FName> Union        = A.Union(B);       // Fire, Ice, Wind, Earth
TSet<FName> Difference   = A.Difference(B);  // Fire
bool bSubset = B.Includes(A);                // false: A has Fire
```

### Iteration

```cpp
TSet<FName> Tags;

for (const FName& Tag : Tags)
{
    UE_LOG(LogMyGame, Log, TEXT("Tag: %s"), *Tag.ToString());
}

for (auto It = Tags.CreateIterator(); It; ++It)
{
    if (It->IsNone()) { It.RemoveCurrent(); }
}
```

---

## TOptional

`TOptional<T>` (`Misc/Optional.h`) holds a value or nothing; it replaces sentinels such as `-1` or `nullptr`.

```cpp
TOptional<int32> MaybeLevel;

if (MaybeLevel.IsSet())
{
    int32 Level = MaybeLevel.GetValue();   // asserts if unset
}
int32 LevelOrOne = MaybeLevel.Get(1);      // default when unset
int32* LevelPtr  = MaybeLevel.GetPtrOrNull();

MaybeLevel = 5;
MaybeLevel.Emplace(7);                     // construct in place
MaybeLevel.Reset();                        // back to unset
MaybeLevel = NullOpt;                      // also unset

TOptional<FVector> FindSpawnPoint(const FString& ZoneName)
{
    if (ZoneName.IsEmpty()) { return NullOpt; }
    return FVector(100.f, 200.f, 0.f);
}

if (TOptional<FVector> Pt = FindSpawnPoint(TEXT("Start")))   // explicit operator bool == IsSet()
{
    FVector Location = Pt.GetValue();
}
```

---

## TVariant

`TVariant<T1, T2, ...>` (`Misc/TVariant.h`) is a type-safe discriminated union. Types must be unique, must not be references, and the first type must be default-constructible (use `FEmptyVariantState` as the first type when none is).

```cpp
#include "Misc/TVariant.h"
#include <type_traits>

TVariant<int32, float, FString> Val;                      // holds a default int32
TVariant<FEmptyVariantState, FVector> Optional;           // starts empty

Val.Set<int32>(42);
Val.Set<FString>(TEXT("Hello"));
Val.Emplace<FString>(TEXT("In place"));
TVariant<int32, float, FString> Built(TInPlaceType<float>(), 1.5f);   // construct holding a float

if (Val.IsType<FString>())
{
    FString& S = Val.Get<FString>();                      // asserts on wrong type
}
if (const FString* Str = Val.TryGet<FString>())            // nullptr on wrong type
{
    UE_LOG(LogMyGame, Log, TEXT("%s"), **Str);
}
float AsFloat = Val.Get<float>(0.f);                      // held float or the default
SIZE_T Index  = Val.GetIndex();                           // index into the type list
constexpr SIZE_T StringIndex = TVariant<int32, float, FString>::IndexOfType<FString>();

// Visit: one generic lambda; select behaviour per held type with if constexpr
Visit([](auto& Held)
{
    using HeldType = std::decay_t<decltype(Held)>;
    if constexpr (std::is_same_v<HeldType, int32>)        { UE_LOG(LogMyGame, Log, TEXT("int %d"), Held); }
    else if constexpr (std::is_same_v<HeldType, float>)   { UE_LOG(LogMyGame, Log, TEXT("float %f"), Held); }
    else if constexpr (std::is_same_v<HeldType, FString>) { UE_LOG(LogMyGame, Log, TEXT("string %s"), *Held); }
}, Val);
```

`Visit(Callable, Variants...)` accepts several variants at once and calls `Callable` with all held values. There is no engine-provided overload-set helper for `Visit`; branch inside a single generic lambda as above.

---

## Sparse and Indirect Arrays

### TSparseArray

`TSparseArray<T>` (`Containers/SparseArray.h`) keeps stable indices with holes where elements were removed. It is the storage behind `TSet`/`TMap`; prefer those directly.

```cpp
struct FMySlot { int32 Id = 0; };
TSparseArray<FMySlot> Slots;
FSparseArrayAllocationInfo AllocInfo = Slots.AddUninitialized();
new (AllocInfo.Pointer) FMySlot();
int32 SlotIndex = AllocInfo.Index;
Slots.RemoveAt(SlotIndex);
```

### TIndirectArray

`TIndirectArray<T>` (`Containers/IndirectArray.h`) owns heap-allocated elements and deletes them on removal and destruction; element addresses stay stable across reallocation.

```cpp
struct FMyNonCopyable { FMyNonCopyable() = default; FMyNonCopyable(const FMyNonCopyable&) = delete; };
TIndirectArray<FMyNonCopyable> Objects;
Objects.Add(new FMyNonCopyable());
```

---

## Containers as UPROPERTY Members

- `TArray<T>`, `TMap<K, V>` and `TSet<T>` of reflected types are supported; object pointers inside them must be `TObjectPtr<T>` to be GC-tracked (keys included).
- Nested containers (`TArray<TArray<T>>`, `TMap<K, TArray<V>>`) are not supported by UnrealHeaderTool; wrap the inner container in a `USTRUCT`.
- A container without `UPROPERTY` is invisible to the GC regardless of element type.

```cpp
// MyContainerHolder.h
#pragma once
#include "CoreMinimal.h"
#include "UObject/Object.h"
#include "Engine/DataAsset.h"
#include "MyContainerHolder.generated.h"

USTRUCT(BlueprintType)
struct MYGAME_API FMyIntRow
{
    GENERATED_BODY()

    UPROPERTY(EditAnywhere, Category="Grid")
    TArray<int32> Values;
};

UCLASS()
class MYGAME_API UMyContainerHolder : public UObject
{
    GENERATED_BODY()
public:
    UPROPERTY()
    TArray<TObjectPtr<UObject>> ManagedObjects;              // GC-tracked

    UPROPERTY()
    TMap<FName, TObjectPtr<UDataAsset>> AssetsByName;        // GC-tracked values

    UPROPERTY(EditAnywhere, Category="Grid")
    TArray<FMyIntRow> Matrix;                                // wrapper struct instead of TArray<TArray<int32>>
};
```

---

## Performance Comparison

| Operation | TArray | TMap | TSet |
|-----------|--------|------|------|
| Add | O(1) amortized | O(1) average | O(1) average |
| Find / Contains | O(n) | O(1) average | O(1) average |
| Remove by value | O(n) | O(1) average | O(1) average |
| Remove by index | O(n) shift, O(1) with swap | — | — |
| Iteration | O(n), cache-friendly | O(n), sparse | O(n), sparse |
| Sorted iteration | `Sort` first, O(n log n) | `KeySort`/`ValueSort` reorder storage | `Sort` |
| Memory | Compact, contiguous | Sparse array + hash | Sparse array + hash |

**Guidelines:**
- `TArray` for ordered data, index access, frequent iteration, small N.
- `TMap` for key lookups where O(1) matters and order is irrelevant.
- `TSet` for uniqueness and membership tests without an associated value.
- `TArray` + `Sort` + binary search (`Algo/BinarySearch.h`) for read-heavy sorted lookups.
