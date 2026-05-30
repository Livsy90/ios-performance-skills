# Launch Taxonomy and Targets

Use this reference when the task depends on launch terminology, measurement scope, target interpretation, or whether the reported number actually represents app launch.

This file should stay focused on classification and targets. For implementation details, use the other references:

* Use `pre-main-dyld-and-static-initializers.md` for dyld, runtime setup, `+load`, constructors, and static initialization.
* Use `linking-strategy.md` for static, dynamic, and mergeable library trade-offs.
* Use `launch-orchestration-and-dependency-graph.md` for critical path analysis, startup step dependencies, safe parallelism, and dependency-chain optimization.
* Use `appdelegate-scenedelegate-and-first-frame.md` for app lifecycle code, root UI creation, and first-frame work.
* Use `swiftui-app-launch.md` for SwiftUI `App`, root view, observable state, and lifecycle-triggered async work.
* Use `third-party-sdks-at-launch.md` for SDK startup policy.
* Use `metrics-instruments-xctest-metrickit.md` for tool-specific instructions, APIs, CI baselines, and production monitoring.

## Core Definition

Treat app launch as the user-visible transition from an external launch request to the first useful visible UI and early responsiveness.

For measurement purposes, the most common launch boundary is:

```text
user opens the app
→ system and process startup
→ runtime and app initialization
→ first app-rendered frame is drawn
→ optional extended launch work
→ app becomes responsive enough for the intended first interaction
```

The launch screen storyboard or static launch image is not the app's first rendered frame. It is a system-provided placeholder while the app is still starting.

## Launch Scenarios

Classify the scenario before diagnosing or comparing numbers.

### Cold Launch

Use **cold launch** when the app process is not already running and little relevant system, binary, data, or service state is warm.

Common cases:

* after device reboot
* after the app has not been launched for a long time
* after the app was terminated and system caches are not helpful
* after install or update, depending on the test setup

Cold launch is the worst-case user experience, but it can be noisy. It is useful for validating the app under unfavorable conditions and for catching launch work that is hidden by warm caches.

### Warm Launch

Use **warm launch** when the app still starts a new process, but relevant system state, binary pages, services, or app data may already be warm.

Common cases:

* the app was recently terminated and launched again
* repeated XCTest launch iterations
* repeated manual launches during local profiling

Warm launch is often more repeatable than cold launch. It is useful for regression testing, but it must not be reported as cold-launch performance.

### Prewarmed Launch

Use **prewarmed launch** when iOS has performed part of the launch preparation before the user explicitly opens the app.

Prewarming can make manual timing misleading because some early work may happen before the point where custom app timing begins. Do not infer pre-main cost from app-level timestamps alone when prewarming may be involved.

When prewarming is possible, prefer system tools and production metrics that understand launch phases, or clearly label custom measurements as app-defined intervals rather than full launch duration.

### First Install, First Run, and Post-Update Launch

Use **first install**, **first run**, or **post-update launch** when the app starts after installation, update, data migration, schema migration, or a major app-version change.

Do not automatically treat these as ordinary cold launches. They may include extra app-defined work:

* initial database creation
* schema migration
* cache creation
* onboarding or permission routing
* first authentication/session resolution
* feature flag bootstrap
* local data reindexing
* migration from older app state

An agent may not be able to prove that a measurement came from one of these scenarios unless the trace, test setup, logs, CI script, or launch metadata make it explicit. When evidence is missing, classify the scenario as unknown and ask for measurement context.

During code review, treat first-run and post-update work as launch-path risk if it can execute before first frame or first interaction. Compare first-run launches only against other first-run launches with a similar setup.

### Resume / Already-Running Return

Use **resume** or **already-running return** when the app process already exists and the user returns from the Home Screen, app switcher, notification, universal link, or another foreground transition.

This is not a full launch. It does not run the same process creation, dyld, static initialization, or `didFinishLaunching` path as a new process launch.

Analyze resume separately when the user asks about foregrounding latency, app switcher return, scene activation, stale UI, authentication refresh, or background-to-foreground responsiveness.

Avoid using **hot launch** as a primary category. If the user says "hot launch," clarify whether they mean warm launch of a new process or resume of an already-running process.

### Background Launch

Use **background launch** when the system starts the app process for background work without immediately presenting UI.

Examples:

* background fetch
* silent push
* location event
* background processing task
* background URLSession event

Do not evaluate background launch with first-frame targets. If background work later leads to visible UI, split the analysis into background processing and the later foreground/resume path.

### Unknown Scenario

Use **unknown** when the report does not say how the app was launched, whether the process was already running, which device was used, or whether the number came from Instruments, XCTest, MetricKit, Organizer, custom logging, or manual timing.

In this case, do not make strong conclusions. First normalize the measurement setup or give advice that is explicitly conditional.

## Launch Entry Point

Classify how the launch was initiated, because the entry point can change the required first screen, routing work, and data requirements.

Common entry points:

* app icon
* notification tap
* universal link or custom URL scheme
* widget
* Siri or shortcut
* file open or document handoff
* background event that later becomes visible

Do not compare app-icon launch with deep-link, notification, widget, or document launch unless those paths are intentionally part of the same launch target. Deep links and notification launches may require authentication, routing, navigation, data loading, or state restoration that the normal app-icon path does not require.

## Measurement Boundary

Always identify what the measurement includes.

Common boundaries:

* app icon tap to first app-rendered frame
* process start to first app-rendered frame
* `main` entry to first screen
* first frame to first responsiveness
* app icon tap to completion of extended launch tasks
* background-to-foreground resume time
* custom business milestone such as "home data loaded"

Different tools and custom measurements may use different boundaries. Do not compare a custom `main`-to-home measurement with system time-to-first-draw metrics unless the difference is intentional and clearly labeled.

## What to Compare

Only compare numbers that represent the same scenario and boundary.

Good comparisons:

* cold launch before vs cold launch after on the same device class
* warm XCTest launch baseline vs warm XCTest launch after a change
* first-run launch before vs first-run launch after with the same data/migration setup
* MetricKit launch distribution for app version A vs app version B
* p50/p90/p95 for the same metric, same launch scenario, and comparable audience

Bad comparisons:

* cold launch vs warm launch
* launch vs resume
* first-run or migration-heavy launch vs ordinary returning-user launch
* app-icon launch vs notification or universal-link launch
* debug simulator measurement vs release device measurement
* high-end device measurement vs oldest supported device measurement
* custom `main`-to-first-screen logging vs MetricKit time-to-first-draw
* one manual run vs a distribution of repeated measurements

If the user provides a single number without context, treat it as a symptom, not evidence.

## Targets and Budgets

### First-Frame Target

Use roughly **400 ms to first visible app frame** as an aggressive user-experience target, not as a universal correctness rule.

Interpretation:

* The goal is for the app to show its first app-rendered frame during the system launch animation.
* The first frame can be minimal, cached, placeholder-based, or partially loaded.
* The first frame should still be valid and useful enough that the app can become responsive quickly.
* The target is not a watchdog termination threshold.
* The target is not a pre-main-only budget.

If pre-main alone approaches or exceeds this budget, treat it as a strong signal that launch-critical work is happening too early, but do not label the budget as "pre-main must be under 400 ms." The end-to-end launch experience matters.

### First Frame vs Final Frame

Do not require the first frame to contain all final data.

A good launch can render:

* cached content
* placeholders
* skeleton UI
* partial data
* last-known authenticated state
* a lightweight routing state

Then it can complete secondary work after the first frame, as long as the app remains responsive and does not immediately block the user.

### Extended Launch

Some apps have work that starts at launch and finishes after the first frame. Treat this as **extended launch** when it still affects early usability.

Examples:

* loading enough data for the first meaningful interaction
* resolving initial navigation or deep link state
* refreshing session state needed by the first screen
* finishing initial database access needed by visible UI

Do not hide important launch latency by moving everything after first draw. If post-first-frame work blocks interaction, it is still part of the user-perceived startup experience.

### Responsive Launch

A launch is not successful just because the first frame appeared.

Check whether the app can respond to the first expected interaction shortly after the launch animation completes. If the first frame appears quickly but the main thread remains blocked, classify the issue as early post-launch responsiveness rather than solved launch performance.

## Measurement Setup Checklist

Use this checklist only to classify and normalize launch measurements. Tool-specific workflows belong in `metrics-instruments-xctest-metrickit.md`.

Before accepting a launch number, identify:

* device model
* iOS version
* app version and build configuration
* Debug, Release, or Profile build
* simulator or physical device
* cold, warm, prewarmed, resume, background launch, or unknown scenario
* app icon, notification, deep link, widget, shortcut, document, or other entry point
* fresh install, returning user, or upgraded app
* app data container reset or preserved
* migration path executed or already completed
* authenticated or unauthenticated state
* app data size and account state
* network state and server dependency
* Low Power Mode on/off
* thermal state, if relevant
* battery level or charging state, if the measurement is unexpectedly noisy
* number of runs and variance
* measurement source: Instruments, XCTest, MetricKit, Organizer, custom logging, or manual timing

Prefer release-like builds on physical devices. Include older supported devices when judging user impact, because launch behavior can differ significantly across CPU, memory, storage, OS version, power state, thermal state, and app data size.

## Reporting Guidance for the Agent

When using this reference, include a short launch classification before recommendations.

Use this format when helpful:

```markdown
### Launch classification
Cold launch / warm launch / prewarmed launch / first-run launch / post-update launch / resume / background launch / unknown.

### Entry point
App icon / notification / universal link / widget / shortcut / document / background event / unknown.

### Measurement boundary
What the number appears to measure: app icon to first draw, process start to first draw, `main` to first screen, first draw to responsiveness, extended launch, resume, custom milestone, or unknown.

### Target interpretation
Whether the number should be judged against first-frame, extended launch, resume, background processing, or first-run expectations.

### Caution
Any reason the number may not be comparable: build type, device, data set, first-run work, migration, prewarming, simulator, Low Power Mode, thermal state, one-off run, mixed entry points, or mixed launch scenarios.
```

## Common Mistakes

* Treating resume as launch.
* Calling warm launch "cold" because the app was manually killed.
* Treating background launch as first-frame launch.
* Treating the launch screen storyboard as the first app frame.
* Using 400 ms as a watchdog threshold.
* Using 400 ms as a pre-main-only budget.
* Optimizing to first draw while leaving the app unresponsive immediately afterward.
* Comparing Debug/simulator numbers to Release/device numbers.
* Comparing app-icon launch with notification, universal-link, widget, or document launch.
* Mixing first-run or post-update migration launches with ordinary returning-user launches.
* Averaging mixed cold, warm, prewarmed, first-run, resume, and background events into one conclusion.
* Trusting custom app timestamps without considering prewarming and system-side work.
* Reporting average launch time without checking p90/p95, variance, device class, or scenario.
* Reporting a single launch number without percentiles, variance, device class, entry point, or scenario.

## Boundary With Other References

This reference should answer:

* What kind of launch is this?
* How was the launch initiated?
* What does the number mean?
* What target or expectation applies?
* Is the measurement comparable?
* Is this actually launch, first-run setup, extended launch, resume, background launch, or early responsiveness?

This reference should not explain:

* how dyld works in detail
* how to remove `+load`
* whether to use static or dynamic linking
* how to build launch orchestration or dependency resolution
* how to restructure `didFinishLaunching`
* how to configure `XCTApplicationLaunchMetric`
* how to read MetricKit histograms
* how to use the Instruments App Launch template
