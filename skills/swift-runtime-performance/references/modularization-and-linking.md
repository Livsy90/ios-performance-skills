# Modularization and Linking

Use this reference when a review involves Swift module boundaries, cross-module optimization, `@inlinable`, `@usableFromInline`, `@frozen`, library evolution, static linking, dynamic frameworks, mergeable libraries, app launch time, binary size, or build-time/runtime trade-offs.

The goal is not to recommend fewer modules or static linking by default. The goal is to separate architectural boundaries from compiler visibility, linking behavior, ABI resilience, launch cost, and build performance.

## Core model

Modularization affects several independent concerns:

* team ownership
* build graph size
* incremental build behavior
* public API design
* optimizer visibility
* generic specialization
* inlining opportunities
* ABI and library evolution
* launch-time dynamic linking
* binary size
* dependency hygiene

Do not collapse all of these into one rule such as "more modules are faster" or "dynamic frameworks are slow."

Ask which problem is being solved:

* build time
* runtime speed
* app launch time
* binary size
* API stability
* dependency isolation
* team ownership
* testability
* distribution outside the app

The right module shape depends on the dominant constraint.

## Review questions

Start with these questions:

1. Is the module boundary architectural, organizational, or accidental?
2. Is hot code crossing a public module boundary?
3. Does the optimizer see the implementation of the hot code?
4. Is the API public only because of the module split?
5. Is library evolution enabled?
6. Is this code distributed as a binary framework?
7. Does the app use many dynamic frameworks?
8. Is the measured issue launch time, CPU time, binary size, or build time?
9. Are optimization attributes being used as design tools or as measured fixes?
10. Is the proposed change worth the API, ABI, and maintenance cost?

Do not change module boundaries without knowing which cost is being optimized.

## Module boundaries and optimizer visibility

Within a module, the Swift optimizer usually has more visibility. It can analyze function bodies, specialize generic functions, inline small functions, devirtualize calls, remove abstraction, and optimize ARC traffic more aggressively.

Across module boundaries, the optimizer may see only the public interface unless the module exposes additional implementation details through features such as `@inlinable` or cross-module optimization settings.

This matters most for:

* small generic helpers in hot loops
* protocol-heavy APIs
* collection algorithms
* numeric or geometry code
* serializers and parsers
* frequently called public wrappers
* type-erased abstractions
* performance-sensitive UI layout code

Review rule:

* Keep hot implementation details internal when possible.
* Avoid making APIs public only because the module boundary is inconvenient.
* If a public API is hot and generic, check whether the optimizer can specialize it.
* If the optimizer cannot see enough, consider redesigning the boundary before adding attributes.

## Internal implementation vs public API

A common performance issue is exposing implementation details as public API just to share code across modules.

Example:

```swift
public protocol FeedFiltering {
    func accepts(_ item: FeedItem) -> Bool
}

public func filteredItems(
    _ items: [FeedItem],
    using filter: any FeedFiltering
) -> [FeedItem] {
    items.filter { filter.accepts($0) }
}
```

This may be the right API if filters are runtime-pluggable and heterogeneous. It may be unnecessary overhead if every call site uses one concrete filter known at compile time.

A more optimizer-friendly internal path might keep the hot implementation concrete or generic:

```swift
struct RegionFeedFilter {
    let region: Region

    func accepts(_ item: FeedItem) -> Bool {
        item.region == region
    }
}

func filteredItems<F>(
    _ items: [FeedItem],
    using filter: F
) -> [FeedItem] where F: FeedFilter {
    items.filter { filter.accepts($0) }
}
```

Use this only when it preserves the intended design. Do not remove dynamic behavior that the architecture actually needs.

## Cross-module hot paths

When hot code crosses a module boundary, inspect:

* whether the function is public or internal
* whether the function is generic
* whether the caller sees the concrete type
* whether the implementation body is visible
* whether the module is built with library evolution
* whether the hot path goes through `any Protocol`
* whether optimized SIL shows specialization
* whether profiling points to abstraction overhead

Prefer these fixes before attributes:

* move the hot loop closer to the concrete implementation
* expose a higher-level operation instead of many tiny public calls
* keep the performance-critical helper internal
* add a concrete fast path behind a public abstraction
* reduce type erasure inside the inner loop
* batch work across the boundary

Example:

```swift
public protocol RouteScoring {
    func score(_ route: Route) -> Double
}

public func bestRoute(
    from routes: [Route],
    scorer: any RouteScoring
) -> Route? {
    routes.max { left, right in
        scorer.score(left) < scorer.score(right)
    }
}
```

If `bestRoute` runs often on large arrays and every caller uses the same concrete scorer, the existential boundary may be unnecessary. If scorers are loaded dynamically or configured at runtime, the existential may be the correct design.

## `@inlinable`

`@inlinable` makes the body of a function available to clients of the module so that the optimizer in the client module can use it.

Use it carefully.

Good candidates:

* small public functions
* stable utility functions
* performance-critical generic wrappers
* simple computed properties
* thin forwarding functions where cross-module inlining matters
* algorithms where client-side specialization is important

Poor candidates:

* large functions
* unstable implementations
* functions that reveal private design
* code that changes frequently
* functions with complex dependencies
* APIs where binary compatibility matters more than speed

Review questions:

* Is this function public because clients need it, or only because the module split requires it?
* Is the function small enough that exposing the body is reasonable?
* Is the implementation stable enough to become part of the module interface?
* Does optimized SIL or benchmarking show that cross-module visibility matters?
* Would an internal concrete fast path be cleaner?
* Would moving the hot loop into the defining module avoid the need for `@inlinable`?

Do not use `@inlinable` as a default performance switch. It is a visibility and resilience decision as much as an optimization tool.

## `@usableFromInline`

`@usableFromInline` allows an internal declaration to be referenced from an `@inlinable` public declaration.

It is not ordinary `internal`.

A `@usableFromInline` declaration becomes part of the ABI-visible surface of the module. It is not part of the source-level public API, but clients may depend on its binary interface through inlined code.

Use it when:

* an `@inlinable` function needs a helper
* the helper should not be source-public
* the helper's signature is stable enough for ABI exposure
* the performance benefit justifies the extra commitment

Avoid it when:

* the helper is unstable
* the type shape may change soon
* the helper exposes implementation details that should remain fully private
* `@inlinable` was not justified in the first place

Example:

```swift
@inlinable
public func normalizedProgress(_ value: Double) -> Double {
    _clamp(value, lower: 0, upper: 1)
}

@usableFromInline
func _clamp(_ value: Double, lower: Double, upper: Double) -> Double {
    min(max(value, lower), upper)
}
```

This can be appropriate for a small, stable helper. It would be a poor choice for a large implementation detail that the library author expects to redesign.

## `@frozen`

`@frozen` is a library evolution tool. It tells clients that the stored layout of a public struct or the cases of a public enum will not change in future binary-compatible versions.

It can allow better optimization because clients can make assumptions about layout or enum cases. The cost is reduced evolution flexibility.

Use `@frozen` only when the type's layout or cases are intentionally stable.

Good candidates:

* small value types with stable stored properties
* low-level performance types
* enums whose cases are part of a stable domain model
* types in libraries where ABI stability matters and layout stability is acceptable

Poor candidates:

* models that may gain fields
* enums likely to gain cases
* DTOs controlled by changing backend payloads
* public types still evolving
* types where resilience is more valuable than optimization

Review questions:

* Is library evolution enabled or relevant for this module?
* Is the type public across a binary boundary?
* Are stored properties or enum cases truly stable?
* Is client-side layout knowledge worth the loss of flexibility?
* Is this being added for a measured reason or just because the type looks small?

Do not use `@frozen` as a casual optimization hint in app-internal code.

## Library evolution and resilience

Library evolution allows a binary framework to evolve without breaking existing clients. This flexibility can reduce some optimization opportunities because clients cannot assume all implementation details.

When library evolution is enabled, public types and functions need to preserve binary compatibility across versions. The compiler must respect that resilience boundary.

Common effects:

* public struct layout may be hidden from clients unless frozen
* public enum cases may be resilient unless frozen
* public function bodies may be hidden unless inlinable
* some calls may require indirect access patterns
* clients may have less room for specialization and inlining

Review questions:

* Is this module shipped as a binary framework?
* Does it need stable ABI across versions?
* Is `BUILD_LIBRARY_FOR_DISTRIBUTION` enabled because it is truly needed?
* Is this an app-internal module that does not require resilience?
* Are performance-sensitive APIs accidentally behind resilient public boundaries?

Do not disable library evolution just for speed if the framework is distributed in a way that requires binary compatibility.

## Module stability vs library evolution

Do not confuse these concepts.

Module stability concerns whether clients built with different compiler versions can import a module interface.

Library evolution concerns whether a binary framework can change implementation details in later versions without breaking existing clients.

In Xcode projects, "Build Libraries for Distribution" is often the setting that brings these concerns into the build. It can be necessary for binary distribution, but it should not be enabled for every internal module by default without understanding the trade-offs.

Review rule:

* Enable distribution-oriented settings for modules that are actually distributed.
* For app-internal modules, prefer simpler settings unless the project has a clear distribution requirement.

## Whole-module optimization

Whole-module optimization gives the compiler visibility across files within the same module.

It helps with:

* inlining across files
* generic specialization
* devirtualization
* ARC optimization
* dead code elimination
* abstraction removal

It does not automatically solve cross-module optimization. A module boundary is still a visibility boundary unless additional mechanisms expose implementation details.

Review questions:

* Is the issue within one module or across modules?
* Is release built with optimization enabled?
* Is whole-module optimization or an equivalent optimized build mode active?
* Is the hot function public across a module boundary?
* Would moving a helper into the same module solve the issue?

Do not reason about module performance from Debug builds.

## Cross-module optimization build settings

Some toolchains and build configurations provide cross-module optimization modes that can improve optimizer visibility across Swift modules.

Treat these as build-system-level tools, not API design substitutes.

Use them when:

* the project owns the relevant modules
* release build time impact is acceptable
* the optimization benefit is measured
* module boundaries are architecturally useful and should remain

Do not rely on cross-module optimization to justify poor API boundaries. It may not apply equally to binary dependencies, third-party frameworks, or distribution scenarios.

## Static libraries and static frameworks

Static linking copies library code into the final linked binary.

Potential benefits:

* less dynamic loader work at app launch
* more link-time visibility in some configurations
* simpler runtime dependency graph
* fewer embedded dynamic frameworks

Potential costs:

* larger final app binary in some dependency graphs
* duplicate code if the same static library is linked into multiple dynamic products
* slower clean release links
* less flexibility for separately loaded components
* possible dependency-management complexity

Use static linking when:

* app launch time is sensitive
* the dependency is app-internal
* separate dynamic loading is not needed
* the build system can avoid duplicate linkage
* binary distribution does not require a dynamic framework

Avoid blanket rules. Static linking can improve launch behavior, but it can also create binary-size or dependency-graph problems.

## Dynamic frameworks

Dynamic frameworks are loaded by the dynamic loader at runtime. They can support independent build products, binary distribution, runtime loading patterns, and clearer dependency boundaries.

Potential benefits:

* separate build products
* useful binary distribution model
* better separation for large teams
* faster development builds in some project structures
* runtime sharing for system frameworks
* clearer boundaries for SDKs and plugins

Potential costs:

* launch-time loading and linking work
* embedded framework management
* less optimizer visibility across boundaries
* public/resilient API pressure
* more complex dependency graph
* potential binary size overhead from metadata and boundaries

Use dynamic frameworks when:

* the module is distributed independently
* the framework boundary is a real product boundary
* dynamic loading or replacement is needed
* development/build workflow benefits outweigh launch cost
* the code is an SDK or plugin boundary

Avoid dynamic frameworks for small app-internal modules when launch time is critical and the dynamic boundary has no product or architecture value.

## Mergeable libraries

Mergeable libraries are a tool for combining some benefits of dynamic and static approaches.

They can allow a project to use dynamic-library-like workflows during development while producing release builds with launch-time behavior closer to static linking.

Use mergeable libraries when:

* the project has many dynamic app-internal frameworks
* development builds benefit from dynamic boundaries
* release launch time matters
* the team wants to reduce dynamic loader work without fully redesigning the module graph
* the deployment and Xcode requirements fit the project

Review questions:

* Is the current launch cost caused by many app-internal dynamic frameworks?
* Is the project using Xcode support for mergeable libraries correctly?
* Are libraries merged in the intended build configuration?
* Is the app measuring release launch behavior, not only Debug behavior?
* Does merging preserve debugging and development workflow needs?

Do not assume mergeable libraries fix every modularization problem. They address part of the linking and launch-time trade-off, not API design, specialization, or architectural coupling by themselves.

## Launch-time cost

Launch-time cost can come from many sources:

* dynamic library loading
* symbol binding
* Objective-C runtime setup
* Swift metadata and protocol conformance work
* static initializers
* global lazy initialization triggered early
* dependency graph depth
* framework count
* large binary size
* early synchronous work in app startup

Do not attribute launch regressions only to "too many modules" without a launch trace.

Review questions:

* Is the launch regression measured in a release-like build?
* Is the cost pre-main, app initialization, first frame, or first usable screen?
* Which frameworks are loaded before first frame?
* Are app-internal dynamic frameworks necessary?
* Are static initializers doing real work?
* Are global singletons initialized during launch?
* Would static linking, mergeable libraries, or fewer dynamic boundaries help?
* Would lazy loading or moving work after first frame help more?

Validation tools:

* app launch template in Instruments
* dyld statistics when available
* signposts around startup phases
* release-like device measurements
* binary size reports
* dependency graph inspection

## Build-time trade-offs

Modularization can improve or hurt build time depending on the project.

It can help when:

* modules have clear ownership
* changes are localized
* dependencies point in one direction
* large features can build independently
* public interfaces are stable
* unnecessary recompilation is reduced

It can hurt when:

* modules are too fine-grained
* dependency graph is dense
* public interfaces change often
* generic-heavy APIs cross many boundaries
* code generation or macros run in many targets
* dynamic framework packaging dominates incremental builds
* the app target depends on everything

Review questions:

* Does this module reduce rebuild scope?
* How often does its public interface change?
* Is the dependency graph shallow or tangled?
* Are modules split by ownership or by arbitrary folders?
* Is the build bottleneck compilation, linking, indexing, code generation, or packaging?
* Are test targets forcing broad rebuilds?

Do not use runtime performance arguments to justify build-time modularization without separating the two concerns.

## Binary size

Module and linking choices can affect binary size.

Size can increase because of:

* generic specialization copies
* duplicated static libraries
* large inlined functions
* public metadata
* protocol conformance metadata
* embedded dynamic frameworks
* debug symbols in non-release artifacts
* unused code not stripped due to visibility

Size can decrease because of:

* dead stripping
* removing dynamic framework overhead
* avoiding duplicated abstractions
* merging small libraries
* reducing unnecessary public API
* avoiding aggressive inlining of large functions

Review questions:

* Is the measured size app binary, app bundle, download size, or installed size?
* Are symbols and debug info included?
* Are generic specializations causing code growth?
* Are static libraries duplicated through multiple dynamic products?
* Are many small dynamic frameworks adding metadata overhead?
* Would reducing visibility help dead stripping?

Do not optimize size by removing abstraction until the size report identifies the source.

## Dependency visibility

A module's public interface can expose dependencies unintentionally.

Example:

```swift
import PaymentsCore

public struct CheckoutSummary {
    public var paymentMethod: PaymentMethod
    public var total: Money
}
```

If `PaymentMethod` and `Money` come from another module, clients of this module may become coupled to that dependency through the public interface.

Review questions:

* Does the public API expose implementation dependencies?
* Can dependency types be kept internal?
* Would a local DTO or facade reduce transitive coupling?
* Is `@_implementationOnly import` appropriate for hiding an implementation dependency?
* Is the dependency stable enough to expose publicly?

Use implementation-only imports cautiously because they impose restrictions: public and inlinable APIs cannot expose hidden dependency types.

## `@_implementationOnly import`

`@_implementationOnly import` can hide an imported module from the public interface of the importing module.

It can help reduce transitive dependency exposure when a dependency is only used internally.

Use it when:

* the imported module is an implementation detail
* no public API mentions its types
* no `@inlinable` code exposes or depends on its private details in a way clients need
* the team accepts the underscored attribute's constraints and evolution risk

Be cautious because underscored attributes are not as stable as ordinary language features.

Review rule:

* Prefer clean public API boundaries first.
* Use implementation-only imports to enforce dependency hiding, not to paper over poor module design.

## Public API shape and performance

Public API design affects performance because it controls what clients can see and what the optimizer can assume.

A high-level API can sometimes be faster than many low-level public calls because it keeps the hot loop inside the defining module.

Less ideal:

```swift
public protocol TokenReader {
    func hasMoreTokens() -> Bool
    func readNextToken() -> Token
}
```

If clients call these methods repeatedly across a dynamic or existential boundary, the cost may accumulate.

Often better:

```swift
public protocol TokenReader {
    func readAllTokens() -> [Token]
}
```

Or:

```swift
public func parseTokens(
    from source: Source,
    options: ParseOptions
) -> [Token]
```

This keeps the loop and representation choices inside the module.

Review rule:

* Prefer APIs that move hot loops into the module that owns the implementation.
* Avoid forcing clients to perform many tiny calls across abstraction boundaries.

## Common patterns to flag

Flag these patterns for review:

* public generic helpers used in hot paths without `@inlinable` or another visibility strategy
* public protocols used only by one concrete implementation
* `any Protocol` in tight loops across module boundaries
* many tiny public methods called repeatedly
* app-internal code split into many dynamic frameworks without launch-time justification
* `BUILD_LIBRARY_FOR_DISTRIBUTION` enabled for modules that are never distributed
* `@frozen` added to unstable models
* `@inlinable` added to large implementation-heavy functions
* `@usableFromInline` exposing unstable helpers
* dynamic framework boundaries used only to mirror folders
* dependency types leaking through public APIs
* static libraries duplicated into multiple dynamic frameworks
* launch-time initialization hidden in global state

## Recommended fixes

Choose the smallest fix that addresses the measured problem.

Possible fixes:

* move a hot loop into the defining module
* make a helper internal instead of public
* provide a concrete fast path
* keep public abstraction but add an internal specialized implementation
* reduce type erasure in inner loops
* batch calls across module boundaries
* use `@inlinable` for small stable public hot functions
* use `@usableFromInline` for stable helpers required by `@inlinable`
* use `@frozen` only for stable public value types in library-evolution contexts
* switch app-internal frameworks from dynamic to static when launch time matters
* adopt mergeable libraries when the project fits that model
* reduce dynamic framework count before first frame
* hide implementation dependencies from public API
* split or merge modules based on measured build graph behavior

Avoid:

* adding `@inlinable` broadly
* freezing unstable public models
* making everything static without checking duplicate linkage
* making everything dynamic for architecture purity
* disabling library evolution for distributed binary frameworks
* merging modules solely because of a theoretical optimization concern
* splitting modules solely because a folder is large
* optimizing Debug build launch behavior instead of release-like launch behavior

## Example: public generic helper

Problematic when hot and unspecialized across modules:

```swift
public func firstMatch<Element>(
    in elements: [Element],
    where predicate: (Element) -> Bool
) -> Element? {
    for element in elements {
        if predicate(element) {
            return element
        }
    }

    return nil
}
```

Possible review:

* If this is a tiny stable utility used heavily across module boundaries, `@inlinable` may be reasonable.
* If the predicate captures large state or cannot inline, `@inlinable` may not help much.
* If only one module uses this helper, keep it internal.
* If this is part of a larger algorithm, move the algorithm into the module that owns the data representation.

Do not add `@inlinable` before checking whether this function is actually hot.

## Example: API that forces repeated boundary crossing

Less efficient API shape for heavy use:

```swift
public protocol PermissionStore {
    func contains(_ permission: Permission) -> Bool
}
```

If clients call this thousands of times, each call may cross an abstraction boundary.

A batched API may be better:

```swift
public protocol PermissionStore {
    func containsAny(_ permissions: some Sequence<Permission>) -> Bool
}
```

or:

```swift
public protocol PermissionStore {
    func intersection(with permissions: Set<Permission>) -> Set<Permission>
}
```

This allows the implementation to choose the right representation and keeps more work on one side of the boundary.

## Example: launch-sensitive app modules

If an app has many app-internal dynamic frameworks and launch traces show dynamic loader work before first frame, possible options include:

* convert some internal frameworks to static libraries/frameworks
* use mergeable libraries in supported Xcode configurations
* merge tiny modules that do not represent real ownership boundaries
* delay loading optional features until after first frame
* remove launch-time work from global initializers
* reduce dependency chains among startup modules

Do not decide based only on framework count. Use launch measurements.

## Signs to look for in optimized SIL

For cross-module optimization issues, inspect whether the hot path contains:

* unspecialized generic calls
* `witness_method` in an inner loop
* `open_existential_*` around hot values
* repeated `apply` calls to small wrappers
* `partial_apply` for closure creation
* `class_method` where concrete type should be known
* retained type-erased wrappers
* missed inlining of tiny public functions

SIL should support the performance hypothesis. It should not replace user-visible measurement.

## Signs to look for in launch measurements

Look for:

* many app-internal dynamic libraries loaded before first frame
* expensive static initializers
* early initialization of feature modules not needed at launch
* large dependency chains from the app target
* Objective-C runtime setup from unnecessary classes/categories
* Swift metadata and protocol conformance work around startup
* synchronous work hidden behind global accessors

Separate pre-main work from application initialization and first-screen work.

## Decision rules

### If you see `@inlinable`

Ask:

* Is the function public or otherwise crossing a module boundary?
* Is the function small and stable?
* Is it performance-critical?
* Does the client benefit from seeing the body?
* Are all referenced helpers public or `@usableFromInline`?
* Is this exposing implementation details unnecessarily?

Keep `@inlinable` only when cross-module visibility is intentional.

### If you see `@usableFromInline`

Ask:

* Which `@inlinable` API needs this helper?
* Is the helper's interface stable enough to be ABI-visible?
* Could the helper be public instead, or should the outer function not be `@inlinable`?
* Does the helper expose private representation?

Use it sparingly.

### If you see `@frozen`

Ask:

* Is library evolution relevant?
* Is this type public across a binary boundary?
* Are stored properties or enum cases stable?
* Is optimization worth losing layout/case flexibility?

Do not freeze evolving models casually.

### If you see many dynamic frameworks

Ask:

* Are they app-internal or distributed independently?
* Are they loaded before first frame?
* Is launch time measured?
* Would static linking or mergeable libraries reduce launch cost?
* Are the dynamic boundaries useful for development or product distribution?

Do not convert all dynamic frameworks blindly.

### If you see many modules

Ask:

* Is the issue build time, runtime speed, launch time, or architecture?
* Are modules split by ownership and dependency direction?
* Are hot paths crossing public boundaries?
* Is the dependency graph shallow or dense?
* Are public interfaces stable?

Do not merge or split modules without identifying the target cost.

## Common gotchas

* More modules do not automatically mean better build time.
* Fewer modules do not automatically mean better runtime performance.
* Static linking can improve launch behavior but may increase binary size or duplicate code.
* Dynamic frameworks can help distribution and development workflow but can add launch work.
* Mergeable libraries address linking and launch trade-offs, not poor API boundaries.
* `@inlinable` exposes implementation to clients and should be treated as a commitment.
* `@usableFromInline` is not ordinary internal visibility.
* `@frozen` trades evolution flexibility for optimization opportunities.
* Library evolution is important for binary frameworks but often unnecessary for app-internal modules.
* Public APIs can prevent optimization even when the code is in the same repository.
* A high-level public API can be faster than many tiny public calls.
* Debug launch behavior may differ from release-like launch behavior.
* Framework count alone is not proof of launch cost.
* Optimized SIL can explain a cost, but user-visible measurement should drive decisions.

## Output guidance

When this reference is used, include:

```markdown
## Boundary being reviewed

State whether the issue is module boundary, public API, library evolution, linking, launch time, binary size, or build time.

## Suspected cost

Classify the cost:
cross-module specialization / missed inlining / resilient layout / dynamic loading / binary size / build graph / dependency exposure.

## Evidence to check

Recommend:
optimized SIL, launch trace, build timing, binary size report, dependency graph, Instruments, or release-like device measurement.

## Findings

Explain how the boundary creates or may create the cost.

## Recommended change

Suggest the smallest change:
internalize, batch, specialize, add a concrete fast path, use `@inlinable`, use `@usableFromInline`, freeze, switch linking mode, adopt mergeable libraries, or reshape modules.

## Trade-offs

Call out API stability, ABI commitment, binary size, build time, launch time, team ownership, and maintainability.

## Validation

Explain how to confirm improvement.
```

If the concern is theoretical, say so and avoid changing module structure or ABI attributes without measurement.
