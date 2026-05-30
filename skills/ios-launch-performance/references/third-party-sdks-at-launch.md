# Third-Party SDKs at Launch

Use this reference when a launch investigation involves vendor SDKs or app-owned wrappers around vendor SDKs: analytics, crash reporting, ads, attribution, remote config, push, feature flags, experimentation, security, fraud prevention, consent, logging, monitoring, or other services initialized during startup.

Keep this file focused on **SDK initialization strategy**. Do not use it as the primary guide for dyld internals, static vs dynamic linking, app/scene lifecycle mechanics, SwiftUI root setup, or measurement tooling.

## Scope Boundary

This file covers:

- whether a vendor SDK must start before first frame
- whether an SDK can be split into minimal configuration and deferred startup
- SDK startup in `AppDelegate`, `SceneDelegate`, SwiftUI app entry points, dependency containers, and bootstrap coordinators
- analytics, logging, crash reporting, ads, attribution, remote config, feature flags, push, security, fraud, A/B testing, experimentation, and monitoring SDKs
- launch-specific correctness risks caused by delaying a vendor
- vendor-supported lazy, deferred, offline, cached, minimal, or background initialization modes
- how to review SDK wrappers without guessing vendor behavior

This file does not cover:

- dyld loading, Objective-C `+load`, constructors, runtime registration, or static initializer mechanics; use `pre-main-dyld-and-static-initializers.md`
- static frameworks, dynamic frameworks, mergeable libraries, package manager layout, or shipped binary inspection; use `linking-strategy.md`
- general `UIApplicationDelegate`, `UISceneDelegate`, root window, first-screen routing, or main-thread launch work; use `appdelegate-scenedelegate-and-first-frame.md`
- SwiftUI `@main App`, root view, observable state, `.task`, or `.onAppear` behavior; use `swiftui-app-launch.md`
- Instruments, XCTest, MetricKit, Organizer, signpost design, or CI baseline setup; use `metrics-instruments-xctest-metrickit.md`
- general privacy, legal, compliance, or security architecture review unless the decision changes launch-time SDK startup

## Core Model

Every SDK initialized on the launch path should answer one question:

```text
What user-visible or correctness problem happens if this SDK starts after the first frame or after first interaction?
```

Do not assume that every SDK can be deferred. Also do not assume that every vendor instruction to initialize in `didFinishLaunching` means the whole SDK must perform all work synchronously before the first frame.

Many SDKs can be split into smaller phases:

```text
register lightweight configuration
→ install critical handlers or delegates
→ expose cached/default state
→ render the first UI
→ start network/session/upload/preload work later
→ initialize feature-specific modules on first use
```

The review goal is to keep only the truly launch-critical portion on the startup path and move the rest to explicit, measurable readiness points.

## What the Agent Can Inspect

When repository access is available, inspect real initialization sites instead of giving generic advice.

Search for startup code and bootstrap containers:

```sh
rg "didFinishLaunching|willFinishLaunching|configurationForConnecting|willConnectTo|@main|AppDelegate|SceneDelegate|Bootstrap|AppStartup|AppInitializer|DependencyContainer|ServiceLocator|Assembler|CompositionRoot" .
```

Search for common SDK categories and vendor wrappers:

```sh
rg "Analytics|Crash|Crashlytics|Sentry|Bugsnag|Firebase|Amplitude|Mixpanel|AppsFlyer|Adjust|Branch|AdMob|GoogleMobileAds|RemoteConfig|FeatureFlag|Experiment|ABTest|Push|Notification|OneSignal|Security|Fraud|Jailbreak|Attestation|Monitoring|Logger|Telemetry|Consent|Tracking" .
```

Inspect package and dependency manifests when they exist:

- `Package.swift`
- `Package.resolved`
- `Podfile`
- `Podfile.lock`
- `Cartfile`
- `Cartfile.resolved`
- Tuist, Bazel, Buck, or custom project generator manifests
- vendored `.framework` and `.xcframework` folders
- app target build settings and initialization scripts

Inspect app-owned wrappers first. Many codebases hide vendor startup behind names such as:

- `AnalyticsService`
- `CrashReporter`
- `RemoteConfigProvider`
- `FeatureFlagClient`
- `PushNotificationService`
- `AttributionService`
- `SecurityProvider`
- `FraudDetector`
- `MonitoringClient`
- `SDKManager`
- `ThirdPartyServices`
- `AppBootstrapper`

When vendor documentation is not available in the repository, state the assumption clearly and recommend checking the vendor's official initialization modes before changing startup order.

## SDK Startup Classification

Classify each SDK or SDK wrapper before recommending deferral.

### Launch-critical and synchronous

Use this classification only when delaying initialization can break correctness, security, crash capture, launch routing, or regulatory requirements.

Examples:

- crash reporter handler installation needed to catch crashes during startup
- security or fraud gate required before showing sensitive content
- deep link or notification router needed to decide the initial screen
- feature flag defaults required to choose the launch surface
- notification categories or delegates required before notification handling
- mandatory compliance or consent state needed before any tracking starts

Even then, review whether the synchronous portion can be reduced.

### Launch-critical but reducible

Use this when the SDK must participate in launch but does not need its full workload immediately.

Typical strategy:

- install handlers or delegates early
- configure keys and local options early
- load cached state synchronously only if cheap and required
- postpone uploads, session sync, network fetches, device scans, preloads, and large persistence work

### First-interaction required

Use this when the SDK is needed before the user performs an action, but not before the first visible UI.

Examples:

- analytics event pipeline before the first tappable screen event
- experiment assignment before a specific interaction
- fraud or risk check before login, payment, transfer, or checkout
- push permission request before a notification onboarding step

### Post-first-frame acceptable

Use this when the SDK improves observability, personalization, monetization, or background readiness but is not required for the first frame.

Examples:

- analytics session upload
- remote config refresh when cached/default values exist
- ad SDK initialization when the first screen has no ad slot
- attribution network sync when cached routing is enough
- noncritical monitoring setup
- secondary logging sinks

### Feature-specific and lazy

Use this when the SDK is only required after entering a feature.

Examples:

- ad mediation for an ad-supported screen
- chat/support SDK for the help center
- payment risk SDK for checkout or transfer
- map/search/location vendor for a later tab
- social login SDK for the login method screen

### Background-only or maintenance

Use this when the work should not affect launch readiness.

Examples:

- cache pruning
- log upload
- pending analytics flush
- remote config refresh that does not affect launch UI
- ad preloading for later screens
- token sync that can retry later
- SDK health checks

## Category Guidance

### Analytics and event pipelines

Analytics usually does not need full synchronous startup before first frame.

Prefer:

- cheap local event buffering
- cached user/session identity when already available
- nonblocking session start
- upload/flush after visible UI or later
- explicit consent gating before tracking
- no synchronous network call for user properties, experiments, or device metadata during launch

Keep synchronous launch work only when the app must record a launch event before any possible crash or termination and the local write is cheap.

Red flags:

- analytics initialization performs network I/O before first frame
- analytics reads large local stores synchronously
- analytics blocks on consent, user profile, remote config, or attribution
- multiple analytics SDKs start separately instead of using a small app-owned event buffer

### Crash reporting and diagnostics

Crash reporting often has a valid reason to install handlers early. That does not mean all crash SDK work must block launch.

Prefer:

- minimal handler installation early
- deferred crash report upload
- deferred symbol, attachment, breadcrumb, or log enrichment
- bounded file reads for previous crash state
- no synchronous network work during launch
- no large log database scan before first frame

Be careful before deferring the handler itself. If startup crashes matter, delaying handler installation may reduce diagnostic coverage.

Red flags:

- uploading previous crash reports synchronously during startup
- scanning large log directories before first frame
- attaching large device/app state snapshots immediately
- starting multiple overlapping crash or monitoring SDKs without a clear owner

### Ads and monetization SDKs

Ad SDKs are usually poor candidates for launch-critical synchronous initialization unless the first visible screen immediately contains an ad and the product explicitly accepts that trade-off.

Prefer:

- initialize after first visible UI
- initialize on first ad surface entry
- preload ads only after interaction-critical work is complete
- respect consent and tracking authorization state before ad requests
- keep mediation/network adapter startup out of the critical path

Red flags:

- initializing ad mediation in `didFinishLaunching` when no ad appears on the first screen
- loading or preloading ads before root UI is visible
- blocking first frame on consent, tracking, or ad configuration fetch
- initializing many ad network adapters unconditionally

### Attribution, install tracking, and deep linking

Attribution SDKs often combine two different concerns: routing and measurement. Do not treat them as one indivisible launch task.

Prefer:

- parse launch URL, universal link, or notification payload early when it determines the first screen
- route with local payload data when possible
- defer network attribution sync when it does not change immediate routing
- use cached campaign or install state when safe
- time-bound any attribution call that can affect first-screen routing

Red flags:

- blocking root UI until a network attribution response returns
- starting attribution only to upload install/session data before first frame
- mixing deep-link routing with analytics upload in one synchronous SDK startup call
- failing to define fallback routing when attribution is delayed or unavailable

### Remote config

Remote config should not make launch depend on the network unless the app has a strict correctness requirement.

Prefer:

- safe local defaults
- last-known-good cached config
- explicit minimum config needed for launch
- bounded local reads
- asynchronous refresh after first frame
- staged application of new config after the app is ready

Launch-critical remote config should be rare and justified. If a value is required before first frame, prefer shipping a default in the app bundle or storing the last-known-good value locally.

Red flags:

- waiting for remote config fetch before creating root UI
- decoding a large config file synchronously on the main thread
- coupling remote config refresh to dependency container creation
- failing closed in a way that blocks launch for noncritical features

### Feature flags and experimentation

Feature flags can be launch-critical when they choose the initial app surface, enable a safety kill switch, or control a high-risk path. Most flag refresh work is not launch-critical.

Prefer:

- local defaults compiled into the app
- cached flag snapshots loaded cheaply
- deterministic fallback behavior
- separating read-only launch flag access from remote refresh
- refreshing and activating new values after first frame or at a controlled readiness point
- avoiding network waits on the launch path

For experiments, avoid blocking first frame on assignment unless the experiment controls the first visible screen and product explicitly accepts the trade-off.

Red flags:

- remote flag fetch before root UI
- JSON parsing of a large flag payload on the main thread
- flag client initialization that starts analytics, attribution, and remote config transitively
- hidden feature-flag reads that force SDK initialization during root view/model creation

### Push notifications

Push support can have legitimate launch-time configuration requirements. Separate lightweight notification configuration from vendor network startup.

Prefer:

- register notification categories and delegates at launch when the app relies on actionable notifications or launch-time notification routing
- parse notification launch payload early if it decides the initial route
- defer token upload, vendor sync, topic subscription refresh, and marketing automation startup when they are not required for the first frame
- avoid prompting for notification permission during generic app launch unless the onboarding flow intentionally does so
- preserve correct behavior for launches caused by tapping a notification

Red flags:

- starting a full push marketing SDK synchronously only to upload a token
- blocking first frame on token registration or vendor subscription sync
- losing notification launch routing because push setup was deferred without a fallback
- requesting notification permission before the UI explains the value

### Security, fraud, attestation, and integrity SDKs

Security-related SDKs require more caution than analytics or ads. Some apps must gate sensitive content, payments, transfers, login, or regulated flows before allowing interaction.

Prefer:

- minimal synchronous checks only for gates that are truly required before showing the first screen
- cached or locally computed risk state when acceptable
- asynchronous risk enrichment after first frame
- feature-level checks before sensitive actions
- explicit timeout and fallback behavior for network-based checks
- clear product/security decision about fail-open vs fail-closed behavior

Do not defer security SDKs blindly. In high-risk apps, a small amount of launch-time security work may be correct. The performance review should narrow and bound the work, not remove the control.

Red flags:

- network attestation blocking launch with no timeout
- expensive device scan before showing any non-sensitive UI
- security SDK initialized multiple times through different wrappers
- unclear fallback when security service is slow or unavailable
- doing large keychain or file-system scans on the main thread

### Logging, monitoring, and observability

Observability is important, but launch should not be blocked by heavy logging infrastructure.

Prefer:

- tiny in-memory or local buffered logging during early launch
- deferred remote transport setup
- deferred upload of previous logs
- sampling where appropriate
- avoiding synchronous persistence on the main thread

Red flags:

- opening a large logging database before first frame
- compressing or uploading logs during launch
- initializing multiple monitoring SDKs with duplicate device/environment collection

### Consent and privacy SDKs

Consent state may be required before analytics, ads, attribution, or tracking starts. That does not always mean a full consent SDK must block first frame.

Prefer:

- cheap cached consent state for startup decisions
- no tracking before consent when consent is required
- showing consent UI as part of an explicit onboarding or privacy flow
- deferred vendor startup until consent is known
- separating consent read from vendor initialization

Red flags:

- starting tracking SDKs before consent is resolved
- blocking launch on a network consent refresh when cached state is available
- coupling consent SDK startup to unrelated SDK initialization

## Vendor Initialization Strategy

Use this procedure when reviewing a vendor SDK on the launch path.

### 1. Identify the entry point

Find where the SDK is started and who calls it.

Look for:

- direct calls from app/scene lifecycle methods
- calls from a bootstrapper or dependency container
- eager singleton construction
- property wrappers or lazy globals that are touched during root UI creation
- SwiftUI root model initialization that starts services
- Objective-C categories or runtime hooks that register work implicitly

If the SDK starts through `+load`, constructor functions, or static initializers, switch to `pre-main-dyld-and-static-initializers.md` for that part of the investigation.

### 2. Split configuration from work

For each SDK, separate:

- local configuration
- delegate/handler installation
- cached state read
- network request
- upload/flush
- device scan
- database open or migration
- permission prompt
- feature module preload
- background maintenance

Only keep the required subset before first frame.

### 3. Check vendor-supported modes

Look for official support for:

- delayed initialization
- manual start
- offline mode
- disable automatic session tracking
- disable automatic event collection
- cached configuration
- lazy module loading
- background upload
- separate handler installation and transport startup
- test or no-op mode for CI

Do not invent unsupported initialization sequences for security, crash, payment, or attribution SDKs without checking vendor documentation.

### 4. Define the readiness point

Replace vague delay rules with explicit readiness points.

Good readiness points include:

- after first visible UI is committed, verified by measurement
- after first interaction-critical work is complete
- after authentication state is known
- after consent state is available
- when a feature screen is entered
- when network becomes available
- when the app becomes active and idle enough for background startup

Avoid arbitrary fixed delays as the only mechanism. A delay may hide the cost in one trace but still cause jank during early interaction.

### 5. Preserve correctness contracts

Before deferring an SDK, check whether it affects:

- startup crash capture
- notification or deep-link routing
- security gating
- compliance or consent
- kill switches
- initial screen selection
- payment, login, transfer, or regulated flows
- data integrity or migration safety

If correctness depends on the SDK, reduce the work rather than blindly moving it later.

### 6. Add measurement around app-owned wrappers

When the app owns a wrapper such as `SDKManager.start()`, add app-level timing around each vendor startup step rather than measuring only the combined wrapper.

Prefer measuring:

- time spent before first frame
- main-thread time
- synchronous file/keychain/database time
- network requests started during launch
- first interaction latency after deferral
- whether deferred startup causes hitches shortly after launch

Detailed tool setup belongs in `metrics-instruments-xctest-metrickit.md`.

## Decision Rules

- Do not use blanket rules such as “defer all SDKs” or “initialize all SDKs in `didFinishLaunching`.” Classify each SDK by launch correctness and user impact.
- Keep the minimal launch-critical part early; move network, upload, preload, scan, refresh, and maintenance work later when safe.
- Prefer cached/default state over launch-time network dependency.
- Prefer app-owned facades that can buffer events, expose cached state, and initialize vendors lazily.
- Do not block first frame on analytics, ads, attribution upload, log upload, remote config refresh, or noncritical experiment refresh unless the product explicitly accepts the trade-off.
- Do not defer crash handler installation, notification routing, security gates, or first-screen feature flags without understanding the correctness impact.
- Treat vendor SDK documentation as the source of truth for supported initialization modes.
- Treat SDK initialization as main-thread work until measurement proves otherwise.
- Avoid starting SDKs from global/static initializers, Objective-C `+load`, constructors, or root view/model initializers unless there is a strong reason.
- If an SDK must start early, make that path small, deterministic, bounded, and visible in measurement.

## Red Flags

Flag these during review:

- one `initializeAllSDKs()` method called synchronously during launch with no per-SDK classification
- vendor initialization hidden inside dependency container construction
- SDK startup that performs synchronous network, keychain, file, database, or JSON parsing on the main thread
- crash, analytics, logs, monitoring, and attribution all starting separate device/environment scans
- ad mediation initialized before any ad surface exists
- remote config or feature flag fetch blocking root UI
- push token upload blocking first frame
- notification routing broken by deferring push setup without another routing path
- security SDK network check blocking launch without timeout or fallback
- consent SDK and tracking SDK initialized in the wrong order
- SDK wrappers started from SwiftUI root view/model initialization
- launch performance claims based only on “moved to async” without trace evidence

## Recommended Review Output

When this reference is used, return SDK-specific findings instead of generic launch advice.

Use this structure:

```markdown
### SDK launch classification

- SDK or wrapper:
- Current startup point:
- Required before first frame: yes/no/unknown
- Required before first interaction: yes/no/unknown
- Correctness risk if deferred:
- Main launch cost suspicion:

### Recommendation

- Keep early:
- Move later:
- Make lazy:
- Replace with cached/default state:
- Vendor documentation to verify:

### Validation

- Local trace area to inspect:
- App-level signpost or timing to add:
- Production metric or regression check:
- Risk to retest after deferral:
```

If the answer depends on unavailable vendor documentation, say so clearly and recommend the safest classification rather than guessing.

## Non-Goals

Do not turn this reference into a generic SDK integration guide. The only question here is how vendor startup affects app launch and early responsiveness.
