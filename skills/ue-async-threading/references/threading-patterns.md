# Threading Patterns Reference

Complete templates for UE 5.8 async and threading patterns. Each snippet compiles against the 5.8 headers named in its include list; user types follow the `FMy*`/`AMy*` convention.

---

## FRunnable Subclass Template

Dedicated thread with cooperative shutdown through `std::atomic<bool>` and an event so the loop never spins.

```cpp
#include "HAL/Runnable.h"
#include "HAL/RunnableThread.h"
#include "HAL/Event.h"
#include "HAL/PlatformProcess.h"
#include "Containers/MpscQueue.h"
#include <atomic>

class FMyBackgroundWorker : public FRunnable
{
public:
    FMyBackgroundWorker() = default;

    ~FMyBackgroundWorker()
    {
        StopThread();
    }

    // Call from the game thread
    void StartThread()
    {
        if (Thread == nullptr)
        {
            Thread = FRunnableThread::Create(
                this,
                TEXT("MyBackgroundWorker"),
                0,                                   // InStackSize: 0 = platform default
                TPri_BelowNormal,                    // stay below the game thread
                FPlatformAffinity::GetPoolThreadMask(),
                EThreadCreateFlags::None);
        }
    }

    // Call from the game thread; safe to call twice
    void StopThread()
    {
        if (Thread != nullptr)
        {
            Thread->Kill(true);                      // calls Stop(), then blocks until Run() returns
            delete Thread;
            Thread = nullptr;
        }
    }

    // Any thread
    void Submit(int32 WorkItem)
    {
        Queue.Enqueue(WorkItem);
        WakeEvent->Trigger();
    }

    // --- FRunnable (HAL/Runnable.h:32-61) ---
    virtual bool Init() override
    {
        return true;                                 // new thread; return false to abort
    }

    virtual uint32 Run() override
    {
        while (!bStopRequested.load(std::memory_order_relaxed))
        {
            int32 Item = 0;
            while (Queue.Dequeue(Item))
            {
                ProcessItem(Item);
            }
            WakeEvent->Wait(100);                    // ms; wakes early on Trigger()
        }
        return 0;
    }

    virtual void Stop() override
    {
        bStopRequested.store(true, std::memory_order_relaxed);   // called from the killing thread; signal only
        WakeEvent->Trigger();
    }

    virtual void Exit() override
    {
        // new thread, after Run() returns: release thread-local resources
    }

private:
    void ProcessItem(int32 Item)
    {
        // pure data work; no UObject access here
    }

    FRunnableThread* Thread = nullptr;
    FEventRef WakeEvent{ EEventMode::AutoReset };    // HAL/Event.h:136 — pooled FEvent, released in the destructor
    TMpscQueue<int32> Queue;
    std::atomic<bool> bStopRequested{ false };
};
```

`FRunnableThread::Create(FRunnable*, const TCHAR* ThreadName, uint32 InStackSize = 0, EThreadPriority InThreadPri = TPri_Normal, uint64 InThreadAffinityMask = FPlatformAffinity::GetNoAffinityMask(), EThreadCreateFlags InCreateFlags = EThreadCreateFlags::None)` (`HAL/RunnableThread.h:44`). If the platform reports `FPlatformProcess::SupportsMultithreading() == false`, `GetSingleThreadInterface()` must return an `FSingleThreadRunnable*` that the engine ticks instead, otherwise return `nullptr` and the feature is unavailable there.

---

## FNonAbandonableTask + FAsyncTask Template

Thread-pool work unit. `FAsyncTask<T>` constructs `T` from the forwarded arguments; you read the result through `GetTask()`. Construction happens inside `FAsyncTask<T>`, so the friend declaration lets the constructor be private; `GetTask()` reads happen in your code, so the result members must be `public`.

```cpp
#include "Async/AsyncWork.h"

class FMyChunkProcessTask : public FNonAbandonableTask
{
public:
    friend class FAsyncTask<FMyChunkProcessTask>;
    friend class FAutoDeleteAsyncTask<FMyChunkProcessTask>;

    FMyChunkProcessTask(TArray<FVector> InRawVertices, float InScale)
        : RawVertices(MoveTemp(InRawVertices))
        , Scale(InScale)
    {}

    // Result: read after IsDone()/EnsureCompletion()
    TArray<FVector> ProcessedVertices;

    void DoWork()
    {
        ProcessedVertices.Reserve(RawVertices.Num());
        for (const FVector& V : RawVertices)
        {
            ProcessedVertices.Add(V * Scale);
        }
    }

    FORCEINLINE TStatId GetStatId() const
    {
        RETURN_QUICK_DECLARE_CYCLE_STAT(FMyChunkProcessTask, STATGROUP_ThreadPoolAsyncTasks);
    }

private:
    TArray<FVector> RawVertices;
    float Scale;
};

// --- Owner-managed: keep the pointer, poll, then collect ---
TArray<FVector> Vertices;
FAsyncTask<FMyChunkProcessTask>* Task = new FAsyncTask<FMyChunkProcessTask>(MoveTemp(Vertices), 2.0f);
Task->StartBackgroundTask(GBackgroundPriorityThreadPool);     // default pool is GThreadPool
// ... later, once per frame:
if (Task->IsDone())
{
    TArray<FVector> Result = MoveTemp(Task->GetTask().ProcessedVertices);
    delete Task;
    Task = nullptr;
}
// ... or force completion (runs inline if not started yet):
// Task->EnsureCompletion(); delete Task;

// --- Fire-and-forget: deletes itself after DoWork, no result retrieval ---
(new FAutoDeleteAsyncTask<FMyChunkProcessTask>(MoveTemp(Vertices), 2.0f))->StartBackgroundTask();
```

Member signatures (`Async/AsyncWork.h`): `StartBackgroundTask(FQueuedThreadPool* InQueuedPool = GThreadPool, EQueuedWorkPriority InQueuedWorkPriority = EQueuedWorkPriority::Normal, EQueuedWorkFlags InQueuedWorkFlags = EQueuedWorkFlags::None, int64 InRequiredMemory = -1, const TCHAR* InDebugName = nullptr)` (`:423`), `StartSynchronousTask(...)` (`:415`), `EnsureCompletion(bool bDoWorkOnThisThreadIfNotStarted = true, bool bIsLatencySensitive = false)` (`:433`), `bool Cancel()` (`:486`), `bool WaitCompletionWithTimeout(float TimeLimitSeconds)` (`:512`), `bool IsDone()` (`:544`), `bool IsWorkDone() const` (`:558`), `TTask& GetTask()` (`:631`).

---

## TGraphTask Template with Prerequisites

Legacy TaskGraph class-based task; still compiles and interoperates with `UE::Tasks` (an `FGraphEventRef` is a valid prerequisite for `UE::Tasks::Launch`). Prefer `UE::Tasks` for new code.

```cpp
#include "Async/TaskGraphInterfaces.h"

class FMySmoothPathTask
{
public:
    explicit FMySmoothPathTask(TArray<FVector>& InOutPath) : Path(InOutPath) {}

    static ESubsequentsMode::Type GetSubsequentsMode() { return ESubsequentsMode::TrackSubsequents; }
    ENamedThreads::Type GetDesiredThread() { return ENamedThreads::AnyBackgroundThreadNormalTask; }
    TStatId GetStatId() const { RETURN_QUICK_DECLARE_CYCLE_STAT(FMySmoothPathTask, STATGROUP_TaskGraphTasks); }

    void DoTask(ENamedThreads::Type CurrentThread, const FGraphEventRef& MyCompletionGraphEvent)
    {
        for (FVector& Point : Path)
        {
            Point = SmoothPoint(Point);
        }
    }

private:
    TArray<FVector>& Path;
};

// Step A: no prerequisites
TArray<FVector> PathData;
FGraphEventRef StepA = TGraphTask<FMySmoothPathTask>::CreateTask(nullptr).ConstructAndDispatchWhenReady(PathData);

// Step B: same task type again, after A (FGraphEventArray = TArray<FGraphEventRef, TInlineAllocator<4>>)
FGraphEventArray StepAPrereq;
StepAPrereq.Add(StepA);
FGraphEventRef StepB = TGraphTask<FMySmoothPathTask>::CreateTask(&StepAPrereq).ConstructAndDispatchWhenReady(PathData);

// Lambda form without a class (Async/TaskGraphInterfaces.h:1135)
FGraphEventRef StepC = FFunctionGraphTask::CreateAndDispatchWhenReady(
    [&PathData]() { FinalizePath(PathData); }, TStatId(), &StepAPrereq, ENamedThreads::AnyThread);

// Wait on the game thread (Async/TaskGraphInterfaces.h:414)
FTaskGraphInterface::Get().WaitUntilTaskCompletes(StepC, ENamedThreads::GameThread);

// Or feed the legacy event into UE::Tasks: a single FGraphEventRef is a valid prerequisites argument (Tasks/TaskPrivate.h:266)
UE::Tasks::TTask<void> After = UE::Tasks::Launch(UE_SOURCE_LOCATION, []() { PublishPath(); }, StepC);
```

---

## UE::Tasks Chain with FTaskEvent and Cancellation

```cpp
#include "Tasks/Task.h"
#include "Tasks/Pipe.h"                     // FPipe (Tasks/Task.h only forward-declares it)
#include "Tasks/TaskConcurrencyLimiter.h"   // FTaskConcurrencyLimiter

using namespace UE::Tasks;

// Step 1: produce
TTask<TArray<FVector>> PosTask = Launch(UE_SOURCE_LOCATION, []() -> TArray<FVector>
{
    TArray<FVector> Positions;
    FillPositions(Positions);
    return Positions;
});

// Step 2: consume step 1 (capture the handle by value; GetResult() is non-const so the lambda is mutable)
TTask<TArray<FVector>> SmoothTask = Launch(UE_SOURCE_LOCATION, [PosTask]() mutable -> TArray<FVector>
{
    TArray<FVector> Raw = PosTask.GetResult();
    SmoothPositions(Raw);
    return Raw;
}, Prerequisites(PosTask), ETaskPriority::BackgroundNormal);

// Step 3: gated by an event the game thread triggers later
FTaskEvent PublishGate{ UE_SOURCE_LOCATION };
FCancellationToken Cancel;
TTask<void> PublishTask = Launch(UE_SOURCE_LOCATION, [SmoothTask, &Cancel]() mutable
{
    if (Cancel.IsCanceled()) { return; }
    const TArray<FVector>& Final = SmoothTask.GetResult();
    PublishPositions(Final);
}, Prerequisites(SmoothTask, PublishGate));

PublishGate.Trigger();                                            // releases step 3 once step 2 is done

// Nested task: parent is not complete until the nested one is, without blocking a worker
TTask<void> Parent = Launch(UE_SOURCE_LOCATION, []()
{
    TTask<void> Child = Launch(UE_SOURCE_LOCATION, []() { ChildWork(); });
    AddNested(Child);
});

// Group wait with timeout; WaitAny returns the index of the first completed task or INDEX_NONE
TArray<FTask> All{ PublishTask, Parent };
const bool bDone = Wait(All, FTimespan::FromMilliseconds(2.0));
const int32 FirstDone = WaitAny(All, FTimespan::Zero());

// Game-thread body via extended priority (no AsyncTask needed)
Launch(UE_SOURCE_LOCATION, []() { check(IsInGameThread()); }, ETaskPriority::Normal, EExtendedTaskPriority::GameThreadNormalPri);

// Pipe: serialize access to one resource
FPipe StatsPipe{ TEXT("StatsPipe") };
StatsPipe.Launch(UE_SOURCE_LOCATION, []() { AccumulateStats(0); });
StatsPipe.Launch(UE_SOURCE_LOCATION, []() { AccumulateStats(1); });   // runs strictly after the previous pipe task
StatsPipe.WaitUntilEmpty();

// Concurrency limiter: at most 3 tasks in flight, each gets a unique slot index for scratch buffers
FTaskConcurrencyLimiter Limiter(3, ETaskPriority::BackgroundHigh);
for (int32 Index = 0; Index < 32; ++Index)
{
    Limiter.Push(UE_SOURCE_LOCATION, [Index](uint32 Slot) { DecompressInto(Index, Slot); });
}
Limiter.Wait(FTimespan::FromSeconds(10.0));
```

---

## ParallelFor Variants

```cpp
#include "Async/ParallelFor.h"

TArray<UStaticMesh*> Meshes;

// Basic — equal-cost iterations (Async/ParallelFor.h:526)
ParallelFor(Meshes.Num(), [&Meshes](int32 Index) { ProcessMesh(Meshes[Index]); });

// Named, with MinBatchSize — avoids task overhead on small ranges (:543)
ParallelFor(TEXT("MeshProcess"), Meshes.Num(), 128, [&Meshes](int32 Index) { ProcessMesh(Meshes[Index]); });

// Flags — variable-cost iterations at background priority
ParallelFor(Meshes.Num(), [&Meshes](int32 Index) { ProcessMesh(Meshes[Index]); },
    EParallelForFlags::Unbalanced | EParallelForFlags::BackgroundPriority);

// Per-task context — one FMyScratch per worker task, Body(ContextType&, int32) (:792)
struct FMyScratch
{
    TArray<FVector> TempBuffer;
};
TArray<FMyScratch> Contexts;
ParallelForWithTaskContext(TEXT("GenNormals"), Contexts, Meshes.Num(), 64,
    [&Meshes](FMyScratch& Scratch, int32 Index)
    {
        Scratch.TempBuffer.Reset();
        ComputeNormals(Meshes[Index], Scratch.TempBuffer);
    });
// After the call, Contexts holds every worker's scratch — reduce here on the calling thread.

// Custom context constructor — ContextConstructor(int32 ContextIndex, int32 NumContexts) (:694)
ParallelForWithTaskContext(TEXT("GenNormalsSized"), Contexts, Meshes.Num(),
    [](int32 ContextIndex, int32 NumContexts) { FMyScratch S; S.TempBuffer.Reserve(1024); return S; },
    [&Meshes](FMyScratch& Scratch, int32 Index) { ComputeNormals(Meshes[Index], Scratch.TempBuffer); });

// Reuse contexts you already own (:815)
ParallelForWithExistingTaskContext(MakeArrayView(Contexts), Meshes.Num(), 64,
    [&Meshes](FMyScratch& Scratch, int32 Index) { ComputeNormals(Meshes[Index], Scratch.TempBuffer); });

// Do caller-side work first, then help with the loop (:571)
ParallelForWithPreWork(Meshes.Num(), [&Meshes](int32 Index) { ProcessMesh(Meshes[Index]); },
    []() { PrepareCaches(); });
```

CVar `Async.ParallelFor.DisableOversubscription` (backed by `GParallelForDisableOversubscription`, `Async/ParallelFor.h:43`) prevents `ParallelFor` from waking additional workers when the scheduler is already saturated.

---

## Async() with EAsyncExecution Modes

```cpp
#include "Async/Async.h"
#include "Misc/FileHelper.h"

// ThreadPool — CPU work
TFuture<int32> F1 = Async(EAsyncExecution::ThreadPool, []() { return HeavyCompute(); });

// TaskGraphMainTick — game thread, inside a Tick; safe for UObject code and delegates
TFuture<void> F2 = Async(EAsyncExecution::TaskGraphMainTick, []() { check(IsInGameThread()); });

// Thread — dedicated thread for blocking I/O (FFileHelper::LoadFileToArray, Misc/FileHelper.h:79)
TFuture<TArray<uint8>> F3 = Async(EAsyncExecution::Thread, []()
{
    TArray<uint8> Bytes;
    FFileHelper::LoadFileToArray(Bytes, TEXT("C:/Temp/Input.bin"), 0);
    return Bytes;
});

// Completion callback — runs on the thread that finished the work
TFuture<float> F4 = Async(EAsyncExecution::TaskGraph, []() { return 1.0f; }, []() { NotifyDone(); });

// Convenience wrappers (Async/Async.h:407, :430) — AsyncPool takes FQueuedThreadPool&
TFuture<int32> F5 = AsyncPool(*GThreadPool, []() { return Compute(); }, nullptr, EQueuedWorkPriority::Low);
TFuture<void> F6 = AsyncThread([]() { BlockingIOWork(); }, 0 /*StackSize*/, TPri_Normal);
```

---

## TPromise/TFuture Producer-Consumer

```cpp
#include "Async/Future.h"
#include "Async/Async.h"

struct FMyStats
{
    int32 PlayerCount = 0;
    TArray<float> FrameTimes;
};

// Producer — move the promise into the work, hand the future to one consumer below
TFuture<FMyStats> StartGatherStats()
{
    TPromise<FMyStats> Promise;
    TFuture<FMyStats> Future = Promise.GetFuture();           // exactly once per promise

    Async(EAsyncExecution::ThreadPool, [P = MoveTemp(Promise)]() mutable
    {
        FMyStats Stats;
        Stats.PlayerCount = GatherPlayerMetrics();
        GatherFrameMetrics(Stats.FrameTimes);
        P.SetValue(MoveTemp(Stats));                          // or P.EmplaceValue(...)
    });
    return Future;
}

// Consume each future ONE way: Then()/Next() move its state into the continuation and
// invalidate it ("This invalidate this future", Async/Future.h:669); a second call asserts.

// Consumer A — continuation with the future (Then) or the value (Next); runs where the promise was fulfilled
void ConsumeWithThen(TFuture<FMyStats> Future)
{
    Future.Then([](TFuture<FMyStats> Completed)
    {
        const FMyStats& Stats = Completed.Get();              // Get() keeps the future valid
        UE_LOG(LogMyGame, Log, TEXT("Players: %d"), Stats.PlayerCount);
    });
}

// Consumer B — hop to the game thread before touching UObjects
void ConsumeOnGameThread(TFuture<FMyStats> Future, AMyActor* MyActor)
{
    Future.Next([Weak = TWeakObjectPtr<AMyActor>(MyActor)](FMyStats Stats)
    {
        AsyncTask(ENamedThreads::GameThread, [Weak, Stats = MoveTemp(Stats)]()
        {
            if (AMyActor* Actor = Weak.Get()) { Actor->ApplyStats(Stats); }
        });
    });
}

// Consumer C — poll from Tick instead of blocking
void PollFromTick(TFuture<FMyStats>& Future)
{
    if (Future.IsReady()) { FMyStats Stats = Future.Consume(); }   // Consume() moves out and invalidates
}
```

---

## FTSTicker and FTimerManager

```cpp
#include "Containers/Ticker.h"
#include "TimerManager.h"
#include "GameFramework/Actor.h"
#include "MyActor.generated.h"

struct FMyStats;                                               // defined in the TPromise/TFuture example above

UCLASS()
class MYGAME_API AMyActor : public AActor
{
    GENERATED_BODY()

public:
    virtual void BeginPlay() override;
    virtual void EndPlay(const EEndPlayReason::Type EndPlayReason) override;

    void OnFire();
    void OnNextTick();
    void ApplyResult(float Value);
    void ApplyStats(const FMyStats& Stats);

private:
    bool PollNetwork(float DeltaTime);

    FTSTicker::FDelegateHandle PollHandle;
    FTimerHandle FireHandle;
    FTimerHandle OnceHandle;
};

void AMyActor::BeginPlay()
{
    Super::BeginPlay();

    // Engine-wide ticker; return true to keep ticking (Containers/Ticker.h:45)
    PollHandle = FTSTicker::GetCoreTicker().AddTicker(
        FTickerDelegate::CreateUObject(this, &AMyActor::PollNetwork), 0.0f);

    FTimerManager& Timers = GetWorldTimerManager();

    // Method pointer: (Handle, Obj, Method, Rate, bLoop = false, FirstDelay = -1)  TimerManager.h:167
    Timers.SetTimer(FireHandle, this, &AMyActor::OnFire, 1.0f, true);

    // Delegate: (Handle, Delegate, Rate, bLoop, FirstDelay = -1)  TimerManager.h:178
    Timers.SetTimer(OnceHandle, FTimerDelegate::CreateWeakLambda(this, [this]() { OnFire(); }), 3.0f, false);

    // Parameters struct  TimerManager.h:124,211
    FTimerManagerTimerParameters Params;
    Params.bLoop = true;
    Params.bMaxOncePerFrame = true;                            // collapse catch-up ticks after a hitch
    Params.FirstDelay = 0.5f;
    Timers.SetTimer(FireHandle, this, &AMyActor::OnFire, 0.1f, Params);

    // Next tick  TimerManager.h:247
    Timers.SetTimerForNextTick(this, &AMyActor::OnNextTick);
}

void AMyActor::EndPlay(const EEndPlayReason::Type EndPlayReason)
{
    FTSTicker::RemoveTicker(PollHandle);                       // static (Containers/Ticker.h:66)
    GetWorldTimerManager().ClearAllTimersForObject(this);      // TimerManager.h:291
    Super::EndPlay(EndPlayReason);
}

bool AMyActor::PollNetwork(float DeltaTime)
{
    return true;                                               // false removes the ticker
}

void AMyActor::OnFire() {}
void AMyActor::OnNextTick() {}
void AMyActor::ApplyResult(float Value) {}
void AMyActor::ApplyStats(const FMyStats& Stats) {}
```

Queries (`TimerManager.h:304-444`): `PauseTimer(Handle)`, `UnPauseTimer(Handle)`, `IsTimerActive(Handle)`, `IsTimerPaused(Handle)`, `TimerExists(Handle)`, `GetTimerRate(Handle)`, `GetTimerElapsed(Handle)`, `GetTimerRemaining(Handle)`. `FTimerHandle::IsValid()` / `Invalidate()` (`Engine/TimerHandle.h:24,30`). Without an Actor: `GetWorld()->GetTimerManager()` (`Engine/World.h:4289`) or `UGameInstance::GetTimerManager()` (`Engine/GameInstance.h:424`) for timers that must survive level transitions.

---

## Lock-Free Producer/Consumer with TMpscQueue

```cpp
#include "Containers/MpscQueue.h"

struct FMyMessage
{
    int32 Id = 0;
    FVector Position = FVector::ZeroVector;
};

TMpscQueue<FMyMessage> Messages;               // many producers, exactly one consumer

// Producers — any thread; Enqueue forwards constructor arguments
Messages.Enqueue(FMyMessage{ 7, FVector(1.f, 2.f, 3.f) });

// Consumer — one thread only (game thread Tick)
FMyMessage Msg;
while (Messages.Dequeue(Msg))
{
    HandleMessage(Msg);
}
if (const FMyMessage* Front = Messages.Peek()) { }             // consumer-only peek, nullptr when empty
```

`TSpscQueue<T>` (`Containers/SpscQueue.h`) has the same API for the single-producer case. Neither queue provides `Num()`; track counts with a `std::atomic<int32>` if needed.
