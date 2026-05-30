# Profiling and Validation Reference

Use this reference when the user asks for SwiftUI profiling help or provides performance artifacts such as Instruments traces, `xctrace` output, signpost logs, XCTest benchmark results, MetricKit payloads, screen recordings, memory graphs, console logs, or screenshots.

This reference is a validation layer for the main SwiftUI performance skill. It should not replace code review. Use profiling to confirm or reject a specific hypothesis, not to produce generic advice.

## Core Rule

Do not claim that a performance issue was measured unless the task includes actual evidence:

- an Instruments trace
- `xctrace` output
- signpost intervals
- XCTest performance output
- MetricKit payloads
- a memory graph
- a screen recording
- user-provided timing logs
- a command the agent actually ran

If evidence is missing, phrase findings as risks or hypotheses.

Prefer:

```md
This is a likely hot path because the row formats values during `body`. To confirm it, profile the scroll interaction with the SwiftUI instrument or Time Profiler and look for long body updates or formatter samples on the main thread.
```

Avoid:

```md
This causes a 200 ms hitch.
```

unless the artifact or command output actually shows that number.

## What the Agent Can Do

When the environment supports it, the agent may:

- inspect the project structure
- identify the app target and scheme
- run build commands
- list simulators and devices
- run `xcrun xctrace` commands
- create trace files
- export trace metadata or tables
- run XCTest performance tests
- parse text logs, signpost exports, MetricKit JSON, and benchmark output
- compare before/after results

When the environment does not support local profiling, the agent should:

- provide exact local steps for the user
- suggest instrumentation code
- request or describe the most useful artifact to export
- explain what to look for in Instruments
- keep findings clearly labeled as hypotheses

Do not pretend that GUI Instruments was used if no trace was opened or no artifact was provided.

## Capability Check Before Running Profiling

Before attempting profiling, check whether the task has:

- macOS host
- full Xcode installation, not only Command Line Tools
- selected Xcode path
- buildable project
- runnable target
- known scheme/configuration
- available simulator or device
- reproducible scenario
- permission to run shell commands
- permission to create trace/log artifacts

Useful checks:

```bash
xcode-select -p
xcodebuild -version
xcrun simctl list devices available
xcrun xctrace list templates
xcrun xctrace list devices
```

If any of these fail, do not force profiling. Provide a local plan instead.

## First Step: Define the Scenario

Every profiling task needs a reproducible scenario.

Capture:

- target screen
- start state
- exact user action
- expected symptom
- device or simulator
- OS version
- build configuration
- data size
- network/cache state
- number of repetitions

Examples:

```md
Scenario: Open Portfolio, scroll the holdings list from top to bottom, tap Load More once, then continue scrolling for 10 seconds.
```

```md
Scenario: Cold-launch the app after termination, wait until the first account balance is visible, then stop measurement.
```

```md
Scenario: Type five characters into search with 2,000 local rows loaded and observe whether rows update on every keystroke.
```

Avoid measuring vague interactions such as “the app feels slow.” Convert them into a concrete scenario first.

## Build and Runtime Rules

Prefer release-like conditions for performance validation:

- Release configuration when possible
- same device class for before/after comparisons
- same OS version
- same account/data set
- same network conditions
- same app state before each run
- multiple runs for noisy metrics

Debug builds can be useful for diagnosis, logging, and `_printChanges()`, but they are not reliable proof of production performance.

Simulator results are useful for quick iteration and code-path discovery. Prefer physical devices for final conclusions about scrolling smoothness, animation hitches, launch time, thermal behavior, and memory pressure.

Low Power Mode can be used as a manual stress signal, not as a formal benchmark.

## Tool Selection

Choose the smallest tool that can answer the question.

| Symptom or question | Prefer | Look for |
|---|---|---|
| Slow SwiftUI updates | SwiftUI instrument when available | long body updates, unnecessary updates, update groups |
| CPU spike during interaction | Time Profiler | main-thread samples, app symbols, expensive transformations |
| Animation hitch or scroll jank | Hangs/Hitches, Animation Hitches, Core Animation, Time Profiler | missed frames, commit/render cost, layout/drawing work |
| Main-thread stalls | Time Profiler, Hangs, signposts | long main-thread work, blocking calls |
| Repeated allocations or memory churn | Allocations | temporary objects, repeated formatter/string/model creation |
| Memory growth or retention | Memory Graph, Leaks, Allocations | retained view models, cycles, caches, large graphs |
| Launch regression | XCTest `XCTApplicationLaunchMetric`, Instruments, MetricKit | first frame, responsiveness, launch phases |
| Production trend | MetricKit, Xcode Organizer, app telemetry | hangs, launch, memory, CPU, energy, animation responsiveness |
| Local algorithmic cost | XCTest performance test | sorting, filtering, grouping, formatting, render model generation |
| User-visible symptom only | Screen recording plus profiling plan | visible hitch timing, reproduction steps |

Do not assume every frame drop is caused by SwiftUI diffing. Rendering, layout, image decoding, main-thread blocking, GPU work, and UIKit/AppKit bridges can all be involved.

## SwiftUI Instrument

Use the SwiftUI instrument when available in the installed Instruments/Xcode version.

It is useful for:

- long view body updates
- unnecessary updates
- frequent update groups
- expensive representable updates
- correlating state changes with SwiftUI work
- checking whether a dependency-narrowing refactor reduced update work

Inspect:

- views with long `body` updates
- views updating after unrelated state changes
- update groups around the target interaction
- repeated row updates during scrolling or pagination
- representable updates if the screen embeds UIKit/AppKit components

Use it with Time Profiler when possible. The SwiftUI instrument can point to the problematic update region; Time Profiler helps identify which app code consumed CPU during that region.

Do not rely on the SwiftUI instrument being available in every environment. If the template is missing, empty, or unsupported for the current Xcode/device combination, fall back to Time Profiler, Hangs/Hitches, signposts, and targeted debug probes.

## Time Profiler

Use Time Profiler to answer:

- Is the main thread blocked?
- Which interaction creates the CPU spike?
- Which app symbols dominate the sampled time?
- Is `body` indirectly calling expensive app code?
- Are formatters, sorters, mappers, decoders, or computed properties visible in the hot path?
- Is pagination append rebuilding old render data?
- Are row builders or view initializers expensive?

Prioritize app-specific symbols over framework noise.

Common SwiftUI findings:

- sorting inside `body`
- filtering inside repeated content
- date or currency formatting per row
- formatter allocation during view updates
- image decoding or resizing on the main thread
- building large render model arrays during rendering
- broad computed properties read by views
- excessive per-row action/menu construction
- expensive UIKit/AppKit representable updates

Interpretation rule:

- A high-cost app function on the main thread during the slow interaction is actionable.
- A large amount of SwiftUI framework time is context, not automatically the root cause.
- Look for app code that triggers expensive updates or makes SwiftUI reconcile too much work.

## Hangs, Hitches, Animation Hitches, and Core Animation

Use these tools when the symptom is visible stutter, delayed gestures, paused animations, scroll jank, or missed frames.

Check whether the hitch aligns with:

- main-thread blocking work
- too much layout work
- expensive drawing
- image decoding or conversion
- deep or frequently rebuilt view/layer hierarchies
- heavy shadows, masks, blurs, overlays, or visual effects
- repeated state changes during animation
- list updates during scrolling
- large transaction/commit phases

For commit-phase hitches, inspect layout, display, prepare, and commit-related work. Heavy view hierarchy mutations, redundant layout invalidation, image preparation, and deep hierarchies can all contribute.

Do not reduce hitch analysis to “SwiftUI redraws too much” unless the trace shows SwiftUI update work as the dominant cause.

## Allocations

Use Allocations when the symptom suggests memory churn, repeated construction, or allocation-heavy updates.

Look for:

- repeated formatter creation
- repeated string creation in rows
- rebuilding large arrays of render models
- temporary collection churn from sorting/filtering/grouping
- image decoding or resizing
- repeated `AnyView` or wrapper construction in hot paths
- copy-on-write structures copied during rendering
- per-row closure-heavy helper objects

Correlate allocation spikes with signposts or user interactions. Allocation volume alone is not enough; tie it to a symptom such as hitching, CPU spikes, memory pressure, or regression.

## Memory Graphs and Leaks

Use memory graphs when the issue is growth, retention, or suspected leaks.

For SwiftUI screens, inspect:

- view models retained after navigation away
- tasks retaining models after cancellation should have happened
- closures capturing models, services, or parent views unexpectedly
- long-lived publishers, notifications, or async streams
- caches without eviction
- UIKit/AppKit representables retaining coordinators or delegates
- image caches retaining decoded images too aggressively

Do not call every retained object a leak. Distinguish:

- expected lifetime
- cache retention
- delayed release
- retain cycle
- unbounded growth

If the user provides only a memory graph screenshot, state what it proves visually and what requires full graph inspection.

## Signposts

Use signposts to mark app-level phases and align them with profiler timelines.

Good boundaries:

- user action started
- network response received
- render models built
- page appended to state
- filter applied
- sort applied
- search text processed
- animation started
- expensive cache lookup started/finished
- image preparation started/finished

Example:

```swift
import os

private let performanceLog = OSLog(
    subsystem: "com.example.app",
    category: "PortfolioPerformance"
)

@MainActor
func appendNextPage(_ page: HoldingPage) {
    os_signpost(.begin, log: performanceLog, name: "AppendHoldingPage")
    pages.append(page)
    os_signpost(.end, log: performanceLog, name: "AppendHoldingPage")
}
```

A signpost around a state mutation marks the app-level phase. It does not measure the full SwiftUI reconciliation, layout, drawing, or rendering cost by itself. Use it to align app events with Instruments timelines.

For async or overlapping operations, prefer signpost IDs:

```swift
import os

private let log = OSLog(subsystem: "com.example.app", category: "Search")

func applySearch(_ query: String) async {
    let id = OSSignpostID(log: log)
    os_signpost(.begin, log: log, name: "ApplySearch", signpostID: id, "query_length=%d", query.count)
    defer {
        os_signpost(.end, log: log, name: "ApplySearch", signpostID: id)
    }

    await model.applySearch(query)
}
```

Avoid logging sensitive values in signpost messages.

## Temporary SwiftUI Debug Probes

Use debug probes only for local diagnosis.

Useful probes:

```swift
var body: some View {
    let _ = Self._printChanges()

    return content
}
```

`_printChanges()` is an underscored diagnostic helper. Use it only as a temporary local debugging probe, preferably under `#if DEBUG`. Do not treat it as production API.

Other probes:

- count row body invocations
- log render model rebuilds
- log filtering/sorting duration
- log duplicate `.onAppear` pagination triggers
- log page append counts
- log task cancellation/restart events

Remove debug probes before production.

## `xctrace` Command-Line Workflow

Use `xctrace` for repeatable command-line profiling when GUI Instruments automation is not necessary.

Start by listing templates and devices:

```bash
xcrun xctrace list templates
xcrun xctrace list devices
```

Record by attaching to a running process:

```bash
xcrun xctrace record \
  --template 'Time Profiler' \
  --time-limit 30s \
  --output /tmp/SwiftUIPerf.trace \
  --attach <pid-or-process-name>
```

Record on a selected device:

```bash
xcrun xctrace record \
  --device '<device-name-or-udid>' \
  --template 'Time Profiler' \
  --time-limit 30s \
  --output /tmp/SwiftUIPerf.trace \
  --attach <pid-or-process-name>
```

Record by launching a target binary when applicable:

```bash
xcrun xctrace record \
  --template 'Time Profiler' \
  --time-limit 30s \
  --output /tmp/SwiftUIPerf.trace \
  --launch -- /path/to/AppOrTool
```

Open the trace manually if GUI inspection is needed:

```bash
open -a Instruments /tmp/SwiftUIPerf.trace
```

Inspect exportable trace contents first:

```bash
xcrun xctrace export \
  --input /tmp/SwiftUIPerf.trace \
  --toc
```

Then export specific tables using the schema/path shown in the table of contents. Do not hardcode export paths across Xcode versions; inspect the `--toc` output first.

Use:

```bash
xcrun xctrace help record
xcrun xctrace help export
```

when exact command syntax is uncertain.

## XCTest Performance Tests

Use XCTest performance tests for repeatable local benchmarks of isolated app code.

Good candidates:

- render model generation
- filtering
- sorting
- grouping
- formatting pipelines
- pagination state updates
- cache lookups
- pure data transformations
- launch metrics with `XCTApplicationLaunchMetric`
- UI hitch measurements with XCTest metrics when supported by the project/toolchain

Example for pure work:

```swift
final class PortfolioRenderingTests: XCTestCase {
    func testRenderModelGenerationPerformance() {
        let positions = TestFixtures.largePositionSet(count: 5_000)

        measure(metrics: [XCTClockMetric(), XCTCPUMetric(), XCTMemoryMetric()]) {
            _ = PortfolioRenderModelBuilder.makeRows(from: positions)
        }
    }
}
```

Example for launch:

```swift
final class LaunchPerformanceTests: XCTestCase {
    func testLaunchPerformance() throws {
        measure(metrics: [XCTApplicationLaunchMetric(waitUntilResponsive: true)]) {
            XCUIApplication().launch()
        }
    }
}
```

Use XCTest metrics to guard regressions in CI, but do not treat microbenchmarks as complete proof of UI smoothness. They isolate code costs; Instruments and real interactions are still needed for frame-time and rendering issues.

When comparing results:

- run the same test multiple times
- avoid changing data sets between runs
- watch variance
- compare Release-like builds when possible
- keep baselines per device class when results are device-dependent

## MetricKit

Use MetricKit for production-level performance signals.

MetricKit can help prioritize investigation by showing trends in areas such as:

- app launch time
- hangs
- responsiveness
- memory
- CPU
- disk writes
- energy
- crash diagnostics

MetricKit is not a replacement for local profiling. It usually tells the agent where to investigate, not exactly which SwiftUI line to change.

Important rules:

- Treat MetricKit payloads as production evidence, but usually aggregated and delayed evidence.
- Correlate MetricKit trends with app versions, device classes, OS versions, and feature rollouts.
- Use local profiling to reproduce and root-cause the issue.
- Do not infer a specific SwiftUI cause from MetricKit alone unless the payload includes enough diagnostic context.

Minimal subscriber shape:

```swift
import MetricKit

final class MetricsSubscriber: NSObject, MXMetricManagerSubscriber {
    func start() {
        MXMetricManager.shared.add(self)
    }

    func stop() {
        MXMetricManager.shared.remove(self)
    }

    func didReceive(_ payloads: [MXMetricPayload]) {
        // Persist or upload aggregated metrics for analysis.
    }

    func didReceive(_ payloads: [MXDiagnosticPayload]) {
        // Persist or upload diagnostics such as hangs or crashes.
    }
}
```

Do not block launch or the main thread while processing payloads. Serialize and upload later if necessary.

## Xcode Organizer and Production Monitoring

Use Organizer or production monitoring dashboards to identify patterns before deep local profiling.

Check:

- app version where regression started
- device families affected
- OS versions affected
- launch type or launch phase if available
- hang rate trends
- memory pressure trends
- animation responsiveness trends
- whether the issue is isolated to a feature rollout

Use production signals to choose the local scenario to reproduce.

Do not use aggregate production metrics as the only proof that a specific code refactor fixed the issue. Confirm with before/after local measurements when possible, then watch production trends after release.

## Screen Recordings

A screen recording is useful evidence for user-visible symptoms, but it rarely proves root cause.

From a screen recording, the agent may infer:

- where the symptom occurs
- whether it is during scrolling, navigation, typing, loading, or animation
- whether the issue looks like hitching, delayed input, blank content, repeated loading, or layout instability
- which scenario to profile

Do not infer exact frame time, CPU cost, memory pressure, or SwiftUI invalidation cause from a recording alone.

Use the recording to create a profiling scenario.

## Interpreting User-Provided Artifacts

When the user provides artifacts, analyze them directly and state what they prove.

For Instruments screenshots:

- identify selected time range
- identify selected instrument/lane
- read visible call tree entries
- distinguish app code from framework code
- ask for or suggest exporting the selected call tree if details are missing

For `.trace` files:

- try to inspect/export with `xctrace` if available
- if unavailable, provide local export steps
- prefer focused exports over huge raw dumps

For signpost logs:

- group intervals by operation
- compare min/median/max if multiple runs exist
- align intervals with user actions
- look for variance and outliers

For XCTest output:

- identify metric type
- compare baseline/current values
- check standard deviation or variance if available
- avoid overreacting to one noisy run

For MetricKit JSON:

- identify payload type
- inspect app version, device/OS dimensions if present
- distinguish metric payloads from diagnostic payloads
- look for repeated patterns rather than single isolated events

For memory graphs:

- identify retained roots and ownership paths
- distinguish expected retention from cycles
- check whether navigation away should have released the object

## Before/After Validation

A good validation loop:

1. Define one scenario.
2. Capture baseline.
3. Form one hypothesis.
4. Apply one targeted refactor.
5. Re-run the same scenario.
6. Compare the same metrics.
7. Keep or revert the change based on evidence.

Avoid changing many variables at once.

For before/after reports, include:

```md
## Scenario

## Baseline evidence

## Hypothesis

## Change made

## New evidence

## Interpretation

## Remaining risks
```

## Mapping Findings Back to SwiftUI Refactors

Use profiling results to choose targeted fixes.

| Evidence | Likely refactor |
|---|---|
| Long body updates with formatter samples | precompute formatted strings or cache formatter output |
| Many unrelated views update | narrow dependencies, split dependency islands, avoid broad model reads |
| Row updates during pagination append | preserve row identity, avoid rebuilding old rows, consider page/section boundaries |
| CPU spike in sorting/filtering | move transformation outside `body`, debounce search, cache derived data |
| Allocation spike on scroll | remove per-row allocation, formatter creation, type erasure, temporary arrays |
| Hitches during animation | reduce main-thread work, simplify layout/drawing, avoid state bursts during animation |
| Memory graph retains model after navigation | inspect tasks, closures, subscriptions, delegates, caches |
| Launch metric regression | inspect startup work, lazy initialization, synchronous I/O, main actor startup tasks |
| MetricKit hang trend | reproduce affected scenario locally, inspect main-thread stalls and hang diagnostics |

## Response Format for Profiling Help

When the user asks for profiling help, answer with:

1. Reproducible scenario
2. Hypothesis
3. Tool choice
4. Exact command or local steps when possible
5. What to look for
6. How to interpret findings
7. Refactor candidates
8. How to compare before and after

## Response Format for Artifact Analysis

When the user provides profiling artifacts, answer with:

1. What the artifact shows
2. What is measured vs inferred
3. Most likely bottleneck
4. Supporting evidence from the artifact
5. What remains uncertain
6. Suggested refactor or next diagnostic step
7. Validation plan

## Common Mistakes

Avoid these mistakes:

- profiling a vague scenario
- comparing Debug before with Release after
- using simulator-only results as final proof of device smoothness
- claiming SwiftUI diffing is the cause without trace evidence
- treating MetricKit as a local profiler
- treating screen recordings as CPU evidence
- treating `_printChanges()` output as production measurement
- adding signposts but forgetting to align them with profiler timelines
- reading only framework symbols and ignoring app code
- optimizing a microbenchmark while the real issue is layout, rendering, or scheduling
- changing architecture before confirming the hot path

## Final Principle

Profiling should make a SwiftUI hypothesis falsifiable.

Start with a concrete symptom, collect the smallest useful evidence, make one targeted change, and compare the same scenario before and after.
