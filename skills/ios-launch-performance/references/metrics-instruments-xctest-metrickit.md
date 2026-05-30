# Metrics, Instruments, XCTest, MetricKit, and Production Monitoring

Use this reference when the task is about measuring, diagnosing, validating, or monitoring iOS launch performance.

This reference is intentionally about evidence and interpretation. It should not explain how to fix every bottleneck. Once the measurement points to a likely cause, route the agent to the focused reference for that cause.

## Scope Boundary

This reference covers:

* Instruments App Launch traces
* Time Profiler during app startup
* signposts and app-owned launch phase markers
* XCTest launch metrics
* `xctrace` exports or textual summaries when provided by the user
* CI launch baselines and regression gates
* MetricKit launch metrics and signpost metrics
* Xcode Organizer production launch reports
* production launch monitoring strategy

This reference does not cover implementation fixes in detail:

* Use `launch-taxonomy-and-targets.md` for cold, warm, prewarmed, resume, first-frame targets, first install/update launch, and measurement comparability.
* Use `pre-main-dyld-and-static-initializers.md` for dyld, static initializers, `+load`, `+initialize`, constructor functions, and runtime registration.
* Use `linking-strategy.md` for static libraries, dynamic frameworks, mergeable libraries, order files, and binary-layout trade-offs.
* Use `launch-orchestration-and-dependency-graph.md` for launch steps, critical path, dependency graphs, safe parallelism, and longest-chain analysis.
* Use `appdelegate-scenedelegate-and-first-frame.md` for UIKit lifecycle startup, root UI creation, first-frame readiness, and main-thread launch work.
* Use `swiftui-app-launch.md` for SwiftUI `App`, root view setup, observable state, `.task`, `.onAppear`, and environment initialization.
* Use `third-party-sdks-at-launch.md` for analytics, crash reporting, ads, attribution, push, remote config, feature flags, security SDKs, and vendor startup policy.

## Agent Capabilities

The agent can:

* ask for an Instruments trace, `xctrace` export, XCTest output, MetricKit payload, Organizer screenshot/export, CI trend, or production dashboard summary
* inspect code that defines launch tests, signposts, MetricKit subscribers, logging, metric upload, and test launch arguments
* propose where to place app-owned signposts around launch phases
* identify ambiguity in scenario classification, readiness boundary, device class, build configuration, or data set
* explain what each measurement layer can and cannot prove
* recommend a measurement plan before optimization
* suggest CI baseline and regression-gate strategy
* connect a metric regression to the next reference to inspect

The agent should not claim it measured the app unless trace, test, payload, or monitoring data is present in the task context. Without evidence, it should say what to measure, why to measure it, and how to interpret possible outcomes.

## Measurement Layers

Use several layers of evidence because each answers a different question.

### Local diagnosis: Instruments

Use Instruments when the question is **why this launch is slow on this device/build**.

Best for:

* locating expensive launch phases
* inspecting call stacks and thread activity
* seeing main-thread work before first frame
* correlating app-owned signposts with system timelines
* validating that a change moved, shortened, or removed launch-path work
* comparing before/after traces under controlled conditions

Limitations:

* a local trace may not represent production devices or data sets
* simulator traces are not enough for launch conclusions
* Debug builds can distort results
* a single trace can be noisy
* Instruments explains a local run, not the whole user population

### Automated regression: XCTest launch metrics

Use XCTest launch metrics when the question is **did this commit or release candidate regress a known launch path**.

Best for:

* repeatable launch checks on controlled devices
* CI or scheduled device-farm baselines
* comparing one scenario across commits
* catching large regressions before release
* validating that a targeted fix did not make the measured launch path worse

Limitations:

* launch tests measure a configured scenario, not every production scenario
* results are sensitive to device state, OS, test data, cache state, account state, network, and feature rollout
* CI device pools can be noisy
* XCTest reports a symptom and boundary, not the exact source line

### Field trend: MetricKit

Use MetricKit when the question is **how launch behaves on real user devices after distribution**.

Best for:

* app-version and build-to-build launch trends
* device-class and OS-version segmentation
* detecting production regressions that local testing missed
* understanding distributions instead of isolated lab samples
* separating launch, prewarmed launch, resume, and extended launch histograms

Limitations:

* payloads are aggregated and delayed
* histograms do not identify the exact function or source line
* payloads reflect many sessions, device conditions, and user states
* interpretation needs app version, build, OS, device, and scenario context

### Product visibility: Xcode Organizer

Use Xcode Organizer when the question is **which production performance issue should the team prioritize**.

Best for:

* quick launch trend visibility for distributed builds
* comparing versions and builds
* spotting device-specific or OS-specific issues
* using percentile views to avoid average-only decisions
* deciding whether a local regression also matters in the field

Limitations:

* data is delayed and aggregated
* availability depends on distribution and eligible users
* Organizer helps prioritize and triage; it is not a replacement for local traces

## Measurement Setup Checklist

Before accepting or comparing a launch number, identify:

* launch scenario: cold, warm, prewarmed, resume, first install, first run after update, or unknown
* measurement boundary: tap-to-first-draw, process-start-to-first-draw, `main`-to-first-screen, launch-to-responsive, or custom interval
* device model and OS version
* build configuration: Debug, Release, Profile, TestFlight, or App Store
* simulator or physical device
* app version and build number
* account state: logged out, logged in, restored session, enterprise profile, migration state
* data size: fresh install, small cache, large cache, large database, upgraded schema
* network mode: live network, mocked, disabled, captive/slow network, or airplane mode
* feature flags, remote config, experiments, or rollout cohort
* number of runs and variance
* measurement source: Instruments, XCTest, MetricKit, Organizer, custom logging, or manual timing

If these are unknown, treat the number as a symptom rather than conclusive evidence.

## Instruments Guidance

When diagnosing launch locally, start with Instruments before changing code.

Recommended approach:

1. Use a release-like build configuration.
2. Prefer a physical device. Include an older supported device when possible.
3. Capture the same launch scenario before and after the change.
4. Use the App Launch template when available.
5. Add Time Profiler when CPU-heavy startup work is suspected.
6. Add signpost-related instruments when app-owned launch markers exist.
7. Capture more than one run when results are surprising or noisy.

Inspect:

* first visible app frame location in the timeline
* work before app lifecycle entry
* main-thread call tree during app and scene initialization
* long gaps between lifecycle entry, root UI setup, first draw, and responsiveness
* static initialization or framework work if visible in the trace
* CPU-heavy startup work
* blocking synchronization on the launch path
* file, database, keychain, image decoding, or JSON parsing work before first frame
* duplicated initialization
* third-party SDK startup regions
* app-owned signpost intervals

Interpretation rules:

* A hot function is not automatically a launch bug. Confirm it runs before first frame or first interaction.
* Main-thread work before first frame is usually more launch-critical than background work that does not block readiness.
* Background work can still hurt launch if it competes for CPU, I/O, locks, memory, or priority.
* If the expensive region appears before app lifecycle entry, route to `pre-main-dyld-and-static-initializers.md` or `linking-strategy.md`.
* If the expensive region is a sequence of app-owned startup steps, route to `launch-orchestration-and-dependency-graph.md`.

## Time Profiler During Launch

Use Time Profiler when the App Launch trace identifies a broad expensive region or when startup appears CPU-bound.

Look for:

* main-thread CPU work before first frame
* repeated setup paths
* expensive constructors or setup functions visible after app code begins
* parsing, decoding, formatting, hashing, sorting, or graph construction
* lock contention or semaphore waits
* synchronous service initialization
* third-party SDK startup functions
* launch work that appears on background queues but blocks readiness indirectly

When reporting Time Profiler findings, include:

* thread where the work occurs
* whether it runs before first frame, before first interaction, or after launch
* whether it blocks the critical path
* whether it is app code, system code, or third-party code
* whether the trace is stable across repeated runs

## Signposts and Launch Phase Markers

Use signposts to make app-owned startup phases visible in Instruments. Use MetricKit signpost metrics only for a small number of stable, important intervals.

Good signpost boundaries:

* app lifecycle entry to root UI configured
* scene connection to window visibility
* root UI construction
* dependency container bootstrap
* session or auth state resolution
* local feature flag read
* database open or migration check
* root view model construction
* first-screen cached data load
* post-first-frame startup work that can affect responsiveness

Poor signpost boundaries:

* one giant interval covering the entire launch
* intervals with unclear ownership
* intervals that start and end on unrelated async paths
* very small functions that create trace noise
* high-cardinality names or categories
* names containing user IDs, account IDs, request IDs, URLs, tokens, or dynamic strings

Naming guidance:

* Use stable names across releases.
* Keep names short and phase-oriented.
* Prefer a small number of meaningful intervals over many noisy intervals.
* Do not include private or user-specific data in names or metadata.
* Make the boundary clear enough that a future engineer can compare traces without reading the implementation.

MetricKit-specific caution:

* Ordinary `OSSignposter` / `os_signpost` usage is excellent for Instruments.
* MetricKit custom signpost metrics require MetricKit-compatible signpost APIs/log handles and should be reserved for important intervals.
* Avoid high-volume signposts in MetricKit payloads.
* Treat MetricKit signpost metrics as production distributions, not as function-level profiling.

## XCTest Launch Metrics

Use XCTest launch metrics when the team needs repeatable regression checks.

Prefer `XCTApplicationLaunchMetric` for app launch measurement.

Interpretation:

* `XCTApplicationLaunchMetric()` measures application launch to first frame and extended launch tasks.
* `XCTApplicationLaunchMetric(waitUntilResponsive: true)` is useful when the readiness boundary includes early responsiveness, not only first draw.
* Older signpost-based XCTest metrics can be useful for custom regions, but launch-specific tests should prefer the launch metric when available.

When reviewing a launch test, check:

* the measured launch scenario is named clearly
* test launch arguments stabilize data and disable unrelated variability
* the app starts from a known state
* process launch is actually measured, not resume
* first install, logged-out, logged-in, deep link, push, and restored-session launches are not mixed into one baseline
* the readiness boundary matches the product expectation
* live network dependency is mocked, disabled, or deliberately part of the scenario
* device model, OS, build configuration, and scenario are stored with the result
* CI tracks history rather than relying on one pass/fail result

Avoid:

* placing unrelated cleanup inside the measured launch block
* relying on simulator-only results
* comparing Debug results with Release/Profile history
* using one global threshold for all devices and scenarios
* failing PRs for changes inside normal variance
* calling a launch test stable when it depends on remote config, random cache state, or live backend latency

## CI Baselines and Regression Gates

A launch performance test is useful only when the team treats it as a trend.

Recommended baseline strategy:

1. Choose a small set of representative launch scenarios.
2. Run on stable physical devices or a controlled device farm.
3. Store results with commit, app version, build configuration, device model, OS version, data set, and scenario.
4. Track median and high-percentile values when sample size allows it.
5. Alert on meaningful regression windows rather than one noisy run.
6. Require a local trace or `xctrace` artifact for regressions above the agreed threshold.
7. Rebaseline only after an intentional launch-path change is reviewed and documented.

Useful scenario dimensions:

* cold vs warm classification
* first install vs returning user
* first run after app update
* logged out vs logged in
* small cache vs large cache
* no local database vs migrated database
* mocked network vs real network
* normal mode vs Low Power Mode when used as a stress signal
* older supported device vs current flagship

Regression-gate guidance:

* Use per-device and per-scenario baselines.
* Prefer percentage and absolute thresholds together.
* Treat high-percentile regression as important even if the average looks acceptable.
* Do not block every PR on slow/noisy UI launch tests if the device pool is unstable; scheduled or nightly runs can be more reliable.
* Store raw measurements, not only summarized pass/fail output.

## MetricKit Launch Metrics

Use MetricKit when the app needs production launch monitoring or build-to-build launch regression detection.

The relevant payload area is `MXMetricPayload.applicationLaunchMetrics`, which provides `MXAppLaunchMetric` histograms. Important launch-related properties include:

* `histogrammedTimeToFirstDraw`
* `histogrammedOptimizedTimeToFirstDraw`
* `histogrammedApplicationResumeTime`
* `histogrammedExtendedLaunch`

Interpretation guidance:

* `histogrammedTimeToFirstDraw` represents ordinary launch to first draw.
* `histogrammedOptimizedTimeToFirstDraw` represents launches affected by system optimization/prewarming and must be tracked separately.
* `histogrammedApplicationResumeTime` is resume from background, not full process launch.
* `histogrammedExtendedLaunch` helps detect work that continues beyond first draw and still affects the launch experience.

Do not collapse these histograms into one number. Keep ordinary launch, optimized/prewarmed launch, resume, and extended launch separate.

When reviewing MetricKit integration, check:

* subscriber registration is early enough to receive payloads but does not perform heavy processing on the launch path
* payload handling avoids main-thread work during startup
* upload is deferred, resilient, and batched
* app version, build number, OS version, device model, and environment metadata are preserved where available
* histogram bucket information is stored, not flattened into one average too early
* P50/P90/P95-style views are available where sample size supports them
* launch metrics are compared build-to-build and version-to-version
* privacy-sensitive information is not added to metric records
* resume metrics are not presented as launch metrics
* optimized/prewarmed launch metrics are not mixed with ordinary launch metrics

MetricKit cannot answer:

* which exact source line caused the regression
* whether a specific function is slow in one local trace
* whether a local optimization fixed every production device class
* whether a custom timestamp covers system-side launch work

Use MetricKit to prioritize and verify field trends; use Instruments and reproducible tests for root cause.

## Xcode Organizer

Use Xcode Organizer as a production trend and prioritization tool for distributed builds.

Recommend checking Organizer when:

* local tests look good but users report slow launch
* launch regressions appear only after TestFlight or App Store distribution
* the team needs device-specific production visibility
* the issue may affect older devices or a specific OS version
* the team needs to compare app versions and builds
* field data is needed before prioritizing a launch investigation

Organizer guidance:

* Check launch trends by app version and build.
* Prefer percentile and device-segment views when available.
* Treat the data as delayed and aggregated.
* Use Organizer to decide priority and scope, then use local traces or reproducible tests for root cause.
* Do not use Organizer screenshots as a replacement for stored CI/performance history.

## Custom App Telemetry

Custom launch telemetry can complement Apple tools, but it must be labeled carefully.

Useful custom context:

* app version and build
* launch scenario inferred by app-owned state where safe
* logged-in/logged-out state
* first install, returning user, or post-update run
* data size bucket, not raw user data
* feature rollout or experiment cohort where privacy-safe
* first screen route or launch entry point category
* high-level signpost phase durations

Cautions:

* App timestamps usually cannot see system-side work before the app starts executing.
* Prewarming can make app-defined intervals look better than user-perceived launch.
* Avoid sending user identifiers, account identifiers, request identifiers, tokens, or raw URLs.
* Do not use app telemetry as the only source of truth for first draw unless the boundary is validated against system tools.
* Keep event names and dimensions low-cardinality.

## Production Monitoring Strategy

A good launch monitoring setup combines local diagnosis, automated regression checks, and field metrics.

Recommended model:

* **Local diagnosis**: Instruments App Launch, Time Profiler, signposts
* **Automated regression**: XCTest launch metrics in CI or scheduled physical-device runs
* **Production trend**: MetricKit and Xcode Organizer
* **App-owned context**: stable phase markers, launch scenario tags, build metadata, and privacy-safe rollout context

Track separately:

* ordinary launch-to-first-draw distribution
* optimized/prewarmed launch distribution
* resume distribution
* extended launch distribution
* device model and OS version breakdown
* app version/build trend
* first install / first run after update when app telemetry can safely infer it
* first screen or entry-point category when useful and privacy-safe

Escalation model:

1. Production trend shows regression.
2. CI or local controlled test attempts to reproduce it.
3. Instruments identifies the phase and call stack.
4. The fix is validated locally.
5. CI baseline confirms no regression.
6. MetricKit or Organizer confirms field recovery in a later build.

## Recommended Agent Output

When using this reference, include a measurement section.

Suggested structure:

```markdown
### Measurement plan
- Local diagnosis:
- Automated regression:
- Production monitoring:

### Evidence needed
- Trace/test/payload needed:
- Scenario classification:
- Device/build requirements:

### Interpretation
- Metric to watch:
- What improvement would prove the fix:
- What ambiguity remains:

### Handoff
- If the issue is in pre-main:
- If the issue is in linking strategy:
- If the issue is in launch orchestration:
- If the issue is in app lifecycle startup:
- If the issue is in SwiftUI root setup:
- If the issue is in third-party SDK startup:
```

Keep recommendations tied to evidence. Avoid generic performance advice when the task only needs a measurement plan.

## Common Mistakes

* Treating resume time as launch time.
* Comparing cold, warm, prewarmed, first install, post-update, and resume measurements as if they were interchangeable.
* Treating a single local trace as a production conclusion.
* Using MetricKit histograms to claim a specific function is slow.
* Running launch tests only on simulator or Debug builds.
* Measuring a path that depends on live network state without labeling it as such.
* Failing CI for tiny noise-level differences.
* Tracking averages while ignoring high-percentile users.
* Processing MetricKit payloads synchronously during startup.
* Adding signposts with unstable names or private user data.
* Mixing optimized/prewarmed launch and ordinary launch in one dashboard.
* Forgetting to verify a launch fix with both local evidence and field trends.
* Rebaselining after a regression without a documented decision.

## Boundary With Other References

This reference should stop at measurement, interpretation, validation, and monitoring.

If the answer needs to explain how to reduce dyld cost, remove static initialization, change linking strategy, restructure launch steps, defer AppDelegate work, simplify SwiftUI root setup, or change SDK initialization policy, route to the corresponding reference instead of expanding that guidance here.
