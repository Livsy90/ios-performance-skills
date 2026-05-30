# Launch Taxonomy and Targets

Use this reference when the task depends on launch terminology, measurement scope, target interpretation, or whether the reported number actually represents app launch.

This file should stay focused on classification and targets. For implementation details, use the other references:

- Use `pre-main-dyld-and-static-initializers.md` for dyld, runtime setup, `+load`, constructors, and static initialization.
- Use `linking-strategy.md` for static, dynamic, and mergeable library trade-offs.
- Use `appdelegate-scenedelegate-and-first-frame.md` for app lifecycle code, root UI creation, and first-frame work.
- Use `swiftui-app-launch.md` for SwiftUI `App`, root view, observable state, and lifecycle-triggered async work.
- Use `third-party-sdks-at-launch.md` for SDK startup policy.
- Use `metrics-instruments-xctest-metrickit.md` for tool-specific instructions, APIs, CI baselines, and production monitoring.

## Core Definition

Treat app launch as the user-visible transition from an external launch request to the first useful visible UI and early responsiveness.

For measurement purposes, the most common launch boundary is:

```text
user opens the app
→ system and process startup
→ runtime and app initialization
→ first frame is drawn
→ optional extended launch work
→ app becomes responsive enough for the intended first interaction
```

The launch screen storyboard or static launch image is not the app's first rendered frame. It is a system-provided placeholder while the app is still starting.

## Launch Scenarios

Classify the scenario before diagnosing or comparing numbers.

### Cold Launch

Use **cold launch** when the app process is not already running and little launch-related state is warm.

Common cases:

- after device reboot
- after the app has not been launched for a long time
- after the app was terminated and system caches are not helpful
- after install or update, depending on the test setup

Cold launch is the worst-case user experience, but it can be noisy. It is useful for validating the app under unfavorable conditions and for catching launch work that is hidden by warm caches.

### Warm Launch

Use **warm launch** when the app still starts a new process, but parts of the system, binary pages, services, or dependency state may already be warm.

Common cases:

- the app was recently terminated and launched again
- repeated XCTest launch iterations
- repeated manual launches during local profiling

Warm launch is often more repeatable than cold launch. It is useful for regression testing, but it must not be reported as cold-launch performance.

### Prewarmed Launch

Use **prewarmed launch** when iOS has performed part of the launch preparation before the user explicitly opens the app.

Prewarming can make manual timing misleading because some early work may happen before the point where custom app timing begins. Do not infer pre-main cost from app-level timestamps alone when prewarming may be involved.

When prewarming is possible, prefer system tools and production metrics that understand launch phases, or clearly label custom measurements as app-defined intervals rather than full launch duration.

### Resume / Already-Running Return

Use **resume** or **already-running return** when the app process already exists and the user returns from the Home Screen, app switcher, notification, universal link, or another foreground transition.

This is not a full launch. It does not run the same process creation, dyld, static initialization, or `didFinishLaunching` path as a new process launch.

Analyze resume separately when the user asks about foregrounding latency, app switcher return, scene activation, stale UI, authentication refresh, or background-to-foreground responsiveness.

Avoid using **hot launch** as a primary category. If the user says "hot launch," clarify whether they mean warm launch of a new process or resume of an already-running process.

### Unknown Scenario

Use **unknown** when the report does not say how the app was launched, whether the process was already running, which device was used, or whether the number came from Instruments, XCTest, MetricKit, Organizer, custom logging, or manual timing.

In this case, do not make strong conclusions. First normalize the measurement setup or give advice that is explicitly conditional.

## What to Compare

Only compare numbers that represent the same scenario and boundary.

Good comparisons:

- cold launch before vs cold launch after on the same device class
- warm XCTest launch baseline vs warm XCTest launch after a change
- MetricKit launch distribution for app version A vs app version B
- p50/p90/p95 for the same metric, same launch scenario, and comparable audience

Bad comparisons:

- cold launch vs warm launch
- launch vs resume
- debug simulator measurement vs release device measurement
- high-end device measurement vs oldest supported device measurement
- custom `main`-to-first-screen logging vs MetricKit time-to-first-draw
- one manual run vs a distribution of repeated measurements

If the user provides a single number without context, treat it as a symptom, not evidence.

## Targets and Budgets

### First-Frame Target

Use roughly **400 ms to first visible app frame** as an aggressive user-experience target, not as a universal correctness rule.

Interpretation:

- The goal is for the app to show its first app-rendered frame during the system launch animation.
- The first frame can be minimal, cached, placeholder-based, or partially loaded.
- The first frame should still be valid and useful enough that the app can become responsive quickly.
- The target is not a watchdog termination threshold.
- The target is not a pre-main-only budget.

If pre-main alone approaches or exceeds this budget, treat it as a strong signal that launch-critical work is happening too early, but do not label the budget as "pre-main must be under 400 ms." The end-to-end launch experience matters.

### First Frame vs Final Frame

Do not require the first frame to contain all final data.

A good launch can render:

- cached content
- placeholders
- skeleton UI
- partial data
- last-known authenticated state
- a lightweight routing state

Then it can complete secondary work after the first frame, as long as the app remains responsive and does not immediately block the user.

### Extended Launch

Some apps have work that starts at launch and finishes after the first frame. Treat this as **extended launch** when it still affects early usability.

Examples:

- loading enough data for the first meaningful interaction
- resolving initial navigation or deep link state
- refreshing session state needed by the first screen
- finishing initial database access needed by visible UI

Do not hide important launch latency by moving everything after first draw. If post-first-frame work blocks interaction, it is still part of the user-perceived startup experience.

### Responsive Launch

A launch is not successful just because the first frame appeared.

Check whether the app can respond to the first expected interaction shortly after the launch animation completes. If the first frame appears quickly but the main thread remains blocked, classify the issue as early post-launch responsiveness rather than solved launch performance.

## Measurement Setup Checklist

Use this checklist only to classify and normalize launch measurements. Tool-specific workflows belong in `metrics-instruments-xctest-metrickit.md`.

Before accepting a launch number, identify:

- device model
- iOS version
- app version and build configuration
- Debug, Release, or Profile build
- simulator or physical device
- cold, warm, prewarmed, resume, or unknown scenario
- fresh install, returning user, or upgraded app
- authenticated or unauthenticated state
- app data size and account state
- network state and server dependency
- number of runs and variance
- measurement source: Instruments, XCTest, MetricKit, Organizer, custom logging, or manual timing

Prefer release-like builds on physical devices. Include older supported devices when judging user impact, because launch behavior can differ significantly across CPU, memory, storage, OS version, and app data size.

## Reporting Guidance for the Agent

When using this reference, include a short launch classification before recommendations.

Use this format when helpful:

```markdown
### Launch classification
Cold launch / warm launch / prewarmed launch / resume / unknown.

### Measurement boundary
What the number appears to measure: app icon to first draw, process start to first draw, `main` to first screen, first draw to responsiveness, or unknown.

### Target interpretation
Whether the number should be judged against first-frame, extended launch, or resume expectations.

### Caution
Any reason the number may not be comparable: build type, device, data set, prewarming, simulator, one-off run, or mixed launch scenarios.
```

## Common Mistakes

- Treating resume as launch.
- Calling warm launch "cold" because the app was manually killed.
- Treating the launch screen storyboard as the first app frame.
- Using 400 ms as a watchdog threshold.
- Using 400 ms as a pre-main-only budget.
- Optimizing to first draw while leaving the app unresponsive immediately afterward.
- Comparing Debug/simulator numbers to Release/device numbers.
- Averaging mixed cold, warm, prewarmed, and resume events into one conclusion.
- Trusting custom app timestamps without considering prewarming and system-side work.
- Reporting a single launch number without percentiles, variance, device class, or scenario.

## Boundary With Other References

This reference should answer:

- What kind of launch is this?
- What does the number mean?
- What target or expectation applies?
- Is the measurement comparable?
- Is this actually launch, extended launch, resume, or early responsiveness?

This reference should not explain:

- how dyld works in detail
- how to remove `+load`
- whether to use static or dynamic linking
- how to restructure `didFinishLaunching`
- how to configure `XCTApplicationLaunchMetric`
- how to read MetricKit histograms
- how to use the Instruments App Launch template
