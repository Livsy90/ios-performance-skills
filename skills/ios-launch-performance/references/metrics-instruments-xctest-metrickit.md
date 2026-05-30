# Metrics, Instruments, XCTest, MetricKit, and Production Monitoring

## Scope Boundary

Use this reference when the task is about measuring, diagnosing, validating, or monitoring iOS launch performance.

This reference covers:

- Instruments App Launch traces
- Time Profiler during startup
- signposts and launch phase markers
- XCTest launch metrics
- CI launch baselines
- MetricKit launch metrics
- Xcode Organizer launch reports
- production monitoring strategy

This reference does not explain how to fix specific launch bottlenecks. When measurement points to a cause, hand off to the relevant reference:

- `pre-main-dyld-and-static-initializers.md` for dyld, static initializers, `+load`, constructors, and runtime registration
- `linking-strategy.md` for static, dynamic, and mergeable library trade-offs
- `appdelegate-scenedelegate-and-first-frame.md` for UIKit lifecycle startup and first-frame work
- `swiftui-app-launch.md` for SwiftUI `App`, root view, observable state, `.task`, and `.onAppear`
- `third-party-sdks-at-launch.md` for analytics, crash reporting, remote config, ads, attribution, push, feature flags, and security SDKs
- `launch-taxonomy-and-targets.md` for cold, warm, prewarmed, resume, first-frame targets, and measurement setup

## Agent Capabilities

The agent can:

- ask for an Instruments trace, XCTest output, MetricKit payload, Organizer screenshot/export, or CI trend
- inspect code that defines launch tests, signposts, MetricKit subscribers, logging, and metric upload paths
- propose where to place app-owned signposts around startup phases
- recommend a measurement plan before optimization
- explain what each measurement layer can and cannot prove
- identify ambiguity in launch classification, test setup, and metric interpretation
- suggest CI gating and production monitoring practices
- connect a metric regression to the next reference to inspect

The agent should not claim it has directly measured the app unless it has actual trace, metric, test, or payload data in the task context. Without evidence, it should describe what to measure and what each result would imply.

## Measurement Layers

Use launch measurements at multiple layers. Each layer answers a different question.

### Instruments

Use Instruments for local diagnosis.

Best for:

- finding which launch phase is expensive
- inspecting call stacks
- seeing main-thread work during startup
- validating whether a code change moved or removed launch work
- correlating custom signposts with system timelines

Limitations:

- local traces may not represent production devices or real user data
- simulator traces are not sufficient for launch conclusions
- Debug builds can exaggerate or hide costs
- a single trace can be noisy

### XCTest launch metrics

Use XCTest launch metrics for repeatable local and CI regression checks.

Best for:

- preventing launch regressions in pull requests or nightly runs
- comparing the same app flow across commits
- tracking launch duration under controlled conditions
- validating that a change did not make launch worse

Limitations:

- results are sensitive to device state, OS version, data set, network, and account state
- UI tests usually measure a configured launch path, not the full diversity of production launches
- CI devices can be noisy unless the environment is controlled
- XCTest metrics show a symptom, not the exact bottleneck

### MetricKit

Use MetricKit for production launch distributions collected from real devices.

Best for:

- monitoring launch trends by version/build
- detecting regressions after release
- comparing launch behavior across device classes and OS versions
- understanding real-world distributions rather than a single lab number

Limitations:

- MetricKit launch metrics are aggregated and delayed
- histograms do not identify the exact source line or function
- payloads may combine many sessions and device conditions
- payload interpretation needs app version, build, device, OS, and scenario context

### Xcode Organizer

Use Xcode Organizer for App Store-distributed production metrics.

Best for:

- quick product-level visibility into launch time trends
- comparing app versions and builds
- checking percentile views such as typical and high-impact users
- segmenting by device category where available

Limitations:

- data is available only for distributed builds and eligible users
- there is a delay before metrics appear
- Organizer is a prioritization and trend tool, not a local root-cause debugger

## Instruments Guidance

When diagnosing launch locally, start with Instruments before changing code.

Recommended approach:

1. Use a release-like build configuration.
2. Prefer a physical device, including an older supported device when possible.
3. Capture the launch with the App Launch template when available.
4. Add Time Profiler if CPU-heavy startup work is suspected.
5. Add signpost-related instruments when app-owned launch markers exist.
6. Compare traces before and after the change under the same conditions.

What to inspect:

- timeline location of first visible UI
- work before `main` or before app lifecycle entry
- main-thread call tree during app and scene initialization
- CPU-heavy work before first frame
- synchronous file, database, keychain, or decoding work if visible in stacks
- long intervals between lifecycle entry and first frame
- app-owned signpost intervals around startup phases
- repeated or duplicated initialization

Do not over-interpret one trace. If a trace is surprising, capture more than one run and check whether the same phase remains expensive.

## Time Profiler During Launch

Use Time Profiler when launch is CPU-bound or when the App Launch trace shows a broad expensive region that needs call-stack detail.

The agent should look for:

- heavy work on the main thread
- repeated initialization paths
- expensive constructors or setup functions visible after app code begins
- large parsing, decoding, formatting, hashing, sorting, or dependency construction
- blocking synchronization on the launch path
- third-party SDK startup functions

Useful interpretation rules:

- A hot function is not automatically a launch bug; confirm it runs before first frame or first interaction.
- Main-thread time before first frame usually matters more than background work that does not block readiness.
- Background work can still hurt launch if it competes for CPU, I/O, locks, memory, or priority with first-frame work.
- If heavy work appears before app lifecycle entry, use the pre-main reference.

## Signposts and Launch Phase Markers

Use signposts to make app-owned startup phases visible in Instruments and, where appropriate, in MetricKit custom metrics.

Good signpost boundaries:

- app lifecycle entry to root UI configured
- scene connection to window visible
- dependency container bootstrap
- session/auth state resolution
- feature flag local read
- database open or migration check
- root view model construction
- first-screen cached data load
- post-first-frame noncritical startup work

Poor signpost boundaries:

- one giant `launch` interval covering everything
- intervals that cross unrelated asynchronous work without clear ownership
- high-cardinality names that include user IDs, account IDs, request IDs, or dynamic strings
- signposts around every tiny function, creating noise
- signposts that start and end on unrelated code paths

Recommended naming style:

```swift
import os

private let startupLog = Logger(subsystem: "com.example.app", category: "startup")
private let startupSignposter = OSSignposter(logger: startupLog)

func prepareInitialSession() async {
    let id = startupSignposter.makeSignpostID()
    let state = startupSignposter.beginInterval("InitialSessionPreparation", id: id)
    defer { startupSignposter.endInterval("InitialSessionPreparation", state) }

    await sessionStore.loadCachedSession()
}
```

Keep signpost names stable across releases. Stable names make trace comparison and dashboards easier.

## XCTest Launch Metrics

Use XCTest launch metrics when the team needs repeatable regression checks.

Recommended pattern:

```swift
import XCTest

final class LaunchPerformanceTests: XCTestCase {
    func testLaunchToResponsiveState() {
        let app = XCUIApplication()
        app.launchArguments += ["-uiTesting", "1", "-useStableStartupData", "1"]

        measure(metrics: [XCTApplicationLaunchMetric(waitUntilResponsive: true)]) {
            app.launch()
            app.terminate()
        }
    }
}
```

Notes for the agent:

- `XCTApplicationLaunchMetric()` records launch to first frame and extended launch tasks.
- `XCTApplicationLaunchMetric(waitUntilResponsive: true)` is useful when the team cares about early responsiveness, not only first draw.
- Keep test data stable. A launch test that randomly depends on network, account state, feature rollout, or cache size is not a reliable baseline.
- Prefer physical devices for meaningful launch baselines.
- Use consistent OS/device pools when comparing results.
- Do not mix first install, logged-out, logged-in, deep link, push launch, and restored-session launches in one baseline.

When the agent reviews launch tests, it should check:

- whether the test uses launch arguments to stabilize data and disable unrelated variability
- whether the measured scenario is named clearly
- whether the app is terminated between iterations when measuring process launch
- whether the test waits for the right readiness boundary
- whether CI stores trends rather than relying only on one failed run
- whether baselines are device-specific rather than global across all hardware

## CI Baselines and Regression Gates

A launch performance test is useful only when the team treats it as a trend, not a single absolute number.

Recommended baseline strategy:

1. Choose a small set of representative launch scenarios.
2. Run on stable physical devices or a controlled device farm.
3. Store historical results by app version, commit, device model, OS version, build configuration, and scenario.
4. Track median and high-percentile values when enough samples exist.
5. Alert on meaningful regressions rather than tiny noise-level changes.
6. Require an Instruments trace for regressions that exceed the agreed threshold.

Good baseline dimensions:

- cold or warm classification
- first install vs returning user
- logged-out vs logged-in
- cached data size
- network mocked, disabled, or real
- device model
- OS version
- app build configuration
- locale if startup formatting/localization is heavy

Avoid:

- one universal launch budget for every device and scenario
- comparing simulator results with physical-device history
- comparing Debug builds with Release builds
- using averages alone when variance is high
- failing pull requests for tiny changes within normal noise
- accepting a large regression because a single later run happened to pass

## MetricKit Launch Metrics

Use MetricKit when the app needs production launch monitoring or build-to-build launch regression detection.

The relevant payload area is `MXMetricPayload.applicationLaunchMetrics`, which provides `MXAppLaunchMetric` histograms. Useful launch-related properties include:

- `histogrammedTimeToFirstDraw`
- `histogrammedOptimizedTimeToFirstDraw`
- `histogrammedApplicationResumeTime`
- `histogrammedExtendedLaunch`

Interpretation guidance:

- `histogrammedTimeToFirstDraw` is the primary production signal for launch to first draw.
- `histogrammedOptimizedTimeToFirstDraw` should be treated separately from ordinary launch measurements because system prewarming/optimization can change the path.
- `histogrammedApplicationResumeTime` is about returning from background, not full process launch.
- `histogrammedExtendedLaunch` can reveal work that continues beyond the first draw but still belongs to the launch experience.

Do not collapse all MetricKit launch histograms into one number. Keep launch, optimized/prewarmed launch, resume, and extended launch separate.

When reviewing MetricKit integration, check:

- subscriber registration occurs early enough to receive reports, but without heavy processing on the launch path
- payload processing is lightweight or moved off the main thread
- payload upload is deferred and resilient
- app version, build number, OS version, device model, and environment metadata are preserved where available
- histograms are stored without losing bucket information
- privacy-sensitive information is not added to metric records
- dashboards separate P50/P90/P95-style views where possible
- launch metrics are compared build-to-build, not only as raw global averages

## Xcode Organizer

Use Xcode Organizer as a production trend and prioritization tool.

The agent can recommend checking Organizer when:

- local tests look good but users report slow launch
- launch regressions appear only after App Store/TestFlight distribution
- the team needs device-specific production visibility
- the issue may affect only older devices or a specific OS version
- the team needs to compare app versions and builds

Organizer guidance:

- Check launch time trends by version/build.
- Compare typical and high-impact percentile views when available.
- Segment by device class if the UI allows it.
- Treat Organizer data as delayed and aggregated.
- Use Organizer to decide priority, then use Instruments or reproducible tests for root cause.

## Production Monitoring Strategy

A good launch monitoring setup combines local diagnosis, automated regression checks, and field metrics.

Recommended model:

- **Local diagnosis**: Instruments App Launch, Time Profiler, signposts
- **Automated regression**: XCTest launch metrics in CI or scheduled device runs
- **Production trend**: MetricKit and Xcode Organizer
- **App-owned context**: stable signposts, launch scenario tags, build metadata, feature rollout state where privacy-safe

Track at least:

- launch-to-first-draw distribution
- optimized/prewarmed launch distribution if available
- resume distribution separately
- extended launch distribution if relevant
- device model and OS version breakdown
- app version/build trend
- startup scenario classification when app-owned telemetry can safely infer it

Avoid using production monitoring as a replacement for local traces. Production tells the team where and how often launch is slow; local tools explain why.

## Recommended Agent Output

When using this reference, include a measurement section in the answer.

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
- If the issue is in pre-main, read ...
- If the issue is in app lifecycle startup, read ...
```

## Common Mistakes

- Treating resume time as launch time.
- Comparing cold, warm, prewarmed, and resume measurements as if they were interchangeable.
- Using a single local trace as a production conclusion.
- Using MetricKit histograms to claim a specific function is slow.
- Running launch tests only on simulator or Debug builds.
- Failing CI on tiny noise-level differences.
- Measuring a launch path that depends on live network state.
- Adding signposts with unstable names or private user data.
- Processing MetricKit payloads synchronously during startup.
- Optimizing based on averages while ignoring high-percentile users.
- Forgetting to validate a launch fix with both local traces and production trends.

## Boundary With Other References

This reference should stop at measurement, interpretation, and monitoring.

If the answer needs to explain how to reduce dyld cost, move static initialization, change linking strategy, defer AppDelegate work, restructure SwiftUI root setup, or change third-party SDK initialization, use the corresponding reference instead of expanding that guidance here.
