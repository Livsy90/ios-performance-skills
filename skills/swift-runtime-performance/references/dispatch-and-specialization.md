# Dispatch and Specialization

Use this reference when a review involves method dispatch, protocol dispatch, generics, existentials, opaque types, devirtualization, inlining, specialization, Objective-C interop, or cross-module optimizer visibility.

The goal is not to replace every abstraction with concrete code. The goal is to identify whether a call path pays for dynamic behavior, whether that dynamic behavior is required, and whether the optimizer can remove the abstraction in optimized builds.

## Core model

Swift performance questions around dispatch usually come down to three questions:

1. Is the call target known to the compiler?
2. Is dynamic behavior part of the design?
3. Can the optimizer see enough code to inline, devirtualize, or specialize?

Do not assume that source syntax alone tells the final runtime cost. Swift optimization happens after type checking, and optimized SIL often gives a better picture than source-level guessing.

## Dispatch categories

Useful categories for review:

* Direct/static dispatch: the compiler can resolve the target without runtime lookup.
* Class dispatch: a call goes through a class dispatch mechanism because overriding may be possible.
* Witness table dispatch: a call goes through protocol conformance information.
* Objective-C message dispatch: a call uses Objective-C runtime messaging.
* Dynamic replacement / dynamic Swift dispatch: a call is intentionally kept dynamically replaceable or dynamically resolved.

These categories are mental models, not always one-to-one source annotations. The optimizer can transform a more dynamic-looking source call into a more direct call when it can prove the target.

## Direct and static dispatch

Direct dispatch is possible when the compiler knows the exact implementation.

Common cases:

* functions on concrete value types
* `final` class methods
* static methods
* private/internal functions visible to the optimizer
* non-requirement protocol extension methods dispatched through the static type
* devirtualized calls after optimization

Example:

```swift
struct Checksum {
    func combine(_ lhs: UInt64, _ rhs: UInt64) -> UInt64 {
        lhs &* 31 &+ rhs
    }
}

func checksum(_ values: [UInt64], hasher: Checksum) -> UInt64 {
    values.reduce(0) { partial, value in
        hasher.combine(partial, value)
    }
}
```

This code exposes a concrete `Checksum` value. In optimized builds, the compiler may inline and simplify the call path.

Review questions:

* Is this call concrete at the use site?
* Is the implementation visible to the optimizer?
* Is the call small enough to inline profitably?
* Is this path actually hot enough to care?

Do not force direct dispatch if the abstraction is needed and not performance-critical.

## Class dispatch

Non-final class methods may need dynamic dispatch because subclasses can override behavior.

Example:

```swift
class FeedSorter {
    func sort(_ items: [FeedItem]) -> [FeedItem] {
        items.sorted { $0.date > $1.date }
    }
}

final class PinnedFirstSorter: FeedSorter {
    override func sort(_ items: [FeedItem]) -> [FeedItem] {
        items.sorted {
            ($0.isPinned, $0.date) > ($1.isPinned, $1.date)
        }
    }
}
```

This hierarchy may be valid when overriding is part of the design.

Review questions:

* Is subclassing intentional?
* Is the base class an extension point?
* Is the call in a hot path?
* Does the hierarchy model domain behavior or only implementation convenience?
* Can internal code use a concrete final implementation while preserving a public abstraction?

Do not flatten a useful hierarchy only because dynamic dispatch exists.

## `final` as semantic default

Prefer `final` for app-level classes that are not designed for inheritance.

`final` communicates that subclassing is not part of the type's contract. It also gives the optimizer more freedom to devirtualize and inline calls when the concrete type is visible.

Use non-final classes when inheritance is intentional:

* UIKit/AppKit subclassing patterns
* framework extension points
* public or `open` APIs designed for external subclassing
* test seams that intentionally rely on subclassing
* class hierarchies that model real polymorphic behavior
* Objective-C interoperability patterns requiring dynamic behavior

Example:

```swift
final class ReceiptFormatter {
    func title(for receipt: Receipt) -> String {
        "\(receipt.storeName) · \(receipt.total)"
    }
}
```

Here `final` is primarily a design statement. The type is a concrete implementation, not a subclassing point.

Review rule:

* Do not require a measured hot path just to mark an app-level non-inheritable class `final`.
* Require evidence only when presenting `final` as a performance optimization.
* Do not use `final` where subclassing is part of the API contract.

## Protocol witness dispatch

A protocol requirement call may use a witness table. The witness table maps protocol requirements to the concrete implementation for a conforming type.

Example:

```swift
protocol EventEncoder {
    func encode(_ event: AnalyticsEvent) -> Data
}

struct JSONEventEncoder: EventEncoder {
    func encode(_ event: AnalyticsEvent) -> Data {
        event.jsonData
    }
}

func encodeAll<E: EventEncoder>(
    _ events: [AnalyticsEvent],
    using encoder: E
) -> [Data] {
    events.map { encoder.encode($0) }
}
```

The generic function may initially be expressed in terms of a protocol witness call. In optimized builds, if the concrete type is visible and specialization happens, the witness call may disappear.

Review questions:

* Is this protocol requirement called in a hot loop?
* Is the concrete type known at the call site?
* Is the generic function internal to the module?
* Does optimized SIL show specialization?
* Would a concrete helper or generic constraint preserve the design while improving optimizer visibility?

Do not assume every protocol call remains dynamic in final code. Check optimized SIL when it matters.

## Existential dispatch

An existential such as `any EventEncoder` hides the concrete type at that point. Calls through the existential may require opening the existential and dispatching through protocol witness information.

Example:

```swift
func encodeWithSelectedEncoder(
    _ events: [AnalyticsEvent],
    encoder: any EventEncoder
) -> [Data] {
    events.map { encoder.encode($0) }
}
```

This is appropriate when runtime flexibility is required. It may be less ideal when the hot path is actually homogeneous and the existential only exists for local convenience.

Review questions:

* Is runtime heterogeneity required?
* Is the existential stored or only passed through?
* Is the call inside a hot loop?
* Would a generic parameter keep the API clear?
* Does the existential cross an architecture boundary where type erasure is intentional?
* Is boxing, opening, or witness dispatch visible in optimized SIL or Allocations?

Do not automatically replace `any` with generics. Existentials are useful when runtime polymorphism is the design.

## Protocol extension dispatch gotcha

Protocol extension methods that are not protocol requirements are dispatched based on the static type.

Example:

```swift
protocol BadgeProvider {
    var badge: String { get }
}

extension BadgeProvider {
    func decoratedBadge() -> String {
        "[\(badge)]"
    }
}

struct PremiumBadgeProvider: BadgeProvider {
    var badge: String { "Premium" }

    func decoratedBadge() -> String {
        "★ \(badge)"
    }
}

func render(_ provider: any BadgeProvider) -> String {
    provider.decoratedBadge()
}
```

The call to `decoratedBadge()` uses the extension implementation because `decoratedBadge` is not a protocol requirement. The method in `PremiumBadgeProvider` is not an override of a protocol requirement.

If dynamic behavior is intended, put the method in the protocol requirement list:

```swift
protocol DynamicBadgeProvider {
    var badge: String { get }
    func decoratedBadge() -> String
}

extension DynamicBadgeProvider {
    func decoratedBadge() -> String {
        "[\(badge)]"
    }
}
```

Review rule:

* If a protocol extension method is expected to be dynamically customized by conforming types, make it a protocol requirement.
* If static extension behavior is intended, keep it as an extension-only helper.
* Do not confuse this correctness issue with performance alone.

## Generics and specialization

Generics let code be written once while still allowing the optimizer to create concrete specialized versions when it has enough visibility.

Example:

```swift
protocol ScoreRule {
    func score(_ input: Candidate) -> Int
}

func rank<R: ScoreRule>(
    candidates: [Candidate],
    rule: R
) -> [Candidate] {
    candidates.sorted {
        rule.score($0) > rule.score($1)
    }
}
```

When `R` is known and the function body is visible, the optimizer may specialize `rank` for the concrete rule type, inline `score`, and remove witness dispatch.

Specialization is not guaranteed.

It depends on factors such as:

* optimization level
* module boundaries
* visibility of the function body
* code size heuristics
* whether the call site uses concrete types
* whether closures or type-erased wrappers hide the implementation
* whether the optimizer considers specialization profitable

Review questions:

* Is the generic function in the same module as its hot call sites?
* Is the function public across a module boundary?
* Is the generic code called with a small number of concrete types?
* Does specialization increase code size too much?
* Does optimized SIL show specialized versions?
* Is the cost dominated by the generic abstraction or by work inside the function?

Do not say "generic means static dispatch" without checking whether specialization actually occurs.

## Opaque types: `some Protocol`

Opaque types hide the concrete type from API users while preserving a single underlying concrete type for the implementation.

Example:

```swift
protocol ReceiptRenderer {
    func render(_ receipt: Receipt) -> RenderedReceipt
}

func makeRenderer() -> some ReceiptRenderer {
    CompactReceiptRenderer()
}
```

`some ReceiptRenderer` is useful when the caller does not need to know the concrete type, but the implementation can still keep a concrete underlying type.

Use `some` when:

* the implementation returns one concrete type per declaration
* API users should not depend on the concrete type
* existential storage is not needed
* the abstraction is still statically shaped

Do not use `some` when:

* different concrete types must be returned from the same function branch without wrapping
* values of different conforming types must be stored together
* runtime replacement is the purpose of the API

Example requiring existential or type erasure:

```swift
func makeRenderer(compact: Bool) -> any ReceiptRenderer {
    if compact {
        CompactReceiptRenderer()
    } else {
        DetailedReceiptRenderer()
    }
}
```

Review rule:

* Prefer `some` for opaque return positions with a single underlying type.
* Prefer `any` when runtime heterogeneity is required.
* Prefer generics when the caller should provide the concrete type and specialization matters.

## Objective-C message dispatch and Swift interop

Objective-C interoperability can introduce a more dynamic dispatch model.

Common cases:

* calls through Objective-C classes and protocols
* `@objc` APIs used by selectors
* KVO-compatible members
* target-action
* method swizzling
* dynamic replacement patterns
* APIs marked `dynamic`

Be precise:

* `@objc` exposes a declaration to Objective-C.
* `dynamic` tells Swift to use dynamic dispatch.
* Objective-C runtime features such as selectors, KVO, and swizzling require dynamic behavior.
* A Swift-to-Swift call to an `@objc` member is not automatically a reason to rewrite code. Inspect the actual call path if performance matters.

Example:

```swift
final class SearchButtonHandler: NSObject {
    @objc func didTapSearchButton(_ sender: Any) {
        submitSearch()
    }

    private func submitSearch() {
        // Swift implementation details
    }
}
```

The selector-facing method is dynamic by design. Keep the dynamic boundary small and move implementation details into Swift methods when useful.

Review questions:

* Is Objective-C runtime behavior required?
* Is the dynamic call on a hot path or only an event boundary?
* Can dynamic interop be isolated at the edge?
* Is `dynamic` present because of a real runtime requirement?
* Would removing dynamic behavior break KVO, target-action, swizzling, or framework integration?

Do not remove Objective-C interop from framework boundaries where it is required.

## Inlining

Inlining replaces a function call with the function body. It can remove call overhead and expose further optimization opportunities such as constant propagation, ARC elimination, and specialization.

Inlining can also increase code size.

Review questions:

* Is the function tiny and called frequently?
* Does inlining expose more optimization?
* Is the function large or cold?
* Is code size a concern?
* Did the optimizer already inline it?
* Does `@inline(__always)` improve measured performance or only make the source look faster?

Guidance:

* Do not use `@inline(__always)` as a default performance fix.
* Consider `@inline(never)` for large cold paths when code size, compile time, or profiling clarity matters.
* Prefer measuring and inspecting optimized SIL before adding inlining attributes.
* Treat inlining attributes as last-mile tuning, not design tools.

Example:

```swift
@inline(never)
func buildDebugDescription(for payload: LargePayload) -> String {
    // Cold diagnostic path.
    payload.debugDescription
}
```

This can be reasonable when the function is cold and keeping it out of hot code helps code size or profiling clarity.

## Devirtualization

Devirtualization turns a dynamic-looking call into a direct call when the optimizer can prove the target.

Common enablers:

* `final` classes or methods
* private/internal visibility
* whole-module optimization
* concrete local types
* generic specialization
* sealed internal hierarchies visible to the optimizer

Example:

```swift
final class LocalTaxCalculator {
    func tax(for subtotal: Money) -> Money {
        subtotal.percent(8)
    }
}

func totals(for orders: [Order], calculator: LocalTaxCalculator) -> [Money] {
    orders.map { order in
        order.subtotal + calculator.tax(for: order.subtotal)
    }
}
```

Because the concrete final type is visible, the optimizer has more opportunity to inline or devirtualize the call.

Review rule:

* Prefer design clarity first.
* Use `final` where non-inheritance is the intended contract.
* Check optimized SIL when devirtualization matters to a hot path.

## Cross-module specialization

Module boundaries can limit optimizer visibility.

Inside a module, the optimizer often has more freedom to inline, devirtualize, and specialize. Across modules, public APIs may hide implementation details unless the body is made available to clients.

Relevant tools:

* `@inlinable`: exposes a function body to client modules.
* `@usableFromInline`: allows internal declarations to be used from `@inlinable` code.
* `@frozen`: exposes layout assumptions for certain public value types.
* Cross-module optimization: allows more optimization across module boundaries when enabled by the build.

Use these tools carefully.

Example:

```swift
@inlinable
public func clamp<Value: Comparable>(
    _ value: Value,
    lower: Value,
    upper: Value
) -> Value {
    min(max(value, lower), upper)
}
```

This may be reasonable for a tiny, stable, performance-sensitive public helper. It would be a poor default for large functions or APIs whose implementation should remain private.

Review questions:

* Is this public function called in a hot path from another module?
* Is the function small and stable?
* Is the implementation safe to expose as part of the module interface?
* Would `@inlinable` leak implementation details?
* Is code size likely to increase?
* Would moving the hot implementation internal be cleaner?
* Is the issue actually cross-module specialization or something else?

Do not add `@inlinable` only because a function is slow. First identify that the module boundary is the relevant limitation.

## Type erasure

Type erasure can intentionally hide concrete types behind a stable wrapper. It often uses closures, boxes, classes, or existentials internally.

Example:

```swift
struct AnyReceiptRenderer: ReceiptRenderer {
    private let _render: (Receipt) -> RenderedReceipt

    init<R: ReceiptRenderer>(_ renderer: R) {
        _render = renderer.render
    }

    func render(_ receipt: Receipt) -> RenderedReceipt {
        _render(receipt)
    }
}
```

This design may be useful for storage, dependency inversion, or API stability. It may also add closure storage and dynamic calls.

Review questions:

* Why is type erasure needed?
* Is the erased value stored long-term?
* Is rendering called in a hot loop?
* Can the hot path stay generic while the boundary remains erased?
* Is the cost visible in Time Profiler, Allocations, or SIL?
* Would a simpler existential be enough?
* Would a concrete implementation be enough internally?

Do not remove type erasure when it is the correct architectural boundary. Instead, keep erased boundaries at the edges and preserve concrete/generic code inside hot loops.

## Closure dispatch and `partial_apply`

Closures can add both allocation and dispatch cost, especially when escaping, stored, or repeatedly created.

Look for:

* closure creation inside loops
* escaping closures stored as properties
* higher-order functions in measured hot paths
* captured mutable variables
* type-erased callbacks
* `partial_apply` in optimized SIL

Example:

```swift
func makePredicate(
    minimumScore: Int,
    source: ScoreSource
) -> (Candidate) -> Bool {
    { candidate in
        source.score(for: candidate) >= minimumScore
    }
}
```

This is fine as ordinary code. It becomes relevant when many predicates are created repeatedly or used in a very hot loop.

Review questions:

* Is the closure escaping?
* Is it allocated repeatedly?
* Does it capture references?
* Does the closure body inline?
* Would a concrete predicate type make the hot path clearer?
* Is the cost measured or only suspected?

Do not rewrite readable higher-order code unless profiling points to closure overhead or missed optimization.

## SIL patterns

When dispatch or specialization matters, inspect optimized SIL.

Useful signs:

* `function_ref`: direct function reference
* `class_method`: class method dispatch
* `witness_method`: protocol witness dispatch
* `open_existential_*`: existential opening
* `partial_apply`: closure creation
* `apply`: function application
* `alloc_ref`: class allocation
* `alloc_box`: boxed variable
* `strong_retain` / `strong_release`: ARC traffic
* specialized function names containing specialization markers
* absence or presence of the concrete implementation in the hot path

Interpretation:

* `witness_method` in generic source may disappear after specialization.
* `class_method` may disappear after devirtualization.
* `partial_apply` may indicate a closure object or partial application.
* Direct calls can still be expensive if the function body is expensive.
* Dynamic dispatch can be irrelevant if the call is not frequent.

Use SIL to explain measured behavior, not to justify speculative rewrites.

## Decision rules

### If you see a protocol in a hot path

Ask:

* Is the call through a generic parameter or an existential?
* Is the concrete type homogeneous?
* Is runtime heterogeneity required?
* Does optimized SIL show `witness_method`?
* Does specialization remove the witness call?
* Would a generic helper preserve the public API?

Prefer:

* generic/concrete hot path for homogeneous work
* existential/type-erased boundary for storage and runtime flexibility
* measuring before replacing architecture

### If you see `any Protocol`

Ask:

* Is the existential stored, passed, or called in a loop?
* Is boxing visible?
* Is dynamic runtime choice required?
* Would `some` work in a return position?
* Would a generic function work for the hot path?
* Is the existential an intentional module or dependency boundary?

Do not replace `any` automatically.

### If you see `some Protocol`

Ask:

* Does the function really return one underlying concrete type?
* Is API opacity the goal?
* Does the caller need to store heterogeneous values?
* Would `any` be clearer for runtime heterogeneity?

Do not use `some` when the design requires multiple runtime types in one value slot.

### If you see a non-final class

Ask:

* Is subclassing intended?
* Is this a UIKit/AppKit/framework extension point?
* Is it public/open API?
* Would `final` better express the contract?
* Is dynamic dispatch relevant to performance?

Prefer `final` by default for app-level classes not designed for inheritance. Treat performance benefit as a bonus unless the hot path is measured.

### If you see `@objc` or `dynamic`

Ask:

* Is Objective-C runtime behavior required?
* Is this an edge callback or a frequent internal call?
* Can the dynamic boundary be kept small?
* Would removing it break selectors, KVO, swizzling, or framework integration?

Do not remove required interop.

### If you see `@inlinable`

Ask:

* Is this API small, stable, and performance-sensitive?
* Is the body safe to expose to clients?
* Does it depend on internal implementation details?
* Is code size acceptable?
* Is this solving a real cross-module optimization issue?

Do not use `@inlinable` as a general speed annotation.

### If you see `@inline(__always)`

Ask:

* Is there evidence the optimizer failed to inline a hot tiny function?
* Did adding the attribute improve measured performance?
* Did code size grow?
* Is this hiding a larger design issue?

Do not recommend it as a first-line fix.

## Common gotchas

* Dynamic-looking source code may become direct after optimization.
* Concrete-looking code can still allocate or retain heavily.
* `final` is a good semantic default for non-inheritable app-level classes.
* `final` should not be oversold as a standalone performance fix.
* Protocol extension methods are not dynamically overridden unless they are protocol requirements.
* `any Protocol` is not wrong; it is runtime polymorphism with a cost.
* `some Protocol` is opaque, not heterogeneous storage.
* Generics are not automatically faster unless specialization and inlining happen where they matter.
* Specialization can improve speed but may increase code size.
* `@inlinable` is an API and ABI commitment.
* `@inline(__always)` can hurt code size and instruction-cache behavior.
* Objective-C interop is often required at framework boundaries.
* Dispatch overhead rarely matters more than allocation, copying, I/O, parsing, layout, or algorithmic cost unless the call is extremely frequent.
* Debug builds are not useful for final dispatch and specialization conclusions.

## Output guidance

When this reference is used, include:

```markdown
## Dispatch model

Identify whether the relevant call is direct, class-dispatched, witness-dispatched, existential, Objective-C dynamic, or optimized away.

## Specialization/devirtualization opportunity

Explain whether the optimizer can see the concrete implementation and whether specialization or devirtualization is plausible.

## Design requirement

State whether dynamic behavior is required by the API, architecture, framework, or runtime behavior.

## Recommendation

Suggest the smallest change:
`final`, generic helper, concrete internal path, `some`, `any`, erased boundary, moving code inside a module, or no change.

## Validation

Recommend optimized SIL, Time Profiler, benchmark, or binary-size check.
```

If the dispatch concern is theoretical and the call is not in a hot path, say so and avoid unnecessary rewrites.
