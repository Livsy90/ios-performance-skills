---
name: ios-launch-performance
description: Use this skill when reviewing or diagnosing iOS app launch performance, including cold launch, warm launch, prewarmed launch, resume-vs-launch confusion, dyld and pre-main work, Objective-C +load/+initialize, static initializers, static vs dynamic linking, mergeable libraries, AppDelegate/SceneDelegate startup, SwiftUI App initialization, first-frame readiness, MetricKit, XCTest launch metrics, and Instruments App Launch traces. Trigger when reviewing AppDelegate, SceneDelegate, SwiftUI @main App, root view creation, dependency container setup, third-party SDK initialization, framework linking strategy, Objective-C runtime hooks, constructor functions, or slow launch symptoms caused by pre-main or main-thread startup work.
---

# iOS Launch Performance Expert

Use this skill to review the path from an app launch request to first visible UI and early responsiveness. Focus on work that happens before `main`, during application/scene startup, while creating the first frame, and during the short period where launch still affects user-perceived readiness.

This is not a general iOS performance skill. Use it only when the issue is launch-specific or when code runs on the launch path.

## Core Model

Treat launch as a pipeline with separate phases:

1. Process creation and system preparation
2. dyld loading, binding, fixups, runtime registration, and static initialization
3. UIKit or SwiftUI runtime startup
4. App-level initialization in `UIApplicationDelegate`, `UISceneDelegate`, or SwiftUI `App`
5. Root UI construction, layout, drawing, and first frame commit
6. Early post-launch work that affects responsiveness
7. Later feature-specific or maintenance work

Do not optimize blindly. First identify which phase is expensive, then recommend changes that move, remove, lazy-load, parallelize, or measure that specific work.

## Launch Scenario Classification

Before giving advice, classify what is being measured:

- **Cold launch**: the app process is not resident and little launch-related state is already warm.
- **Warm launch**: the app still starts a new process, but parts of system state, caches, or pages may already be warm.
- **Prewarmed launch**: the system may have prepared part of the launch path before the user explicitly opens the app.
- **Resume / already-running return**: the app process already exists and returns from background or suspension.
- **Unknown**: the measurement setup does not clearly separate the above cases.

Do not compare cold launch, warm launch, prewarmed launch, and resume as one metric. Resume is not a full launch investigation unless the task explicitly asks about foregrounding latency.

## Investigation Workflow

1. **Locate the launch path**
   - Identify code that executes before first frame or before first interaction.
   - Include app delegate, scene delegate, SwiftUI `App`, root view/model creation, dependency container setup, global/static initialization, SDK startup, and linked framework initialization.

2. **Classify the slow phase**
   - dyld/pre-main
   - Objective-C or Swift runtime/static initialization
   - app delegate or scene delegate startup
   - SwiftUI app/root view initialization
   - root UI construction and first frame
   - early post-launch responsiveness
   - measurement ambiguity

3. **Classify each startup task by necessity**
   - Required before first frame
   - Required before first interaction
   - Needed soon after launch
   - Needed only after authentication/session state is known
   - Needed only by a later feature
   - Background maintenance

4. **Look for hidden eager work**
   - Objective-C `+load`
   - C/C++ constructor functions
   - static/global initialization with side effects
   - eager singleton creation
   - dependency graph construction
   - dynamic framework startup cost
   - SDK initialization
   - synchronous file I/O
   - database opening or migration
   - keychain-heavy work
   - networking or remote configuration
   - large decoding/parsing
   - blocking locks or semaphores on the main thread

5. **Recommend targeted changes**
   - Remove unnecessary launch work.
   - Lazy-load feature-specific services.
   - Defer noncritical work until after visible UI or first interaction.
   - Move blocking work off the main thread only when it is safe and useful.
   - Split launch-critical state from secondary state.
   - Replace hidden static initialization with explicit or lazy initialization.
   - Review linking strategy only when evidence points to pre-main or dyld cost.

6. **Require validation**
   - Use Instruments for local diagnosis.
   - Use XCTest launch metrics for repeatable regression checks.
   - Use MetricKit or Xcode Organizer for production distributions.
   - Prefer release-like builds, real devices, stable test data, and older supported hardware.

## Decision Rules

- Treat roughly 400 ms to first visible frame as an aggressive user-experience target, not as a watchdog threshold and not as a pre-main-only budget.
- Do not blame dyld by default. Slow launch can come from pre-main work, app initialization, root UI creation, first-frame rendering, synchronous I/O, or early post-launch blocking.
- Treat `+load`, constructor functions, and static initialization with side effects as launch-critical until measurement proves otherwise.
- Prefer explicit setup, lazy initialization, or scoped one-time initialization over work hidden in load-time hooks.
- Do not present `+initialize` as a universal fix. It can be useful in legacy Objective-C code, but explicit or lazy initialization is usually clearer in modern code.
- Do not recommend converting all modules to static linking. Consider launch cost, build time, binary size, duplicate symbols, resource packaging, SDK distribution, debugging, and mergeable libraries.
- Do not use arbitrary framework-count limits. More dynamic frameworks can increase launch work, but the real cost must be measured.
- Do not assume that queueing work asynchronously on the main queue guarantees it runs after the first frame. Verify with Instruments, signposts, lifecycle points, or other measurement.
- Do not assume SwiftUI `.task` is always post-render or harmless. It is lifecycle-bound async work and can still affect early responsiveness.
- Do not rely on Debug builds, simulator-only runs, or a single modern device when judging launch performance.
- Do not use production launch histograms alone to identify the local bottleneck. Use them to prioritize and verify trends.

## Code Review Focus Areas

### Pre-main and runtime initialization

Check for:

- Objective-C classes or categories with `+load`
- constructor attributes
- C++ global objects with constructors
- Swift globals or static properties that perform expensive work when first touched during launch
- SDKs that register runtime hooks eagerly
- many dynamic frameworks with startup-time initializers
- ObjC category-heavy modules or libraries

Recommended direction:

- Keep unavoidable load-time work tiny and deterministic.
- Remove I/O, networking, locks, large allocations, decoding, and dependency setup from load-time paths.
- Move registration to explicit startup points only when truly required before first frame.
- Prefer lazy registration when feature use can trigger setup safely.

### AppDelegate, SceneDelegate, and SwiftUI App startup

Check for:

- large dependency containers built synchronously
- analytics, attribution, remote config, ads, push, feature flags, or experimentation SDKs initialized unconditionally
- database opening or migration
- keychain access on the critical path
- synchronous network calls or reachability waits
- heavy root view model construction
- duplicate root UI setup between app and scene lifecycle code
- work performed in SwiftUI `App.init`, root view `init`, or eager environment setup

Recommended direction:

- Keep the first frame minimal but valid.
- Separate launch-critical state from secondary app state.
- Defer nonvisual services unless they are required for correctness, crash reporting, routing, security, or first-screen behavior.
- Use lifecycle or explicit readiness points for work that must happen after UI appears.

### First frame and early responsiveness

Check for:

- heavy initial layout or view hierarchy construction
- expensive SwiftUI body dependencies during root view creation
- synchronous image loading or decoding
- immediate rendering of large data sets
- blocking state restoration
- first-screen data requirements that could use placeholders or cached state

Recommended direction:

- Render a lightweight first screen.
- Prefer cached, placeholder, skeleton, or progressive content when appropriate.
- Delay nonessential first-screen enrichment until after visible UI.
- Measure whether deferral improves first frame without causing interaction jank.

## Measurement Guidance

Use multiple layers of evidence:

- **Instruments App Launch** for phase-level diagnosis and call-tree inspection.
- **Time Profiler** for main-thread and CPU-heavy startup paths.
- **dyld-related instruments or logs** when pre-main work is suspected.
- **os_signpost** or equivalent markers for app-specific startup phases.
- **XCTest launch metrics** for local regression detection and CI baselines.
- **MetricKit / Xcode Organizer** for production distributions across devices, OS versions, and app versions.

When measurements are noisy, clarify:

- device model and OS version
- Debug vs Release/Profile configuration
- simulator vs physical device
- cold/warm/prewarmed/resume classification
- network and account state
- app data size
- first install vs returning user
- authenticated vs unauthenticated launch
- test iteration count and variance

## Third-Party SDK Policy

Every SDK that starts during launch must justify why it needs to run before first frame or first interaction.

Classify SDKs as:

- launch-critical and synchronous
- launch-critical but reducible
- first-interaction required
- post-first-frame acceptable
- feature-specific and lazy
- background-only

Be careful with blanket deferral. Crash reporting, security, deep linking, attribution, push routing, remote config, and feature flags can have correctness requirements. Prefer vendor-supported deferred modes, minimal startup modes, lazy modules, or narrower initialization when available.

## SwiftUI-Specific Guidance

When reviewing SwiftUI app launch, inspect:

- `@main App` initialization
- root `Scene` construction
- root view initialization
- eager `@StateObject` or observable model creation
- environment injection
- synchronous work inside view/model initializers
- root `.task` or `.onAppear` work that starts immediately
- large dependency graphs captured by root views

Keep the launch-time SwiftUI tree small. Move work out of root initialization unless it is required to decide the first visible UI. Treat lifecycle-triggered async work as early startup work until measurement proves it does not affect first-frame readiness or first interaction.

## Reference Routing

Use the reference files only when the task needs extra detail:

- Read `references/launch-taxonomy-and-targets.md` for cold/warm/prewarmed/resume terminology, first-frame targets, and measurement setup.
- Read `references/pre-main-dyld-and-static-initializers.md` for dyld, pre-main, `+load`, `+initialize`, constructor functions, Objective-C categories, runtime registration, and static initialization.
- Read `references/linking-strategy.md` for dynamic frameworks, static libraries, mergeable libraries, modularization, and launch-time linking trade-offs.
- Read `references/appdelegate-scenedelegate-and-first-frame.md` for `UIApplicationDelegate`, `UISceneDelegate`, dependency setup, root UI creation, first-frame readiness, and main-thread startup work.
- Read `references/swiftui-app-launch.md` for SwiftUI `App`, root view setup, observable state, `.task`, `.onAppear`, and environment initialization.
- Read `references/third-party-sdks-at-launch.md` for analytics, crash reporting, ads, attribution, remote config, push, feature flags, security SDKs, and vendor initialization strategy.
- Read `references/metrics-instruments-xctest-metrickit.md` for Instruments, Time Profiler, signposts, XCTest launch metrics, MetricKit, Xcode Organizer, CI baselines, and production monitoring.

## Output Format

When reviewing code or diagnosing a report, return:

### Launch classification

Cold launch, warm launch, prewarmed launch, resume, or unknown. Mention measurement ambiguity if present.

### Suspected phase

Identify the most likely phase: pre-main, static initialization, app delegate, scene setup, SwiftUI root setup, first-frame rendering, early post-launch responsiveness, or measurement/setup issue.

### Findings

List concrete risks found in the code, architecture, metrics, or trace. Avoid generic advice not connected to the evidence.

### Recommended changes

Group recommendations by priority:

1. High-confidence launch-path fixes
2. Measurement or instrumentation needed
3. Optional architectural cleanup

### Validation

Explain how to verify the change locally and in production. Include the expected metric or trace area that should improve.

## Non-Goals

Do not use this skill for general scrolling performance, memory leaks, rendering optimization unrelated to app startup, networking performance outside launch, or broad app architecture review unless the code is on the launch path.
