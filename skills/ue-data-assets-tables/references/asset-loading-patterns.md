# Asset Loading Patterns

Async loading, `FStreamableManager`, load grouping and handle lifecycle. Target engine: **UE 5.8**, verified against `Engine/Classes/Engine/StreamableManager.h` and `Engine/Classes/Engine/AssetManager.h`.

---

## FStreamableManager Access

`FStreamableManager` is a member of `UAssetManager`. Do not create your own instances in game code — share the one the Asset Manager owns.

```cpp
FStreamableManager& SM = UAssetManager::GetStreamableManager();   // Engine/AssetManager.h:105
```

There is no `AssetManager` Build.cs module: `UAssetManager` and `FStreamableManager` both live in `Engine`.

```csharp
PublicDependencyModuleNames.AddRange(new string[] { "Core", "CoreUObject", "Engine" });
```

Priority constants are public members of `FStreamableManager`:

```cpp
FStreamableManager::DefaultAsyncLoadPriority   // 0
FStreamableManager::AsyncLoadHighPriority      // 100
```

---

## Pattern 1: Single Asset, Callback on Completion

```cpp
// MyCharacter.h — members
TSharedPtr<FStreamableHandle> LoadHandle;

UPROPERTY(EditDefaultsOnly, Category = "Art")
TSoftObjectPtr<UStaticMesh> WeaponMeshSoft;
```

```cpp
void AMyCharacter::BeginLoadWeaponMesh()
{
    FStreamableManager& SM = UAssetManager::GetStreamableManager();

    LoadHandle = SM.RequestAsyncLoad(
        WeaponMeshSoft.ToSoftObjectPath(),
        FStreamableDelegate::CreateUObject(this, &AMyCharacter::OnWeaponMeshReady));
}

void AMyCharacter::OnWeaponMeshReady()
{
    if (UStaticMesh* Mesh = WeaponMeshSoft.Get())
    {
        WeaponMeshComponent->SetStaticMesh(Mesh);
    }
    // Safe to release now: the component holds a hard reference.
    LoadHandle.Reset();
}
```

`RequestAsyncLoad` accepts a single `FSoftObjectPath`, a `TArray<FSoftObjectPath>`, an `FStreamableDelegate`, an `FStreamableDelegateWithHandle` or a bare callable.

---

## Pattern 2: Multiple Assets, Single Callback

```cpp
void UMyLoadSystem::LoadUIAssets(const TArray<TSoftObjectPtr<UTexture2D>>& IconRefs)
{
    TArray<FSoftObjectPath> Paths;
    Paths.Reserve(IconRefs.Num());
    for (const TSoftObjectPtr<UTexture2D>& Ref : IconRefs)
    {
        if (!Ref.IsNull())
        {
            Paths.Add(Ref.ToSoftObjectPath());
        }
    }

    if (Paths.IsEmpty())
    {
        return;
    }

    FStreamableManager& SM = UAssetManager::GetStreamableManager();
    LoadUIHandle = SM.RequestAsyncLoad(
        Paths,
        FStreamableDelegate::CreateUObject(this, &UMyLoadSystem::OnUIAssetsLoaded),
        FStreamableManager::AsyncLoadHighPriority,
        /*bManageActiveHandle=*/false,
        /*bStartStalled=*/false,
        TEXT("Inventory icons"));
}

void UMyLoadSystem::OnUIAssetsLoaded()
{
    TArray<UTexture2D*> LoadedIcons;
    LoadUIHandle->GetLoadedAssets(LoadedIcons);   // entries are null where the cast failed

    for (UTexture2D* Icon : LoadedIcons)
    {
        if (Icon)
        {
            CacheIcon(Icon);
        }
    }
}
```

---

## Pattern 3: Params Struct

`FStreamableAsyncLoadParams` replaces the long argument list and is the only way to bind cancel/update delegates at request time, or to opt into the just-in-time async loader.

```cpp
FStreamableAsyncLoadParams Params;
Params.TargetsToStream = MoveTemp(Paths);
Params.Priority = FStreamableManager::AsyncLoadHighPriority;
Params.bManageActiveHandle = false;
Params.bStartStalled = false;
Params.bUseJustInTimeAsyncLoader = true;
Params.OnComplete = FStreamableDelegateWithHandle::CreateWeakLambda(this,
    [this](TSharedPtr<FStreamableHandle> Handle)
    {
        ApplyLoadedAssets(Handle);
    });
Params.OnCancel = FStreamableDelegateWithHandle::CreateWeakLambda(this,
    [this](TSharedPtr<FStreamableHandle> Handle)
    {
        AbortLoadingScreen();
    });

LoadHandle = UAssetManager::GetStreamableManager().RequestAsyncLoad(
    MoveTemp(Params), TEXT("Level transition"));
```

Fields: `TargetsToStream`, `OnComplete`, `OnCancel`, `OnUpdate`, `Priority`, `bManageActiveHandle`, `bStartStalled`, `bUseJustInTimeAsyncLoader`, `DownloadParams`.

---

## Pattern 4: Synchronous Fallback

```cpp
// Option A: RequestSyncLoad — blocks until done. Loading screens and init only.
TSharedPtr<FStreamableHandle> Handle = UAssetManager::GetStreamableManager().RequestSyncLoad(
    AssetPath,
    /*bManageActiveHandle=*/false,
    TEXT("Level transition sync load"));

UObject* Asset = Handle->GetLoadedAsset();

// Option B: block on an already-started async load.
if (LoadHandle.IsValid() && LoadHandle->IsLoadingInProgress())
{
    // Timeout in seconds; 0 waits forever. Returns EAsyncPackageState::Type.
    LoadHandle->WaitUntilComplete(5.f, /*bStartStalledHandles=*/true);
}
```

`FStreamableManager::LoadSynchronous(Target, bManageActiveHandle, RequestHandlePointer)` is the one-asset shortcut that returns the object directly.

---

## Pattern 5: Lambda Callback with Captured State

```cpp
const FPrimaryAssetId AssetId(FPrimaryAssetType(TEXT("MyWeaponDefinition")), FName(TEXT("DA_Rifle")));

TSharedPtr<FStreamableHandle> Handle = UAssetManager::Get().LoadPrimaryAsset(
    AssetId,
    TArray<FName>{ TEXT("Game") },
    FStreamableDelegate::CreateWeakLambda(this, [this, AssetId]()
    {
        UMyWeaponDefinition* Def =
            UAssetManager::Get().GetPrimaryAssetObject<UMyWeaponDefinition>(AssetId);
        if (Def)
        {
            EquipWeapon(Def);
        }
    }));
```

A bare `CreateLambda` that captures `this` is unsafe: the callback can fire after the object is destroyed. Use `CreateUObject` (binds to a `UObject` method and checks validity) or `CreateWeakLambda(this, Lambda)` (drops the call if the owner died).

---

## Pattern 6: Progress Polling

`UObject` has no `Tick`; poll from a timer, an actor tick, or a widget's `NativeTick`.

```cpp
// MyLoadingScreen.h — members
TSharedPtr<FStreamableHandle> LoadHandle;
FTimerHandle ProgressTimer;
```

```cpp
void UMyLoadingScreen::StartLoadWithProgress(const TArray<FSoftObjectPath>& Assets)
{
    FStreamableManager& SM = UAssetManager::GetStreamableManager();

    LoadHandle = SM.RequestAsyncLoad(
        Assets,
        FStreamableDelegate::CreateUObject(this, &UMyLoadingScreen::OnLoadComplete));

    GetWorld()->GetTimerManager().SetTimer(
        ProgressTimer, this, &UMyLoadingScreen::PollProgress, 0.1f, true);
}

void UMyLoadingScreen::PollProgress()
{
    if (LoadHandle.IsValid() && LoadHandle->IsLoadingInProgress())
    {
        const float Progress = LoadHandle->GetLoadProgress();   // 0.0 to 1.0
        SetProgressPercent(Progress);
    }
}

void UMyLoadingScreen::OnLoadComplete()
{
    GetWorld()->GetTimerManager().ClearTimer(ProgressTimer);
    SetProgressPercent(1.f);
    LoadHandle.Reset();
    HideLoadingScreen();
}
```

`GetLoadedCount(int32& LoadedCount, int32& RequestedCount)` gives the raw counts if you prefer a fraction of assets over the streaming progress value.

---

## Pattern 7: Bundle State Transitions

Bundle state changes are the idiomatic way to move a whole set of primary assets between gameplay states.

```cpp
UAssetManager& AM = UAssetManager::Get();

TArray<FPrimaryAssetId> WeaponIds;
AM.GetPrimaryAssetIdList(FPrimaryAssetType(TEXT("MyWeaponDefinition")), WeaponIds);

// Main menu: UI bundle only.
AM.LoadPrimaryAssets(WeaponIds, TArray<FName>{ TEXT("UI") });

// Entering gameplay: add Game, drop UI.
AM.ChangeBundleStateForPrimaryAssets(
    WeaponIds,
    TArray<FName>{ TEXT("Game") },   // AddBundles
    TArray<FName>{ TEXT("UI") },     // RemoveBundles
    /*bRemoveAllBundles=*/false,
    FStreamableDelegate::CreateUObject(this, &UMyItemSubsystem::OnGameplayBundlesReady));

// Leaving gameplay: swap every loaded asset that currently has Game back to UI.
AM.ChangeBundleStateForMatchingPrimaryAssets(
    TArray<FName>{ TEXT("UI") },     // NewBundles
    TArray<FName>{ TEXT("Game") });  // OldBundles

AM.UnloadPrimaryAssets(WeaponIds);
```

All four Asset Manager entry points (`LoadPrimaryAsset(s)`, `LoadPrimaryAssetsWithType`, `ChangeBundleStateForPrimaryAssets`, `ChangeBundleStateForMatchingPrimaryAssets`) also take an `FAssetManagerLoadParams&&` overload with `OnComplete`, `OnCancel`, `OnUpdate`, `Priority` and optional `DownloadParams`. Never pass the trailing `UE::FSourceLocation Location` argument — the compiler fills it in for the streaming debug tools.

---

## Handle Lifecycle Rules

| Query | Meaning |
|---|---|
| `IsActive()` | Request exists, not released and not cancelled |
| `IsLoadingInProgress()` | Async load still running |
| `HasLoadCompleted()` | All targets loaded (the completion delegate may not have run yet) |
| `WasCanceled()` | `CancelHandle()` was called; the completion delegate never fires |
| `HasError()` | One or more targets failed |
| `IsCombinedHandle()` | Handle wraps child handles |
| `IsUsingJustInTimeAsyncLoader()` | The request is being trickled to the async loader |
| `GetDebugName()` | The `DebugName` passed at request time |

Ownership:

- `TSharedPtr<FStreamableHandle>` keeps the loaded assets pinned while it is alive.
- Letting the `TSharedPtr` go out of scope releases the request — after completion if the load is still running (`Engine/StreamableManager.h:316-320`); the manager keeps the handle alive until the completion delegate has run.
- `ReleaseHandle()` before completion is deferred; the completion delegate still runs.
- `CancelHandle()` stops the load immediately; the completion delegate is skipped and the cancel delegate fires if bound.
- `bManageActiveHandle = true` makes the manager hold the handle until you release it explicitly — use it when you have nowhere to store the pointer.
- Primary assets loaded through `UAssetManager` stay resident until `UnloadPrimaryAsset(s)`, independent of the returned handle.

```cpp
Handle->BindCancelDelegate(FStreamableDelegate::CreateWeakLambda(this, [this]()
{
    OnLoadCanceled();
}));
Handle->CancelHandle();
```

---

## Combined Handles

```cpp
FStreamableManager& SM = UAssetManager::GetStreamableManager();

const FSoftObjectPath PathA = BodyMeshSoft.ToSoftObjectPath();
const FSoftObjectPath PathB = HeadMeshSoft.ToSoftObjectPath();

TSharedPtr<FStreamableHandle> HandleA = SM.RequestAsyncLoad(PathA);
TSharedPtr<FStreamableHandle> HandleB = SM.RequestAsyncLoad(PathB);

TSharedPtr<FStreamableHandle> Merged = SM.CreateCombinedHandle(
    TArray<TSharedPtr<FStreamableHandle>>{ HandleA, HandleB },
    TEXT("Character kit"),
    EStreamableManagerCombinedHandleOptions::MergeDebugNames);

Merged->BindCompleteDelegate(FStreamableDelegate::CreateWeakLambda(this, [this]()
{
    OnKitReady();
}));
```

`CreateCombinedHandle(TConstArrayView<TSharedPtr<FStreamableHandle>> ChildHandles, FString DebugName, EStreamableManagerCombinedHandleOptions Options, FStreamableAsyncLoadParams&& Params)` holds the children as hard references while the combined handle is active. `EStreamableManagerCombinedHandleOptions` values: `None`, `MergeDebugNames`, `RedirectParents`, `SkipNulls`.

---

## Just-in-Time Async Loading

`UE_ENABLE_STREAMABLE_JIT_ASYNC_LOADING` (`Engine/StreamableManager.h:20`) is a compile-time gate that defaults to `0`. With it compiled in:

- `s.StreamableEnableJITAsyncLoading` — honours `FStreamableAsyncLoadParams::bUseJustInTimeAsyncLoader` per request.
- `s.StreamableEnableJITAsyncLoadingGlobally` — forces JIT for every request regardless of the per-request flag.

JIT trickles requests into the async loading queue instead of submitting them all at once, so later requests are queued at up-to-date handle priorities and cancelled requests are never submitted at all. It helps most when a large batch is enqueued and priorities change while it drains.

---

## Anti-Patterns

### Releasing the handle before consuming the assets

```cpp
// BAD: after Reset() nothing holds the mesh once this callback returns; the next GC collects it.
LoadHandle = SM.RequestAsyncLoad(MeshSoft.ToSoftObjectPath(), FStreamableDelegate::CreateWeakLambda(this, [this]()
{
    LoadHandle.Reset();
    CachedMesh = MeshSoft.Get();   // raw pointer / non-UPROPERTY cache: dangles after GC
}));

// GOOD: hand ownership to something else first.
LoadHandle = SM.RequestAsyncLoad(MeshSoft.ToSoftObjectPath(), FStreamableDelegate::CreateWeakLambda(this, [this]()
{
    MeshComponent->SetStaticMesh(MeshSoft.Get());
    LoadHandle.Reset();
}));
```

### Storing the handle in a local

```cpp
// BAD: the load still completes, but the handle is released right after completion,
// so the mesh is collectable at the next GC unless something hard-references it.
void AMyActor::LoadStuff()
{
    TSharedPtr<FStreamableHandle> Handle =
        UAssetManager::GetStreamableManager().RequestAsyncLoad(MeshSoft.ToSoftObjectPath());
}

// GOOD: TSharedPtr<FStreamableHandle> LoadHandle; as a class member.
```

### `LoadSynchronous()` in `Tick`

```cpp
// BAD: stalls the game thread on every frame where the asset is not resident.
void AMyActor::Tick(float DeltaTime)
{
    Super::Tick(DeltaTime);
    UStaticMesh* Mesh = MeshSoft.LoadSynchronous();
}
```

Start one async load and apply the result in the callback.

### Ignoring failures

```cpp
if (Handle->HasError())
{
    ShowAssetLoadFailure();
}
```
