# AppDelegate, SceneDelegate, and First Frame

Use this reference when a launch investigation points to application lifecycle code, scene lifecycle code, dependency setup during startup, root UI creation, first-screen preparation, or main-thread work that delays the first visible app frame.

Keep this reference focused on work that happens after the app enters its own lifecycle code and before the first useful UI becomes visible and responsive. Do not use it as the primary guide for dyld/pre-main, framework linking strategy, SwiftUI `@main App` startup, third-party SDK policy, or measurement tool setup.

## Scope Boundary

This file covers:

- `UIApplicationDelegate` launch callbacks
- `UISceneDelegate` scene connection callbacks
- root `UIWindow` and root view controller setup
- first-screen view controller creation
- dependency setup that happens in app or scene lifecycle code
- main-thread startup work before first frame
- state restoration, launch options, deep links, notifications, and routing when they affect first-screen readiness
- deferral decisions for app-owned launch work

This file does not cover:

- dyld, pre-main, Objective-C `+load`, `+initialize`, constructors, or static initializer internals; use `pre-main-dyld-and-static-initializers.md`
- static vs dynamic linking, mergeable libraries, or framework count; use `linking-strategy.md`
- SwiftUI `@main App`, root SwiftUI view initialization, `.task`, `.onAppear`, or observable state setup; use `swiftui-app-launch.md`
- SDK-specific startup policy for analytics, ads, crash reporting, attribution, push, remote config, feature flags, or security vendors; use `third-party-sdks-at-launch.md`
- Instruments, XCTest, MetricKit, Organizer, signpost design, or CI baseline setup; use `metrics-instruments-xctest-metrickit.md`
- general scrolling, rendering, memory, networking, or architecture review unless the work is on the launch path

## Core Model

After pre-main work completes, UIKit starts the app lifecycle and gives the app a chance to configure shared process-level state, connect scenes, build a window, create the first screen, and submit the first frame.

For launch performance, treat this area as a critical path:

```text
UIApplicationDelegate startup
→ optional scene session configuration
→ UISceneDelegate scene connection
→ window and root controller creation
→ first screen model/state preparation
→ first layout/draw/commit
→ early responsiveness after the first frame
```

The goal is not to make lifecycle methods empty. The goal is to ensure that only work required for a correct first frame or first interaction stays on this path.

## Lifecycle Responsibilities

### App Delegate

Use the app delegate for process-level startup and app-wide lifecycle coordination.

Review code in:

- `application(_:willFinishLaunchingWithOptions:)`
- `application(_:didFinishLaunchingWithOptions:)`
- `application(_:configurationForConnecting:options:)`
- app delegate initializers and stored property initialization
- custom bootstrap objects called from app delegate methods

Typical launch-safe responsibilities:

- install a minimal root coordinator or bootstrap object
- configure app-wide behavior that must exist before scenes connect
- decide high-level launch routing state when it is cheap and local
- register critical services that must observe launch or crash immediately
- return quickly enough for scene connection and first UI creation to continue

Common launch risks:

- building the full application dependency graph synchronously
- opening databases before the first screen needs them
- running migrations on the main thread
- performing keychain-heavy checks synchronously
- waiting for network, remote config, reachability, token refresh, or feature flag fetches
- parsing large local files
- registering many observers, notification handlers, or routers eagerly
- initializing every subsystem because it is convenient from one central method
- doing UI work in both app delegate and scene delegate for the same window

### Scene Delegate

Use the scene delegate for scene-specific UI creation and scene lifecycle coordination.

Review code in:

- `scene(_:willConnectTo:options:)`
- `sceneWillEnterForeground(_:)`
- `sceneDidBecomeActive(_:)`
- custom scene coordinators called during connection
- root window and root controller factories

Typical launch-safe responsibilities:

- create a `UIWindow` for the connected `UIWindowScene`
- create a minimal root controller or root coordinator
- apply scene-specific routing input from connection options
- make the window visible
- schedule noncritical scene work outside the first-frame path

Common launch risks:

- duplicating root UI construction already done by the app delegate
- doing heavy routing before any placeholder/root UI exists
- creating multiple complete navigation stacks during startup
- restoring complex UI state synchronously
- loading large first-screen data sets before showing any UI
- performing synchronous image loading, decoding, or layout-heavy setup
- treating every scene connection as a fresh app launch without checking the scenario

## What the Agent Can Inspect

When repository access is available, inspect concrete launch-path code instead of giving generic advice.

Search for lifecycle entry points:

```sh
rg "didFinishLaunching|willFinishLaunching|configurationForConnecting|willConnectTo|sceneDidBecomeActive|sceneWillEnterForeground" .
```

Search for common startup work inside lifecycle files and bootstrap code:

```sh
rg "bootstrap|configure|start|initialize|setup|register|migrate|open|load|restore|resolve|container|coordinator" .
```

Search for blocking or expensive operations near startup paths:

```sh
rg "Data\(|contentsOf:|FileManager|Keychain|SecItem|UserDefaults|JSONDecoder|PropertyListDecoder|sleep|wait\(|semaphore|sync\(|performAndWait|DispatchQueue\.main\.sync" .
```

Search for root UI creation and duplication:

```sh
rg "UIWindow\(|rootViewController|makeKeyAndVisible|UINavigationController\(|UITabBarController\(|UIHostingController" .
```

Use search results carefully. A match is a lead, not proof of launch impact. Confirm whether the matched code actually runs before first frame and whether it blocks the main thread.

The agent can:

- trace lifecycle call chains from app/scene delegates into bootstrap code
- identify synchronous work on the launch path
- classify startup tasks by necessity
- suggest moving work to explicit readiness points
- propose lazy initialization for feature-scoped services
- suggest minimal root UI or placeholder state where appropriate
- add or recommend lightweight app-owned phase markers when measurement is missing
- propose code patches if the repository is available and the change is local and safe

The agent cannot reliably:

- prove first-frame timing without a trace or measurement
- assume `DispatchQueue.main.async` means work runs after the first displayed frame
- know third-party SDK internal launch cost without documentation, symbols, traces, or vendor guidance
- decide that a security, crash reporting, routing, or compliance requirement can be deferred without product/context validation
- convert a synchronous dependency into async/lazy setup without checking call-site semantics

## Startup Task Classification

Before recommending deferral, classify each task by when the app truly needs it.

### Required before first frame

Keep only work that is necessary to show a correct first visible UI.

Examples:

- creating the window and minimal root controller
- selecting the initial shell or high-level route from already-local state
- reading tiny, already-local state that determines whether the first screen is login, locked, onboarding, or main shell
- setting critical app-wide invariants required before any UI code runs
- starting a crash reporter or security component when the product requires it before any user interaction

Even required work should be reviewed for main-thread blocking, unnecessary breadth, and avoidable eager dependency construction.

### Required before first interaction

Some work does not need to block first frame but should complete before the user taps or before the first screen becomes fully interactive.

Examples:

- preparing the first screen's minimal view model state
- verifying a local session expiration timestamp
- restoring lightweight navigation context
- loading cached data needed for immediate controls
- enabling critical observers for first-screen actions

Move this work out of `didFinishLaunching` when possible, but ensure the UI has a clear loading, disabled, or placeholder state if interaction depends on it.

### Needed soon after launch

Some work is useful shortly after launch but should not delay first frame.

Examples:

- refreshing cached user data
- warming small caches used by the first few screens
- scheduling noncritical observers
- refreshing local configuration that has a safe default
- preparing background sync state

Prefer explicit post-visible or post-first-interaction scheduling, and verify that the work does not create early interaction jank.

### Feature-specific and lazy

Feature-specific services should start when the feature is opened or when its data is actually needed.

Examples:

- search index setup for a screen not shown at launch
- map, camera, media, or payment modules not visible on the first screen
- secondary tab data providers
- optional personalization engines
- admin/debug tools

Do not initialize feature modules from app delegate simply because they are globally reachable.

### Background maintenance

Maintenance tasks should not block visible UI.

Examples:

- cache cleanup
- log compaction
- old file deletion
- database vacuuming
- analytics batch upload
- nonurgent sync repair

Schedule these with appropriate priority, cancellation, and battery/network awareness. Avoid starting all maintenance immediately after first frame if it competes with early user interaction.

## AppDelegate Review Rules

When reviewing app delegate code, ask these questions:

1. Does this method create only the app-wide state needed before the first scene or first frame?
2. Does any call chain perform synchronous I/O, network waiting, database work, keychain access, migration, parsing, locking, or large allocation?
3. Is a full dependency graph built when only a small root subset is needed?
4. Are launch options handled cheaply, or do they trigger heavy routing and data loading?
5. Is UI setup duplicated with scene delegate code?
6. Is the app doing work here only because this method is a convenient central place?
7. Is the work safe to move later, lazy-load, or split into critical and noncritical parts?

Prefer:

- a thin app delegate
- explicit bootstrap objects with narrow responsibilities
- cheap local state reads over remote decisions
- small root dependency sets
- lazy service registration for feature-specific services
- clear boundaries between process-level startup and scene-specific UI

Avoid:

- a single `configureEverything()` call with hidden work
- unconditional service startup for every app subsystem
- blocking the main thread while waiting for remote state
- long synchronous migrations during launch
- creating UI in multiple lifecycle locations for the same scene
- using app delegate as a general dependency container initializer

## SceneDelegate Review Rules

When reviewing scene delegate code, ask these questions:

1. Does `scene(_:willConnectTo:options:)` create only the scene-specific UI needed now?
2. Can the root controller appear before heavy data or secondary modules are ready?
3. Does scene connection restore too much navigation or data synchronously?
4. Are connection options handled without blocking visible UI?
5. Is each scene treated independently when the app supports multiple windows?
6. Does a returning or additional scene accidentally rerun app-wide bootstrap?
7. Are root controllers and coordinators cheap to construct?

Prefer:

- a minimal root controller or shell
- route selection from cheap local state
- progressive first-screen loading
- scene-specific coordinators with explicit dependencies
- delayed restoration for heavy secondary state
- idempotent scene setup

Avoid:

- recreating app-wide services per scene
- building every tab or navigation branch before the first frame
- performing blocking authentication refresh during scene connection
- loading large images, files, or data sets synchronously
- doing expensive view hierarchy construction before any lightweight root is visible

## Root UI and First Frame

The first frame should be minimal, valid, and visually stable. It does not need to contain every final piece of data.

Review root UI creation for:

- expensive root view controller initializers
- view models that start work in `init`
- coordinators that construct many screens immediately
- tab controllers that eagerly build every tab's full content
- navigation stacks restored with heavy screens before any shell appears
- synchronous image decoding or asset processing
- complex Auto Layout setup on the first screen
- first-screen tables or collections populated with large data sets synchronously
- blocking state restoration

Prefer first-frame strategies such as:

- shell-first UI with progressive data loading
- lightweight login/locked/onboarding/main-shell decision
- cached or placeholder state when remote data is not required for correctness
- lazy tab or feature construction
- minimal root view model state
- delayed enrichment after visible UI
- explicit loading states instead of blocking launch

Be careful: showing an empty or broken shell is not a launch improvement. The first frame must still be correct for the user's state and safe for the product domain.

## Routing, Launch Options, and State Restoration

Launch options, universal links, notifications, shortcuts, handoff, and restoration can force early routing decisions. Handle them without turning launch into full feature initialization.

Recommended approach:

1. Parse launch input cheaply.
2. Decide the minimal initial route or pending route.
3. Show a valid root UI quickly.
4. Resolve expensive data or authorization checks asynchronously when the UI has a safe loading state.
5. Complete deep-link or notification navigation only after required state is available.

Review for:

- deep-link handlers that load full feature modules before root UI appears
- notification handlers that fetch remote data synchronously during launch
- restoration code that recreates complex screens before showing a shell
- authentication refresh that blocks route selection
- dependency cycles between router, session manager, and root UI creation

Prefer pending-route models over blocking route completion during `didFinishLaunching` or scene connection.

## Dependency Setup on the Launch Path

Dependency containers are common launch bottlenecks because they make it easy to construct the whole app graph at once.

Review for:

- container `build`, `registerAll`, `resolveAll`, or `assemble` calls during launch
- services registered by constructing concrete instances instead of factories
- eager singletons that start I/O or threads during registration
- root view models that resolve broad dependencies
- feature modules registered before they are reachable
- hidden work in property initializers of dependency objects

Prefer:

- registration of lightweight factories over eager instances
- separating launch-critical services from feature services
- constructing only the root dependency slice needed for the first screen
- lazy module loading for secondary features
- explicit async preparation steps after first UI when needed
- narrow protocols for first-screen dependencies

Avoid turning dependency setup into global hidden work. If the dependency graph is too broad, splitting it often improves both launch performance and architectural clarity.

## Main-Thread Work

App delegate and scene delegate callbacks run as part of startup and commonly lead to main-thread work. Main-thread work is not automatically bad: UI setup must happen there. The problem is blocking, CPU-heavy, I/O-heavy, or unnecessarily broad work that prevents first frame or early interaction.

High-risk patterns:

- synchronous file reads or writes
- keychain calls on the critical path
- database open/migration on the main thread
- large JSON or property list decoding
- image decoding or resizing
- waiting on semaphores, groups, locks, or synchronous dispatch
- synchronous network wrappers
- expensive logging or analytics formatting
- large collection diffing before first frame
- rebuilding localization, theme, or configuration objects with broad side effects

When moving work off the main thread, verify:

- the API is thread-safe
- UI state is updated on the correct actor/thread
- the first screen has a valid loading state
- cancellation and priority are appropriate
- background work does not immediately compete with early interaction

Do not recommend backgrounding work simply to hide it. Work that is required for correctness may need a better first-frame design rather than an unsafe thread move.

## Deferral Patterns

Use explicit deferral points instead of vague "do it later" advice.

Possible deferral points:

- after minimal root UI is visible
- after first screen appears
- after first user interaction
- after authentication/session state is known
- after the first critical data request completes
- when a specific feature is opened
- when the app is idle or in a background-friendly maintenance window

When suggesting deferral, specify:

- what moves
- why it is not needed before first frame
- what state the UI shows before it completes
- what event triggers it
- how failure is handled
- how the improvement will be measured

Avoid relying on `DispatchQueue.main.async` as proof that work happens after first frame. It only moves work out of the current synchronous call stack. Use lifecycle points, explicit readiness signals, or measurement to confirm timing.

## Safe Patch Heuristics

When the agent is allowed to edit code, prefer small, reversible changes.

Good patch candidates:

- split a large startup method into named phases
- move noncritical work behind an explicit method called after root UI setup
- replace eager service construction with a factory when call sites already tolerate laziness
- add a lightweight placeholder state for first screen data
- delay feature module creation until route selection actually needs it
- add app-owned phase markers around lifecycle and root UI construction
- remove duplicate root UI setup when the correct lifecycle owner is clear

Risky patch candidates requiring extra care:

- changing authentication, security, crash reporting, payment, or compliance startup order
- making a synchronous API async across many call sites
- moving database migration or keychain logic without checking data correctness
- changing deep-link routing order
- changing multi-window behavior
- replacing root navigation architecture
- deferring feature flags or remote config when they gate visible behavior

If correctness is uncertain, recommend a minimal instrumentation or decomposition patch first, then a behavior-changing optimization after evidence is available.

## Review Checklist

Use this checklist when reviewing lifecycle startup code.

- [ ] Is the launch scenario known, or could this be resume/prewarmed/warm launch confusion?
- [ ] Is app-wide bootstrap separated from scene-specific UI creation?
- [ ] Does `didFinishLaunching` avoid broad synchronous initialization?
- [ ] Does `scene(_:willConnectTo:options:)` create a minimal valid root UI?
- [ ] Is root UI creation performed in one lifecycle owner, not duplicated?
- [ ] Are launch options and deep links parsed cheaply before expensive route completion?
- [ ] Are dependency containers registering factories instead of constructing the whole graph?
- [ ] Are first-screen view models cheap to create?
- [ ] Are database, keychain, file, parsing, and migration work absent from the first-frame critical path unless required?
- [ ] Are background tasks scheduled with priority and correctness in mind rather than all started immediately?
- [ ] Does every deferred task have a clear trigger and UI state before completion?
- [ ] Is the recommendation connected to a launch phase and validation plan?

## Output Guidance for This Reference

When this reference is used, report findings in this shape:

```markdown
### Lifecycle phase
AppDelegate / SceneDelegate / root UI / first frame / early responsiveness.

### Critical-path work
List concrete calls or responsibilities that appear to block first frame or first interaction.

### Necessity classification
Classify each task as first-frame, first-interaction, soon-after-launch, feature-lazy, or maintenance.

### Recommended change
Describe the smallest safe change and the deferral point or lazy boundary.

### Correctness risk
Mention anything that may affect routing, auth, crash reporting, security, state restoration, or multi-window behavior.

### Validation
Say what trace, signpost, launch metric, or manual comparison should improve.
```
