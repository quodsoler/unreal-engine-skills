---
name: ue-testing-debugging
description: "Use when writing or running Unreal Engine tests, adding logging, or profiling a build. Also use when the user mentions 'UE_LOG', 'log category', 'DEFINE_LOG_CATEGORY', 'UE_LOGFMT', 'verbosity', 'check', 'ensure', 'ensureMsgf', 'verify', 'IMPLEMENT_SIMPLE_AUTOMATION_TEST', 'automation test', 'TEST_CLASS', 'CQTest', 'AFunctionalTest', 'latent command', 'Unreal Insights', 'TRACE_CPUPROFILER_EVENT_SCOPE', 'TRACE_BOOKMARK', 'stat unit', 'SCOPE_CYCLE_COUNTER', 'memreport', 'DrawDebugLine', 'UE_VLOG', 'gameplay debugger', 'ShowDebug', 'console command', 'CVar', or 'why is my frame time bad'. For module and target wiring, see ue-module-build-system; for editor tooling, see ue-editor-tools."
metadata:
  version: "2.0.0"
  engine: "5.8"
---

# UE Testing, Logging & Profiling

Target engine: **UE 5.8**. APIs below are verified against the 5.8 headers; older forms are listed under "Deprecated — do not use".

This skill owns logging, assertions, automation/functional testing, console commands and performance profiling for the repo. Logging, assertions, stats, trace and CVars live in `Core` (`Logging/`, `Misc/AssertionMacros.h`, `Stats/Stats.h`, `ProfilingDebugging/`, `HAL/IConsoleManager.h`). Debug drawing, the Visual Logger and `AHUD::ShowDebug` live in `Engine`. Test frameworks are the developer modules `AutomationController`, `CQTest`, `FunctionalTesting` and `AutomationDriver`; the Gameplay Debugger is the runtime module `GameplayDebugger`.

## Context

Read `.agents/ue-project-context.md` if it exists (module names, conventions, enabled plugins, GAS/networking setup). Do not stop if it is missing. Identify the area from the request and the codebase; ask only when two plausible readings would produce different code.

| Request is about… | Go to |
|---|---|
| `UE_LOG`, categories, verbosity, structured logging | [Logging](#logging) |
| `check`, `ensure`, `verify`, crash on bad state | [Assertions](#assertions) |
| Unit tests, `IMPLEMENT_SIMPLE_AUTOMATION_TEST`, specs, latent commands | [Automation Tests](#automation-tests) |
| `TEST_CLASS`, `ASSERT_THAT`, fixture-style tests | [CQTest](#cqtest) |
| Tests placed in a map, `AFunctionalTest` | [Functional Tests](#functional-tests) |
| Insights, `stat`, cycle counters, CSV, memory, hitches | [Profiling](#profiling) |
| `DrawDebug*`, `UE_VLOG`, Gameplay Debugger, `ShowDebug` | [Debug Visualization](#debug-visualization) |
| `Exec` functions, `FAutoConsoleCommand`, CVars | [Console Commands and CVars](#console-commands-and-cvars) |

## Logging

### Declaring categories
```cpp
#include "Logging/LogMacros.h"
// MyGameLogging.h — DECLARE_LOG_CATEGORY_EXTERN(Name, DefaultVerbosity, CompileTimeVerbosity)
DECLARE_LOG_CATEGORY_EXTERN(LogMyGame, Log, All);

DEFINE_LOG_CATEGORY(LogMyGame);   // MyGameLogging.cpp
```

| Macro | Where | Scope |
|---|---|---|
| `DECLARE_LOG_CATEGORY_EXTERN(Name, Default, CompileTime)` | header | the module and everything that includes it |
| `DEFINE_LOG_CATEGORY(Name)` | one `.cpp` | pairs with the `EXTERN` form |
| `DEFINE_LOG_CATEGORY_STATIC(Name, Default, CompileTime)` | one `.cpp` | that translation unit only |
| `DECLARE_LOG_CATEGORY_CLASS(Name, Default, CompileTime)` | class body | class static; pair with `DEFINE_LOG_CATEGORY_CLASS(Class, Name)` |

`DefaultVerbosity` must be `<=` `CompileTimeVerbosity` or the `static_assert` inside the category macro fires. Anything above `CompileTimeVerbosity` is deleted by the compiler, so `DECLARE_LOG_CATEGORY_EXTERN(LogMyGame, Log, Log)` removes every `Verbose`/`VeryVerbose` call site. Monolithic builds can cap all categories at once by defining `COMPILED_IN_MINIMUM_VERBOSITY`.

### Verbosity levels

| Level | Console | File | Notes |
|---|---|---|---|
| `Fatal` | yes | yes | crashes the process, even when logging is disabled |
| `Error` | yes | yes | collected by commandlets and the editor; fails a commandlet |
| `Warning` | yes | yes | can be promoted to an error |
| `Display` | yes | yes | use this when the user must see the line |
| `Log` | **no** | yes | file only — the default for development messages |
| `Verbose` | no | yes | only once the category is raised to `Verbose` |
| `VeryVerbose` | no | yes | only once the category is raised to `VeryVerbose` |

`ELogVerbosity::All` is an alias for `VeryVerbose` and is the usual `CompileTimeVerbosity` argument.

### Emitting
```cpp
UE_LOG(LogMyGame, Display, TEXT("Player %s spawned"), *PlayerName);
UE_LOG(LogMyGame, Warning, TEXT("Inventory full, dropped %s"), *ItemName);
UE_LOG(LogMyGame, Error,   TEXT("Failed to load asset %s"), *AssetPath);
UE_LOG(LogMyGame, Fatal,   TEXT("Save data unrecoverable"));   // crashes

// Condition and arguments are evaluated only when the category is active.
UE_CLOG(Health <= 0.f, LogMyGame, Warning, TEXT("%s has zero health"), *GetName());
```

Wrap an expensive report in `if (UE_LOG_ACTIVE(LogMyGame, Verbose)) { … }` so the work itself is skipped, not just the formatting.

### Structured logging

`UE_LOGFMT` emits named fields that survive into Insights and log analysis; use `{Name}` placeholders, not `%s`.
```cpp
#include "Logging/StructuredLog.h"

// Named fields: order is irrelevant, extra fields are allowed.
UE_LOGFMT(LogMyGame, Warning, "Loading '{Name}' failed with error {Error}",
    ("Name", AssetName), ("Error", ErrorCode), ("Flags", LoadFlags));

// Positional fields: the values must match the placeholders exactly.
UE_LOGFMT(LogMyGame, Display, "Spawned {Count} actors in {Seconds}s", SpawnCount, Elapsed);
UE_CLOGFMT(bFailed, LogMyGame, Error, "Retry {Attempt} failed", ("Attempt", Attempt));
```

Field names must match `[A-Za-z0-9_]+` and be unique per event; values serialize through `SerializeForLog` or `operator<<(FCbWriter&, T)`. `UE_LOGFMT` takes at most 16 fields — use `UE_LOGFMT_EX` with `UE_LOGFMT_FIELD` / `UE_LOGFMT_VALUE` beyond that.

### Shipping and Test builds

`NO_LOGGING` is `!USE_LOGGING_IN_SHIPPING` in **both** `Test` and `Shipping`, and `0` in `Debug` and `Development`. Under `NO_LOGGING`, `UE_LOG`/`UE_CLOG`/`UE_LOGFMT` collapse to a `Fatal`-only path: every non-fatal call disappears, arguments included. Set `bUseLoggingInShipping = true;` in `MyGame.Target.cs` to define `USE_LOGGING_IN_SHIPPING=1` and get logging back.

### Controlling verbosity at runtime

```
-LogCmds="LogMyGame Verbose, LogNet Warning"   # command line, applied at boot
Log list [Filter]       # every category with its current verbosity
Log LogMyGame Verbose   # raise one category
Log global none         # silence everything
Log reset               # restore defaults
```

From code use `UE_SET_LOG_VERBOSITY(LogMyGame, Verbose)`; from config use `[Core.Log]` in `DefaultEngine.ini`.

## Assertions

All in `Misc/AssertionMacros.h`. Choose by *who* is wrong: `check` for programmer error, `ensure` for a recoverable surprise worth reporting, a plain `if` for user or data input.
```cpp
check(Component != nullptr);
checkf(Index >= 0 && Index < Items.Num(), TEXT("Index %d out of [0,%d)"), Index, Items.Num());
checkNoEntry();                             // unreachable branch
checkNoReentry();                           // must not be entered twice
checkSlow(ValidateExpensiveInvariant());    // Debug only
verify(Manager->Initialize());              // expression ALWAYS evaluated

if (ensure(Component != nullptr))
{
    Component->Activate();                  // ensure() returns the condition
}
ensureMsgf(Health > 0.f, TEXT("%s has health %.1f"), *GetName(), Health);
ensureAlways(bInitialized);                 // reports every failure, not just the first
```

`ensure`/`ensureMsgf` report **once per call site per session**; `ensureAlways`/`ensureAlwaysMsgf` report every time. All four return the condition, so they compose with `if`.

| Build | `DO_CHECK` | `DO_ENSURE` | `DO_GUARD_SLOW` | `NO_LOGGING` |
|---|---|---|---|---|
| Debug | 1 | 1 | 1 | 0 |
| Development | 1 | 1 | 0 | 0 |
| Test **and** Shipping | `USE_CHECKS_IN_SHIPPING` | `USE_ENSURES_IN_SHIPPING` | 0 | `!USE_LOGGING_IN_SHIPPING` |

`USE_CHECKS_IN_SHIPPING` defaults to `0` and is set by `bUseChecksInShipping` in the target rules, so **Test builds also strip `check` and `ensure` by default**. When `DO_CHECK` is 0, `check(expr)` degrades to `CA_ASSUME(expr)` and the expression is never evaluated; `verify(expr)` still evaluates it and only drops the halt.

## Automation Tests

Test sources live in a `Private/Tests` folder of the module under test, or in a dedicated test module. Wrap each test file in `#if WITH_DEV_AUTOMATION_TESTS` (or `WITH_AUTOMATION_TESTS` = `WITH_DEV_AUTOMATION_TESTS || WITH_PERF_AUTOMATION_TESTS`) — the `IMPLEMENT_*` macros alone still compile the class, they only skip registration when `WITH_AUTOMATION_WORKER` is 0 (`Misc/AutomationTest.h:4366`); UBT turns both on for every configuration except Test and Shipping, overridable with `bForceCompileDevelopmentAutomationTests`, `bForceCompilePerformanceAutomationTests` and `bForceDisableAutomationTests` (`UEBuildTarget.cs:6309-6325`). Module and target wiring, the full pattern library and the command lines for running tests are in [references/automation-test-patterns.md](references/automation-test-patterns.md).
```cpp
// Source/MyGame/Private/Tests/MyInventoryTest.cpp — whole file inside #if WITH_DEV_AUTOMATION_TESTS
#include "Misc/AutomationTest.h"
#include "MyInventoryComponent.h"

IMPLEMENT_SIMPLE_AUTOMATION_TEST(FMyInventoryAddTest, "MyGame.Inventory.AddItem",
    EAutomationTestFlags::EditorContext | EAutomationTestFlags::ProductFilter)

bool FMyInventoryAddTest::RunTest(const FString& Parameters)
{
    UMyInventoryComponent* Inv = NewObject<UMyInventoryComponent>(GetTransientPackage());
    UE_RETURN_ON_ERROR(Inv != nullptr, TEXT("Inventory created"));

    Inv->AddItem(FName("Sword"), 1);
    TestEqual(TEXT("Count after add"), Inv->GetItemCount(FName("Sword")), 1);
    TestTrue(TEXT("Has sword"), Inv->HasItem(FName("Sword")));
    TestFalse(TEXT("No axe"), Inv->HasItem(FName("Axe")));
    return true;
}
```

`IMPLEMENT_COMPLEX_AUTOMATION_TEST` adds `virtual void GetTests(TArray<FString>& OutBeautifiedNames, TArray<FString>& OutTestCommands) const` and runs `RunTest` once per entry. `IMPLEMENT_CUSTOM_SIMPLE_AUTOMATION_TEST(TClass, TBaseClass, PrettyName, TFlags)` declares a simple test whose base is a shared fixture you derive from `FAutomationTestBase`.

### Flags

`EAutomationTestFlags` is an `enum class` with `ENUM_CLASS_FLAGS`, so combine with `|`. Two `static_assert`s enforce **at least one context flag** and **exactly one filter flag**.
| Group | Values |
|---|---|
| Context (≥1 required) | `EditorContext`, `ClientContext`, `ServerContext`, `CommandletContext`, `ProgramContext` |
| Filter (exactly 1) | `SmokeFilter`, `EngineFilter`, `ProductFilter`, `PerfFilter`, `StressFilter`, `NegativeFilter` |
| Priority | `CriticalPriority`, `HighPriority`, `MediumPriority`, `LowPriority` |
| Other | `NonNullRHI`, `RequiresUser`, `Disabled`, `SupportsAutoRTFM` |

`EAutomationTestFlags_ApplicationContextMask` covers every context flag in one token.

### Assertions inside a test

```cpp
TestEqual(TEXT("Label"), Actual, Expected);              // int32/int64/SIZE_T/FName/FText/FString/FStringView…
TestEqual(TEXT("Approx"), ActualF, ExpectedF, 0.001f);   // float/double/FVector/FRotator/FTransform take a tolerance
TestTrue(TEXT("Label"), bCondition);                     // also TestFalse, TestNotEqual
TestNotNull(TEXT("Label"), Ptr);                         // also TestNull, TestSamePtr, TestNotSamePtr
TestValid(TEXT("Label"), SharedOrWeakPtr);               // also TestInvalid

AddError(FString::Printf(TEXT("Unexpected damage %d"), Damage));   // also AddWarning, AddInfo
AddErrorIfFalse(bOk, TEXT("Setup failed"));
UE_RETURN_ON_ERROR(Ptr != nullptr, TEXT("Ptr must not be null")); // adds error, returns false

// Expect (and swallow) a log line the code under test is supposed to emit.
AddExpectedMessage(TEXT("Inventory is full"), ELogVerbosity::Warning,
                   EAutomationExpectedMessageFlags::Contains);
```

The pattern is a regex unless you use the `…Plain` variants (`AddExpectedMessagePlain`, `AddExpectedErrorPlain`). `AddExpectedError(Pattern, MatchType, Occurrences, bIsRegex)` is the shorthand that matches warnings and errors (it forwards `ELogVerbosity::Warning`, which is inclusive); the `AddExpectedMessage` overload without a verbosity matches every severity.

### Latent commands

`RunTest` returns before latent commands run. Enqueue them, then let `Update()` return `true` when the step is finished. Enqueuing after `return true;` is unreachable code.
```cpp
DEFINE_LATENT_AUTOMATION_COMMAND_ONE_PARAMETER(FMyWaitSecondsCommand, float, Duration);
bool FMyWaitSecondsCommand::Update()
{
    return GetCurrentRunTime() >= Duration;
}

ADD_LATENT_AUTOMATION_COMMAND(FMyWaitSecondsCommand(2.0f));   // inside RunTest, before `return true;`
```

`DEFINE_LATENT_AUTOMATION_COMMAND` through `..._FIVE_PARAMETER` exist. To assert *after* the wait, derive from `IAutomationLatentCommand` and capture the `FAutomationTestBase*`.

### Specs

`BEGIN_DEFINE_SPEC(TClass, PrettyName, TFlags)` / member declarations / `END_DEFINE_SPEC(TClass)` declares a BDD-style test whose body is `void TClass::Define()`. Inside `Define()` use `Describe`, `It`, `LatentIt`, `BeforeEach`, `LatentBeforeEach`, `AfterEach` (and `xIt`/`xDescribe` to disable a block). `DEFINE_SPEC(TClass, PrettyName, TFlags)` replaces the pair when the spec needs no members. Worked example in [references/automation-test-patterns.md](references/automation-test-patterns.md).

## CQTest

`CQTest` (module `CQTest`) is the fixture-based front end over the same framework. `Assert` is a `FNoDiscardAsserter`, so every assertion goes through `ASSERT_THAT`, which returns from the method on failure.
```cpp
#include "CQTest.h"

TEST_CLASS(FMyInventoryTests, "MyGame.Inventory")
{
    UMyInventoryComponent* Inventory = nullptr;

    BEFORE_EACH() { Inventory = NewObject<UMyInventoryComponent>(GetTransientPackage()); }

    TEST_METHOD(AddItem_WhenEmpty_StoresOne)
    {
        Inventory->AddItem(FName("Sword"), 1);
        ASSERT_THAT(AreEqual(1, Inventory->GetItemCount(FName("Sword"))));
    }

    TEST_METHOD_WITH_TAGS(AddItem_WhenFull_Rejects, "[Slow][Inventory]")
    {
        Inventory->SetCapacity(0);
        ASSERT_THAT(IsFalse(Inventory->AddItem(FName("Sword"), 1)));
    }
};
```

`TEST_CLASS(ClassName, TestDir)` uses `FNoDiscardAsserter` and `DefaultFlags` (`EAutomationTestFlags_ApplicationContextMask | EAutomationTestFlags::ProductFilter`). Variants: `TEST_CLASS_WITH_FLAGS(ClassName, TestDir, Flags)`, `TEST_CLASS_WITH_TAGS`, `TEST_CLASS_WITH_BASE`, `TEST_CLASS_WITH_ASSERTS`. Lifecycle macros are `BEFORE_ALL()`/`AFTER_ALL()` (static, once per fixture) and `BEFORE_EACH()`/`AFTER_EACH()` (override `Setup()`/`TearDown()`). Asserter helpers: `IsTrue`, `IsFalse`, `IsNull`, `IsNotNull`, `AreEqual`, `AreNotEqual`, `AreEqualIgnoreCase`, `AreNotEqualIgnoreCase`, `IsNear(Expected, Actual, Epsilon)` — each also takes a trailing failure message.

World, actor and async helpers (`TestCommandBuilder`, `FSpawnHelper`, `FActorTestSpawner`, `FMapTestSpawner`, `TObjectBuilder`, `UTestGameInstance`) are covered in [references/automation-test-patterns.md](references/automation-test-patterns.md).

## Functional Tests

`AFunctionalTest` (module `FunctionalTesting`) is an actor placed in a test map. `PrepareTest`, `StartTest`, `IsReady_Implementation` and `OnTimeout` are **protected** virtuals; `FinishTest` and the `Assert*` helpers are public.
```cpp
// MyFunctionalTest.h
#pragma once
#include "CoreMinimal.h"
#include "FunctionalTest.h"
#include "MyFunctionalTest.generated.h"

UCLASS()
class MYGAME_API AMyFunctionalTest : public AFunctionalTest
{
    GENERATED_BODY()

protected:
    // PrepareTest() sets TimeLimit / PreparationTimeLimit; IsReady_Implementation()
    // gates StartTest(); StartTest() asserts and calls FinishTest().
    virtual void PrepareTest() override;
    virtual bool IsReady_Implementation() override;
    virtual void StartTest() override;

private:
    UPROPERTY(EditInstanceOnly, Category = "MyGame")
    TObjectPtr<AActor> TargetActor;
};
```

Each override calls `Super::` first. `FinishTest(EFunctionalTestResult TestResult, const FString& Message)` ends the run; `EFunctionalTestResult` is `Default`, `Invalid`, `Error`, `Running`, `Failed`, `Succeeded`, and `Default` lets the recorded assertions decide. Assertions are `AssertTrue`/`AssertFalse` (`bool Condition, const FString& Message, const UObject* ContextObject = nullptr`), `AssertIsValid(UObject*, const FString&, const UObject* = nullptr)`, `AssertEqual_Int/Float/Bool/Name/Object/Vector/Rotator/Transform/Quat` and `AssertValue_Int/Float/Double/DateTime` (with an `EComparisonMethod`). Also `TimeLimit`, `PreparationTimeLimit`, `SetTimeLimit(float, EFunctionalTestResult)`, `LogMessage(const FString&)` and `StartStep`/`FinishStep`. Full `.cpp` in [references/automation-test-patterns.md](references/automation-test-patterns.md).

Run every functional test in the current world with the Blueprint node `UFunctionalTestingManager::RunAllFunctionalTests(WorldContextObject, bNewLog, bRunLooped, FailedTestsReproString)` (the class is `MinimalAPI` and the function is not `FUNCTIONALTESTING_API`, so calling it from another module's C++ does not link), or from the command line:
```
UnrealEditor-Cmd.exe MyGame.uproject -ExecCmds="Automation RunTests MyGame.Functional" -unattended -nopause -testexit="Automation Test Queue Empty" -log
```

`Automation` also accepts `List`, `RunTest`, `RunFilter <Filter>`, `RunAll`, `SetFilter`, `SetPriority`, `SetMinimumPriority`, `Quit` and `SoftQuit`; chain them with `;`.

## Profiling

Full command tables, the CSV and LLM workflows and the bottleneck-triage procedure are in [references/profiling-commands.md](references/profiling-commands.md).

### Unreal Insights
```
# Capture at launch
UnrealEditor.exe MyGame.uproject -trace=cpu,frame,bookmark,counters,gpu,loadtime,log -tracehost=127.0.0.1
UnrealEditor.exe MyGame.uproject -trace=cpu,frame -tracefile=MyCapture.utrace

# Or from the console, mid-session
Trace.File D:/Captures/Run1.utrace cpu,frame,bookmark    # or Trace.Send 127.0.0.1 cpu,frame
Trace.Enable counters
Trace.Bookmark WaveStarted
Trace.Stop                    # also Trace.Pause, Trace.Resume, Trace.Status
```

Channel names drop the `Channel` suffix and lowercase: `cpu`, `gpu`, `frame`, `bookmark`, `counters`, `log`, `loadtime`, `task`, `memalloc`, `callstack`, `module`, `metadata`, `net`, `rhicommands`, `rendercommands`, `slate`, `assetmetadata`, `iostore`, `animation`, `region`, `screenshot`.

### Instrumenting code

```cpp
#include "Stats/Stats.h"                              // DECLARE_*_STAT, SCOPE_CYCLE_COUNTER
#include "HAL/PlatformMisc.h"                         // SCOPED_NAMED_EVENT
#include "ProfilingDebugging/CpuProfilerTrace.h"      // TRACE_CPUPROFILER_EVENT_SCOPE
#include "ProfilingDebugging/MiscTrace.h"             // TRACE_BOOKMARK
#include "ProfilingDebugging/CountersTrace.h"         // TRACE_*_COUNTER, TRACE_INT_VALUE
#include "ProfilingDebugging/CsvProfiler.h"           // CSV_*
#include "ProfilingDebugging/ScopedTimers.h"          // FScopedDurationTimer

DECLARE_STATS_GROUP(TEXT("MyGame"), STATGROUP_MyGame, STATCAT_Advanced);
DECLARE_CYCLE_STAT(TEXT("MyGame Tick"), STAT_MyGameTick, STATGROUP_MyGame);
CSV_DEFINE_CATEGORY(MyGame, true);
TRACE_DECLARE_INT_COUNTER(MyGameActiveEnemies, TEXT("MyGame/ActiveEnemies"));

void UMySubsystem::Tick(float DeltaTime)
{
    SCOPE_CYCLE_COUNTER(STAT_MyGameTick);              // stat system + Insights
    TRACE_CPUPROFILER_EVENT_SCOPE(UMySubsystem::Tick); // Insights only, very cheap
    CSV_SCOPED_TIMING_STAT(MyGame, SubsystemTick);     // one CSV column
    SCOPED_NAMED_EVENT(MyGame_Tick, FColor::Orange);   // also visible to PIX/Razor

    TRACE_COUNTER_SET(MyGameActiveEnemies, ActiveEnemyCount);
    CSV_CUSTOM_STAT(MyGame, Enemies, ActiveEnemyCount, ECsvCustomStatOp::Set);
}

void UMySubsystem::BeginWave(int32 WaveNumber)
{
    TRACE_BOOKMARK(TEXT("WaveStarted_%d"), WaveNumber);  // vertical marker in Insights
    CSV_EVENT(MyGame, TEXT("WaveStarted"));

    double LoadSeconds = 0.0;
    {
        FScopedDurationTimer Timer(LoadSeconds);        // accumulates into LoadSeconds
        LoadWaveAssets(WaveNumber);
    }
    UE_LOGFMT(LogMyGame, Display, "Wave {Wave} loaded in {Seconds}s",
        ("Wave", WaveNumber), ("Seconds", LoadSeconds));
}
```

`QUICK_SCOPE_CYCLE_COUNTER(STAT_MyAdHocScope)` declares a stat inline for a one-off measurement; `TRACE_INT_VALUE(TEXT("MyGame/Spawned"), Count)` and `TRACE_FLOAT_VALUE` emit a counter with no declaration; `SCOPED_NAMED_EVENT_TEXT("Name", FColor::Yellow)` takes a literal and `SCOPED_NAMED_EVENT_F` formats one.

### stat commands

`stat unit` first: it splits frame time into Game, Draw, RHIT and GPU. Then drill into the dominant thread with `stat game`, `stat scenerendering`, `stat initviews`, `stat slate`, `stat threading`, `stat streaming`, `stat net`, or `stat MyGame` for your own group. `stat unitgraph` plots the same numbers over time, `stat fps` shows frame rate, `stat hitches` flags spikes, `stat namedevents` emits stat names as profiler events for external tools.

### Memory

```
stat memory / stat memoryplatform    # engine categories / OS physical and virtual
memreport -full                      # dump to Saved/Profiling/MemReports
obj list class=Texture2D             # loaded objects of a class, with sizes
```

Launch with `-LLM` and `-trace=memalloc,callstack,module` for Memory Insights; scope allocations with `LLM_SCOPE(ELLMTag::EngineMisc)` or a custom tag declared via `LLM_DEFINE_TAG`/`LLM_DECLARE_TAG` and entered with `LLM_SCOPE_BYTAG`. On Windows, non-editor builds default to `MallocBinned3`; override with `-binnedmalloc2`, `-binnedmalloc`, `-ansimalloc` or `-stompmalloc`.

### Hitch capture and UObject count

`snapshothitches -start` arms an automatic trace snapshot whenever the stats system detects a hitch; `snapshothitches -stop` disarms it. It needs `STATS` and a non-Shipping build, and disables the `Screenshot` channel while armed so a stale screenshot cannot land in the snapshot tail.

Compiling with `CSV_TRACK_UOBJECT_COUNT=1` (with CSV profiling enabled) records the live `UObject` count every frame as CSV stat `Total` in category `ObjectCount`, read from `UObjectStats::GetUObjectCount()`. Use it to catch object leaks over a long session without a full memory capture.

## Debug Visualization

### Debug drawing

Everything in `DrawDebugHelpers.h` sits behind `ENABLE_DRAW_DEBUG`, which is `UE_ENABLE_DEBUG_DRAWING` = `(!(UE_BUILD_SHIPPING || UE_BUILD_TEST) || WITH_EDITOR)`. In Test or Shipping game builds they become empty inline stubs (`DrawDebugHelpers.h:180-185`; defining `SHIPPING_DRAW_DEBUG_ERROR=1` removes them so stray calls fail to compile), but arguments are still evaluated — wrap expensive call sites in `#if ENABLE_DRAW_DEBUG`.
```cpp
#include "DrawDebugHelpers.h"

// LifeTime: -1 (default) or 0 draws one frame; > 0 is seconds.
// bPersistentLines = true keeps the shape until FlushPersistentDebugLines.
UWorld* World = GetWorld();
DrawDebugLine(World, Start, End, FColor::Red, false, 2.0f, 0, 2.0f);
DrawDebugPoint(World, Location, 8.0f, FColor::White, false, 2.0f, 0);
DrawDebugDirectionalArrow(World, Start, End, 40.0f, FColor::Cyan, false, 2.0f, 0, 2.0f);
DrawDebugBox(World, Center, Extent, FQuat::Identity, FColor::Blue, false, 2.0f, 0, 1.0f);
DrawDebugSphere(World, Center, 100.0f, 12, FColor::Green, false, 2.0f, 0, 1.0f);
DrawDebugCapsule(World, Center, 88.0f, 34.0f, FQuat::Identity, FColor::Yellow, false, 2.0f, 0, 1.0f);
DrawDebugString(World, Location, TEXT("Label"), nullptr, FColor::White, 2.0f, true, 1.0f);
FlushPersistentDebugLines(World);   // and FlushDebugStrings(World)
```

The Blueprint-facing `UKismetSystemLibrary::DrawDebugLine/Arrow/Sphere/Capsule/String` gate themselves, take `FLinearColor` and an `EDrawDebugSceneDepthPriorityGroup`, and are joined by `PrintString(WorldContextObject, InString, bPrintToScreen, bPrintToLog, TextColor, Duration, Key)`.

### Visual Logger

`ENABLE_VISUAL_LOG` is `PLATFORM_DESKTOP && !NO_LOGGING && UE_ENABLE_DEBUG_DRAWING`. Open the timeline with **Window > Visual Logger**.
```cpp
#include "VisualLogger/VisualLogger.h"

UE_VLOG(this, LogMyGame, Log, TEXT("State: %s"), *StateName);
UE_VLOG_UELOG(this, LogMyGame, Warning, TEXT("Lost target %s"), *TargetName);  // also hits UE_LOG
UE_VLOG_LOCATION(this, LogMyGame, Verbose, GetActorLocation(), 20.0f, FColor::Green, TEXT("Here"));
UE_VLOG_SEGMENT(this, LogMyGame, Log, PathStart, PathEnd, FColor::Red, TEXT("Path"));
UE_VLOG_SPHERE(this, LogMyGame, Verbose, PatrolPoint, 50.0f, FColor::Blue, TEXT("Patrol"));
UE_VLOG_ARROW(this, LogMyGame, Log, GetActorLocation(), AimPoint, FColor::Yellow, TEXT("Aim"));  // also _BOX, _CAPSULE
UE_CVLOG(bIsAlerted, this, LogMyGame, Log, TEXT("Alerted"));   // also UE_CVLOG_SEGMENT, UE_CVLOG_LOCATION

FVisualLogger::Get().SetIsRecording(true);       // and SetIsRecordingToFile / SetIsRecordingToTrace
```

Every macro early-outs on `FVisualLogger::IsRecording()`, so the arguments cost nothing when recording is off.

### Gameplay Debugger and ShowDebug

Press `'` in PIE. Register a category from your module's `StartupModule` (add `"GameplayDebugger"` to `PrivateDependencyModuleNames`; wrap the registration and the category class in `#if WITH_GAMEPLAY_DEBUGGER`, which is 0 in Test/Shipping game builds by default — `GameplayDebuggerCategory.h:10-11`, `TargetRules.cs:1125`):
```cpp
#include "GameplayDebugger.h"   // in FMyGameModule::StartupModule()
if (IGameplayDebugger::IsAvailable())
{
    IGameplayDebugger& Debugger = IGameplayDebugger::Get();
    Debugger.RegisterCategory(TEXT("MyGame"),
        IGameplayDebugger::FOnGetCategory::CreateStatic(&FMyDebuggerCategory::MakeInstance),
        EGameplayDebuggerCategoryState::EnabledInGame);
    Debugger.NotifyCategoriesChanged();
}
```

`FMyDebuggerCategory` derives from `FGameplayDebuggerCategory` and overrides `virtual void CollectData(APlayerController* OwnerPC, AActor* DebugActor)` and `virtual void DrawData(APlayerController* OwnerPC, FGameplayDebuggerCanvasContext& CanvasContext)`, feeding them with `AddTextLine` and `AddShape`. Call `UnregisterCategory(TEXT("MyGame"))` in `ShutdownModule`.

`AHUD::ShowDebug(FName DebugType)` toggles a named debug page (`ShowDebug AI`, `ShowDebug Physics`); `ShowDebugToggleSubCategory(FName)` toggles a sub-page and `ShowDebugForReticleTargetToggle(TSubclassOf<AActor>)` retargets it at whatever the reticle is over. Override `virtual void ShowDebugInfo(float& YL, float& YPos)` or bind `AHUD::OnShowDebugInfo` for your own page. The separate `display`, `displayall` and `displayclear` console commands print a named property of an object or class onto the HUD.

## Console Commands and CVars

```cpp
// In the AMyPlayerController class body. Exec works on PlayerController, Pawn, HUD,
// GameMode, GameState, CheatManager (or a UCheatManagerExtension) and GameInstance.
UFUNCTION(Exec)
void ToggleGodMode();
```

```cpp
// MyGameDebug.cpp — console objects register at static init
#include "HAL/IConsoleManager.h"

static FAutoConsoleCommand GMyDumpStatsCmd(
    TEXT("MyGame.DumpStats"), TEXT("Dump gameplay stats"),
    FConsoleCommandDelegate::CreateStatic(&FMyStatsDumper::Dump));

// World + args; prefer this for anything that touches gameplay in PIE.
static FAutoConsoleCommandWithWorldAndArgs GMySpawnItemCmd(
    TEXT("MyGame.SpawnItem"), TEXT("Usage: MyGame.SpawnItem <Name>"),
    FConsoleCommandWithWorldAndArgsDelegate::CreateStatic(&FMySpawner::SpawnFromConsole));

static TAutoConsoleVariable<float> CVarMyDamageScale(
    TEXT("MyGame.DamageScale"), 1.0f, TEXT("Scales all outgoing damage."), ECVF_Cheat);

// FAutoConsoleVariableRef binds a console name to an existing int32/float/bool/FString/FName.
static float GMyDrawDistance = 5000.0f;
static FAutoConsoleVariableRef CVarMyDrawDistance(
    TEXT("MyGame.DrawDistance"), GMyDrawDistance, TEXT("Gameplay draw distance."), ECVF_Default);
```

Read a CVar with `GetValueOnGameThread()`, `GetValueOnRenderThread()` or `GetValueOnAnyThread()`. Common flags: `ECVF_Default`, `ECVF_Cheat` (disabled on shipping targets), `ECVF_ReadOnly`, `ECVF_Scalability`, `ECVF_RenderThreadSafe`. When a command's lifetime is tied to an object, register at runtime with `IConsoleManager::Get().RegisterConsoleCommand(Name, Help, Delegate)` and release the returned `IConsoleCommand*` with `IConsoleManager::Get().UnregisterConsoleObject(Cmd)`.

## Deprecated — do not use

| Do not emit | Use in 5.8 | Source |
|---|---|---|
| `UE_TRACE_BOOKMARK(...)` | `TRACE_BOOKMARK(Format, ...)` | `ProfilingDebugging/MiscTrace.h` — no `UE_TRACE_BOOKMARK` exists |
| `Trace.Start [Channels]` | `Trace.File [Path] [Channels]` | "'Trace.Start' is being deprecated in favor of 'Trace.File'" in `TraceAuxiliary.cpp` |
| `stat startfile` / `stat stopfile` / `stat startfileraw` (`.uestats`) | `Trace.File` + `UnrealInsights.exe` | gated behind `UE_ENABLE_STATS_FILE_DEPRECATED_IN_5_8` (default `0`) in `Stats/StatsFile.h` |
| `stat gpu` | `ProfileGPU`, `DumpGPU`, or the per-queue group `stat GPU0_Graphics0` | GPU groups are built as `STATGROUP_%s` from `GPU%d_%s%d` in `RHI/Private/GPUProfiler.cpp` |
| Unscoped flag names such as `EditorContext` | scope them: `EAutomationTestFlags::EditorContext` | `EAutomationTestFlags` is an `enum class` in `Misc/AutomationTest.h` |
| `EAutomationExpectedErrorFlags::…` as a distinct type | `EAutomationExpectedMessageFlags::…` | the former is now a namespace alias of the latter |
| `UE_LOG_REF`, `UE_LOG_CLINKAGE` | `UE_LOG` | marked "DO NOT USE … will be deprecated" in `Logging/LogMacros.h` |
| `-csvstatfile=Out.csv` | `-csvCaptureFrames=N` plus `CsvProfile StartFile=<name>` | `CsvProfiler.cpp` parses `csvCaptureFrames=`, `csvCategories=`, `csvMetadata=`, `csvGpuStats`, `csvExecCmds=` and others, but no `csvstatfile` |

## Common Mistakes

**Assuming Test builds keep `check`/`ensure`:** `DO_CHECK` and `DO_ENSURE` are `USE_CHECKS_IN_SHIPPING` / `USE_ENSURES_IN_SHIPPING` in **both** Test and Shipping, and both default to `0`. Use `verify()` when the expression must always run, or set `bUseChecksInShipping`.

**Logic inside a log or assert:** `UE_LOG(LogMyGame, Log, TEXT("%d"), AdvanceCounter());` stops advancing once `NO_LOGGING` is on, and `check(Init())` stops initializing once `DO_CHECK` is 0. Compute into a local first, then log or assert on it.

**Using `Log` and expecting console output:** `Display` prints to the console; `Log` only reaches the log file. "My log line never appears" is almost always a `Log`-verbosity message.

**Missing filter flag:** every `IMPLEMENT_*_AUTOMATION_TEST`, `DEFINE_SPEC` and `BEGIN_DEFINE_SPEC` needs exactly one of `SmokeFilter`/`EngineFilter`/`ProductFilter`/`PerfFilter`/`StressFilter`/`NegativeFilter` plus at least one context flag, or the `static_assert` fails to compile.

**Dropping a CQTest assertion result:** the asserter methods are `[[nodiscard]]`, so `Assert.IsTrue(x);` warns and keeps running after a failure. Always write `ASSERT_THAT(IsTrue(x));`.

**`DrawDebug*` in a Shipping or Test game build:** the calls become no-op stubs but their arguments (traces, string formatting) still run. Wrap the whole block in `#if ENABLE_DRAW_DEBUG`.

**Skipping `Super::StartTest()`:** `AFunctionalTest::StartTest` fires the Blueprint `ReceiveStartTest` event, so omitting `Super::` silently disables every Blueprint-authored step in the same test actor.

## Related Skills

- `ue-cpp-foundations` — UObject basics, delegates, containers, the UFUNCTION/UPROPERTY specifier tables
- `ue-module-build-system` — Build.cs and Target.cs wiring, `bBuildDeveloperTools`, test module setup, plugin layout
- `ue-editor-tools` — editor utilities, commandlets and Slate tooling that host test and debug UI
- `ue-physics-collision` — trace and sweep queries whose results you visualize with `DrawDebug*`
- `ue-ai-navigation` — behaviour trees, EQS and the AI-side Gameplay Debugger categories
- `ue-async-threading` — task graph and async work that latent commands and the `task` trace channel observe
- `ue-gameplay-framework` — GameMode, PlayerController, HUD and CheatManager, the hosts for `Exec` functions
- `ue-materials-rendering` — material instances, parameter collections, render targets and post process
