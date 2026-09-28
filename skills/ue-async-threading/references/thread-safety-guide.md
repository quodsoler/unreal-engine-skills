# Thread Safety Guide

Rules and patterns for safe cross-thread access in UE 5.8. Header references are relative to `Engine/Source/Runtime`.

---

## Which Thread Am I On

| Function (`Core/Public/CoreGlobals.h:703-745`) | True when |
|---|---|
| `IsInGameThread()` | Game thread (or the main thread during startup) |
| `IsInParallelGameThread()` | A parallel-for-game-thread task |
| `IsInRenderingThread()` | Render thread, or game thread when no separate render thread exists |
| `IsInActualRenderingThread()` | Strictly the render thread |
| `IsInParallelRenderingThread()` | Render thread or one of its parallel workers |
| `IsInRHIThread()` | RHI thread |
| `IsInAudioThread()` / `IsInSlateThread()` | Audio thread / Slate loading thread |

Assert at the boundary rather than deep inside: `check(IsInGameThread());` at the top of any function that touches UObjects and could be reached from a callback.

---

## UObject Access Rules

UObjects are owned by the game thread and collected by the garbage collector. From any other thread, do not: read or write a `UPROPERTY`, call a `UFUNCTION`, call `GetWorld()`, `GetOwner()`, `GetComponentByClass()`, iterate components, spawn or destroy actors, set transforms, bind or broadcast delegates on UObjects, or start timers.

Why it "works sometimes": GC runs at specific points in the frame, so a fast worker rarely overlaps it. Under load, in a long GC, or on slower hardware the race fires as a crash that never reproduces on a developer machine.

### TWeakObjectPtr Hand-Off

Create the weak pointer on the game thread, copy it into the work, resolve it on the game thread.

```cpp
#include "Tasks/Task.h"
#include "Async/Async.h"
#include "GameFramework/Actor.h"
#include "MyActor.generated.h"

UCLASS()
class MYGAME_API AMyActor : public AActor
{
    GENERATED_BODY()

public:
    void ApplyComputedResult(float Value);
    void StartBackgroundCompute();
};

void AMyActor::ApplyComputedResult(float Value) {}

void AMyActor::StartBackgroundCompute()
{
    TWeakObjectPtr<AMyActor> WeakThis(this);

    UE::Tasks::Launch(UE_SOURCE_LOCATION, [WeakThis]()
    {
        const float Result = ExpensiveCompute();               // plain data work

        AsyncTask(ENamedThreads::GameThread, [WeakThis, Result]()
        {
            if (AMyActor* Actor = WeakThis.Get())              // nullptr if collected meanwhile
            {
                Actor->ApplyComputedResult(Result);
            }
        });
    });
}
```

`TWeakObjectPtr<T>` (`Core/Public/UObject/WeakObjectPtrTemplates.h:25`): `Get()`, `IsValid()`, `IsStale()`, `IsExplicitlyNull()`, and `Pin()` returning a `TStrongObjectPtr<T>` (`:151-161`) when you need the object kept alive for a game-thread scope. Delegates offer the same protection: `FTimerDelegate::CreateWeakLambda(this, Lambda)` and `CreateUObject(this, &AMyActor::Method)` stop firing when the object is gone.

### FGCScopeGuard

`FGCScopeGuard` (`CoreUObject/Public/UObject/GarbageCollection.h:117`) prevents GC from starting while it is alive. It is the only sanctioned way to read UObject data from a worker, and it must wrap a short, read-only, non-blocking region:

```cpp
#include "UObject/GarbageCollection.h"

UE::Tasks::Launch(UE_SOURCE_LOCATION, [WeakThis]()
{
    float Snapshot = 0.f;
    {
        FGCScopeGuard NoGC;                                    // blocks GC; never hold across waits or allocations of UObjects
        if (const AMyActor* Actor = WeakThis.Get())
        {
            Snapshot = Actor->GetActorLocation().Z;
        }
    }
    ProcessSnapshot(Snapshot);
});
```

Holding the guard while waiting on another task, or from many workers at once, stalls the game thread when GC is requested. `IsGarbageCollecting()` (`GarbageCollection.h:204`) tells you whether a collection is in progress.

---

## Render Thread

Game-thread code never touches scene proxies or RHI resources directly; it enqueues a command that runs on the render thread with the immediate command list.

```cpp
#include "RenderingThread.h"          // module RenderCore

// FMySceneProxy (your FPrimitiveSceneProxy subclass) declares: void SetColor_RenderThread(const FLinearColor& Color);
FMySceneProxy* SceneProxy = GetMySceneProxy();
const FLinearColor NewColor = FLinearColor::Red;

// RenderCore/Public/RenderingThread.h:1087
ENQUEUE_RENDER_COMMAND(UpdateMyProxy)([Proxy = SceneProxy, NewColor](FRHICommandListImmediate& RHICmdList)
{
    check(IsInRenderingThread());
    Proxy->SetColor_RenderThread(NewColor);
});

FlushRenderingCommands();            // game thread only; blocks until the render thread drains (RenderingThread.h:111)
```

Capture by value: the proxy pointer is owned by the renderer and outlives the command; game-thread UObjects are not. Pass copies of the data you need, never `this`.

---

## Shared Pointer Modes

`TSharedPtr`, `TSharedRef`, `TWeakPtr`, `TSharedFromThis` and `MakeShared` default to `ESPMode::ThreadSafe` (`Core/Public/Templates/SharedPointerFwd.h:24-27`, `SharedPointer.h:2110`). The reference count is atomic; copying and destroying handles across threads is safe.

```cpp
struct FMyComputeBuffer
{
    TArray<float> Samples;
};

TSharedPtr<FMyComputeBuffer> Buffer = MakeShared<FMyComputeBuffer>();          // ThreadSafe by default

// The payload still needs synchronization if two threads mutate it
TSharedPtr<FMyComputeBuffer, ESPMode::NotThreadSafe> HotPathBuffer;             // opt-out only for single-thread hot paths
```

`ESPMode::ThreadSafe` protects the handle operations only. Two threads writing `Buffer->Samples` still race; give the buffer its own lock or hand it over through a queue.

---

## Locks

### FCriticalSection + FScopeLock

`FCriticalSection` is `UE::FPlatformRecursiveMutex` (`HAL/CriticalSection.h:53`), so the same thread may re-enter. `FScopeLock` takes a pointer (`Misc/ScopeLock.h:148`).

```cpp
#include "HAL/CriticalSection.h"
#include "Misc/ScopeLock.h"

class FMyAccumulator
{
public:
    void AddSample(float Value)
    {
        FScopeLock Lock(&Mutex);
        Samples.Add(Value);
        Total += Value;
    }

    float GetAverage() const
    {
        FScopeLock Lock(&Mutex);
        return Samples.Num() > 0 ? Total / Samples.Num() : 0.0f;
    }

private:
    mutable FCriticalSection Mutex;
    TArray<float> Samples;
    float Total = 0.0f;
};
```

`FScopeUnlock` (`Misc/ScopeLock.h:171`) temporarily releases inside a locked scope; `Lock.Unlock()` on an `FScopeLock` releases early. Do not call `Lock()`/`Unlock()` manually: early returns skip the unlock.

### FRWLock for Read-Heavy Data

`FRWLock` is `UE::FPlatformRWLock` (`HAL/CriticalSection.h:56`): many readers or one writer, not recursive.

```cpp
#include "Misc/ScopeRWLock.h"

class FMyPositionCache
{
public:
    FVector Lookup(FName Key) const
    {
        FReadScopeLock ReadLock(RWLock);                       // FReadScopeLock(FRWLock&)  ScopeRWLock.h:92
        const FVector* Found = Positions.Find(Key);
        return Found ? *Found : FVector::ZeroVector;
    }

    void Update(FName Key, const FVector& NewPos)
    {
        FWriteScopeLock WriteLock(RWLock);                     // FWriteScopeLock(FRWLock&)  ScopeRWLock.h:113
        Positions.Add(Key, NewPos);
    }

private:
    mutable FRWLock RWLock;
    TMap<FName, FVector> Positions;
};
```

`FRWScopeLock Lock(RWLock, SLT_ReadOnly)` (`ScopeRWLock.h:198`) picks the mode at runtime and can upgrade with `ReleaseReadOnlyLockAndAcquireWriteLock_USE_WITH_CAUTION()`, which briefly drops the lock, so re-validate after upgrading. Never take a read lock while holding a read lock on the same `FRWLock`, and never take a write lock while holding a read lock.

### UE::FMutex and UE::TUniqueLock

`UE::FMutex` (`Async/Mutex.h:18`) is a one-byte, non-recursive, non-fair mutex: the right default for fine-grained locks inside data structures. `UE::FRecursiveMutex` (`Async/RecursiveMutex.h:19`) when re-entry is unavoidable; `UE::FSharedMutex` (`Async/SharedMutex.h:22`) for a compact readers/writer lock.

```cpp
#include "Async/Mutex.h"
#include "Async/UniqueLock.h"
#include "Async/LockTags.h"

class FMyRegistry
{
public:
    void Register(int32 Id)
    {
        UE::TUniqueLock Lock(Mutex);                           // Async/UniqueLock.h:25 — locks now, unlocks in the destructor
        Ids.Add(Id);
    }

    bool TryRegister(int32 Id)
    {
        if (!Mutex.TryLock())                                  // Async/Mutex.h:38
        {
            return false;
        }
        Ids.Add(Id);
        Mutex.Unlock();
        return true;
    }

    void RegisterLater(int32 Id, bool bLockNow)
    {
        UE::TDynamicUniqueLock<UE::FMutex> Lock(Mutex, UE::DeferLock);   // Async/UniqueLock.h:65, LockTags.h:12 — not locked yet
        if (bLockNow)
        {
            Lock.Lock();                                       // Lock.OwnsLock() is now true; destructor unlocks
        }
        Ids.Add(Id);
    }

private:
    UE::FMutex Mutex;
    TArray<int32> Ids;
};
```

`UE::TScopeLock<MutexType>` (`Misc/ScopeLock.h:25`) is the generic RAII guard for any type exposing `Lock()`/`Unlock()`; `UE::TSharedLock` (`Async/SharedLock.h:21`) is the shared-mode guard for `UE::FSharedMutex`.

---

## Atomics

`std::atomic<T>` is the atomic type in 5.8. `FThreadSafeCounter` (`HAL/ThreadSafeCounter.h:9`), `FThreadSafeBool` (`HAL/ThreadSafeBool.h:9`) and `TAtomic` (`Templates/Atomic.h:13`) carry deprecation comments and are not maintained.

```cpp
#include <atomic>

class FMyBackgroundProcessor
{
public:
    void RequestStop() { bShouldStop.store(true, std::memory_order_relaxed); }
    bool IsStopRequested() const { return bShouldStop.load(std::memory_order_relaxed); }

    void IncrementProcessed() { ProcessedCount.fetch_add(1, std::memory_order_relaxed); }
    int32 GetProcessedCount() const { return ProcessedCount.load(std::memory_order_relaxed); }

private:
    std::atomic<bool> bShouldStop{ false };
    std::atomic<int32> ProcessedCount{ 0 };
};
```

`std::memory_order_relaxed` suffices for flags and counters read in isolation. Use `release` on the store and `acquire` on the load when the atomic publishes other non-atomic writes (see double-buffering below).

---

## Double-Buffering Pattern

Bulk hand-off from a worker to the game thread without contending a lock on the hot path.

```cpp
#include "HAL/CriticalSection.h"
#include "Misc/ScopeLock.h"
#include <atomic>

class FMyDoubleBuffer
{
public:
    // Worker: publish a full frame of data
    void WriteBack(TArray<FVector>&& NewData)
    {
        FScopeLock Lock(&SwapLock);
        BackBuffer = MoveTemp(NewData);
        bSwapPending.store(true, std::memory_order_release);
    }

    // Game thread: once per tick
    const TArray<FVector>& SwapAndRead()
    {
        if (bSwapPending.load(std::memory_order_acquire))
        {
            FScopeLock Lock(&SwapLock);
            Swap(FrontBuffer, BackBuffer);
            bSwapPending.store(false, std::memory_order_relaxed);
        }
        return FrontBuffer;                                    // read lock-free for the rest of the frame
    }

private:
    TArray<FVector> FrontBuffer;
    TArray<FVector> BackBuffer;
    FCriticalSection SwapLock;
    std::atomic<bool> bSwapPending{ false };
};
```

The lock is contended only during the swap. For many producers use `TMpscQueue` instead; for a single owner of a resource use `UE::Tasks::FPipe` and skip the lock entirely.

---

## Lock Ordering Rules

1. Define one global order for locks that can be held together; always acquire in that order.
2. Never call unknown code (delegates, virtuals, callbacks) while holding a lock.
3. Prefer lock-free hand-off (`TMpscQueue`, `std::atomic`, double-buffering, `FPipe`) when it fits.
4. Keep critical sections short: compute outside, lock only to commit.
5. Two locks that must be held simultaneously usually mean the design can be simplified to one.

```cpp
// WRONG — inconsistent order deadlocks
// Thread 1: Lock(A) -> Lock(B)
// Thread 2: Lock(B) -> Lock(A)

// RIGHT — same order everywhere
// Thread 1: Lock(A) -> Lock(B)
// Thread 2: Lock(A) -> Lock(B)
```

---

## Long Game-Thread Work: FScopedSlowTask

When work must stay on the game thread (editor tooling, cooking, blocking loads), report progress instead of freezing silently. `FScopedSlowTask` (`Core/Public/Misc/ScopedSlowTask.h:41`, derives from `FSlowTask`) shows a progress dialog in the editor and is a no-op in Shipping.

```cpp
#include "Misc/ScopedSlowTask.h"

void RebuildAll(const TArray<int32>& Chunks)
{
    FScopedSlowTask Progress(static_cast<float>(Chunks.Num()), NSLOCTEXT("MyGame", "RebuildChunks", "Rebuilding chunks"));
    Progress.MakeDialog(/*bShowCancelButton*/ true);          // Misc/SlowTask.h:124

    for (int32 Chunk : Chunks)
    {
        if (Progress.ShouldCancel())                           // Misc/SlowTask.h:152
        {
            return;
        }
        Progress.EnterProgressFrame(1.f);                      // Misc/SlowTask.h:131
        RebuildChunk(Chunk);
    }
}
```

`MakeDialogDelayed(float Threshold, bool bShowCancelButton = false, bool bAllowInPIE = false)` (`Misc/SlowTask.h:117`) avoids flashing a dialog for fast operations.

---

## Async Loading

Asset streaming is asynchronous by design and belongs to `ue-data-assets-tables`: `FStreamableManager::RequestAsyncLoad`, `UAssetManager::LoadPrimaryAsset`, `TSoftObjectPtr`. Completion delegates run on the game thread; treat them like any other game-thread callback and still guard captured objects with `TWeakObjectPtr`.

---

## ThreadSanitizer

Clang builds define `USING_THREAD_SANITISER` to 1 when ThreadSanitizer is active (`Core/Public/Clang/ClangPlatformCodeAnalysis.h:110-113`); `TSAN_SAFE` (`:149`) marks a function `no_sanitize("thread")` for known-benign races. `FThreadSafeCounter` already switches to a sanitizer-friendly implementation under that macro (`HAL/ThreadSafeCounter.h:25`). Run ThreadSanitizer builds on Linux or Mac to catch the races described above before they ship.
