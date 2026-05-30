# Validation and Testing

Use this reference when reviewing how to verify perceived-performance claims for iOS screens and user flows.

This reference is about validation discipline. It helps the agent distinguish between code-level reasoning, implemented instrumentation, measured local evidence, production evidence, and manual UX judgment.

For staged real-content rendering, read `references/progressive-rendering.md`. For loading states, read `references/loading-states.md`. For optimistic UI, read `references/optimistic-updates.md`. For high-stakes flows, read `references/high-stakes-actions.md`.

## Core Rule

Do not present a perceived-performance recommendation as verified unless there is evidence.

A code change can be plausible without being proven. A screen can have better state modeling without being objectively smoother. A progressive-rendering refactor can reduce blank-screen time without reducing total loading time.

Use precise language:

- “This should reduce blank-screen time.”
- “This gives the user earlier feedback.”
- “This needs validation on a release build.”
- “This should be checked with a screen recording, Instruments trace, or production signal.”

Avoid unsupported claims:

- “This fixes performance.”
- “This is now smooth.”
- “This will feel faster to users.”
- “This works well on older devices.”
- “Low Power Mode proves old-device performance.”

## Agent Capability Boundaries

The agent can usually inspect and improve:

- missing loading, empty, failed, refreshing, pending, and confirmed states
- all-or-nothing rendering
- UI that waits for unrelated async work before updating
- optimistic UI without rollback
- high-stakes flows that show success before confirmation
- missing duplicate-submission guards
- missing retry or recovery states
- missing instrumentation points
- missing performance tests when the repository is available
- missing MetricKit or production diagnostics integration when relevant

The agent can analyze evidence provided by the user:

- screen recordings
- Instruments summaries or trace screenshots
- logs with timestamps
- XCTest or UI test results
- CI performance reports
- MetricKit payload summaries
- Xcode Organizer hangs or hitches summaries
- before/after measurements
- user reports with reproducible steps

The agent cannot directly prove without a runtime environment or user-provided evidence:

- actual smoothness on device
- Low Power Mode behavior
- older-device performance
- thermal behavior
- release-build responsiveness
- animation hitch frequency
- scroll hitch frequency
- production hang or hitch rates
- actual user perception

Do not claim that a manual, device, production, or profiling validation step was performed unless the agent actually ran it in the available environment or the user provided the result.

## Evidence Ladder

Use the strongest available evidence, but be honest about its limits.

### Code-level plausibility

Examples:

- state model supports loading, empty, failed, and refreshing
- UI no longer waits for all sections before rendering primary content
- optimistic update has rollback
- high-stakes action waits for server confirmation
- duplicate submission is guarded

This supports “the design is more likely to feel responsive,” not “performance is proven.”

### Local observation

Examples:

- screen recording
- manual tap-to-feedback timing
- slow network testing
- release build on a real device
- repeated scroll or animation run
- app run with Low Power Mode enabled

This supports local behavioral observations, not universal device coverage.

### Automated local tests

Examples:

- UI tests that verify state transitions
- performance tests with baselines
- measured launch or flow timing
- scroll tests with metrics when supported
- regression checks in CI

This supports repeatability, but still depends on device, simulator, build configuration, and test environment.

### Profiling evidence

Examples:

- Instruments trace
- hang analysis
- Time Profiler
- Animation Hitches
- SwiftUI or UIKit rendering diagnostics
- main-thread analysis

This supports local diagnosis of a specific run.

### Production evidence

Examples:

- Xcode Organizer hangs or hitches
- MetricKit payloads
- app version trends
- device-class breakdowns
- OS-version breakdowns
- production signposts or custom metrics

This supports real-world behavior, but often needs careful aggregation and correlation.

## Manual Validation

Manual validation is appropriate when perceived performance depends on what the user sees and feels.

Use manual validation for:

- tap-to-first-feedback
- time to first meaningful content
- loading-state clarity
- skeleton or placeholder behavior
- layout stability
- scroll smoothness
- animation continuity
- high-stakes confirmation clarity
- optimistic update failure behavior

Suggested scenarios:

- first load
- refresh with existing content
- slow network
- failed request
- retry after failure
- repeated taps
- screen disappearance during work
- app backgrounding during work
- returning to the screen after navigation
- long scrolling session
- animation-heavy transition

The agent can suggest these scenarios. It should not claim they passed unless evidence is provided.

## Screen Recordings

Screen recordings are useful because perceived performance is visual and temporal.

Use recordings to inspect:

- tap-to-first-feedback
- blank-screen duration
- first meaningful content
- layout jumps
- flicker between states
- stale content during refresh
- loading indicator timing
- optimistic update feedback
- rollback or failure transition
- duplicate-submission behavior
- animation hitches visible to the eye

Suggested review method:

1. Start recording before the user action.
2. Perform the action once under normal conditions.
3. Repeat under slow network or constrained conditions when relevant.
4. Compare before and after changes.
5. Note timestamps for first feedback, first meaningful content, and final state.

Correct phrasing:

- “The recording shows feedback appearing immediately after tap.”
- “The screen remains blank until all sections load.”
- “The layout shifts when the secondary section appears.”
- “The rollback state is visible after simulated failure.”

Avoid:

- “Users will definitely perceive this as faster.”
- “This proves all devices are smooth.”
- “This fixes performance globally.”

## Release Builds

Prefer release-like builds for performance validation.

Debug builds are useful for development, but they are not reliable proof of user-facing performance. They may include different compiler optimization levels, additional assertions, debug logging, overlays, and non-production behavior.

Use release or release-like builds when validating:

- scrolling
- animations
- launch or screen transition timing
- rendering-heavy screens
- CPU-heavy transformations
- memory pressure
- perceived responsiveness
- repeated navigation flows

The agent can suggest release-build validation. If the repository is available, it can check whether performance tests or CI jobs run in a release-like configuration.

## Older Devices

Older-device testing is valuable because real devices differ in CPU, GPU, memory bandwidth, thermal behavior, refresh characteristics, OS versions, and storage behavior.

Use older-device testing for:

- scroll-heavy feeds
- media-heavy screens
- animation-heavy flows
- large SwiftUI or UIKit hierarchies
- image decoding or resizing
- expensive layout or compositing
- startup and first-screen rendering
- low memory headroom

The agent cannot simulate an older device from code inspection. It can recommend older-device validation and explain which flows should be tested.

Do not claim that simulator results or a modern device prove older-device performance.

## Low Power Mode

Low Power Mode can be suggested as a cheap manual stress signal for render-heavy screens.

It may make limited performance headroom easier to notice, but it is not an older-device simulator. It does not reproduce older GPU architecture, memory bandwidth, display behavior, OS version, storage behavior, thermal state, or device-specific bottlenecks.

Use Low Power Mode as a signal for:

- frame drops
- animation hitches
- delayed tap response
- expensive layout
- image decoding pressure
- compositing and transparency cost
- render-heavy lists or feeds

Correct phrasing:

- “Try the screen with Low Power Mode enabled as a manual stress signal.”
- “If the screen starts hitching under Low Power Mode, inspect layout, rendering, decoding, or compositing cost.”
- “Low Power Mode does not replace older-device testing.”

Avoid:

- “Low Power Mode proves old-device performance.”
- “The screen passes performance testing because it works in Low Power Mode.”
- “Low Power Mode is equivalent to a low-end device.”

## Scrolling and Animation Checks

Scrolling and animation issues are often perceived as performance problems even when data loading is fast.

Validate:

- first scroll after content appears
- long continuous scrolling
- rapid scroll direction changes
- pull-to-refresh
- navigation push and pop
- modal presentation and dismissal
- expanding and collapsing sections
- animated state transitions
- image-heavy list cells
- video or media-heavy sections
- skeleton-to-content transition
- error-to-retry-to-loaded transition

Look for:

- dropped frames
- visible hitching
- delayed interaction
- layout jumps
- image pop-in
- repeated re-layout
- content offset jumps
- animation stutter
- sudden main-thread stalls

The agent can suggest these checks and can analyze recordings or traces if provided. It should not claim scrolling or animation is smooth without runtime evidence.

## XCTest and UI Performance Tests

Automated tests can help catch regressions, especially when the same flow can be repeated.

Good candidates:

- launch to first screen
- tap to first loading state
- tap to first meaningful content
- refresh flow
- search result rendering
- scroll through a long list
- navigation into and out of a heavy screen
- retry after failure
- optimistic update state transition
- duplicate tap prevention

A simple UI test can verify state transitions even when it does not measure performance:

```swift
func testLoadingStateAppearsBeforeContent() {
    let app = XCUIApplication()
    app.launch()

    app.buttons["Load"].tap()

    XCTAssertTrue(app.staticTexts["Loading"].waitForExistence(timeout: 1))
    XCTAssertTrue(app.staticTexts["Content"].waitForExistence(timeout: 5))
}
```

A performance test can measure repeatable work when the project supports it:

```swift
func testFeedScrollPerformance() {
    let app = XCUIApplication()
    app.launch()

    let feed = app.collectionViews.firstMatch
    XCTAssertTrue(feed.waitForExistence(timeout: 5))

    measure {
        feed.swipeUp()
        feed.swipeDown()
    }
}
```

Use performance tests for regression detection, not as the only proof of perceived quality.

Limitations:

- simulator results may not match device behavior
- CI hardware may be inconsistent
- debug builds can distort performance
- tests may not cover real data size
- tests may miss visual quality issues
- production behavior can differ from local tests

The agent can write or suggest tests when the repository is available. It should state what the test can and cannot prove.

## Instruments

Use Instruments when there is a runtime symptom that code review cannot prove or localize.

Useful when investigating:

- UI hangs
- delayed tap response
- animation hitches
- scroll jank
- high main-thread work
- layout or rendering cost
- image decoding cost
- excessive allocations
- expensive compositing
- long synchronous work
- startup or screen transition delays

The agent can suggest which trace to collect and what to inspect. It can analyze a provided trace summary, screenshot, or exported data. It cannot claim the trace is clean unless it has seen the trace.

Connect symptoms to hypotheses:

- delayed first feedback → main-thread work, no immediate state update, or blocking request path
- blank screen → missing loading state or all-or-nothing rendering
- layout jumps → unstable placeholder or late section size changes
- scroll hitches → expensive cell layout, image decoding, rendering, or main-thread work
- animation hitches → main-thread work, layout invalidation, rendering cost, or compositing
- slow transition → synchronous work during navigation or initial rendering
- task continues after screen disappears → missing cancellation or owner-scoped task cleanup

## Production Signals

Use production signals when the issue appears only in the wild, depends on real data, or needs trend tracking across devices and app versions.

Useful production signals:

- Xcode Organizer hangs
- Xcode Organizer hitches
- MetricKit payloads
- MetricKit hang diagnostics
- custom signposts
- custom screen-stage timings
- backend timing correlated with client stages
- app version trends
- device-class breakdowns
- OS-version breakdowns
- feature-level metrics
- user reports with reproduction details

Recommended breakdowns:

- app version
- device model or device class
- OS version
- screen or feature
- cold vs warm start
- first load vs refresh
- network type when available
- logged-in state or dataset size when relevant

The agent can suggest instrumentation and analyze provided production summaries. It cannot access production data unless the user provides it or the environment has a connected source.

## Custom Instrumentation

Perceived-performance improvements are easier to validate when important stages are named.

Consider logging or signposting:

- user action received
- loading state shown
- first placeholder shown
- first meaningful content shown
- primary action enabled
- secondary content loaded
- refresh started
- refresh completed
- optimistic update applied
- backend confirmation received
- rollback shown
- high-stakes submission started
- final confirmation shown
- unknown outcome detected

Example:

```swift
enum ScreenStage: String {
    case actionReceived
    case loadingShown
    case firstContentShown
    case primaryActionEnabled
    case fullyLoaded
}
```

Use instrumentation to compare before and after changes. Do not log sensitive data. For financial, medical, identity, or security-sensitive flows, review privacy and compliance requirements before adding telemetry.

## Validation by Pattern

Use different validation depending on the practice.

### Progressive rendering

Validate:

- time to first meaningful content
- blank-screen duration
- layout jumps during staged updates
- slow secondary section behavior
- section-level error handling
- stale-while-refreshing behavior

Evidence:

- screen recording
- slow network testing
- state-transition UI tests
- signposts for first content and full content

### Loading states

Validate:

- tap to first visible feedback
- loading vs empty distinction
- error and retry flow
- refresh transition
- placeholder stability
- accessibility labels or announcements when needed

Evidence:

- screen recording
- UI tests
- VoiceOver review
- failure simulation
- slow network testing

### Optimistic updates

Validate:

- immediate local feedback
- pending state
- backend success
- backend failure
- rollback or reconciliation
- repeated taps
- out-of-order responses
- app restart if pending state persists
- offline behavior if supported

Evidence:

- unit tests for state reducer
- UI tests for interaction flow
- mocked API failure
- mocked delayed response
- logs for mutation IDs

### High-stakes actions

Validate:

- confirmation before destructive/significant action
- submitting or authorizing state
- duplicate-submission guard
- authoritative success only after server response
- pending state
- failed state
- unknown outcome
- retry safety
- timeout behavior
- app interruption

Evidence:

- UI tests
- mocked backend states
- manual QA scenarios
- product/legal/security review when needed
- server idempotency confirmation when duplicate submission is harmful

## Review Checklist

Before finalizing a validation recommendation, check:

- [ ] Is the claim code-level, local runtime, automated-test, profiling, or production evidence?
- [ ] Did the answer avoid claiming unmeasured improvement?
- [ ] Is the validation method appropriate for the practice?
- [ ] Is release-like build validation recommended for performance-sensitive claims?
- [ ] Are older devices suggested when device headroom matters?
- [ ] Is Low Power Mode described only as a manual stress signal?
- [ ] Are screen recordings suggested for perceived timing and visual stability?
- [ ] Are scrolling and animation flows validated when relevant?
- [ ] Are failure, retry, and offline cases included when relevant?
- [ ] Are high-stakes unknown outcomes tested?
- [ ] Are production signals suggested for issues seen in the wild?
- [ ] Is instrumentation suggested when before/after comparison would otherwise be vague?
- [ ] Are privacy and compliance concerns mentioned for telemetry in sensitive flows?
- [ ] Does the answer distinguish what the agent can do from what needs manual/runtime validation?

## Review Output Guidance

When using this reference, explain:

```markdown
## Validation scope

State whether the recommendation is based on code inspection, local runtime evidence, profiling evidence, automated tests, or production data.

## What the agent can do

List code changes, instrumentation, tests, or analysis that can be performed from the available repository or artifacts.

## What needs runtime validation

List device, release-build, Low Power Mode, recording, Instruments, or production checks that require execution or user-provided evidence.

## Suggested validation plan

Provide the smallest useful validation plan for the specific pattern: progressive rendering, loading states, optimistic updates, or high-stakes actions.

## Evidence to collect

Name the metrics, recordings, traces, logs, or production signals that would prove or disprove the claim.
```

Use careful language. Prefer “validate,” “measure,” “inspect,” and “compare” over “prove” unless the evidence is actually available.
