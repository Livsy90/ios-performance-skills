---

name: ios-performance-profiling
description: Diagnose iOS performance problems with a profiling-first workflow. Use for launch time, hangs, hitches, scrolling, SwiftUI updates, CPU, memory, leaks, disk I/O, networking, power, MetricKit, XCTest performance tests, and choosing the right Instruments template.
---

# iOS Performance Profiling

## Principle: Evidence Before Optimization

Treat performance work as an investigation, not a guessing exercise.

Follow this loop:

```text
Symptom → reproducible scenario → correct tool → trace or metric → hypothesis → focused fix → re-measure
```

Do not recommend broad rewrites before identifying the measured bottleneck. Code can suggest risks, but traces and metrics confirm cost.

Prefer profiling on:

* a real device
* a Release or release-like build
* realistic data size
* the oldest supported device when possible
* repeated runs, not a single sample

Use the simulator for functional debugging, not final performance conclusions.

## When to use this skill

Use this skill when the user asks about:

* slow launch
* slow screen loading
* UI freezes
* main thread hangs
* dropped frames
* scroll stutter
* SwiftUI over-updates
* expensive `body` recomputation
* high CPU usage
* memory growth
* leaks or retain cycles
* disk writes
* network latency
* battery drain
* MetricKit payloads
* Xcode Organizer metrics
* XCTest performance regressions
* Instruments traces
* choosing the right profiling tool

## When not to use this skill

Do not use this skill for general architecture review unless there is a performance symptom.

Do not use it for speculative micro-optimization without a measurable problem.

Do not claim that a change improves performance unless the reasoning is tied to a trace, metric, reproducible scenario, or explicit verification step.

## Tool Selection Guide

| Symptom                           | Start with                         | Then check                                 |
| --------------------------------- | ---------------------------------- | ------------------------------------------ |
| Slow launch                       | App Launch / Time Profiler         | Organizer, MetricKit, XCTest launch metric |
| App freezes after interaction     | Hangs                              | Time Profiler, MetricKit diagnostics       |
| Scroll stutter or animation hitch | Animation Hitches / Core Animation | Time Profiler                              |
| SwiftUI updates too much          | SwiftUI Instrument                 | Time Profiler, signposts                   |
| High CPU                          | Time Profiler                      | CPU Profiler, signposts                    |
| Memory keeps growing              | Allocations                        | Memory Graph, VM Tracker                   |
| Object does not deallocate        | Memory Graph Debugger              | Allocations, Leaks                         |
| Slow networking                   | Network / HTTP Traffic             | URLSessionTaskMetrics, signposts           |
| Excessive disk writes             | File Activity                      | Organizer, MetricKit                       |
| Battery drain                     | Power Profiler                     | CPU, Network, Location, wakeups            |
| Async tasks pile up               | Swift Concurrency                  | Hangs, Time Profiler                       |
| Actor contention                  | Swift Concurrency                  | Time Profiler                              |
| CI performance regression         | XCTest metrics                     | Saved traces, production metrics           |

## Diagnostic Workflow

### 1. Define the user-visible symptom

Start by naming the problem in user terms.

Good:

```text
The feed starts scrolling smoothly, then stutters when new images arrive.
```

Weak:

```text
The app is slow.
```

Identify:

* affected screen or flow
* user action that triggers the issue
* device and iOS version if known
* Debug vs Release build
* local-only issue or production issue
* expected metric if available

### 2. Choose one primary profiling path

Pick the tool based on the symptom.

Examples:

```text
Tap causes a visible freeze → Hangs first, then Time Profiler.
```

```text
Memory increases after opening and closing Profile screen → Allocations and Memory Graph.
```

```text
SwiftUI list refreshes too often → SwiftUI Instrument, then Time Profiler if work is expensive.
```

### 3. Create a repeatable scenario

Define the exact steps:

```text
1. Launch the app.
2. Open Search.
3. Type a query with at least 100 results.
4. Scroll from top to bottom twice.
5. Stop recording.
```

Keep data, account state, network, and device as stable as possible.

### 4. Add semantic signposts when code is available

Use signposts around business operations so Instruments can align app-level events with system activity.

Good signpost names:

* `Home Initial Load`
* `Search Results Mapping`
* `Checkout Confirmation`
* `Image Decode Batch`
* `Database Save Transaction`
* `Feed Diff Preparation`

Avoid vague names like `Start`, `Step 1`, or `Work`.

Example:

```swift
import OSLog

private let perfLogger = Logger(
    subsystem: "com.example.shop",
    category: "Performance"
)

private let signposter = OSSignposter(logger: perfLogger)

func prepareSearchResults(_ items: [SearchItem]) -> [SearchRowModel] {
    let id = signposter.makeSignpostID()
    let state = signposter.beginInterval("Search Results Mapping", id: id)

    let rows = items.map {
        SearchRowModel(
            title: $0.title,
            subtitle: $0.categoryName,
            priceText: PriceFormatter.shared.string(from: $0.price)
        )
    }

    signposter.endInterval("Search Results Mapping", state)
    return rows
}
```

### 5. Form one hypothesis at a time

A good hypothesis is specific:

```text
The hang is likely caused by synchronous image decoding on the main thread during row configuration.
```

Avoid vague conclusions:

```text
The screen is badly optimized.
```

### 6. Fix one root cause, then re-measure

Prefer narrow changes:

* move CPU-heavy work out of the frame path
* reduce main-thread work
* cache repeated formatting
* stabilize list identity
* precompute display models
* batch disk writes
* avoid duplicate network requests
* bound memory caches
* cancel unnecessary tasks
* reduce actor contention

After the fix, run the same scenario again.

## Time Profiler: CPU and Main Thread Work

### Use when

* CPU is high
* the main thread is busy
* interactions feel delayed
* rendering path performs too much work
* a synchronous operation is suspected

### Inspect

* main thread during UI problems
* hottest app-owned functions
* repeated work inside render paths
* expensive parsing, formatting, sorting, or mapping
* image decoding
* layout and cell configuration
* lock contention
* unnecessary allocations in hot paths

### Reading tips

* Separate work by thread.
* Invert the call tree to find leaf work.
* Hide system libraries when looking for app-owned code.
* Use signposts to focus on the affected flow.

### Common findings

* JSON decoding on the main thread
* date or number formatting in repeated rows
* image decompression during scroll
* sorting or filtering inside SwiftUI `body`
* large model-to-view-model mapping during interaction
* expensive layout triggered repeatedly

### Fix direction

Reduce work on the main thread, move CPU-heavy operations away from frame-sensitive paths, cache stable results, and verify with another trace.

## Hangs: Main Thread Freezes

### Use when

* the app becomes unresponsive
* a tap does not produce feedback quickly
* production diagnostics report hangs
* users describe freezes

### Inspect

* main thread stack during the hang
* synchronous I/O
* blocking locks
* dispatching synchronously to the main queue
* long layout passes
* heavy work isolated to `MainActor`
* actor or task waits that block user progress

### Classify

* busy main thread
* blocked main thread
* lock contention
* actor contention
* async operation that never resumes
* thread pool saturation

### Fix direction

Remove blocking work from the main thread, keep `MainActor` sections small, avoid synchronous I/O in interactions, and verify hang duration before and after.

## Animation Hitches and Rendering

### Use when

* scrolling stutters
* animations drop frames
* gestures feel uneven
* UI is smooth in simple cases but slow with real data

### Inspect

* expensive row or cell setup
* image decoding during scroll
* unstable identity
* repeated layout work
* masks, shadows, blur, transparency, and offscreen rendering
* work that happens inside one frame budget
* frequent SwiftUI invalidations

### Frame budget reminder

At 60Hz, a frame is about 16.7ms.
At 120Hz, a frame is about 8.3ms.

The app does not own the whole budget. Keep frame-path work small.

### Fix direction

Precompute display data, cache decoded or resized images, reduce layout churn, stabilize identity, avoid unnecessary rendering effects, and re-profile the same scroll scenario.

## SwiftUI Instrument: View Updates and Invalidation

### Use when

* SwiftUI views update more often than expected
* a parent state change refreshes too much UI
* `List`, `ForEach`, or lazy containers feel slow
* representable views update frequently
* `body` contains non-trivial work

### Inspect

* update groups
* long body updates
* broad observable state
* environment changes that invalidate large subtrees
* unstable identifiers
* computed properties used by `body`
* formatting, sorting, filtering, mapping, or image work inside `body`
* closures created inside repeated child views
* expensive `UIViewRepresentable` / `UIViewControllerRepresentable` updates

### Fix direction

Narrow state observation, stabilize identity, move expensive transformations out of `body`, precompute row models, keep representable updates minimal, and split views only when it reduces dependencies or improves clarity.

Do not split SwiftUI views mechanically. Smaller files do not automatically mean smaller invalidation.

## Swift Concurrency Instrument

### Use when

* async operations are slow
* tasks stay enqueued for too long
* actors have long queues
* cancellation does not happen
* `MainActor` is overloaded
* concurrency code appears to block UI

### Inspect

* task lifetime
* task cancellation
* actor queues
* work accidentally running on `MainActor`
* unstructured tasks
* detached tasks
* blocking calls inside async functions
* excessive parallelism
* priority inversions

### Fix direction

Keep UI isolation small, move CPU work away from `MainActor`, prefer structured concurrency, add cancellation checks, limit parallelism when necessary, and reduce actor contention.

## Allocations: Memory Growth

### Use when

* memory increases after repeated actions
* memory does not return after leaving a screen
* caches grow without bounds
* large temporary objects cause spikes

### Inspect

* persistent allocations
* transient allocation churn
* large arrays or dictionaries
* decoded images
* cache size
* repeated allocation in hot paths
* objects retained after a flow ends

### Generation-based check

Use a repeated navigation scenario:

```text
1. Start from a stable baseline screen.
2. Mark a memory generation.
3. Open the target screen.
4. Leave the target screen.
5. Mark another generation.
6. Inspect objects that remain alive unexpectedly.
```

Repeat the flow several times. Persistent growth across repetitions is a stronger signal than a single run.

### Fix direction

Release ownership cycles, bound caches, reduce decoded image size, reuse expensive objects carefully, and avoid repeated allocations inside render loops.

## Memory Graph: Retain Cycles and Logical Leaks

### Use when

An object should be gone but is still alive.

Typical cases:

* view controller remains after dismissal
* view model remains after screen closure
* subscription retains owner
* closure captures owner
* delegate is strong
* async sequence or task keeps an object alive

### Inspect

* expected class name
* retaining path
* closure captures
* delegate ownership
* notification observers
* Combine subscriptions
* async tasks and streams
* static containers

### Important distinction

Leaks tools help with unreachable leaked allocations.

Memory Graph is usually better for retain cycles and logical leaks where the object is still referenced, but should no longer be part of the app state.

### Fix direction

Break the ownership chain. Prefer fixing the actual retaining path over adding weak references everywhere.

## Network and HTTP Traffic

### Use when

* screen loading is slow
* requests are duplicated
* cache is not used
* images arrive late
* backend latency dominates the flow

### Inspect

* request waterfall
* duplicate in-flight requests
* DNS and TLS cost
* time to first byte
* payload size
* cache headers
* retry behavior
* sequential requests that could be parallel
* low-priority requests blocking visible content

### Fix direction

Reduce round trips, reuse connections, cache appropriately, avoid duplicate requests, prioritize visible content, reduce payload size, and render progressively when possible.

## Disk I/O

### Use when

* production metrics show excessive writes
* launch reads too much from disk
* UI blocks during persistence
* logs or caches are written too frequently

### Inspect

* synchronous reads or writes on main
* repeated full-file rewrites
* database transaction frequency
* excessive logging
* cache write amplification
* temporary files
* serialization cost

### Fix direction

Batch writes, use transactions, write only changed data, reduce logging, move I/O off the main thread, and avoid disk access on the launch-critical path.

## Power Profiling

### Use when

* battery usage is high
* device heats up
* background work continues too long
* app polls frequently
* location, Bluetooth, sensors, network, or animation work is continuous

### Inspect

* timers
* wakeups
* polling loops
* background tasks
* location accuracy and frequency
* repeated small network requests
* animations running offscreen
* CPU work after UI disappears

### Fix direction

Prefer event-driven updates, reduce timer frequency, stop offscreen work, batch network and disk operations, cancel unnecessary tasks, and lower sensor/location precision when possible.

## Launch Profiling

### Use when

* cold launch is slow
* warm launch regressed
* first content appears late
* app does too much before interaction

### Inspect

* pre-main work
* dynamic library loading
* static initialization
* dependency graph creation
* synchronous disk reads
* database setup
* feature flag loading
* remote config
* first screen construction
* unnecessary network dependency before first render

### Fix direction

Delay non-critical work, reduce startup dependencies, move work after first content, parallelize independent operations, and guard launch time with XCTest or production metrics.

## XCTest Performance Tests

Use XCTest performance tests for regression protection, not deep diagnosis.

Good candidates:

* launch time
* model mapping
* JSON decoding
* database queries
* diff generation
* image processing
* critical screen setup with mocked data

Example:

```swift
import XCTest

final class CatalogPerformanceTests: XCTestCase {
    func testCatalogRowModelCreation() {
        let products = ProductFixtures.makeProducts(count: 2_000)

        measure(metrics: [
            XCTClockMetric(),
            XCTCPUMetric(),
            XCTMemoryMetric()
        ]) {
            _ = products.map(CatalogRowModel.init)
        }
    }
}
```

For launch:

```swift
func testColdLaunchPerformance() {
    measure(metrics: [XCTApplicationLaunchMetric()]) {
        let app = XCUIApplication()
        app.launchArguments = ["--performance-test"]
        app.launch()
    }
}
```

Keep inputs stable. Avoid real network calls in CI performance tests.

## MetricKit and Organizer

Use production metrics when the issue may only appear in the wild.

Use them to answer:

* Did a release regress?
* Which devices are affected?
* Is the issue frequent or rare?
* Are hangs, memory, launch, or disk writes worse in production?
* Is the problem visible at p95 or only in averages?

MetricKit and Organizer identify production signals. Instruments explains local causes.

Do not replace local profiling with production metrics. Use both.

## Output Format

For most tasks, respond with:

```text
## Symptom

...

## Profiling path

Primary:
Secondary:

## What to inspect

...

## Likely hypotheses

...

## Suggested fixes

...

## Verification

...
```

For code reviews, respond with:

```text
## Summary

...

## Findings

### 1. Finding title

Risk:
Why it matters:
Suggested change:
How to verify:

## Profiling checklist

...
```

For traces, reports, or screenshots, respond with:

```text
## What the data shows

...

## Strongest signal

...

## Likely cause

...

## What is not proven yet

...

## Next step

...

## Verification

...
```

## Review Checklist

Use this checklist before finalizing a performance answer:

* Did you identify the user-visible symptom?
* Did you choose the profiling tool based on the symptom?
* Did you avoid claiming certainty without evidence?
* Did you separate local traces from production metrics?
* Did you recommend real-device Release profiling?
* Did you suggest signposts for app-specific operations when useful?
* Did you propose one focused fix at a time?
* Did you include a re-measurement step?
* Did you mention XCTest or production metrics for regression protection when appropriate?
* Did you avoid broad rewrites unless evidence supports them?
