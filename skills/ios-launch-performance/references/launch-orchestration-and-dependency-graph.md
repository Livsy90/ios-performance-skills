# Launch Orchestration and Dependency Graph

Use this reference when launch work has grown into many ordered startup tasks, fragile initialization sequences, unsafe parallelism, or unclear readiness rules.

This file is about organizing post-entry launch work. It should not explain launch taxonomy, dyld internals, framework linking strategy, tool setup, MetricKit APIs, or XCTest configuration. Use the other references for those topics.

## Purpose

Large iOS apps often accumulate launch work over time. What starts as a few setup calls can become a long sequence of services, SDKs, stores, routers, feature gates, session checks, and initial screen dependencies.

When that sequence is stored only as ordered code, comments, or team memory, it becomes hard to optimize safely. Engineers may know that one step must run before another, but the dependency is not represented anywhere the app can validate.

The goal is to turn launch from an implicit list into an explicit readiness model:

```text
startup work
→ dependency graph
→ critical path
→ safe scheduling
→ measurable readiness
```

## When to Use This Reference

Use this reference when reviewing:

- a long `AppDelegate`, `SceneDelegate`, SwiftUI `App`, or launch coordinator
- a startup sequence with many services or SDKs
- launch work currently protected by comments such as "do not reorder"
- an attempted parallel launch implementation
- startup crashes caused by reordered initialization
- launch work that blocks first frame or first interaction
- a slow first screen caused by full app-wide setup
- a launch system with unclear failure handling
- a proposal to move startup work to background queues, tasks, or workers

Do not use this reference only because launch is slow. First decide whether the problem is orchestration, dyld/pre-main, linking, first-frame rendering, SDK policy, or measurement setup.

## Core Model

Model launch work as a set of steps. Each step should have a reason to exist on the launch path.

For each step, identify:

- what it prepares
- whether it is required before first frame
- whether it is required before first interaction
- whether it can run after visible UI
- whether it can be lazy on first feature use
- which other steps it depends on
- which steps depend on it
- whether it must run on the main actor or main thread
- whether it touches shared mutable state
- how failure should affect app readiness
- how long it takes in representative measurements

The key output is not a new abstraction. The key output is clarity: which work is truly launch-critical, which work is merely early, and which work is historical baggage.

## Critical Path vs Secondary Path

Separate launch work by user-visible necessity.

### Critical Path

A step belongs to the critical path only when the app cannot produce a valid first frame or first interaction without it.

Typical critical-path candidates:

- deciding the initial route
- loading minimal session state
- preparing safety or correctness checks required before UI
- installing crash handling when required by product policy
- preparing the smallest state needed by the first visible screen

Do not mark a step critical only because it has always run during launch.

### Secondary Path

A step belongs to the secondary path when it is useful soon after launch but is not needed for the first frame or first interaction.

Common secondary-path candidates:

- nonessential analytics enrichment
- cache cleanup
- secondary feature preloading
- deferred SDK modules
- background synchronization
- warmups for screens not visible at launch
- optional personalization

Secondary work should not compete with first-frame or first-interaction work for the main thread, CPU, I/O, locks, or memory pressure.

### Lazy Path

A step belongs to the lazy path when only a later feature needs it.

Prefer lazy setup when:

- the first screen does not need the subsystem
- the user may never visit the feature in this session
- setup can be hidden behind the feature's own loading state
- partial app readiness is acceptable

Lazy setup is not a free optimization. It moves cost to a later interaction, so the feature must own its loading, failure, cancellation, and retry behavior.

## Dependency Graph

Represent startup dependencies explicitly.

A dependency graph answers:

- Which steps are ready to run now?
- Which steps are blocked?
- Which completed step unblocks other work?
- Which chain determines the minimum possible launch time?
- Which dependencies are real and which are accidental?

A launch step should not depend on another step because of ordering habit. It should depend on another step only when it consumes output, state, registration, configuration, authorization, or side effects produced by that step.

## Dependency Types

Classify dependencies before scheduling work.

### Data Dependency

One step needs data produced by another step.

Review questions:

- Can the consumer work with cached, partial, or stale data?
- Can the producer provide a lightweight initial value?
- Can the consumer request the data lazily later?

### Configuration Dependency

One step needs environment, flags, remote config, region, tenant, build channel, or experiment state.

Review questions:

- Which configuration is required for the first screen?
- Which configuration can be updated after launch?
- Can defaults unblock first frame safely?

### Registration Dependency

One step must register handlers, routes, observers, or integrations before another step can use them.

Review questions:

- Is registration required globally or only for a feature?
- Can registration move closer to feature activation?
- Is registration idempotent?

### Main-Thread Dependency

A step must run on the main thread or main actor because it touches UI, UIKit, SwiftUI state, or a main-thread-only API.

Review questions:

- Is the whole step main-thread-only, or only a small section?
- Can preparation run off the main thread and commit a small result on the main thread?
- Does this step block UI creation or first interaction?

### Side-Effect Dependency

One step relies on another step's side effects rather than returned data.

Review questions:

- Can the side effect become explicit state?
- Can the dependency be expressed as a named readiness condition?
- Is the side effect safe to repeat, cancel, or retry?

### Failure Dependency

One step changes how launch should continue if another step fails.

Review questions:

- Should failure block launch?
- Should failure degrade the first screen?
- Should failure trigger retry in the background?
- Should failure be surfaced to the user?

## Scheduling Policy

Do not introduce parallelism until dependencies are explicit.

A safe scheduler should respect:

- dependency completion
- priority
- main-thread requirements
- resource limits
- cancellation
- failure policy
- duplicate prevention
- deterministic readiness gates

Parallel launch work is useful only when independent steps exist. If the graph is mostly one long chain, adding workers will not help much. In that case, optimize the chain itself or reduce what the first screen requires.

## Bounded Parallelism

Avoid unbounded startup work. Launch is already CPU, I/O, and memory sensitive.

Bounded parallelism means:

- do not start every pending step at once
- keep the main thread available for UI work
- avoid saturating storage with many competing reads
- avoid many services contending for the same locks
- avoid heavy network, disk, and CPU warmups competing at the same time
- prefer small, measurable batches over uncontrolled fan-out

The optimal worker count is not a fixed rule. It depends on device class, work type, thread affinity, I/O pressure, thermal state, and what the first frame needs.

## Main-Thread Blocking Risks

Be very cautious with designs where the main thread waits for parallel startup work to complete.

This pattern can improve a benchmark in some cases but carries several risks:

- the main thread is unavailable for UI creation
- one launch step may need the main thread and deadlock
- old devices have fewer cores to compensate for the blocked main thread
- a slow background step can still delay the whole launch
- error handling often becomes harder to reason about

If the app must wait before showing UI, keep the waiting boundary minimal and explicit. Prefer showing a small valid UI while secondary readiness continues when product behavior allows it.

## Race Conditions and Ordering Bugs

Unsafe parallelism often exposes dependencies that were previously hidden by serial ordering.

Common symptoms:

- startup crashes only on some devices
- first launch fails but second launch succeeds
- flags or configuration sometimes appear unavailable
- database or storage setup races with consumers
- analytics, routing, or push handlers miss early events
- state restoration depends on services that are not ready
- errors disappear when logging or breakpoints are added

When these symptoms appear, do not solve them by returning to a giant serial list by default. First identify the missing dependency and encode it explicitly.

## Cycle Detection

A launch dependency graph must not contain cycles.

A cycle means that step A waits for step B, while step B directly or indirectly waits for step A. Cycles usually reveal one of these design problems:

- two services initialize each other
- configuration and dependency injection are interleaved too tightly
- a feature module requires app-wide setup too early
- a shared singleton hides a dependency that should be explicit
- a first-screen dependency actually belongs to a later feature

When a cycle appears, break it by extracting a smaller readiness condition, using a lightweight placeholder state, or moving a feature-specific dependency out of the launch-critical path.

## Longest Dependency Chain

After the graph is explicit, optimize the longest chain rather than only the largest individual step.

A single expensive step is easy to notice, but a sequence of medium-cost dependent steps can define the minimum possible launch time.

For the longest chain, ask:

- Can any dependency be weakened from required to optional?
- Can a producer emit a minimal initial result earlier?
- Can a consumer accept partial state?
- Can the first screen avoid this chain entirely?
- Can the chain be split into first-frame and post-interaction readiness?
- Can a later feature own the cost instead of global launch?

If the longest chain cannot be parallelized, reduce the first screen's dependency on that chain.

## Partial Readiness

A mature app does not always need full readiness before showing the first screen.

Useful readiness levels:

- app can draw the initial shell
- app can show cached or placeholder first-screen content
- app can respond to basic navigation
- app can complete the first intended interaction
- all feature modules are ready
- background maintenance is complete

Partial readiness requires explicit product behavior. The UI must know what is available, what is still loading, what is disabled temporarily, and how failures are handled.

Do not hide incomplete readiness behind a UI that appears interactive but blocks or fails on first touch.

## Failure Policy

Each launch step should define how failure affects startup.

Classify failures as:

- fatal for app launch
- fatal for authenticated area but not for logged-out UI
- blocking for the first screen only
- degradable with cached/default state
- retryable in the background
- feature-local and not launch-blocking

A launch orchestrator without failure policy often becomes either too fragile or too permissive. It may crash for recoverable issues or silently continue after a critical invariant is broken.

## Cancellation and Timeout Policy

Launch work should not assume unlimited time.

Review whether steps need:

- cancellation when the app moves to background
- timeout when waiting for network, disk, or another subsystem
- retry with backoff
- fallback state
- telemetry for timeout and degraded readiness

Avoid making first frame depend on open-ended network or server behavior.

## Observability

A dependency graph is useful only if the app can explain what happened.

For each significant step or group, collect enough information to answer:

- when it became eligible to run
- when it started
- when it finished
- whether it ran on the main thread or background executor
- whether it waited on dependencies
- whether it failed, timed out, or degraded
- which readiness level it unlocked

Prefer signposts or structured startup events over ad hoc print statements. The goal is to reconstruct the launch timeline and the dependency chain during local profiling and production regression analysis.

Tool-specific guidance belongs in `metrics-instruments-xctest-metrickit.md`.

## Review Checklist

When reviewing a launch orchestration system, check:

- Are launch steps named by responsibility rather than implementation detail?
- Does each step declare why it belongs on the launch path?
- Are dependencies explicit and minimal?
- Are main-thread requirements explicit?
- Are side effects documented as readiness conditions?
- Are failure, timeout, and retry rules defined?
- Can independent work run without blocking first frame?
- Is parallelism bounded?
- Are cycles impossible or detected?
- Is the longest dependency chain visible in measurement?
- Can the first screen render with partial readiness?
- Can later features own their own setup instead of forcing global launch work?

## Agent Guidance

When applying this reference, avoid generic advice such as "run it in parallel" or "move it to a background queue." Instead, produce a graph-oriented review:

```markdown
### Launch graph assessment
Describe whether startup is serial, partially parallel, dependency-driven, or unclear.

### Critical path
Name the work that appears required before first frame or first interaction.

### Hidden dependencies
List dependencies currently implied by ordering, shared state, or side effects.

### Parallelism safety
Explain what can run independently and what must remain ordered.

### Longest chain
Identify the chain most likely to determine launch duration.

### Recommended changes
Suggest changes that reduce launch-critical work, make dependencies explicit, or move feature-specific work out of global startup.

### Validation
Explain how to verify correctness and performance after changing orchestration.
```

## Boundary With Other References

This reference should answer:

- How should startup work be organized?
- Which steps are truly critical?
- Which dependencies are hidden?
- Is parallelism safe?
- What is the longest launch dependency chain?
- Can the app show a useful first screen before full readiness?

This reference should not answer:

- What is cold launch vs warm launch?
- What happens in dyld before `main`?
- Should a module be static, dynamic, or mergeable?
- Which SDKs should start before first frame?
- How should `XCTApplicationLaunchMetric` be configured?
- How should MetricKit histograms be interpreted?
- How should the Instruments App Launch template be used?
