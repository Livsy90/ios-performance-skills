# Pre-main, dyld, and Static Initializers

Use this reference when a launch investigation points to work that happens before the app reaches its own application lifecycle code, or when reviewing code that can execute while images are being loaded and registered.

Keep this reference focused on pre-main behavior. Do not use it as the primary guide for framework linking strategy, AppDelegate/SceneDelegate startup, SwiftUI root initialization, third-party SDK policy, or launch metric setup. Those topics belong in their own reference files.

## Mental Model

Pre-main is the part of launch that happens before the app enters its explicit entry point and before app delegate, scene delegate, or SwiftUI `App` startup code begins.

At a high level, this area can include:

1. Loading the main executable and dependent images.
2. Mapping code and data into the process.
3. Applying dyld fixups and connecting references between images.
4. Registering Objective-C and Swift runtime metadata needed by loaded images.
5. Running load-time initializers such as Objective-C `+load`, C/C++ constructors, clang constructor functions, and functions in binary initializer sections.
6. Handing control to the app entry point.

Do not treat all launch work as pre-main. Work inside `main`, `UIApplicationMain`, `@main App`, `AppDelegate`, `SceneDelegate`, root view initialization, or first-frame rendering is launch-critical, but it is not pre-main.

## Why This Phase Matters

Pre-main work is paid before the app can start building its first UI. If this phase is slow, later deferral in `didFinishLaunching`, `scene(_:willConnectTo:)`, or SwiftUI `.task` cannot recover the time already spent.

The main review goal is to remove hidden eager work from load time and make initialization explicit, lazy, measurable, and scoped to real feature needs.

## Common Sources of Pre-main Work

### Objective-C `+load`

`+load` is called by the Objective-C runtime when a class or category is loaded into the process. It is not tied to whether the app actually uses that class on the first screen.

Treat every `+load` implementation as launch-critical until proven otherwise.

Review for:

- method swizzling
- runtime scanning
- service registration
- singleton creation
- dependency lookup
- logging or analytics setup
- file, keychain, database, or network access
- locks, semaphores, or dispatch synchronization
- large allocations or parsing

Prefer:

- explicit registration from a known startup point
- lazy registration on first feature use
- narrow one-time setup guarded by a cheap path
- compile-time configuration when possible

Acceptable `+load` work should be tiny, deterministic, and independent of app state. Avoid depending on ordering between unrelated classes, categories, or libraries.

### Objective-C `+initialize`

`+initialize` is lazy compared with `+load`; it runs before a class is first used. It can reduce unconditional launch work in legacy Objective-C code, but it is not a universal replacement.

Use it carefully because it still hides work behind a runtime hook and can move latency to the first call site. In modern code, explicit setup or lazy Swift initialization is usually clearer than adding new `+initialize` logic.

Prefer this order when changing code:

1. Remove the need for load-time initialization.
2. Move registration to explicit startup code if it is truly launch-critical.
3. Make setup lazy and feature-scoped if first-frame correctness does not need it.
4. Use `+initialize` only when maintaining Objective-C code where its semantics are appropriate.

### C and C++ Constructors

C++ global objects with nontrivial constructors and functions marked with constructor attributes can run before the app entry point.

Review for:

- global objects that allocate memory
- constructors that open files or databases
- constructors that create threads or queues
- registration systems that scan types or plugins
- hidden dependency setup inside native libraries

Prefer explicit initialization functions that the app calls at a chosen point. If a constructor is unavoidable, keep it local, fast, and free of app-level dependencies.

### Binary Initializer Sections

Functions linked into initializer sections, such as `__DATA,__mod_init_func`, are part of the same load-time risk area. They may come from C, C++, Objective-C, Swift interop code, or third-party binaries.

When a trace shows static initializer time but source code does not obviously contain constructors, inspect linked dependencies and generated/runtime support code as well as application code.

### Objective-C Categories and Runtime Registration

Objective-C classes, categories, protocols, and method metadata must be registered as images load. Categories that implement `+load` are especially important because their load methods run even when the category's methods are not used on the launch path.

Do not assume categories are free just because they contain no visible app startup code. They can still contribute metadata and, when they include `+load`, execute launch-time behavior.

Review category-heavy modules for:

- categories on very common Foundation/UIKit classes
- swizzling performed from category `+load`
- duplicated registration across many modules
- generated Objective-C bridging code from dependencies

### Swift Globals and Static Properties

Swift global variables and static properties are often initialized lazily on first access rather than simply because a module is loaded. Do not label every Swift `static let` as pre-main work.

The launch risk appears when:

- a Swift global or static value is touched by a pre-main initializer
- app startup code touches it before first frame
- initialization performs heavy work
- initialization creates a large dependency graph
- initialization has side effects beyond simple value creation

Prefer small, side-effect-free globals. Keep expensive setup behind explicit async or lazy boundaries that match when the feature is actually needed.

## Review Procedure

When inspecting a pre-main suspicion, follow this sequence:

1. Confirm whether the time is actually pre-main.
   - Use an App Launch trace, dyld Activity, or equivalent evidence.
   - Do not infer pre-main cost only from a slow first frame.

2. Identify which image or initializer owns the time.
   - Main executable
   - Internal framework
   - Third-party SDK
   - System framework
   - Generated code or runtime support

3. Classify the initializer.
   - Objective-C `+load`
   - Objective-C `+initialize`
   - C/C++ constructor
   - function in an initializer section
   - runtime registration/metadata
   - dynamic loading call
   - unknown

4. Decide whether the work must happen before first frame.
   - If no, move it to explicit/lazy setup.
   - If yes, reduce it to the smallest safe operation.
   - If it is from a vendor SDK, look for a documented minimal/deferred initialization mode.

5. Validate the change with the same launch scenario and build configuration.
   - Compare the same device class and OS version.
   - Use release-like builds.
   - Avoid mixing cold, warm, prewarmed, and resume measurements.

## Code Review Checklist

Use this checklist when reviewing Objective-C, C/C++, mixed Swift/Objective-C modules, or binary dependencies on the launch path.

- [ ] No app-level work is hidden inside Objective-C `+load`.
- [ ] Any remaining `+load` methods are tiny, deterministic, and documented.
- [ ] `+load` is not used for networking, disk I/O, database setup, keychain access, analytics setup, or dependency graph construction.
- [ ] Method swizzling from `+load` is justified and limited.
- [ ] There are no C/C++ global constructors doing expensive work.
- [ ] Constructor functions are not used as a substitute for explicit startup APIs.
- [ ] Heavy Swift globals or static values are not touched by pre-main hooks.
- [ ] Objective-C categories with `+load` are audited, especially in third-party frameworks.
- [ ] Initializer ordering assumptions are avoided.
- [ ] Third-party binaries with static initializer cost are tracked separately from application code.
- [ ] Any diagnostic environment variables are used only for debugging, not as production behavior.

## Safer Patterns

### Replace hidden load-time work with explicit registration

Risky pattern:

```objc
@implementation PaymentRouteLoader
+ (void)load {
    RegisterPaymentRoutes();
    StartPaymentDiagnostics();
}
@end
```

Preferred direction:

```objc
void ConfigurePaymentRoutes(void) {
    RegisterPaymentRoutes();
}
```

Then call `ConfigurePaymentRoutes()` from the narrow startup point that actually needs payment routing. Do not initialize diagnostics, networking, or nonessential services as part of route registration.

### Keep constructors trivial

Risky pattern:

```cpp
static SearchIndexBoot boot;

SearchIndexBoot::SearchIndexBoot() {
    openIndexFiles();
    warmCaches();
}
```

Preferred direction:

```cpp
void PrepareSearchIndexIfNeeded() {
    static std::once_flag once;
    std::call_once(once, [] {
        openIndexFiles();
    });
}
```

The second pattern still needs review: it moves cost to first use, so first use must not be the first frame unless that work is essential.

### Keep Swift static initialization cheap

Risky pattern:

```swift
enum BootstrapRegistry {
    static let services = buildCompleteServiceGraph()
}
```

Preferred direction:

```swift
struct BootstrapRegistry {
    func makeLaunchCriticalServices() -> LaunchServices {
        LaunchServices(sessionStore: SessionStore())
    }

    func makeFeatureServices() -> FeatureServices {
        FeatureServices()
    }
}
```

The goal is not to avoid every static value. The goal is to avoid hiding large, side-effecting work behind static access that may happen during launch.

## Diagnostic Hints

Use diagnostics to locate the owner of work before recommending rewrites.

Helpful signals:

- App Launch trace shows a large pre-main or static-initializer region.
- dyld Activity attributes time to static initialization or image loading.
- Time Profiler shows CPU inside runtime registration, constructor functions, or framework startup before app lifecycle code.
- Logs from debug-only dyld or Objective-C runtime environment variables show many loaded images or load methods.
- Binary inspection reveals initializer sections or unexpected linked dependencies.

Potential debug tools and signals:

- Instruments App Launch template
- dyld Activity instrument on Xcode versions that include it
- Time Profiler with system libraries visible when needed
- `os_signpost` around the first app-controlled startup points to separate pre-main from app initialization
- debug-only environment variables such as `DYLD_PRINT_LIBRARIES` or `OBJC_PRINT_LOAD_METHODS`, when supported by the current run environment
- binary inspection tools such as `otool`, `nm`, or `dyld_info` for advanced investigations

Do not overfit to one run. Pre-main measurements can vary by device state, OS version, install state, cache warmth, and build configuration.

## Recommendation Language

Use precise wording:

- "This code can run before app lifecycle code" rather than "this runs before first frame" when discussing `+load` or constructors.
- "Move this out of load time" rather than "put it on a background queue" when the main issue is unconditional pre-main execution.
- "Make initialization explicit or lazy" rather than "replace `+load` with `+initialize`" unless the codebase is legacy Objective-C and that trade-off is justified.
- "Measure the same launch scenario again" rather than "this will improve cold launch by N ms" unless there is trace evidence.

## Boundaries With Other References

Route to another reference when the main issue is outside pre-main:

- Use `linking-strategy.md` for deciding between dynamic frameworks, static libraries, mergeable libraries, modularization, binary size, or build-time trade-offs.
- Use `appdelegate-scenedelegate-and-first-frame.md` for work inside `didFinishLaunching`, scene connection, dependency containers, root UI creation, or main-thread deferral.
- Use `swiftui-app-launch.md` for `@main App`, root `Scene`, root view initialization, observable state, `.task`, or `.onAppear` work.
- Use `third-party-sdks-at-launch.md` for vendor-specific startup policies, deferred SDK modes, crash reporting, attribution, ads, analytics, remote config, push, security, or feature flags.
- Use `metrics-instruments-xctest-metrickit.md` for detailed tool usage, launch metric interpretation, CI baselines, MetricKit, or Organizer.
- Use `launch-taxonomy-and-targets.md` for cold/warm/prewarmed/resume definitions, first-frame targets, and measurement setup.

