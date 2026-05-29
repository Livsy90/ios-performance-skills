---
name: swift-runtime-performance
description: Use this skill when reviewing Swift code for runtime internals and performance trade-offs: allocation, ARC traffic, stack vs heap behavior, method dispatch, protocol witness tables, existentials vs generics, opaque types, copy-on-write, structured concurrency, actor/task overhead, SIL optimizer behavior, unsafe Swift, modularization, linking, launch time, or build-time/runtime trade-offs.
---

# Swift Runtime Performance Expert

Use this skill when analyzing Swift code through the lens of runtime behavior, compiler optimization, memory representation, dispatch, ownership, concurrency scheduling, and module boundaries.

This skill is not a general Swift style guide. It should help identify the runtime cost behind a design choice, explain why the cost may matter, and propose a measurable improvement without sacrificing correctness or API clarity.

## Core principle

Do not optimize from source-code appearance alone.

First determine:

1. Whether the code is in a real hot path.
2. Which runtime cost is suspected.
3. Whether there is evidence from Instruments, benchmarks, SIL, launch traces, or production metrics.
4. Whether the proposed change preserves semantics, readability, and API stability.
5. How the improvement will be validated.

Prefer the least invasive change that removes the measured cost.

## Runtime cost taxonomy

When reviewing code, classify the suspected cost before proposing changes.

Common categories:

* Heap allocation: class instances, closure contexts, boxes, existential storage, temporary buffers.
* ARC traffic: repeated retains/releases, closure captures, reference-heavy value types, weak/unowned lifetime behavior.
* Dispatch: direct calls, class dispatch, witness table dispatch, Objective-C message dispatch.
* Copying and destruction: large values, copy-on-write uniqueness checks, temporary values, ownership traffic.
* Specialization: generic code that does not specialize, type erasure in hot paths, cross-module abstraction.
* Concurrency runtime: actor hops, executor hops, task creation, unbounded task groups, MainActor blocking.
* Unsafe boundary: pointer lifetime, memory binding, aliasing, bounds, escaping buffers.
* Module and linking boundaries: resilience, dynamic framework loading, missing inlining, launch-time costs.

Do not treat any single category as automatically bad. Cost matters when it is frequent, blocking, allocation-heavy, or visible in user-facing latency.

## Review workflow

Use this workflow for performance reviews.

1. Identify the performance question.

   * Is the concern launch time, scrolling, CPU usage, memory pressure, app responsiveness, binary size, build time, or a local hot loop?

2. Locate the hot path.

   * Prefer measured evidence.
   * If evidence is missing, state what should be measured before large rewrites.

3. Classify the runtime cost.

   * Allocation, ARC, dispatch, copying, specialization, concurrency, unsafe boundary, or modularization.

4. Inspect the abstraction boundary.

   * Is the cost caused by a protocol, class hierarchy, closure, actor, module boundary, or type-erased wrapper?

5. Choose the smallest useful change.

   * Avoid broad rewrites.
   * Prefer local changes that improve optimizer visibility, reduce allocation, simplify ownership, or make the intended abstraction explicit.

6. Provide validation.

   * Time Profiler, Allocations, Swift Concurrency instrument, launch trace, benchmark, optimized SIL, or assembly when needed.

## Measurement guidance

Ask for or recommend the right evidence:

* Time Profiler: CPU hot paths, dispatch overhead, repeated small calls.
* Allocations: object churn, closure allocation, existential boxing, temporary buffers.
* Swift Concurrency instrument: actor contention, MainActor blocking, task count, continuation misuse.
* Leaks / Memory Graph: ownership cycles and unexpected retained graphs.
* App launch instruments: dyld work, framework loading, initialization cost.
* Signposts: user-level timing around screen load, parsing, rendering, or background pipelines.
* Optimized SIL: missed specialization, retain/release traffic, witness calls, boxes, closure creation.
* XCTest performance tests or Swift Benchmark: repeatable micro/macro measurements.

Do not use Debug builds to reason about final runtime performance. Many important optimizations require optimized builds.

## Memory: stack, heap, and storage location

Avoid simplistic rules like "struct is stack" or "class is heap and therefore slow."

Use a more precise model:

* Value types are often stored inline in their containing context.
* A value can live in a stack frame, heap object, array buffer, closure context, existential container, or global storage.
* Class instances and actors have identity and are heap-managed.
* Escaping closures often require a heap-allocated capture context.
* Standard value types such as `Array`, `String`, `Dictionary`, `Set`, and `Data` may use reference-backed storage internally.
* A value stored inside a heap object lives inside that heap allocation even if the value itself is a struct.

Review questions:

* Does this code allocate repeatedly in a hot loop?
* Does an escaping closure extend the lifetime of captured values?
* Does a value look cheap but contain reference-backed storage?
* Is an existential or closure forcing boxing?
* Is a class used only to share mutable state that could be modeled differently?

Example:

```swift
struct SearchSnapshot {
    var query: String
    var resultIDs: [ResultID]
    var filters: [Filter]
}

final class SearchScreenState {
    var snapshot: SearchSnapshot
    var selectedID: ResultID?

    init(snapshot: SearchSnapshot) {
        self.snapshot = snapshot
    }
}
```

`SearchSnapshot` has value semantics, but it contains reference-backed storage through `String` and arrays. When stored in `SearchScreenState`, the snapshot value is part of the class instance's heap allocation. Copying the snapshot does not immediately deep-copy all buffers, but ARC and copy-on-write behavior may still appear in hot paths.

## ARC and ownership

ARC cost usually appears as repeated retain/release traffic around class references, reference-backed value storage, escaping closures, or type-erased wrappers.

Look for:

* Reference-heavy values copied through several layers.
* Closures capturing large object graphs.
* Short-lived class wrappers created frequently.
* APIs that force escaping closures when non-escaping would be enough.
* Repeated creation and destruction of temporary reference containers.
* Weak references used in hot paths without a correctness reason.

Do not recommend `weak` or `unowned` as a performance trick.

Use them for ownership semantics:

* `weak` when the referenced object may disappear and nil is a valid state.
* `unowned` only when the referenced object is guaranteed to outlive the reference.
* Strong references when ownership is direct and lifetime should be extended.

Prefer localizing captures when a closure does not need the whole object.

Instead of capturing a large owner implicitly:

```swift
final class ReportCoordinator {
    private let formatter: ReportFormatter
    private let cache: ReportCache

    func makeExporter() -> (Report) -> Data {
        { report in
            self.cache.store(report)
            return self.formatter.encode(report)
        }
    }
}
```

Capture only what the closure needs:

```swift
final class ReportCoordinator {
    private let formatter: ReportFormatter
    private let cache: ReportCache

    func makeExporter() -> (Report) -> Data {
        let formatter = formatter
        let cache = cache

        return { report in
            cache.store(report)
            return formatter.encode(report)
        }
    }
}
```

This does not remove ARC entirely, but it makes ownership explicit and avoids retaining unrelated state through `self`.

## Method dispatch

When dispatch cost is suspected, first check whether the call is actually relevant to the performance question. Most app code should prioritize clear design over dispatch micro-optimizations.

Useful categories:

* Direct/static dispatch: concrete functions where the compiler can resolve the target.
* Class dispatch: calls through overridable class methods.
* Witness table dispatch: calls through protocol requirements.
* Objective-C message dispatch: dynamic Objective-C runtime dispatch, often through interop features.

Review questions:

* Is this call inside a tight loop, parser, renderer, serializer, layout calculation, or frequently executed data pipeline?
* Does dynamic dispatch provide real flexibility here?
* Can the compiler see the concrete type?
* Would `final`, a concrete helper, or a generic function make the intended dispatch model clearer?
* Is this public API boundary intentionally dynamic?

### `final` as the default for non-inheritable types

Prefer `final` for app-level classes that are not explicitly designed for inheritance.

This is primarily a semantic decision: it communicates that subclassing is not part of the type's contract. It also gives the optimizer more freedom to devirtualize calls and inline implementations when the concrete type is visible.

Use non-final classes when inheritance is intentional:

* Framework extension points.
* UIKit/AppKit subclassing patterns.
* Public or `open` APIs designed for external subclassing.
* Class hierarchies that model real polymorphic behavior.
* Test seams that intentionally rely on subclassing.
* Objective-C interoperability patterns that require dynamic behavior.

Do not present `final` as a magic performance fix. In many app-level cases it is simply the correct default. Its performance impact matters most when calls are frequent, dynamic dispatch is visible in a hot path, or the optimizer cannot otherwise prove the concrete implementation.

Example:

```swift
protocol DiscountPolicy {
    func discount(for subtotal: Money) -> Money
}

final class LoyaltyDiscountPolicy: DiscountPolicy {
    func discount(for subtotal: Money) -> Money {
        subtotal.percent(5)
    }
}
```

Here `final` is not a micro-optimization. It states that `LoyaltyDiscountPolicy` is a concrete implementation, not a subclassing point.

When polymorphism is part of the design, keep it explicit:

```swift
class PricingRule {
    func adjustedPrice(_ value: Money) -> Money {
        value
    }
}

final class RegionalPricingRule: PricingRule {
    override func adjustedPrice(_ value: Money) -> Money {
        value.applyingRegionalTax()
    }
}
```

This hierarchy may be valid if the system intentionally models overridable pricing behavior. Do not flatten useful polymorphism without evidence.

## Protocols, existentials, generics, and opaque types

Do not say "existentials are bad" or "generics are always faster."

Use this model:

* Generics are useful when the call path is homogeneous and specialization is likely.
* `some Protocol` preserves a concrete underlying type while hiding it from the API user.
* `any Protocol` is useful when runtime heterogeneity or type erasure is needed.
* Existentials may add witness dispatch, storage abstraction, and sometimes boxing.
* Generics can still fail to specialize when optimizer visibility is limited, especially across module boundaries.

Prefer generics or concrete types when:

* The same concrete type flows through the hot path.
* The function is called frequently.
* The optimizer can see the implementation.
* The abstraction exists only for local convenience.

Prefer `any Protocol` when:

* Values of different concrete types must be stored together.
* Runtime replacement is a real design requirement.
* The value crosses a plugin, dependency inversion, or architectural boundary.
* The code is not performance-critical.
* API simplicity is more important than specialization.

Example:

```swift
protocol PayloadTransform {
    func transform(_ input: Payload) -> Payload
}

func runPipeline<T: PayloadTransform>(
    _ transform: T,
    payloads: [Payload]
) -> [Payload] {
    payloads.map { transform.transform($0) }
}

func runDynamicPipeline(
    _ transforms: [any PayloadTransform],
    payload: Payload
) -> Payload {
    transforms.reduce(payload) { current, transform in
        transform.transform(current)
    }
}
```

The generic version fits a homogeneous hot path. The existential array fits a dynamic pipeline where different transform implementations must be stored together.

## Copy-on-write and large values

Value semantics do not mean "no heap traffic."

Common Swift containers use copy-on-write storage. Copies are often cheap until mutation, but ARC, uniqueness checks, and buffer copies can still matter in hot paths.

Look for:

* Copy-then-mutate patterns in loops.
* Large values passed across many layers and then modified.
* Structs that hide arrays, dictionaries, strings, data buffers, or custom COW storage.
* Repeated transformations that create intermediate collections.
* Accidental mutation of shared buffers.

Example:

```swift
func markVisibleRows(_ rows: [DashboardRow], visibleIDs: Set<RowID>) -> [DashboardRow] {
    var updated = rows

    for index in updated.indices {
        updated[index].isVisible = visibleIDs.contains(updated[index].id)
    }

    return updated
}
```

This may be fine for small inputs. For large arrays in a frequently updated UI path, inspect whether repeated copy-on-write checks and element mutations are contributing to cost. A different representation, batched diff, or in-place mutation at the owning boundary may be better.

Review rule:

* Do not replace value semantics with reference semantics by default.
* First check where copying occurs, whether mutation follows copying, and whether a better ownership boundary exists.

## Structured concurrency and runtime cost

Swift concurrency performance issues often come from task lifetime, actor isolation, blocking work, or unbounded fan-out.

Check for:

* Long-running work on `MainActor`.
* CPU-heavy work inside actor isolation.
* Repeated actor/executor hops in a hot path.
* Unbounded task creation from large collections.
* Blocking synchronous APIs inside async tasks.
* `Task.detached` used as a default background primitive.
* Missing cancellation checks before expensive work.
* Continuations that can resume zero times or more than once.

Prefer structured concurrency:

* `async let` for a small, fixed number of child operations.
* Task groups for dynamic child work.
* Bounded task groups when input size can be large.
* Unstructured tasks only when lifetime is intentionally independent.

Actor guidance:

* Use actors for async consistency boundaries.
* Do not put all work inside an actor just because the actor owns state.
* Read or update actor state while isolated.
* Move pure CPU work outside actor isolation when possible.
* Batch actor operations when repeated calls would cross the same boundary many times.

Example:

```swift
actor ThumbnailIndex {
    private var records: [ImageID: ThumbnailRecord] = [:]

    func records(for ids: [ImageID]) -> [ThumbnailRecord] {
        ids.compactMap { records[$0] }
    }
}
```

A single batched call is usually better than asking the actor for each ID separately from a loop.

For pure work that does not need actor state:

```swift
actor ImportSession {
    private var importedIDs: Set<ItemID> = []

    func register(_ id: ItemID) {
        importedIDs.insert(id)
    }

    nonisolated func normalize(_ raw: RawItem) -> NormalizedItem {
        NormalizedItem(raw)
    }
}
```

Use `nonisolated` only when the function truly does not access isolated state.

For Swift 6.2+ code, do not assume `async` means background execution. Check actor isolation and whether the function should remain caller-isolated, be `nonisolated`, or explicitly leave the current actor with `@concurrent` when that is the intended behavior.

## Blocking and the cooperative pool

Flag blocking operations inside Swift tasks:

* Synchronous file I/O in async code.
* Synchronous network APIs.
* Semaphore waits.
* Condition waits.
* Long `Thread.sleep` calls.
* Long-held locks around expensive work.
* Blocking legacy SDK calls.

Prefer async APIs. If a blocking API must be used, isolate it behind a small adapter, run the blocking operation outside the Swift concurrency cooperative pool, and bridge back with a checked continuation when appropriate.

Use `Task.sleep` instead of `Thread.sleep` inside async code.

## Continuations

When reviewing continuation-based wrappers:

* Prefer checked continuations unless there is measured reason not to.
* Verify every path resumes exactly once.
* Check success, failure, cancellation, timeout, and early-return paths.
* Avoid storing continuations without a clear ownership and cancellation policy.
* Keep the unsafe or legacy callback boundary small.

Bad patterns:

* Callback may never fire.
* Callback may fire multiple times.
* Error path forgets to resume.
* Cancellation does not clean up the underlying operation.
* Continuation escapes into shared mutable state without protection.

## SIL and compiler optimization

When source-level reasoning is not enough, inspect optimized SIL.

Look for:

* `alloc_ref`: class allocation.
* `alloc_box`: boxed storage.
* `partial_apply`: closure creation.
* `strong_retain` / `strong_release`: ARC traffic.
* `class_method`: class dispatch.
* `witness_method`: protocol witness dispatch.
* `open_existential_*`: existential opening.
* `copy_value` / `destroy_value`: ownership traffic.
* Generic functions that remain unspecialized in a hot path.
* Inlining that did or did not happen.

Do not rely on SIL from Debug builds for performance decisions.

Optimization guidance:

* Prefer `final` by default for app-level classes that are not designed for inheritance.
* Treat `final` as a semantic design signal first and an optimization enabler second.
* Prefer concrete or generic hot paths when specialization matters.
* Avoid `@inline(__always)` as a default fix.
* Consider `@inline(never)` for large cold functions if code size or compile time is a concern.
* Use performance attributes only after measuring or inspecting optimized output.

## Unsafe Swift

Unsafe Swift is a boundary management tool, not a default performance strategy.

Use unsafe APIs only when:

* A safe API cannot express the operation efficiently enough.
* Measurement shows abstraction overhead matters.
* Pointer lifetime, binding, alignment, and aliasing assumptions are documented.
* The unsafe region is small.
* The unsafe operation is wrapped behind a safe API.
* Tests cover bounds, lifetime, mutation, and invalid input cases.

Review unsafe code for:

* Pointer escaping beyond valid lifetime.
* Incorrect memory binding.
* Misaligned access.
* Out-of-bounds access.
* Mutation while a buffer is assumed immutable.
* Aliasing assumptions that safe Swift would normally prevent.
* Thread-safety assumptions around shared memory.

Do not recommend unsafe code before considering:

* Algorithmic changes.
* Better data layout.
* Reducing intermediate allocations.
* Borrowing or in-place APIs.
* Specialization.
* Existing standard library APIs.

## Modularization, specialization, and linking

Modularization affects more than build time. It can also affect optimizer visibility, launch time, binary size, and ABI/API evolution.

Separate these concerns:

* Build-time cost.
* Runtime specialization.
* Inlining across module boundaries.
* Public API resilience.
* Dynamic framework loading.
* Launch-time linking.
* Binary size.
* Team ownership and architecture boundaries.

Review questions:

* Does hot generic code cross a module boundary?
* Can the optimizer see the implementation?
* Is the API public only because of module structure?
* Would moving a hot implementation internal improve optimization?
* Is `@inlinable` justified by a stable, small, performance-critical public API?
* Is `@frozen` acceptable, given the loss of layout flexibility?
* Are many dynamic frameworks contributing to launch cost?
* Does the module split improve ownership enough to justify runtime/build trade-offs?

Attribute guidance:

* `@inlinable` exposes a function body to clients and should be treated as an API commitment.
* `@usableFromInline` makes implementation details ABI-visible enough for inlinable code.
* `@frozen` publishes layout assumptions for optimization but reduces future evolution flexibility.
* Do not add these attributes only because a function is slow. First identify the module-boundary problem.

## Decision rules

Use these rules during code review.

### If you see a non-final class

Ask:

* Is subclassing an intentional part of the design?
* Is the type a framework extension point, UIKit/AppKit subclass, test seam, or public/open API?
* Would `final` better express the type's intended usage?
* Is dynamic dispatch relevant in this path?

Prefer `final` by default for app-level classes that are not designed for inheritance.

Do not require a measured hot path just to mark a class `final`. Require evidence only when presenting `final` as a performance optimization.

### If you see a class hierarchy

Ask:

* Is overriding needed?
* Does the hierarchy express domain behavior or only implementation convenience?
* Could concrete final types be used internally while preserving the public abstraction?
* Is the call in a hot path where dynamic dispatch actually matters?

Do not flatten a useful hierarchy without evidence.

### If you see `any Protocol`

Ask:

* Is runtime heterogeneity required?
* Is this in a hot path?
* Would a generic or opaque type preserve the design?
* Is the existential stored long-term, passed briefly, or used inside a loop?
* Is boxing or witness dispatch visible in measurement or SIL?

Do not automatically replace `any` with generics.

### If you see many closures

Ask:

* Are they escaping?
* What do they capture?
* Are they allocated repeatedly?
* Can a value or dependency be captured instead of the whole owner?
* Would a small concrete type make lifetime clearer?

### If you see actors

Ask:

* What state is protected?
* Is long-running work executed while isolated?
* Are callers repeatedly crossing the actor boundary?
* Can requests be batched?
* Can pure work be `nonisolated`?
* Is MainActor doing non-UI work?

### If you see custom COW

Ask:

* Is uniqueness checked at the right boundary?
* Are mutations causing repeated full copies?
* Is thread safety clearly documented?
* Does the type preserve value semantics?
* Are storage references leaking through the API?

### If you see unsafe code

Ask:

* What safe abstraction is being replaced?
* What measured cost justifies this?
* What lifetime and aliasing rules must hold?
* Where is the safe wrapper?
* What tests protect this boundary?

### If you see many modules or frameworks

Ask:

* Is the issue build time, launch time, runtime speed, or architecture?
* Is hot code split across public module boundaries?
* Are dynamic frameworks necessary?
* Would merging, static linking, or internalizing a hot path help?
* Are ABI attributes being used intentionally?

## Common gotchas

* `struct` does not guarantee stack allocation.
* `class` does not automatically mean a performance problem.
* Value semantics can still involve heap storage and ARC.
* `Array`, `String`, `Dictionary`, `Set`, and `Data` can hide reference-backed storage.
* `any Protocol` is a useful abstraction with runtime cost, not a mistake.
* Generics help most when specialization happens.
* `some Protocol` is not a universal replacement for `any Protocol`.
* `final` is a good default for app-level classes that are not designed for inheritance.
* `final` should not be oversold as a standalone performance fix.
* `@inline(__always)` can increase code size and should not be the first fix.
* `@inlinable` is an API and ABI commitment.
* `weak` and `unowned` are ownership tools, not performance tools.
* Actor hops are not the same as OS thread context switches.
* `async` does not automatically mean parallel or background.
* `Task.detached` should not be the default way to leave the main actor.
* Unsafe code can be slower, less optimizable, or incorrect if used casually.
* Debug-build behavior is not reliable evidence for optimized runtime performance.

## Output format

When responding to a review request, use this structure:

```markdown
## Summary

Briefly state whether the code has a likely runtime issue or whether the concern is mostly theoretical.

## Suspected runtime cost

Classify the issue:
allocation / ARC / dispatch / existential / copying / specialization / concurrency / unsafe / modularization.

## Evidence to check

List the measurement or compiler evidence needed:
Instruments, Allocations, Swift Concurrency instrument, optimized SIL, launch trace, benchmark, or XCTest performance test.

## Findings

Explain the concrete source-level pattern and why it may cost runtime work.

## Recommended change

Suggest the smallest safe change. Include code only when it clarifies the recommendation.

## Trade-offs

Mention readability, API flexibility, binary size, ABI stability, build time, or maintainability costs.

## Validation

Explain how to confirm the improvement.
```

If the code is not likely in a hot path, say so and avoid low-level rewrites.

## Reference routing

When reference files are available, read them selectively:

* `references/allocation-and-layout.md` for stack/heap, object layout, closure boxes, existential storage.
* `references/arc-and-ownership.md` for retain/release traffic, closure captures, weak/unowned, lifetime.
* `references/dispatch-and-specialization.md` for direct dispatch, class dispatch, witness dispatch, Objective-C dispatch, generics.
* `references/existentials-generics-opaque-types.md` for `any`, `some`, generic specialization, type erasure.
* `references/cow-and-large-values.md` for copy-on-write, large structs, collection mutation, custom COW.
* `references/concurrency-runtime.md` for actors, tasks, cancellation, priority, continuations, cooperative pool.
* `references/sil-inspection.md` for optimized SIL patterns and compiler diagnostics.
* `references/unsafe-swift.md` for pointer safety, memory binding, aliasing, unsafe wrappers.
* `references/modularization-and-linking.md` for module boundaries, `@inlinable`, `@frozen`, static/dynamic linking, launch cost.

If references are missing, rely on this file and state any uncertainty.
