# SwiftUI App Launch

Use this reference when a launch investigation involves the SwiftUI app lifecycle: `@main App`, root `Scene` construction, root view setup, observable state creation, environment injection, `.task`, `.onAppear`, `scenePhase`, or SwiftUI-to-UIKit delegate bridging.

Keep this reference focused on SwiftUI-specific startup behavior. Do not use it as the primary guide for dyld/pre-main work, framework linking, UIKit app/scene delegate implementation, third-party SDK policy, or launch measurement tooling.

## Scope Boundary

This file covers:

- SwiftUI `@main App` startup code
- `App.init` and stored property initialization in the app type
- `body` and root `Scene` construction
- `WindowGroup` root view creation
- root view `init` and observable model creation
- eager `@StateObject`, `@Observable`, `ObservableObject`, and environment setup
- `.task`, `.task(id:)`, `.onAppear`, and `scenePhase` work that starts near launch
- `@UIApplicationDelegateAdaptor` usage in SwiftUI lifecycle apps
- SwiftUI-specific first-screen readiness and post-visible work classification

This file does not cover:

- dyld, pre-main, Objective-C `+load`, constructor functions, or static initializer internals; use `pre-main-dyld-and-static-initializers.md`
- static vs dynamic linking, mergeable libraries, or framework dependency strategy; use `linking-strategy.md`
- UIKit `UIApplicationDelegate` / `UISceneDelegate` implementation details, root `UIWindow`, or `UIViewController` first-frame setup; use `appdelegate-scenedelegate-and-first-frame.md`
- SDK-specific launch policy for analytics, ads, crash reporting, attribution, push, remote config, feature flags, or security vendors; use `third-party-sdks-at-launch.md`
- Instruments, Time Profiler, XCTest, MetricKit, Organizer, signpost design, or CI baselines; use `metrics-instruments-xctest-metrickit.md`
- general SwiftUI scrolling, diffing, view invalidation, memory, or animation performance unless the issue is on the launch path

## Core Model

In a SwiftUI lifecycle app, the app type is the entry point. SwiftUI creates the `App` instance, evaluates its scenes, creates the root view hierarchy for the active scene, and drives lifecycle callbacks through SwiftUI view and scene mechanisms.

For launch performance, treat this as a critical path:

```text
@main App creation
→ App stored property initialization
→ App.init
→ body / Scene construction
→ WindowGroup root view creation
→ root observable state and environment setup
→ first root view evaluation/layout/draw
→ lifecycle-triggered startup work
→ early responsiveness
```

The goal is not to avoid all initialization in SwiftUI. The goal is to keep the first visible shell cheap, deterministic, and correct while moving secondary work to explicit, measurable readiness points.

## What the Agent Can Inspect

When repository access is available, inspect SwiftUI launch entry points and root dependencies before giving generic advice.

Search for SwiftUI app entry points:

```sh
rg "@main|: App|WindowGroup|DocumentGroup|Settings\s*\{" .
```

Search for UIKit bridge points in SwiftUI lifecycle apps:

```sh
rg "UIApplicationDelegateAdaptor|NSApplicationDelegateAdaptor|WKApplicationDelegateAdaptor|ExtensionDelegateAdaptor" .
```

Search for root startup hooks:

```sh
rg "\.task\s*\{|\.task\s*\(|\.onAppear\s*\{|scenePhase|onChange\s*\(of:.*scenePhase" .
```

Search for eager root state and dependency injection:

```sh
rg "@StateObject|@Observable|ObservableObject|@EnvironmentObject|\.environment\(|\.environmentObject\(|modelContainer|container|resolver|dependencies" .
```

Search for expensive work that may happen in initializers or root tasks:

```sh
rg "init\s*\(|Data\(|contentsOf:|FileManager|Keychain|SecItem|JSONDecoder|PropertyListDecoder|migrate|open|load|fetch|refresh|sync|wait\(|semaphore|DispatchQueue\.main\.sync|Task\.detached" .
```

Use search results as leads. Confirm whether the matched code is reached during launch, whether it runs before the first visible UI, and whether it blocks the main actor or a critical dependency required by the first screen.

The agent can:

- trace the SwiftUI launch path from `@main App` to the first root view
- identify heavy stored properties, `App.init` work, root model construction, and environment setup
- classify root `.task`, `.onAppear`, and `scenePhase` work by necessity
- suggest splitting launch-critical state from secondary app state
- suggest lazy creation of feature-specific services and root view models
- propose idempotent startup gates for view lifecycle callbacks that may run more than once
- recommend moving UIKit delegate responsibilities to the appropriate reference when bridging is involved
- propose small code patches when the repository is available and the change is local and behavior-preserving

The agent cannot reliably:

- prove first-frame timing without measurement or a trace
- assume `.task` always starts after the first frame is displayed
- assume `.onAppear` runs only once
- assume `scenePhase == .active` means a fresh app launch
- move UI-bound work away from the main actor without checking isolation and API requirements
- defer authentication, routing, security, crash reporting, or compliance work without product context
- know whether a SwiftUI root model initializer is expensive without inspecting its call graph

## SwiftUI Launch Review Areas

### `App` stored properties and `App.init`

Review the `@main App` type first. Stored properties and `init` are easy places to hide launch work because they look like simple declarations.

Look for:

- dependency containers created eagerly
- global application coordinators that start subsystems in their initializer
- database or model container setup that happens before the root view appears
- keychain/session reads that expand into slow call chains
- remote configuration, network refresh, or token refresh started from `App.init`
- analytics, logging, push, attribution, or feature flag SDK startup in the app type
- work hidden behind `shared`, `default`, `live`, `make`, `bootstrap`, `configure`, or `start` names

Prefer:

- a small app type that builds only the first visible shell
- cheap local state used only to choose the initial shell or route
- explicit bootstrap phases rather than hidden work in property initialization
- services created lazily when a feature or first interaction needs them
- a narrow launch model instead of a full application graph

Avoid:

- starting every subsystem because `App.init` feels like the only central place
- doing blocking I/O before any scene can render
- making a root dependency container resolve all services eagerly
- hiding expensive work behind property wrappers or static factories

Example risk pattern:

```swift
@main
struct FinanceShellApp: App {
    private let graph = ApplicationGraph.live()

    init() {
        graph.startAllServices()
    }

    var body: some Scene {
        WindowGroup {
            RootScreen(graph: graph)
        }
    }
}
```

Review this as a launch risk even if `startAllServices()` looks clean. The agent should inspect what `ApplicationGraph.live()` and `startAllServices()` actually do.

A safer direction is to make the first shell cheap and move secondary services behind explicit readiness or feature boundaries:

```swift
@main
struct FinanceShellApp: App {
    @StateObject private var launchState = LaunchState()

    var body: some Scene {
        WindowGroup {
            LaunchShell(state: launchState)
                .task {
                    await launchState.prepareNonBlockingStartup()
                }
        }
    }
}
```

This is only safer if `LaunchState.init` is cheap and `prepareNonBlockingStartup()` does not immediately perform long main-actor work. Validate with measurement.

### Root `Scene` and `WindowGroup`

Review the content closure used to create the first SwiftUI hierarchy.

Look for:

- root views that construct many feature modules immediately
- conditional branches that build multiple heavy alternatives before one is selected
- expensive environment values computed inline
- root modifiers that trigger persistence, networking, or SDK setup
- model containers or persistence setup attached at the root without checking whether the first screen needs them
- root navigation state restored synchronously from large local data

Prefer:

- a lightweight shell that can render with minimal local state
- progressive loading for first-screen content
- route decisions based on cheap, already-local information
- dependency injection that passes handles/factories rather than fully initialized feature graphs
- separate first-screen dependencies from later-tab or later-feature dependencies

Avoid treating `WindowGroup` as a replacement for an application bootstrap method. It describes scenes; it should not become a hidden service startup hub.

### Root view initialization

SwiftUI views are values and may be recreated frequently. The launch problem is not that a view has an initializer. The problem is doing expensive, side-effectful, or blocking work while constructing the root hierarchy.

Look for root view initializers that:

- synchronously load files or decode large data
- open databases or perform migrations
- touch keychain or secure storage repeatedly
- resolve many services from a container
- create image caches, formatters, search indexes, or data stores eagerly
- start tasks, timers, notifications, or observers as side effects
- compute first-screen content that could be cached, placeholder-based, or asynchronous

Prefer initializers that only assign already-prepared dependencies or cheap value state.

If setup is required, move it to a model or coordinator with an explicit method and clear lifecycle. Then decide whether that method must run before first frame, before first interaction, or later.

### Observable root state

Review root observable objects and models because their initializers often become launch bottlenecks.

Check:

- `@StateObject` initializers in the root view
- `@Observable` models created in the app or root scene
- `ObservableObject` models injected as environment objects
- global stores or app state objects passed through the root environment
- model initializers that resolve services, fetch data, or subscribe to many publishers/notifications

Prefer:

- cheap model initialization
- explicit async loading after construction
- small launch-specific state separate from full app state
- lazy child models for screens not visible at launch
- cached state for the first visible UI

Avoid:

- making the root app state object own every subsystem immediately
- starting network refreshes, data sync, or cache cleanup in model `init`
- using `@StateObject` as a place to hide expensive startup work
- injecting one massive environment object that constructs everything before the first screen

When suggesting changes, preserve SwiftUI ownership rules. Do not replace `@StateObject` with `@ObservedObject` just to change launch timing. Fix the initializer or ownership boundary instead.

## Lifecycle-Triggered Work

### `.task`

Treat `.task` as lifecycle-bound asynchronous work. It can be a good place to start nonblocking work associated with a visible view, and SwiftUI cancels the task when the view leaves the hierarchy. However, it is not a guarantee that all work starts after the first frame is displayed.

Review `.task` closures for:

- long main-actor sections
- synchronous work before the first `await`
- immediate network calls needed only by later features
- repeated execution because the view is recreated or reinserted
- `.task(id:)` restarts caused by unstable identifiers
- cancellation handling and idempotency
- task priority that competes with early interaction

Prefer:

- short setup before the first suspension point
- nonblocking async work with cancellation support
- stable `.task(id:)` values when restart behavior is intended
- explicit guards for one-time startup when one-time semantics are required
- moving heavy feature work to the feature view instead of the root shell

Do not describe `.task` as automatically post-render. Say that it is asynchronous and lifecycle-bound, then require measurement if the launch impact matters.

### `.onAppear`

Treat `.onAppear` as a visibility callback, not as a one-time launch hook.

Review `.onAppear` closures for:

- repeated startup work when navigation or scene changes reinsert the view
- synchronous main-thread work
- unstructured `Task` creation without cancellation ownership
- duplicate calls also triggered by `.task` or `scenePhase`
- state mutations that cause immediate expensive re-rendering of the root hierarchy

Prefer `.task` for cancellable async work tied to a view lifecycle. Prefer an explicit model/coordinator method with idempotency when the work must run once per app session.

### `scenePhase`

Use `scenePhase` for scene lifecycle reactions, not as a blanket launch detector.

Review `scenePhase` handlers for:

- running launch setup on every `.active` transition
- treating foreground resume as cold launch
- refreshing too much data immediately when returning from background
- duplicating work already started from `.task` or `App.init`
- doing heavy work while the first scene is trying to become interactive

Prefer:

- separate handling for first launch, foreground resume, and background maintenance
- debounced or freshness-based refresh policies
- cheap foreground checks followed by lazy or cancellable refreshes
- explicit state that records whether initial setup already completed

## Delegate Bridging in SwiftUI Lifecycle Apps

SwiftUI lifecycle apps can still bridge to UIKit delegate callbacks with delegate adaptor property wrappers. Treat this bridge as part of the launch path when the adapted delegate starts work during launch.

Review:

- `@UIApplicationDelegateAdaptor` declarations
- adapted delegate initializers
- `application(_:didFinishLaunchingWithOptions:)` in the adapted delegate
- push notification, deep link, security, crash reporting, or SDK setup performed through the delegate
- duplicated initialization between `App.init`, root `.task`, and delegate callbacks

Do not move all delegate logic into SwiftUI views. Use the app/scene delegate reference for UIKit lifecycle responsibilities and the third-party SDK reference for vendor startup policy. In this SwiftUI reference, focus on whether bridging creates hidden eager work or duplicate launch paths.

## First-Screen Readiness in SwiftUI

For launch review, distinguish between rendering a valid first shell and completing all app data loading.

A valid first SwiftUI shell may contain:

- a local session decision
- a locked, login, onboarding, or main shell route
- cached first-screen data
- a placeholder or skeleton state
- disabled controls with clear loading state
- minimal navigation chrome

It does not need to contain:

- refreshed remote configuration when safe defaults exist
- fully synchronized user data
- secondary tab data
- personalized recommendations
- warmed caches for later screens
- feature modules not visible on the first screen
- maintenance work such as cleanup, compaction, or log upload

Be careful with authentication and routing. If the first screen must not be shown until a local security decision is made, keep that decision launch-critical but make it as small and local as possible. Do not replace a required security check with a visual shortcut.

## Review Questions

When reviewing SwiftUI launch code, ask:

1. What is the first SwiftUI view that must become visible?
2. Which state is strictly required to choose that view?
3. Which stored properties and initializers run before that view can be created?
4. Which root models are created eagerly, and what do their initializers do?
5. Which environment values are computed eagerly?
6. Which `.task`, `.onAppear`, or `scenePhase` callbacks start during launch or immediately after the first scene appears?
7. Can this work run more than once because of SwiftUI lifecycle behavior?
8. Is heavy work split into launch-critical, first-interaction, post-visible, and feature-lazy phases?
9. Is any UIKit delegate bridge duplicating work already started from SwiftUI?
10. What measurement will prove that the change improves first frame or early responsiveness?

## Common Findings and Recommended Direction

### Heavy `App.init`

Finding:

- The app type constructs a full graph or starts services before any scene can render.

Recommended direction:

- Keep `App.init` minimal.
- Create only launch-critical state.
- Move secondary startup to explicit async phases or feature-owned lazy initialization.
- Route SDK-specific decisions to `third-party-sdks-at-launch.md`.

### Heavy root observable model

Finding:

- The root `@StateObject` or app state model performs I/O, data fetches, migrations, or broad service registration in `init`.

Recommended direction:

- Make initialization cheap.
- Move work to explicit methods.
- Start only the subset required for first screen correctness.
- Use cached or placeholder state when possible.

### Root `.task` does too much

Finding:

- A root `.task` starts many operations immediately, including work not needed for the first screen.

Recommended direction:

- Split the task into smaller phases.
- Keep the first phase short and cancellable.
- Move feature-specific operations to the feature view or feature coordinator.
- Avoid long main-actor sections.
- Validate early responsiveness, not only time to first frame.

### `.onAppear` used as a launch hook

Finding:

- `.onAppear` starts one-time app setup and can run more than once.

Recommended direction:

- Move one-time session setup to an explicit app/session model with idempotency.
- Use `.task` for cancellable view-bound async work.
- Guard repeated work with state only when repeated execution would be incorrect or expensive.

### `scenePhase` refresh causes resume jank

Finding:

- The app refreshes large state on every `.active` transition.

Recommended direction:

- Separate launch from resume.
- Check freshness before refresh.
- Make resume work cancellable and priority-aware.
- Do not block early interaction on noncritical refresh.

### Environment injection builds too much

Finding:

- Root environment injection creates a complete dependency graph or many environment objects before the first screen.

Recommended direction:

- Inject factories, lightweight handles, or launch-specific dependencies.
- Create feature dependencies when the feature becomes visible.
- Avoid storing full app graphs in environment unless construction is cheap and measured.

## Patch Guidance

When proposing code changes, prefer small, behavior-preserving patches:

- split `App.init` startup into critical and noncritical phases
- replace eager root service construction with lazy factories
- move heavy root model work from `init` to an explicit async method
- add idempotency to root startup methods that are triggered by view lifecycle
- move feature-specific startup from root to feature entry points
- add lightweight app-owned signposts around SwiftUI startup phases if measurement is missing

Do not patch by:

- moving work to a detached task without ownership, cancellation, or priority reasoning
- replacing synchronous security/auth/routing decisions with unsafe placeholders
- changing SwiftUI state ownership wrappers without checking lifecycle semantics
- hiding work behind a different initializer or singleton
- making all startup work background work without considering first interaction correctness

## Validation Handoff

This reference can recommend what to validate, but tool-specific details belong in `metrics-instruments-xctest-metrickit.md`.

For SwiftUI launch changes, validation should answer:

- Did the first visible frame become earlier?
- Did early responsiveness improve, stay the same, or regress?
- Did root `.task` / `.onAppear` work move out of the critical path or merely shift jank later?
- Did the change accidentally duplicate work on resume or scene recreation?
- Did cancellation and idempotency still behave correctly?
- Did authenticated, unauthenticated, first-install, returning-user, deep-link, and notification launches still route correctly?
