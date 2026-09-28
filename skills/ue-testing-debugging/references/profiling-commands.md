# Profiling: Insights, stat, CSV, LLM

Target engine: **UE 5.8**. Companion to `SKILL.md`. Macros come from `Runtime/Core/Public/ProfilingDebugging/`, `Runtime/Core/Public/Stats/Stats.h` and `Runtime/Core/Public/HAL/LowLevelMemTracker.h`; console commands are registered in `Runtime/Core/Private/` and `Runtime/Engine/Private/`.

---

## Triage order

1. `stat unit` — which of Game, Draw, RHIT or GPU owns the frame?
2. Drill into that thread with the matching `stat <group>`.
3. Capture with Insights (`-trace=cpu,frame` or `Trace.File`) and read the flame chart.
4. Add `SCOPE_CYCLE_COUNTER` / `TRACE_CPUPROFILER_EVENT_SCOPE` around the suspect and re-measure.
5. Keep a CSV baseline (`-csvCaptureFrames=600`) so the improvement is provable.

At 30 Hz a frame budget is 33 ms; at 60 Hz it is 16.6 ms. `stat unit` > budget on:

| Line | Means | Look at |
|---|---|---|
| Game | game-thread CPU | `stat game`, `stat threading`, tick counts, AI, blueprint |
| Draw | render-thread CPU | `stat scenerendering`, `stat initviews`, draw call count, primitive count |
| GPU | GPU cost | `ProfileGPU`, `DumpGPU`, Insights `gpu` channel, overdraw, shader complexity |
| RHIT | RHI thread | driver submission, RHI command volume (`-trace=rhicommands`) |

---

## stat commands

Typed into the console (`~`) or passed with `-ExecCmds="…"`.

### Overlays

| Command | Shows |
|---|---|
| `stat fps` | frame rate |
| `stat unit` | Frame / Game / Draw / RHIT / GPU times in ms |
| `stat unitgraph` | the same values as a rolling graph |
| `stat unitmax`, `stat unitcriticalpath`, `stat unittime` | peak values, critical path, raw timings |
| `stat hitches` | flags frames that exceed the hitch threshold |
| `stat detailed`, `stat summary`, `stat raw` | preset overlay verbosity |
| `stat drawcount` | draw calls |
| `stat levels` | streaming level status |
| `stat namedevents`, `stat verbosenamedevents` | emit stat names as profiler events for external tools |
| `stat colorlist`, `stat version`, `stat timecode`, `stat thermals`, `stat tsr` | misc engine overlays |
| `stat none` | disable every group overlay |

### Stat groups

`stat <groupname>` toggles the group's overlay; `stat <groupname>+` shows it hierarchically. Verified group names include `engine`, `game`, `initviews`, `scenerendering`, `memory`, `memoryplatform`, `streaming`, `streamingdetails`, `threading`, `net`, `slate`, `ai`, `anim`, and any `DECLARE_STATS_GROUP` you add (`stat MyGame`).

```
stat group list                     # every registered group
stat group enable MyGame            # enable without drawing the overlay
stat hier -group=scenerendering -sortby=num -maxdepth=4
stat display -font=small            # or -font=tiny
```

### Dumping to the log

```
stat dumpframe -ms=.001 -root=initviews    # one frame, filtered
stat dumpave -num=30 -ms=5.0               # aggregate; also dumpmax, dumpsum
stat dumphitches                           # toggle hitch dumping
stat dumpevents -ms=0.2 -all               # slow events across all threads
stat dumpnonframe                          # non-frame stats, usually memory
stat dumpcpu
stat slow -ms=1.0 -depth=4                 # toggle slow-frame display
stat namedmarker MyMarker                  # insert a marker into the stats stream
```

---

## Unreal Insights

`UnrealInsights.exe` lives at `Engine/Binaries/Win64/UnrealInsights.exe`. Start it first if you are streaming to a recorder; open a `.utrace` by passing it as the first argument.

### Starting a capture

```
# From the command line
-trace=cpu,frame,bookmark,counters                 # channel set; traces to memory by default
-trace=cpu,frame -tracehost=127.0.0.1              # stream to a running recorder
-trace=cpu,frame -tracefile=D:/Captures/Run1.utrace
-tracefiletrunc                                    # overwrite instead of failing
-tracefiletimestamps                               # append a timestamp to the filename
-statnamedevents                                   # also emit stat names as named events
```

### From the console

| Command | Effect |
|---|---|
| `Trace.File [Path] [ChannelSet]` | start tracing to a file |
| `Trace.Send <Host> [ChannelSet]` | start tracing to a trace store |
| `Trace.Stop` | stop tracing |
| `Trace.Pause` / `Trace.Resume` | pause and re-enable the active channels |
| `Trace.Enable <ChannelSet>` / `Trace.Disable [ChannelSet]` | toggle channels mid-session |
| `Trace.Status` | print current trace state |
| `Trace.SnapshotFile [Path]` | write the in-memory ring buffer to disk |
| `Trace.SnapshotSend <Host> <Port>` | send the in-memory ring buffer to a server |
| `Trace.Bookmark <Name>` | emit a `TRACE_BOOKMARK` event |
| `Trace.RegionBegin` / `Trace.RegionEnd`, each taking a name | emit `TRACE_BEGIN_REGION` / `TRACE_END_REGION` |

`Trace.Start` still exists but logs "'Trace.Start' is being deprecated in favor of 'Trace.File'". Use `Trace.File`.

### Channels

A channel declared as `XxxChannel` is named `xxx` on the command line.

| Channel | Contents |
|---|---|
| `cpu` | CPU scope events on every thread |
| `gpu` | GPU pass timings |
| `frame` | per-frame begin/end markers |
| `bookmark` | `TRACE_BOOKMARK` markers |
| `counters` | `TRACE_*_VALUE` / `TRACE_COUNTER_*` series |
| `log` | `UE_LOG` lines inline on the timeline |
| `loadtime` / `assetloadtime` | package and asset load waterfall |
| `task` | task graph and `UE::Tasks` work |
| `memalloc` | allocation tracking for Memory Insights |
| `callstack`, `module`, `metadata` | symbol and context data that `memalloc` needs |
| `net` | replication traffic |
| `rhicommands`, `rendercommands`, `rdg` | RHI / render command lists and the render graph |
| `slate`, `animation`, `mass`, `messaging`, `iostore`, `http`, `cook`, `assetmetadata` | subsystem-specific |
| `region`, `screenshot` | thread-agnostic timespans and embedded screenshots |

Main views: **Timing Insights** (CPU/GPU flame chart), **Memory Insights**, **Asset Loading**, **Networking**, **Counters**.

---

## Code instrumentation

### Cycle stats

```cpp
#include "Stats/Stats.h"

// In a .cpp, once per module.
DECLARE_STATS_GROUP(TEXT("MyGame"), STATGROUP_MyGame, STATCAT_Advanced);
DECLARE_CYCLE_STAT(TEXT("Inventory Tick"), STAT_MyInventoryTick, STATGROUP_MyGame);
DECLARE_DWORD_COUNTER_STAT(TEXT("Active Enemies"), STAT_MyActiveEnemies, STATGROUP_MyGame);
DECLARE_MEMORY_STAT(TEXT("Inventory Memory"), STAT_MyInventoryMemory, STATGROUP_MyGame);

void UMyInventoryComponent::TickComponent(float DeltaTime, ELevelTick TickType,
                                          FActorComponentTickFunction* ThisTickFunction)
{
    Super::TickComponent(DeltaTime, TickType, ThisTickFunction);
    SCOPE_CYCLE_COUNTER(STAT_MyInventoryTick);
    SET_DWORD_STAT(STAT_MyActiveEnemies, ActiveEnemyCount);
}

void UMyInventoryComponent::Compact()
{
    QUICK_SCOPE_CYCLE_COUNTER(STAT_MyInventoryCompact);   // no DECLARE needed
}
```

Use `DECLARE_STATS_GROUP_VERBOSE` for a group that should stay off unless explicitly enabled, and `DECLARE_*_STAT_EXTERN` + `DEFINE_STAT` when the stat is referenced from more than one `.cpp`.

### Insights CPU scopes

```cpp
#include "ProfilingDebugging/CpuProfilerTrace.h"

void UMySubsystem::Process()
{
    TRACE_CPUPROFILER_EVENT_SCOPE(UMySubsystem::Process);              // compile-time name
    for (const FMyWorkItem& Item : Items)
    {
        TRACE_CPUPROFILER_EVENT_SCOPE_STR("ItemPass");                 // literal
        TRACE_CPUPROFILER_EVENT_SCOPE_TEXT(*Item.GetDebugName());      // runtime TCHAR*
    }
}
```

`TRACE_CPUPROFILER_EVENT_SCOPE_CONDITIONAL(Name, Condition)` and the `..._ON_CHANNEL` variants let you gate a scope. `TRACE_CPUPROFILER_EVENT_MANUAL_START(Name)` with a matching `TRACE_CPUPROFILER_EVENT_MANUAL_END()` covers a span that is not lexically scoped.

### Named events

```cpp
#include "HAL/PlatformMisc.h"

SCOPED_NAMED_EVENT(MyGame_Process, FColor::Orange);        // identifier + colour
SCOPED_NAMED_EVENT_TEXT("ItemPass", FColor::Yellow);       // literal
SCOPED_NAMED_EVENT_FSTRING(DebugName, FColor::Green);      // FString
SCOPED_NAMED_EVENT_F(TEXT("Item %d"), FColor::Cyan, Index);
```

Named events appear in Insights and in external profilers (PIX, Razor, Superluminal). They always emit a `TRACE_CPUPROFILER_EVENT_SCOPE*` too; when `ENABLE_NAMED_EVENTS` is 0 that is all they emit (`HAL/PlatformMisc.h:44,204-211`).

### Bookmarks and regions

```cpp
#include "ProfilingDebugging/MiscTrace.h"

TRACE_BOOKMARK(TEXT("WaveStarted_%d"), WaveNumber);   // no UE_ prefix on this one
TRACE_BEGIN_REGION(TEXT("WaveSpawn"));
TRACE_END_REGION(TEXT("WaveSpawn"));
```

The format argument must be a `const TCHAR` array literal — a `static_assert` enforces it, so `TRACE_BOOKMARK(*SomeFString)` does not compile. The whole macro is skipped unless the `bookmark` channel is enabled.

### Counters

```cpp
#include "ProfilingDebugging/CountersTrace.h"

// Inline: no declaration, name resolved once per call site.
TRACE_INT_VALUE(TEXT("MyGame/SpawnedThisFrame"), SpawnedThisFrame);
TRACE_FLOAT_VALUE(TEXT("MyGame/AverageDamage"), AverageDamage);
TRACE_MEMORY_VALUE(TEXT("MyGame/PoolBytes"), PoolBytes);

// Declared: cheaper, and supports increment/decrement.
TRACE_DECLARE_INT_COUNTER(MyGameActiveEnemies, TEXT("MyGame/ActiveEnemies"));
TRACE_COUNTER_SET(MyGameActiveEnemies, ActiveEnemyCount);
TRACE_COUNTER_INCREMENT(MyGameActiveEnemies);
TRACE_COUNTER_DECREMENT(MyGameActiveEnemies);
TRACE_COUNTER_ADD(MyGameActiveEnemies, BatchSize);
```

`TRACE_DECLARE_FLOAT_COUNTER`, `TRACE_DECLARE_MEMORY_COUNTER`, the atomic variants (`TRACE_DECLARE_ATOMIC_INT_COUNTER`) and the extern forms (`TRACE_DECLARE_INT_COUNTER_EXTERN`) all exist. Counters need the `counters` channel.

### Ad-hoc timing

```cpp
#include "ProfilingDebugging/ScopedTimers.h"

double AccumulatedSeconds = 0.0;
{
    FScopedDurationTimer Timer(AccumulatedSeconds);   // adds to the accumulator on scope exit
    DoWork();
}

{
    FAutoScopedDurationTimer Timer;                   // keeps its own accumulator
    DoWork();
    const double Seconds = Timer.GetTime();
}

{
    FScopedDurationTimeLogger Logger(TEXT("Wave load"));   // logs the duration on scope exit
    LoadWave();
}
```

---

## CSV profiling

CSV output goes to `Saved/Profiling/CSV/` (`FCsvProfiler::GetDefaultDirectory()` = `FPaths::ProfilingDir() + "CSV/"`) and is cheap enough to leave on in Test builds.

```cpp
#include "ProfilingDebugging/CsvProfiler.h"

// One per module, in a .cpp. The second argument is "enabled by default".
CSV_DEFINE_CATEGORY(MyGame, true);
// Cross-module: CSV_DEFINE_CATEGORY_MODULE(MYGAME_API, MyGame, true) plus
// CSV_DECLARE_CATEGORY_EXTERN(MyGame) in a header.

void UMySubsystem::Tick(float DeltaTime)
{
    CSV_SCOPED_TIMING_STAT(MyGame, SubsystemTick);          // inclusive time column
    CSV_SCOPED_TIMING_STAT_EXCLUSIVE(SubsystemTickSelf);    // exclusive time column

    CSV_CUSTOM_STAT(MyGame, ActiveEnemies, ActiveEnemyCount, ECsvCustomStatOp::Set);
    CSV_CUSTOM_STAT(MyGame, DamageDealt, FrameDamage, ECsvCustomStatOp::Accumulate);
}

void UMySubsystem::OnLevelLoaded()
{
    CSV_EVENT(MyGame, TEXT("LevelLoaded"));
}
```

`ECsvCustomStatOp` is `Set`, `Min`, `Max`, `Accumulate`. `CSV_CUSTOM_STAT_GLOBAL(StatName, Value, Op)` writes to the global category.

### Driving a capture

```
# Command line
-csvCaptureFrames=600        # capture N frames from startup, then stop
-csvMetadata="build=1234,map=Arena"
-csvGpuStats                 # include GPU stats in the CSV

# Console
CsvProfile Start
CsvProfile Stop
CsvProfile StartFile=Baseline
CsvProfile ExitOnCompletion
CsvCategory MyGame enable    # or disable
```

`[CsvProfiler]` in `DefaultEngine.ini` accepts `EnabledCategories` / `DisabledCategories` arrays and `bEventTimestamps`. Read the output with `Engine/Extras/PerfReportTool` or any spreadsheet.

---

## Memory

```
stat memory                              # engine-side categories
stat memoryplatform                      # OS physical / virtual
stat streaming / stat streamingdetails   # texture and mesh streaming
memreport                                # dump to Saved/Profiling/MemReports
memreport -full                          # extended dump
obj list                                 # every loaded object
obj list class=Texture2D                 # one class, with sizes
```

### Low Level Memory Tracker

```cpp
#include "HAL/LowLevelMemTracker.h"

// Declare a custom tag once, in a .cpp.
LLM_DEFINE_TAG(MyGame_Inventory, TEXT("MyGame/Inventory"), TEXT("MyGame"));
// LLM_DECLARE_TAG(MyGame_Inventory) in a header if other files need it.

void UMyInventoryComponent::Reserve(int32 Count)
{
    LLM_SCOPE_BYTAG(MyGame_Inventory);
    Items.Reserve(Count);
}

void UMySubsystem::LoadAssets()
{
    LLM_SCOPE(ELLMTag::EngineMisc);          // built-in tag
    LLM_SCOPE_BYNAME(TEXT("MyGame/Assets")); // ad-hoc named scope
}
```

Enable with `-LLM` on the command line; add `-trace=memalloc,callstack,module` to see allocations in Memory Insights.

### Allocator selection

On Windows, editor and program builds prefer Mimalloc then TBB; other builds fall back to `MallocBinned3`, then `MallocBinned2`, then `MallocBinned`. Override on the command line (non-Shipping only) with `-libpasmalloc`, `-ansimalloc`, `-tbbmalloc`, `-mimalloc`, `-binnedmalloc3`, `-binnedmalloc2`, `-binnedmalloc` or `-stompmalloc`. `-stompmalloc` turns heap overruns into immediate access violations and is the fastest way to localize a corruption.

### Automatic hitch snapshots

```
snapshothitches -start
snapshothitches -stop
```

Requires `STATS` and a non-Shipping build. While armed it hooks the stats thread's new-frame delegate and writes a trace snapshot whenever a hitch is detected; it disables the `Screenshot` channel so a stale screenshot cannot land in the snapshot tail.

### UObject count

Build with `CSV_TRACK_UOBJECT_COUNT=1` (and CSV profiling enabled). Each frame the engine records `UObjectStats::GetUObjectCount()` as the CSV stat `Total` in the category `ObjectCount`. A monotonically rising line over a long session is an object leak; cross-check with `obj list`.

---

## GPU

| Tool | Use |
|---|---|
| `ProfileGPU` | single-frame per-pass breakdown printed to the log and a viewer |
| `DumpGPU` | dump one frame's intermediate render resources to disk |
| `-trace=gpu` | GPU track in Timing Insights |
| `stat GPU0_Graphics0` | per-queue GPU stat group (groups are named `GPU<n>_<Type><Index>`) |

`ProfileGPU` is shaped by `r.ProfileGPU.Sort`, `r.ProfileGPU.Root`, `r.ProfileGPU.ThresholdPercent`, `r.ProfileGPU.ShowLeafEvents`, `r.ProfileGPU.ShowStats`, `r.ProfileGPU.TableFormatting` and `r.ProfileGPU.UnicodeOutput`. Draw events are compiled out of Test and Shipping unless the target opts back in.

Declare your own GPU stats with `DECLARE_GPU_STAT(MyGamePass)` / `DECLARE_GPU_STAT_NAMED(MyGamePass, TEXT("MyGame Pass"))` from `ProfilingDebugging/RealtimeGPUProfiler.h` (module `RenderCore`).

For draw-call level inspection, enable the RenderDoc plugin, launch with `-AttachRenderDoc`, and capture with the `renderdoc.CaptureFrame` console command (bound to Alt+F12 by default) or `renderdoc.CaptureFrameCount` for a multi-frame capture.

---

## Networking

```
stat net                                 # the Net stat group
-NetTrace=1                              # enable network tracing (higher values = more detail)
-trace=net                               # network track in Insights
```

---

## Profiling on device

Run the recorder on the host and point the device at it:

```
# Host
UnrealInsights.exe

# Device launch arguments
-trace=cpu,frame -tracehost=<host-ip-address>
```

Console platforms expose their own capture tools through platform extensions; `SCOPE_CYCLE_COUNTER` and `SCOPED_NAMED_EVENT` feed them automatically.

---

## Command-line flag reference

| Flag | Effect |
|---|---|
| `-trace=<channels>` | enable trace channels (memory destination by default) |
| `-tracehost=<host>` | stream the trace to a recorder |
| `-tracefile=<path>` | write the trace to a file |
| `-tracefiletrunc` | overwrite an existing trace file |
| `-tracefiletimestamps` | append a timestamp to the trace filename |
| `-statnamedevents` | emit stat names as named events |
| `-LLM` | enable the Low Level Memory Tracker |
| `-csvCaptureFrames=N` | capture N frames of CSV, then stop |
| `-csvMetadata="k=v,k=v"` | attach metadata columns to the CSV |
| `-csvGpuStats` | include GPU stats in the CSV |
| `-NetTrace=<verbosity>` | enable network tracing |
| `-AttachRenderDoc` | load the RenderDoc plugin at startup |
| `-ExecCmds="a, b, c"` | run console commands after startup |
