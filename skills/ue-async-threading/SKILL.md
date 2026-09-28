---
name: ue-async-threading
description: "Use when offloading work off the game thread, dispatching results back to it, running data-parallel loops, or scheduling timers and tickers in UE C++. Also use when the user mentions 'UE::Tasks::Launch', 'FPipe', 'FTaskEvent', 'AsyncTask', 'Async()', 'TFuture', 'TPromise', 'ParallelFor', 'FRunnable', 'FAsyncTask', 'FCriticalSection', 'FRWLock', 'UE::FMutex', 'TMpscQueue', 'IsInGameThread', 'FTSTicker', 'SetTimer', 'thread safety'. For async asset loading, see ue-data-assets-tables; for smart pointers and GC lifetime, see ue-cpp-foundations."
metadata:
  version: "2.0.0"
  engine: "5.8"
---

# UE Async and Threading

Target engine: **UE 5.8**. APIs below are verified against the 5.8 headers; older forms are listed under "Deprecated — do not use".

This skill covers moving work off the game thread and bringing results back: the `UE::Tasks` system (`Tasks/Task.h`, `Tasks/Pipe.h`), `Async`/`TFuture` (`Async/Async.h`, `Async/Future.h`), thread-pool `FAsyncTask` (`Async/AsyncWork.h`), `ParallelFor`, dedicated `FRunnable` threads, locks and lock-free queues, `FTSTicker` and `FTimerManager`. Everything except timers is in the `Core` module; `FTimerManager` is in `Engine`; `ENQUEUE_RENDER_COMMAND` is in `RenderCore`. Build.cs: `PublicDependencyModuleNames.AddRange(new string[] { "Core", "CoreUObject", "Engine" });` and add `"RenderCore"` only when enqueuing render commands.

## Context

Read `.agents/ue-project-context.md` if it exists (module names, conventions, enabled plugins, GAS/networking setup). Do not stop if it is missing.

Identify the area from the request and the codebase. Ask only when two plausible readings would produce different code.

| Request is about… | Go to |
|---|---|
| Which API fits | [Pattern Selection](#pattern-selection) |
| One-shot background work, chaining, prerequisites, events | [UE::Tasks](#uetasks) |
| Serializing access to a resource without a dedicated thread | [FPipe and FTaskConcurrencyLimiter](#fpipe-and-ftaskconcurrencylimiter) |
| Futures, promises, dispatch to the game thread | [Async, TFuture and AsyncTask](#async-tfuture-and-asynctask) |
| Reusable pooled work unit | [FAsyncTask and FAutoDeleteAsyncTask](#fasynctask-and-fautodeleteasynctask) |
| Data-parallel loops | [ParallelFor](#parallelfor) |
| Long-lived dedicated thread | [FRunnable and FRunnableThread](#frunnable-and-frunnablethread) |
| Locks, events, atomics, queues, shared pointers | [Synchronization](#synchronization) |
| Per-frame callbacks and delayed calls | [Tickers and Timers](#tickers-and-timers) |
| UObject, GC and render-thread rules | [Thread Safety Rules](#thread-safety-rules) |
| Old forms | [Deprecated — do not use](#deprecated--do-not-use) |

## Threading Model

| Thread | Check | Owns |
|---|---|---|
| Game thread | `IsInGameThread()` (`CoreGlobals.h`) | All UObject access, Blueprint, gameplay, timers, tickers |
| Render thread | `IsInRenderingThread()` | Scene proxies, render commands (`ENQUEUE_RENDER_COMMAND`) |
| RHI thread | `IsInRHIThread()` | GPU command submission |
| Worker threads | none | `UE::Tasks` scheduler, TaskGraph `AnyThread`, `ParallelFor` |
| Thread pools | none | `GThreadPool`, `GBackgroundPriorityThreadPool`, `GIOThreadPool`; `GLargeThreadPool` only `WITH_EDITOR` (`Misc/QueuedThreadPool.h`) |

Golden rule: UObjects are game-thread-only. Compute off-thread on plain data, then apply results on the game thread through a `TWeakObjectPtr` (see [Thread Safety Rules](#thread-safety-rules)).

## Pattern Selection

| Need | Use | Result |
|---|---|---|
| One-shot background work, dependencies, chaining | `UE::Tasks::Launch` | `TTask<T>` |
| Run a lambda on the game thread from anywhere | `AsyncTask(ENamedThreads::GameThread, ...)` | none |
| Future-style result with execution-context choice | `Async(EAsyncExecution, ...)` | `TFuture<T>` |
| Serialize tasks touching one resource (FIFO) | `UE::Tasks::FPipe` | `TTask<T>` |
| Cap how many tasks run at once | `UE::Tasks::FTaskConcurrencyLimiter` | none |
| Reusable pooled work unit with owner-managed lifetime | `FAsyncTask<T>` / `FAutoDeleteAsyncTask<T>` | via `GetTask()` |
| Data-parallel loop, caller blocks | `ParallelFor` | none |
| Long-lived thread (socket, file watcher, sim loop) | `FRunnable` + `FRunnableThread::Create` | manual |
| Per-frame callback outside an Actor tick | `FTSTicker::GetCoreTicker().AddTicker` | handle |
| Delayed or repeating call on the game thread | `FTimerManager::SetTimer` | `FTimerHandle` |

## UE::Tasks

```cpp
#include "Tasks/Task.h"
// Tasks/Task.h:299
template<typename TaskBodyType>
UE::Tasks::TTask<TInvokeResult_T<TaskBodyType>> Launch(const TCHAR* DebugName, TaskBodyType&& TaskBody,
    ETaskPriority Priority = ETaskPriority::Normal,
    EExtendedTaskPriority ExtendedPriority = EExtendedTaskPriority::None,
    ETaskFlags Flags = ETaskFlags::None);
// Tasks/Task.h:324 — same, with a prerequisites collection as the third parameter
template<typename TaskBodyType, typename PrerequisitesCollectionType>
UE::Tasks::TTask<TInvokeResult_T<TaskBodyType>> Launch(const TCHAR* DebugName, TaskBodyType&& TaskBody,
    PrerequisitesCollectionType&& Prerequisites,
    ETaskPriority Priority = ETaskPriority::Normal,
    EExtendedTaskPriority ExtendedPriority = EExtendedTaskPriority::None,
    ETaskFlags Flags = ETaskFlags::None);
```

```cpp
using namespace UE::Tasks;

TArray<int32> Data;
TTask<int32> Sum = Launch(UE_SOURCE_LOCATION, [Data]() { return ComputeSum(Data); });

// Prerequisites: TaskB runs after TaskA completes. Handles are copyable; GetResult() is non-const.
TTask<FVector> TaskA = Launch(UE_SOURCE_LOCATION, []() { return FVector(1.f, 2.f, 3.f); });
TTask<void> TaskB = Launch(UE_SOURCE_LOCATION,
    [TaskA]() mutable { const FVector Pos = TaskA.GetResult(); ConsumePosition(Pos); },
    Prerequisites(TaskA), ETaskPriority::BackgroundNormal);

// Manual gate: nothing after the event starts until Trigger()
FTaskEvent Gate{ UE_SOURCE_LOCATION };
TTask<void> Gated = Launch(UE_SOURCE_LOCATION, []() { DoWork(); }, Prerequisites(Gate));
Gate.Trigger();

// Wait for a group, with timeout (Tasks/Task.h:393)
TArray<FTask> Group{ TaskB, Gated };
const bool bAllDone = Wait(Group, FTimespan::FromMilliseconds(5.0));

// Cooperative cancellation (Tasks/Task.h: FCancellationToken)
FCancellationToken Token;
Launch(UE_SOURCE_LOCATION, [&Token]() { for (int32 i = 0; i < 1000; ++i) { if (Token.IsCanceled()) { return; } Step(i); } });
Token.Cancel();

// Already-completed task holding a value (useful for uniform interfaces)
TTask<int32> Ready = MakeCompletedTask<int32>(42);
```

**Handle API** (`Tasks/Task.h:36-95`): `IsValid()`, `IsCompleted()`, `Wait()`, `Wait(FTimespan Timeout)` returns `bool`, `TryRetractAndExecute()` runs the task inline if not started, `GetResult()` waits then returns `ResultType&` (`check(IsValid())`). Free functions: `Wait(FTask&)`, `Wait(Collection, FTimespan)`, `WaitAny(Collection, Timeout)` returns the index, `Any(Collection)` returns an `FTask` completed when any input completes, `AddNested(Task)` from inside a running task so the parent is not complete until the nested one is.

**Priorities** (`Async/Fundamental/Task.h:20`, `Tasks/TaskPrivate.h:59-93`): `ETaskPriority::High | Normal (= Default) | BackgroundHigh | BackgroundNormal | BackgroundLow | Inherit`. `EExtendedTaskPriority::None | Inline | TaskEvent | GameThreadNormalPri | GameThreadHiPri | GameThreadNormalPriLocalQueue | GameThreadHiPriLocalQueue | RenderThreadNormalPri | RenderThreadHiPri | RHIThreadNormalPri | RHIThreadHiPri` (plus `LocalQueue` variants). `ETaskFlags::None | DoNotRunInsideBusyWait`. Passing `EExtendedTaskPriority::GameThreadNormalPri` runs the body on the game thread.

A legacy TaskGraph `FGraphEventRef` can be passed directly as the prerequisites argument of `Launch` (`Tasks/TaskPrivate.h:266`), and collections of them work in `WaitAny`/`Any` (`Tasks/Task.h:417, 468`), so `UE::Tasks` can wait on TaskGraph work. Full chained example: [threading-patterns.md](references/threading-patterns.md#uetasks-chain-with-ftaskevent-and-cancellation).

## FPipe and FTaskConcurrencyLimiter

`FPipe` executes its tasks one after another (FIFO when no extra prerequisites), so it replaces a dedicated thread guarding a resource. The pipe must outlive its last task; `~FPipe()` asserts `!HasWork()`.

```cpp
#include "Tasks/Pipe.h"
#include "Tasks/TaskConcurrencyLimiter.h"

UE::Tasks::FPipe SavePipe{ TEXT("SavePipe") };                       // explicit FPipe(const TCHAR* InDebugName)
UE::Tasks::TTask<void> First = SavePipe.Launch(UE_SOURCE_LOCATION, []() { WriteChunk(0); });
UE::Tasks::TTask<bool> Second = SavePipe.Launch(UE_SOURCE_LOCATION, []() { return WriteChunk(1); },
    UE::Tasks::ETaskPriority::BackgroundNormal);
const bool bInsidePipe = SavePipe.IsInContext();                     // true only while a pipe task runs on this thread
SavePipe.WaitUntilEmpty();                                           // before destroying the pipe

UE::Tasks::FTaskConcurrencyLimiter Limiter(4 /*MaxConcurrency*/, UE::Tasks::ETaskPriority::BackgroundHigh);
for (int32 Index = 0; Index < 64; ++Index)
{
    Limiter.Push(UE_SOURCE_LOCATION, [Index](uint32 Slot) { ProcessWithScratch(Index, Slot); }); // Slot in [0, MaxConcurrency)
}
Limiter.Wait();                                                      // Wait(FTimespan Timeout = FTimespan::MaxValue())
```

`FPipe::Launch(const TCHAR*, TaskBody, [Prerequisites,] ETaskPriority = Default, EExtendedTaskPriority = None, ETaskFlags = None)` (`Tasks/Pipe.h:63,90`). `FTaskConcurrencyLimiter` may be destroyed before its tasks finish; its `Wait` is satisfied once and never re-arms (`Tasks/TaskConcurrencyLimiter.h:171-212`).

## Async, TFuture and AsyncTask

```cpp
#include "Async/Async.h"
// Async/Async.h:299
template<typename CallableType>
auto Async(EAsyncExecution Execution, CallableType&& Callable, TUniqueFunction<void()> CompletionCallback = nullptr) -> TFuture<decltype(Forward<CallableType>(Callable)())>;
// Async/Async.h:407 — takes a reference, so pass *GThreadPool
template<typename CallableType>
auto AsyncPool(FQueuedThreadPool& ThreadPool, CallableType&& Callable, TUniqueFunction<void()> CompletionCallback = nullptr, EQueuedWorkPriority InQueuedWorkPriority = EQueuedWorkPriority::Normal);
// Async/Async.h:430
template<typename CallableType>
auto AsyncThread(CallableType&& Callable, uint32 StackSize = 0, EThreadPriority ThreadPri = TPri_Normal, TUniqueFunction<void()> CompletionCallback = nullptr);
// Async/Async.h:463
CORE_API void AsyncTask(ENamedThreads::Type Thread, TUniqueFunction<void()> Function);
```

| `EAsyncExecution` (`Async/Async.h:27`) | Runs on |
|---|---|
| `TaskGraph` | Worker thread, short tasks |
| `TaskGraphMainThread` | Game thread, may run inside GC or PostLoad waits — only for code safe anywhere |
| `TaskGraphMainTick` | Game thread inside a Tick — the safe choice for delegates and UObject code |
| `Thread` | New dedicated thread, long-running or blocking I/O |
| `ThreadIfForkSafe` | As `Thread`, fork-aware |
| `ThreadPool` | `GThreadPool` |
| `LargeThreadPool` | `GLargeThreadPool`, `WITH_EDITOR` only |

```cpp
TFuture<FMyResult> Future = Async(EAsyncExecution::ThreadPool, []() { return ComputeResult(); });
if (Future.IsReady()) { UseResult(Future.Get()); }       // non-blocking check
FMyResult Copy = Future.Get();                            // blocks; does NOT invalidate (Async/Future.h:226)
Future.Next([](FMyResult Value) { UseResult(Value); });   // continuation receives the value; runs on the completing thread

TPromise<FMyResult> Promise;
TFuture<FMyResult> FromPromise = Promise.GetFuture();   // call once
Async(EAsyncExecution::Thread, [P = MoveTemp(Promise)]() mutable { P.SetValue(ComputeResult()); });

AMyActor* MyActor = FindMyActor();
AsyncTask(ENamedThreads::GameThread, [WeakActor = TWeakObjectPtr<AMyActor>(MyActor), Copy]()
{
    if (AMyActor* Actor = WeakActor.Get()) { Actor->ApplyResult(Copy); }
});
```

**`TFuture<T>`** (`Async/Future.h:210-440`): `Get()`, `IsReady()`, `IsValid()`, `Wait()`, `WaitFor(const FTimespan&)`, `WaitUntil(const FDateTime&)`, `Then(Func)` receives `TFuture<T>`, `Next(Func)` receives `T` (both move the state out and invalidate this future, `Async/Future.h:669`), `Consume()` moves the value out and invalidates, `Share()` gives `TSharedFuture<T>`, `Reset()`. **`TPromise<T>`** (`:527`): `GetFuture()`, `SetValue(const T&)`, `SetValue(T&&)`, `EmplaceValue(Args&&...)`. Continuations run on whichever thread completes the promise — hop to the game thread explicitly.

## FAsyncTask and FAutoDeleteAsyncTask

Reusable work unit on a `FQueuedThreadPool`. Subclass `FNonAbandonableTask` (`Async/AsyncWork.h:666`), implement `DoWork()` and `GetStatId()`. `FAsyncTask<T>` constructs `T` inside its own constructor, so with the `friend` declaration `T`'s constructor may be private (the engine's own example in `AsyncWork.h:26-42` does this). Members your code reads through `Task->GetTask()` must be `public`, because that access happens outside the friend.

```cpp
#include "Async/AsyncWork.h"

class FMyChunkTask : public FNonAbandonableTask
{
public:
    friend class FAsyncTask<FMyChunkTask>;
    friend class FAutoDeleteAsyncTask<FMyChunkTask>;

    explicit FMyChunkTask(TArray<int32> InInput) : Input(MoveTemp(InInput)) {}

    int32 Result = 0;

    void DoWork()
    {
        for (int32 Value : Input) { Result += Value; }
    }

    FORCEINLINE TStatId GetStatId() const
    {
        RETURN_QUICK_DECLARE_CYCLE_STAT(FMyChunkTask, STATGROUP_ThreadPoolAsyncTasks);
    }

private:
    TArray<int32> Input;
};

// Owner-managed lifetime
TArray<int32> Numbers;
FAsyncTask<FMyChunkTask>* Task = new FAsyncTask<FMyChunkTask>(MoveTemp(Numbers));
Task->StartBackgroundTask();                       // (FQueuedThreadPool* = GThreadPool, EQueuedWorkPriority = Normal, EQueuedWorkFlags = None, int64 RequiredMemory = -1, const TCHAR* DebugName = nullptr)
const bool bReady = Task->IsDone();                // poll once per frame, never spin
Task->EnsureCompletion();                          // (bool bDoWorkOnThisThreadIfNotStarted = true, bool bIsLatencySensitive = false)
const int32 Sum = Task->GetTask().Result;
delete Task;

// Fire-and-forget: deletes itself after DoWork
(new FAutoDeleteAsyncTask<FMyChunkTask>(MoveTemp(Numbers)))->StartBackgroundTask();
```

Other members (`Async/AsyncWork.h:415-558`): `StartSynchronousTask(...)` runs inline; `Cancel()` returns `true` if it was still queued; `WaitCompletionWithTimeout(float TimeLimitSeconds)`; `IsWorkDone()` is the cheap non-blocking check. Pass `GBackgroundPriorityThreadPool` as the pool for low-priority work. Full template: [threading-patterns.md](references/threading-patterns.md#fnonabandonabletask--fasynctask-template).

## ParallelFor

```cpp
#include "Async/ParallelFor.h"
// Async/ParallelFor.h:526, :543
inline void ParallelFor(int32 Num, TFunctionRef<void(int32)> Body, EParallelForFlags Flags = EParallelForFlags::None);
inline void ParallelFor(const TCHAR* DebugName, int32 Num, int32 MinBatchSize, TFunctionRef<void(int32)> Body, EParallelForFlags Flags = EParallelForFlags::None);
// Async/ParallelFor.h:792 — Body is called as Body(ContextType&, int32 Index)
template <typename ContextType, typename ContextAllocatorType, typename FunctionType>
inline void ParallelForWithTaskContext(const TCHAR* DebugName, TArray<ContextType, ContextAllocatorType>& OutContexts, int32 Num, int32 MinBatchSize, const FunctionType& Body, EParallelForFlags Flags = EParallelForFlags::None);
```

```cpp
TArray<UStaticMesh*> Meshes;
ParallelFor(Meshes.Num(), [&Meshes](int32 Index) { ProcessMesh(Meshes[Index]); });

ParallelFor(TEXT("ProcessMeshes"), Meshes.Num(), 64, [&Meshes](int32 Index) { ProcessMesh(Meshes[Index]); },
    EParallelForFlags::Unbalanced | EParallelForFlags::BackgroundPriority);

struct FMyScratch { TArray<FVector> Buffer; };
TArray<FMyScratch> Contexts;                                          // one per worker task, reused across iterations
ParallelForWithTaskContext(TEXT("Normals"), Contexts, Meshes.Num(), 32,
    [&Meshes](FMyScratch& Scratch, int32 Index) { Scratch.Buffer.Reset(); ComputeNormals(Meshes[Index], Scratch.Buffer); });
```

| `EParallelForFlags` (`Async/ParallelFor.h:46`) | Effect |
|---|---|
| `None` | Default |
| `ForceSingleThread` | Run sequentially on the caller (debugging) |
| `Unbalanced` | Iterations have very different costs; smaller batches |
| `PumpRenderingThread` | Caller pumps render commands while waiting |
| `BackgroundPriority` | Workers run at background priority |

Also available: `ParallelForTemplate(...)` (no `TFunctionRef` indirection, `:496`), `ParallelForWithPreWork(...)` (`:571`, run caller-side work before helping), `ParallelForWithTaskContext(OutContexts, Num, ContextConstructor, Body, Flags)` (`:722`), `ParallelForWithExistingTaskContext(TArrayView<ContextType> Contexts, Num, MinBatchSize, Body, Flags)` (`:815`). The caller participates and blocks until every iteration finishes. CVar `Async.ParallelFor.DisableOversubscription` (`GParallelForDisableOversubscription`, `Async/ParallelFor.h:43`) stops `ParallelFor` from waking extra workers.

## FRunnable and FRunnableThread

Use only for a dedicated, long-lived thread. Lifecycle on the new thread: `Init()` → `Run()` → `Exit()`. `Stop()` is called from outside by `Kill()`; it must only signal.

```cpp
// HAL/Runnable.h:32-69 — override verbatim
virtual bool Init();
virtual uint32 Run() = 0;
virtual void Stop();
virtual void Exit();
virtual class FSingleThreadRunnable* GetSingleThreadInterface();   // return a fallback for -nothreading platforms, or nullptr

// HAL/RunnableThread.h:44
static CORE_API FRunnableThread* Create(
    class FRunnable* InRunnable,
    const TCHAR* ThreadName,
    uint32 InStackSize = 0,
    EThreadPriority InThreadPri = TPri_Normal,
    uint64 InThreadAffinityMask = FPlatformAffinity::GetNoAffinityMask(),
    EThreadCreateFlags InCreateFlags = EThreadCreateFlags::None);
virtual bool Kill(bool bShouldWait = true) = 0;    // calls Stop(); with bShouldWait blocks until Run() returns
virtual void WaitForCompletion() = 0;
virtual void Suspend(bool bShouldPause = true) = 0;
virtual void SetThreadPriority(EThreadPriority NewPriority) = 0;
```

`EThreadPriority` (`GenericPlatform/GenericPlatformAffinity.h:25`): `TPri_Normal, TPri_AboveNormal, TPri_BelowNormal, TPri_Highest, TPri_Lowest, TPri_SlightlyBelowNormal, TPri_TimeCritical`. Always `Kill(true)` then `delete` the `FRunnableThread*`; killing without waiting leaks and can deadlock (header comment `HAL/RunnableThread.h:77-79`). Sleep with `FPlatformProcess::Sleep(float Seconds)` or block on an `FEventRef` instead of spinning. Full template with shutdown: [threading-patterns.md](references/threading-patterns.md#frunnable-subclass-template).

## Synchronization

| Need | Type | RAII guard | Header |
|---|---|---|---|
| General mutex, recursive | `FCriticalSection` (= `UE::FPlatformRecursiveMutex`) | `FScopeLock Lock(&Mutex)`; `FScopeUnlock` to release inside a scope | `HAL/CriticalSection.h:53`, `Misc/ScopeLock.h:140` |
| Many readers, one writer, not recursive | `FRWLock` (= `UE::FPlatformRWLock`) | `FReadScopeLock(FRWLock&)`, `FWriteScopeLock(FRWLock&)`, `FRWScopeLock(Lock, SLT_ReadOnly / SLT_Write)` | `HAL/CriticalSection.h:56`, `Misc/ScopeRWLock.h:92-198` |
| One-byte, non-recursive, unfair, fastest | `UE::FMutex` | `UE::TUniqueLock<UE::FMutex>`; `UE::TDynamicUniqueLock` with `UE::DeferLock` | `Async/Mutex.h:18`, `Async/UniqueLock.h:19,48`, `Async/LockTags.h:12` |
| Small recursive mutex | `UE::FRecursiveMutex` | `UE::TUniqueLock` | `Async/RecursiveMutex.h:19` |
| Small readers/writer | `UE::FSharedMutex` (`LockShared/UnlockShared`) | `UE::TSharedLock`, `UE::TUniqueLock` | `Async/SharedMutex.h:22`, `Async/SharedLock.h:21` |
| Guard any type with `Lock()/Unlock()` | `UE::TScopeLock<MutexType>` | — | `Misc/ScopeLock.h:25` |

```cpp
TArray<FVector> Points;
mutable FCriticalSection Mutex;              // mutable so const getters can lock
void Add(const FVector& P) { FScopeLock Lock(&Mutex); Points.Add(P); }

TMap<FName, FVector> Cache;
mutable FRWLock CacheLock;
FVector Read(FName Key) const { FReadScopeLock Lock(CacheLock); return Cache.FindRef(Key); }
void Write(FName Key, FVector V) { FWriteScopeLock Lock(CacheLock); Cache.Add(Key, V); }

int32 Counter = 0;
UE::FMutex SmallMutex;
void Bump() { UE::TUniqueLock Lock(SmallMutex); ++Counter; }
```

**Events:** `FEventRef Event(EEventMode::AutoReset)` (`HAL/Event.h:129-139`) is the RAII pooled `FEvent`: `Event->Trigger()`, `Event->Wait()`, `Event->Wait(uint32 WaitTimeMs)`, `Event->Reset()`. Raw pooling: `FEvent* E = FPlatformProcess::GetSynchEventFromPool(bool bIsManualReset = false)` / `FPlatformProcess::ReturnSynchEventToPool(E)` (`GenericPlatformProcess.h:786,799`). `UE::FManualResetEvent` (`Async/ManualResetEvent.h`): `Notify()`, `Wait()`, `WaitFor(FMonotonicTimeSpan)`, `Reset()`. Between tasks prefer `UE::Tasks::FTaskEvent` — waiting on it does not block a worker.

**Atomics:** `std::atomic<T>` (`<atomic>`, already included by `Templates/Atomic.h`). `FThreadSafeCounter`, `FThreadSafeBool` and `TAtomic` are marked deprecated in their headers (see [Deprecated](#deprecated--do-not-use)). Use `std::memory_order_relaxed` for pure flags and counters, `acquire`/`release` when the atomic publishes other data.

**Queues:** `TMpscQueue<T>` (`Containers/MpscQueue.h`) and `TSpscQueue<T>` (`Containers/SpscQueue.h`): `Enqueue(Args&&...)`, `bool Dequeue(T& Out)`, `TOptional<T> Dequeue()`, `T* Peek()`, `IsEmpty()`. Single consumer only. `TQueue<T, EQueueMode>` still compiles but is marked "planned for deprecation" (`Containers/Queue.h:11`).

**Shared pointers:** `TSharedPtr`, `TSharedRef`, `TWeakPtr` and `MakeShared` default to `ESPMode::ThreadSafe` (`Templates/SharedPointerFwd.h:24-27`, `SharedPointer.h:2110`); the refcount is atomic, the pointee is not protected. Opt into `ESPMode::NotThreadSafe` only for hot single-thread paths.

## Tickers and Timers

Both run on the game thread. `FTSTicker` is engine-wide and survives level changes; `FTimerManager` is per `UWorld` and pauses with it.

```cpp
#include "Containers/Ticker.h"
// Inside AMyActor, which declares: void Poll(float DeltaTime); void OnFire();
// Containers/Ticker.h:45,56,66 — delegate returns true to keep ticking, false to remove itself
FTSTicker::FDelegateHandle TickHandle = FTSTicker::GetCoreTicker().AddTicker(
    FTickerDelegate::CreateWeakLambda(this, [this](float DeltaTime) { Poll(DeltaTime); return true; }), 0.0f /*InDelay*/);
FTSTicker::FDelegateHandle Named = FTSTicker::GetCoreTicker().AddTicker(TEXT("MyPoll"), 0.5f, [](float DeltaTime) { return true; });
FTSTicker::RemoveTicker(TickHandle);            // static; safe with an expired handle
```

`FTickerDelegate` is `DECLARE_DELEGATE_RetVal_OneParam(bool, FTickerDelegate, float)` (`Containers/Ticker.h:21`). Subclass `FTSTickerObjectBase` and override `virtual bool Tick(float DeltaTime) = 0` for an object that registers itself (`:136-158`).

```cpp
#include "TimerManager.h"
// Engine/Public/TimerManager.h:167-237, 247-268, 281-291
FTimerHandle FireHandle, OnceHandle, DelegateHandle;                  // normally UPROPERTY-free members of AMyActor
FTimerManager& Timers = GetWorldTimerManager();                       // AActor; elsewhere GetWorld()->GetTimerManager()
Timers.SetTimer(FireHandle, this, &AMyActor::OnFire, 1.0f, /*InbLoop*/ true, /*InFirstDelay*/ -1.f);
Timers.SetTimer(OnceHandle, FTimerDelegate::CreateWeakLambda(this, [this]() { OnFire(); }), 2.0f, false);
Timers.SetTimer(DelegateHandle, FTimerDelegate::CreateUObject(this, &AMyActor::OnFire), 1.0f, false, 0.25f);

FTimerManagerTimerParameters Params;                                  // TimerManager.h:124
Params.bLoop = true; Params.bMaxOncePerFrame = true; Params.FirstDelay = 0.5f;
Timers.SetTimer(FireHandle, this, &AMyActor::OnFire, 0.1f, Params);

FTimerHandle NextTick = Timers.SetTimerForNextTick(this, &AMyActor::OnFire);
Timers.PauseTimer(FireHandle); Timers.UnPauseTimer(FireHandle);
const float Remaining = Timers.GetTimerRemaining(FireHandle);          // -1 if not found
Timers.ClearTimer(FireHandle);                                        // invalidates the handle
Timers.ClearAllTimersForObject(this);                                 // clears timers bound to this object; CreateLambda/TFunction timers need ClearTimer(Handle)
```

`FTimerDelegate` is `TDelegate<void(), FNotThreadSafeNotCheckedDelegateUserPolicy>` (`TimerManager.h:23`); create it with `CreateUObject`, `CreateWeakLambda`, `CreateLambda`, `CreateSP`, `CreateStatic` (`Delegates/DelegateSignatureImpl.inl`). `SetTimer` overloads take a method pointer, `FTimerDelegate`, `FTimerDynamicDelegate`, `TFunction<void(void)>&&`, or no callback (handle-only countdown). Blueprint-facing: `UKismetSystemLibrary::K2_SetTimer(UObject* Object, FString FunctionName, float Time, bool bLooping, bool bMaxOncePerFrame = false, float InitialStartDelay = 0.f, float InitialStartDelayVariance = 0.f)` and `K2_ClearAndInvalidateTimerHandle(const UObject* WorldContextObject, UPARAM(ref) FTimerHandle& Handle)` (`Kismet/KismetSystemLibrary.h:902,832`). Timer examples: [threading-patterns.md](references/threading-patterns.md#ftsticker-and-ftimermanager).

## Thread Safety Rules

1. **Game thread only:** any `UPROPERTY` read or write, `UFUNCTION` call, `GetWorld()`, spawning, destroying, component changes, delegates on UObjects, timers, tickers. Guard entry points with `check(IsInGameThread())`.
2. **Never capture raw `UObject*` or `this` into deferred work.** Capture `TWeakObjectPtr<T>` (`UObject/WeakObjectPtrTemplates.h:25`) and resolve with `Get()` on the game thread, or build delegates with `CreateWeakLambda`.
3. **GC can run between the launch and the callback.** `FGCScopeGuard` (`UObject/GarbageCollection.h:117`) blocks GC for a scope; use it only for short read-only access from a worker, never around blocking waits.
4. **Render thread:** `ENQUEUE_RENDER_COMMAND(MyCommand)([Data](FRHICommandListImmediate& RHICmdList) { UploadOnRenderThread(RHICmdList, Data); });` (`RenderCore/Public/RenderingThread.h:1087`, module `RenderCore`); `FlushRenderingCommands()` from the game thread drains it. Check with `IsInRenderingThread()`.
5. **Shared data needs its own lock** even inside a thread-safe `TSharedPtr`; the refcount is atomic, the payload is not.
6. **Prefer lock-free hand-off:** `TMpscQueue`, `std::atomic`, double-buffering, or one `FPipe` per resource.

Full patterns, lock ordering, double buffering, `FScopedSlowTask` and sanitizer notes: [thread-safety-guide.md](references/thread-safety-guide.md). Async asset loading (`FStreamableManager::RequestAsyncLoad`) belongs to `ue-data-assets-tables`.

## Deprecated — do not use

| Do not emit | Use in 5.8 | Source |
|---|---|---|
| `FTicker::GetCoreTicker()` | `FTSTicker::GetCoreTicker()` | `FTicker` is absent from the 5.8 headers; only `FTSTicker` exists (`Containers/Ticker.h:26`) |
| `FThreadSafeCounter` | `std::atomic<int32>` | header comment "DEPRECATED. Please use `std::atomic<int32>`" (`HAL/ThreadSafeCounter.h:9`) |
| `FThreadSafeBool` | `std::atomic<bool>` | header comment "DEPRECATED" (`HAL/ThreadSafeBool.h:9`) |
| `TAtomic<T>` | `std::atomic<T>` | "planned for deprecation" (`Templates/Atomic.h:13`, `:528`); no `UE_DEPRECATED` macro yet |
| `FExternalMutex` | `TIntrusiveMutex<Params>` (`Async/IntrusiveMutex.h:60`) | `UE_DEPRECATED(5.7)` in `Async/ExternalMutex.h:73` |
| `TExternalMutex<Params>` | `TIntrusiveMutex<Params>` | `UE_DEPRECATED(5.8)` in `Async/ExternalMutex.h:23` |
| `FPlatformProcess::CreateSynchEvent(...)` | `GetSynchEventFromPool` / `ReturnSynchEventToPool`, or `FEventRef` | `UE_DEPRECATED(5.0)` in `GenericPlatform/GenericPlatformProcess.h:776` |
| `TQueue<T, EQueueMode::Mpsc>` / `EQueueMode::Spsc` | `TMpscQueue<T>` / `TSpscQueue<T>` | "planned for deprecation" (`Containers/Queue.h:11`) |
| `ParallelFor(Num, Body, bool bForceSingleThread, bool bPumpRenderingThread)` | `ParallelFor(Num, Body, EParallelForFlags)` | bool overload kept at `Async/ParallelFor.h:481`; flags form `:526` is the documented one |
| `AsyncPool(GThreadPool, ...)` | `AsyncPool(*GThreadPool, ...)` | parameter is `FQueuedThreadPool&` (`Async/Async.h:407`); the pointer form does not compile |
| `TSharedPtr<T, ESPMode::NotThreadSafe>` "because the default is not thread-safe" | `TSharedPtr<T>` — the default is `ESPMode::ThreadSafe` | `Templates/SharedPointerFwd.h:25` |
| `TGraphTask<T>` / `FGraphEventRef` for new work | `UE::Tasks::Launch` / `FTask` (recommendation, not a deprecation) | no `UE_DEPRECATED` in `Async/TaskGraphInterfaces.h`; `FGraphEventRef` still accepted as a `UE::Tasks` prerequisite (`Tasks/Task.h:360`) |

## Common Mistakes

**Touching a UObject from a worker:** GC and other game-thread writes race with you.
```cpp
// WRONG — Health is a UPROPERTY on this AMyActor
Async(EAsyncExecution::ThreadPool, [this]() { Health = ComputeHealth(); });
// RIGHT
Async(EAsyncExecution::ThreadPool, [Weak = TWeakObjectPtr<AMyActor>(this)]()
{
    const float NewHealth = ComputeHealth();
    AsyncTask(ENamedThreads::GameThread, [Weak, NewHealth]() { if (AMyActor* A = Weak.Get()) { A->ApplyResult(NewHealth); } });
});
```

**Blocking the game thread right after launching:** `Task.GetResult()` or `Future.Get()` on the next line turns async into sync. Poll `IsCompleted()`/`IsReady()` in Tick, chain with `Prerequisites`, or hop back with `AsyncTask(ENamedThreads::GameThread, Lambda)`.

**Private members in an `FNonAbandonableTask`:** `friend class FAsyncTask<T>` covers the constructor (it runs inside `FAsyncTask`), but not `GetTask().Result` read from your code. Make the result members public.

**Nested `FRWLock` acquisition:** `FRWLock` is not recursive; a read lock inside a read lock (or a write inside a read) deadlocks. Acquire once per call path or switch to `FCriticalSection`.

**Shared mutable state inside `ParallelFor`:**
```cpp
TArray<int32> Data;
// WRONG
int32 Total = 0; ParallelFor(Data.Num(), [&](int32 i) { Total += Data[i]; });
// RIGHT
std::atomic<int32> Total{ 0 }; ParallelFor(Data.Num(), [&](int32 i) { Total.fetch_add(Data[i], std::memory_order_relaxed); });
```

**Destroying an `FPipe` or `FRunnable` owner with work in flight:** call `Pipe.WaitUntilEmpty()` and `Thread->Kill(true)` before the destructor body runs; `~FPipe()` asserts `!HasWork()`.

**Blocking inside `FRunnable::Stop()`:** `Stop()` runs on the caller's thread while `Run()` is still executing; only set an atomic flag or trigger an event, then let `Kill(true)` wait.

**Raw-delegate timers on a dying actor:** `FTimerDelegate::CreateLambda([this]{})` keeps calling after `EndPlay`; use `CreateWeakLambda`/`CreateUObject` and `ClearAllTimersForObject(this)`.

## Related Skills

- `ue-cpp-foundations` — `TSharedPtr`/`TWeakObjectPtr`/`TStrongObjectPtr` semantics, GC lifetime, subsystems table
- `ue-data-assets-tables` — `FStreamableManager`, `UAssetManager`, async asset loading and soft references
- `ue-testing-debugging` — Unreal Insights task and thread traces, `stat` commands, logging, automation tests for async code
- `ue-procedural-generation` — long-running generation on `FAsyncTask`/`UE::Tasks`, `ProceduralMeshComponent` hand-off to the game thread
- `ue-mass-entity` — `ParallelForEachEntityChunk`, `EParallelExecutionFlags`, processor threading rules
- `ue-networking-replication` — RPC and replication callbacks always run on the game thread
- `ue-blueprint-cpp-interop` — exposing C++ to Blueprint: UFUNCTION/UPROPERTY meta keys, latent actions and async nodes
- `ue-niagara-effects` — Niagara systems, user parameters, data interfaces and data channels
- `ue-serialization-savegames` — USaveGame, FArchive, actor snapshots and config persistence
