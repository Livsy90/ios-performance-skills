# Linking Strategy and Launch Cost

Use this reference when the launch investigation points to dynamic library loading, framework count, modularization strategy, embedded frameworks, static libraries, mergeable libraries, or dependency graph shape.

This reference is only about **linking decisions that can affect launch**. Do not use it as a general modular architecture guide, a build-system optimization guide, or a full dyld/static-initializer guide.

## Scope Boundary

This file covers:

- static libraries and static frameworks
- dynamic frameworks and dynamic libraries
- mergeable libraries
- embedded framework count and dependency depth
- first-party vs third-party dependency strategy
- release-build bundle inspection
- launch-risk review of linking choices

This file does not cover:

- detailed dyld internals, fixups, rebasing, binding, or initializer order; use `pre-main-dyld-and-static-initializers.md`
- Objective-C `+load`, `+initialize`, constructor functions, or static initializer contents; use `pre-main-dyld-and-static-initializers.md`
- AppDelegate, SceneDelegate, root UI, or startup work deferral; use `appdelegate-scenedelegate-and-first-frame.md`
- SwiftUI root app initialization; use `swiftui-app-launch.md`
- SDK-specific startup policy; use `third-party-sdks-at-launch.md`
- metric setup, XCTest, Instruments workflow, or production monitoring; use `metrics-instruments-xctest-metrickit.md`

## Core Model

Linking strategy affects where cost is paid.

- **Static linking** copies needed library code into the consuming binary at build time.
- **Dynamic linking** records a dependency on a separate binary that dyld must locate, map, connect, and initialize at launch or at runtime.
- **Mergeable libraries** are dynamic libraries built with metadata that allow the linker to merge them into a consuming binary, commonly preserving dynamic-library ergonomics during development while reducing separate dynamic images in release builds.

Do not turn this into a universal rule such as “static is always faster” or “dynamic is always bad.” The correct recommendation depends on the dependency graph, build configuration, app size, launch measurements, distribution model, and whether the dependency is actually present as a separate dynamic image in the shipped app.

## What the Agent Can Inspect

When repository access is available, inspect the project instead of guessing.

Look for:

- `*.xcodeproj/project.pbxproj`
- `*.xcworkspace`
- `Package.swift`
- `Package.resolved`
- `Podfile`, `Podfile.lock`
- `Cartfile`, `Cartfile.resolved`
- Tuist manifests such as `Project.swift` and `Workspace.swift`
- Bazel files such as `BUILD`, `MODULE.bazel`, or `WORKSPACE`
- generated project settings if the repository uses a project generator
- build scripts that copy, embed, strip, merge, or re-sign frameworks
- vendored `.framework` or `.xcframework` artifacts
- app extension targets that share dependencies with the main app

When a built product is available, inspect the app bundle and binaries.

Useful checks, when the tools exist in the environment:

```sh
find MyApp.app -maxdepth 3 -type d -name "*.framework" | sort
otool -L MyApp.app/MyApp
dyld_info -dependents MyApp.app/MyApp
file MyLibrary.framework/MyLibrary
```

Use these commands as inspection aids, not as proof of launch impact. The final recommendation still needs measurement in a release-like build.

## Classify Each Dependency

For each dependency that appears on the launch path or in the linked binary graph, classify it before recommending changes.

### Ownership

- Apple system framework
- first-party internal module
- third-party source dependency
- third-party binary dependency
- vendored SDK with required early initialization
- shared dependency used by app extensions
- optional feature dependency

### Linkage

- static library
- static framework
- dynamic framework or dylib
- mergeable library
- Swift Package product with explicit static/dynamic type
- Swift Package product with automatic type selection
- CocoaPods static framework/library
- CocoaPods dynamic framework
- runtime-loaded library or plugin
- resource-only bundle

### Launch relevance

- required by the initial executable at launch
- embedded but not directly needed before first frame
- needed only by a later feature
- used only by an app extension
- dynamically loaded after launch
- unclear without inspecting the build product

## When Linking Strategy Is Likely Relevant

Investigate linking strategy when any of the following are true:

- Instruments or launch logs point to high dyld/pre-main time.
- The app embeds many first-party dynamic frameworks.
- The dependency graph contains chains of dynamic frameworks that depend on more dynamic frameworks.
- A modularized app ships most internal feature modules as dynamic frameworks.
- The release `.app` bundle contains many non-system frameworks under `Frameworks/`.
- A framework is linked and embedded even though it is only needed by a later feature.
- A third-party SDK is pulled into the app binary even though only an optional feature uses it.
- A dependency is included both by the main app and by one or more app extensions.
- Recent modularization or package-manager changes correlate with launch regression.

Do not blame linking strategy just because launch is slow. If the slow work is in AppDelegate, root view construction, database startup, keychain access, network waits, or first-frame rendering, route to the relevant reference instead.

## Static Linking: When It Helps

Static linking can reduce launch-time dynamic loading work because fewer separate dynamic images need to be located, mapped, connected, and initialized by dyld.

It is often worth considering for:

- small first-party modules
- internal feature modules that do not need independent binary boundaries
- launch-critical shared code used immediately by the app
- modules that are currently dynamic only because of historical project setup
- source dependencies that do not need runtime loading or independent distribution
- dependency graphs where many small dynamic frameworks are embedded in the final app

Recommended agent behavior:

- Suggest static linking as an option, not as an automatic fix.
- Prefer applying it first to first-party modules the team controls.
- Avoid hard framework-count thresholds.
- Ask for release-build measurements before and after the change.
- Check for app extensions and duplicate inclusion risk before recommending broad conversion.

## Static Linking: Risks and Trade-offs

Static linking can introduce costs or correctness issues.

Check for:

- longer clean or incremental link times
- larger final executable size
- duplicate symbols when the same static library is included through multiple paths
- Objective-C class duplication if a static library is copied into multiple dynamic frameworks
- resource packaging differences when converting from dynamic framework to static library/framework
- Swift module and binary distribution requirements
- symbol visibility and exported-symbol expectations
- app extension duplication and binary size growth
- debugging and symbolication workflow changes
- third-party license or vendor support constraints

Do not recommend converting vendored binary frameworks to static form unless the vendor provides a supported static or mergeable distribution.

## Dynamic Linking: When It Is Reasonable

Dynamic frameworks are not inherently wrong. They may be appropriate when the project needs:

- faster local iteration for large modules
- independently distributed binary SDKs
- clean binary boundaries for teams or vendors
- runtime loading of optional functionality
- shared code between multiple executables where duplication would be expensive
- plugin-like architecture on platforms and contexts where it is supported
- compatibility with vendor tooling, resources, or signing requirements

Recommended agent behavior:

- Do not tell the team to remove all dynamic frameworks.
- Ask whether a dynamic boundary is intentional or accidental.
- Distinguish development-time convenience from release-time launch cost.
- Verify whether the framework is still embedded as a separate image in the shipped product.
- Prefer reducing unnecessary dynamic images over destroying useful module boundaries.

## Dynamic Linking: Launch Risks

Dynamic linking can affect launch because every dependent dynamic image can add work for dyld and the kernel. The exact cost depends on the binary, OS, device, fixup format, caches, initializer work, exported symbols, Objective-C metadata, Swift metadata, and dependency graph.

Review dynamic frameworks for:

- unnecessary embedding in the main app target
- deep dependency chains
- frameworks that are linked by the app but only used after launch
- frameworks that contain static initializers or Objective-C load-time hooks
- duplicated SDKs across app and extension targets
- debug-only frameworks accidentally present in release builds
- large binary SDKs initialized or loaded too early
- package-manager defaults that produce dynamic frameworks unintentionally

Avoid claims like “each framework adds N ms.” The agent should not invent per-framework timing. Measure the actual app.

## Mergeable Libraries

Mergeable libraries are often the best option to evaluate for large modular apps using many dynamic frameworks.

They can help when the team wants:

- dynamic-library ergonomics during development
- fewer separate dynamic images in release builds
- better launch characteristics without fully abandoning modular framework targets
- gradual migration from many embedded internal frameworks
- reduced release bundle complexity

Recommended agent behavior:

- Suggest mergeable libraries when the project uses Xcode versions and deployment targets that support the team’s desired workflow.
- Treat them as a candidate solution, not as a guaranteed drop-in fix.
- Verify Debug and Release behavior separately.
- Confirm which dependencies are merged and which remain as separate dynamic frameworks.
- Check app-extension boundaries before merging dependencies shared across multiple executables.
- Ensure the shipped app does not still embed merged dependencies unnecessarily.

Do not give detailed setup steps unless the task asks for implementation. For this skill, the important launch-performance question is whether release builds still ship many separate dynamic images and whether mergeable libraries can reduce that without unacceptable trade-offs.

## Package Manager Notes

### Swift Package Manager

Inspect `Package.swift` for product type:

```swift
.library(name: "FeatureCore", type: .static, targets: ["FeatureCore"])
.library(name: "FeatureUI", type: .dynamic, targets: ["FeatureUI"])
.library(name: "SharedKit", targets: ["SharedKit"])
```

When `type` is omitted, the package does not explicitly force static or dynamic linkage. The final linkage may depend on the consumer and build system. Do not assume it is static or dynamic without inspecting the generated project or built product.

### CocoaPods

Inspect `Podfile` for linkage choices, especially:

```ruby
use_frameworks! :linkage => :static
use_frameworks! :linkage => :dynamic
```

Also check `vendored_frameworks` and `vendored_libraries`. A pod can be source-built, static, dynamic, or vendored. Do not assume all pods behave the same way.

### Carthage and vendored XCFrameworks

Carthage and manually integrated `.xcframework` artifacts often produce binary frameworks. Determine whether the actual slice used by the app is static or dynamic with binary inspection when possible.

## App Extensions and Shared Dependencies

Be careful when the app has extensions.

Before recommending static linking, merging, or framework removal, check:

- whether the dependency is used by the main app, an extension, or both
- whether static linking would duplicate code into multiple executables
- whether a merged dependency would increase extension binary size
- whether the extension requires a smaller subset of the dependency
- whether the framework contains APIs unavailable to extensions
- whether the dependency must remain separately packaged for signing or distribution reasons

For shared app/extension code, there may be no single best answer. The agent should explain the trade-off and recommend measuring both app launch and extension size/startup behavior.

## Review Procedure

When asked to review linking strategy for launch performance, follow this procedure:

1. **Identify evidence**
   - Is there a launch trace, dyld/pre-main regression, or only a suspicion?
   - Is the issue observed in Debug, Release, TestFlight, or production?

2. **List shipped dynamic images**
   - Inspect the built `.app` bundle or project settings.
   - Separate Apple system frameworks from embedded first-party and third-party frameworks.

3. **Map ownership and necessity**
   - Mark which frameworks are first-party, third-party, optional, launch-critical, or feature-specific.

4. **Find accidental dynamic boundaries**
   - Look for small internal modules that became dynamic by default or history rather than need.
   - Look for modules that could be static or mergeable in release builds.

5. **Check for duplicate inclusion risks**
   - Pay attention to app extensions, multiple dynamic frameworks consuming the same static library, and SDKs included through more than one package manager path.

6. **Recommend the smallest safe change**
   - Start with removing unused links/embeds.
   - Then consider converting first-party modules to static or mergeable form.
   - Leave vendor SDKs alone unless supported alternatives exist.

7. **Require validation**
   - Compare release-like builds before and after.
   - Validate launch traces and bundle contents.
   - Watch build time, app size, extension size, and symbolication impact.

## Decision Rules

- Prefer removing an unnecessary dependency over changing its linkage.
- Prefer narrowing where a dependency is linked over linking it everywhere.
- Prefer first-party modules as the first optimization target because the team controls their build settings.
- Prefer static or mergeable linkage for small internal modules that do not need a runtime boundary.
- Prefer dynamic linkage when independent distribution, runtime loading, debugging workflow, or shared binary boundary matters more than launch cost.
- Prefer mergeable libraries for large modular apps that want dynamic-style development and static-like release launch behavior.
- Do not recommend converting all modules at once.
- Do not use arbitrary numeric limits for dynamic frameworks.
- Do not assume Debug bundle contents represent Release/TestFlight behavior.
- Do not assume a package manager’s label tells the final binary truth; inspect the built product when possible.

## Common Findings

Use concrete findings like these:

- “The app embeds many first-party dynamic frameworks, but several appear to be small internal modules with no obvious runtime boundary requirement. Consider static or mergeable linkage for those modules first.”
- “This dependency is only used after login but is linked into the main app at launch. Consider moving it behind a feature boundary or runtime loading strategy if the platform and architecture support it.”
- “The framework appears in `Embed Frameworks`, but the app does not need it as a separate runtime image in release builds. Verify whether it can be merged, statically linked, or removed from embedding.”
- “The app and extension both consume this dependency. A static conversion may duplicate code across executables, so validate extension size and startup impact before changing it.”
- “The project uses a dynamic binary SDK from a vendor. Do not rewrite linkage manually; check whether the vendor provides static, dynamic, or mergeable variants.”

## Output Guidance

When this reference is used, include a focused section in the final review:

```markdown
### Linking strategy

**Current shape:** static / dynamic / mergeable / unclear.

**Launch risk:** why the linking graph may affect launch.

**Safe next check:** what to inspect in project settings or the built app.

**Candidate change:** remove unused link/embed, convert first-party module to static, enable mergeable libraries, or keep dynamic.

**Trade-offs:** build time, binary size, app extensions, resources, SDK support, debugging, symbolication.

**Validation:** release-like launch measurement and bundle/binary inspection.
```

Keep the recommendation tied to evidence. If there is no launch trace or built-product inspection, label the finding as a hypothesis.
