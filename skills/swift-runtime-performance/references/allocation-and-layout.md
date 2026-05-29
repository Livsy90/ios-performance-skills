# Allocation and Layout

Use this reference when a review involves stack vs heap behavior, value storage, class/actor allocation, closure capture contexts, existential storage, object layout, or unexpected allocations in Swift code.

The goal is not to label types as "fast" or "slow". The goal is to identify where values are stored, what lifetime they require, whether allocation is repeated, and whether the storage model matters for the measured performance problem.

## Core model

Avoid simple rules such as:

* `struct` means stack.
* `class` means slow.
* value type means no heap allocation.
* reference type means performance problem.
* generic code means zero allocation.
* existential code always allocates.

Use a storage-and-lifetime model instead.

A Swift value can be stored in many places:

* a stack frame
* a heap object
* a closure capture context
* an array or dictionary buffer
* an existential container
* a global or static storage location
* task or async state that survives suspension
* an optimizer-created temporary
* a box created to extend lifetime or support mutation

The same source-level type can appear in different storage locations depending on context.

## What to inspect first

Before proposing a change, identify:

1. Is this allocation on a hot path?
2. Is allocation repeated frequently?
3. Is the value escaping its local scope?
4. Is the value stored inside another heap object?
5. Is the value captured by an escaping closure?
6. Is the value passed through an existential?
7. Is the value inside reference-backed storage such as `Array`, `String`, `Dictionary`, `Set`, or `Data`?
8. Is the optimizer able to remove or stack-promote the temporary?
9. Is there evidence from Instruments, optimized SIL, or benchmarks?

Do not rewrite storage models without evidence that allocation or layout matters.

## Stack storage

Stack storage is usually associated with local values whose lifetime is known and bounded by the current function activation.

A stack frame may contain:

* local value variables
* function parameters
* temporary values
* inline fields of local structs or enums
* non-escaping closure context when the optimizer can keep it local

Stack allocation is cheap, but do not describe it as literally free. It can still contribute to frame size, register pressure, copying, and optimizer decisions.

Do not optimize for stack placement directly unless allocation or copying appears in measurement. The optimizer may already scalar-replace, eliminate, or move storage.

Example:

```swift
struct LayoutMetrics {
    var width: Double
    var height: Double
    var scale: Double
}

func area(for metrics: LayoutMetrics) -> Double {
    metrics.width * metrics.height * metrics.scale
}
```

`LayoutMetrics` is a small value used locally. In optimized code, this may be passed in registers, kept in the stack frame, or optimized away entirely. Do not assume a specific physical layout without inspecting optimized output.

## Heap storage

Heap storage commonly appears when a value needs identity, shared ownership, dynamic lifetime, or storage that cannot be represented as a simple local temporary after optimization.

Common heap-backed or heap-related cases:

* class instances
* actor instances
* escaping closure contexts
* boxed captured variables
* reference-backed standard library buffers
* existential boxes for values that cannot fit inline
* async/task state that must survive suspension and may be stored outside a simple synchronous stack frame
* Objective-C and Core Foundation objects
* storage created by type-erasure wrappers

Heap allocation matters most when it is frequent, large, contended, or visible in user-facing latency.

The important review question is usually allocation frequency and object graph shape, not the mere presence of a class.

Example:

```swift
final class FeedRenderState {
    var visibleIDs: [PostID]
    var scrollOffset: Double

    init(visibleIDs: [PostID], scrollOffset: Double) {
        self.visibleIDs = visibleIDs
        self.scrollOffset = scrollOffset
    }
}
```

`FeedRenderState` is a heap object because it is a class instance. Its `visibleIDs` field is stored as part of that object, but the array elements live in separate reference-backed storage managed by the array.

## Value types inside heap objects

A struct stored in a class instance is not stored on the stack just because it is a struct. It is stored inline within the class object's heap allocation, unless its own fields point to separate storage.

Example:

```swift
struct PlaybackPosition {
    var seconds: Double
    var rate: Double
}

final class PlayerSession {
    var position: PlaybackPosition

    init(position: PlaybackPosition) {
        self.position = position
    }
}
```

`PlaybackPosition` has value semantics, but the `position` property is part of the `PlayerSession` heap object.

Review rule:

* Distinguish value semantics from physical storage location.
* A value can preserve independent copy behavior while being stored inside heap memory.

## References inside value types

A struct can contain references or reference-backed values.

Example:

```swift
struct TimelineSection {
    var title: String
    var itemIDs: [ItemID]
    var metadata: SectionMetadata
}

final class SectionMetadata {
    var source: String

    init(source: String) {
        self.source = source
    }
}
```

Copying `TimelineSection` copies the value container, but it may also retain shared storage for `String`, `Array`, and `SectionMetadata`.

Review questions:

* Does the value type contain class references?
* Does it contain large copy-on-write storage?
* Is the value copied across many architectural layers?
* Is it mutated after being copied?
* Does it accidentally provide value-looking syntax over shared mutable state?

## Class references inside structs

When a struct stores a class reference, copying the struct copies the reference, not the referenced object.

Example:

```swift
final class ImageDecodeCache {
    private var entries: [ImageKey: DecodedImage] = [:]
}

struct ImagePipeline {
    var cache: ImageDecodeCache
    var configuration: DecodeConfiguration
}
```

Two copies of `ImagePipeline` can point to the same `ImageDecodeCache`. The struct itself may have value semantics for its stored fields, but the cache has shared identity.

Review rule:

* Do not infer independent state from struct syntax when the struct contains references.
* Check whether shared reference state is intentional and thread-safe.

## Closure allocation and capture contexts

Closures are a common source of hidden allocation and lifetime extension.

A non-escaping closure can often remain local, inline, or disappear after optimization. An escaping closure usually requires a capture context whose lifetime can extend beyond the current function call.

Look for:

* closures created inside loops
* closures stored in arrays or dictionaries
* closures assigned to object properties
* callbacks crossing async or actor boundaries
* closures capturing `self` when only one dependency is needed
* closures capturing mutable local variables that require boxing

Example:

```swift
final class MetricsReporter {
    private let sink: MetricsSink
    private let clock: Clock

    init(sink: MetricsSink, clock: Clock) {
        self.sink = sink
        self.clock = clock
    }

    func makeHandler() -> (Event) -> Void {
        let sink = sink
        let clock = clock

        return { event in
            sink.record(event, at: clock.now)
        }
    }
}
```

The escaping closure still needs storage, but its captured state is explicit and narrow. It does not retain the whole reporter object.

Review questions:

* Is the closure escaping?
* What values does it capture?
* Is the closure created repeatedly?
* Could it be replaced with a concrete helper type?
* Could captured dependencies be narrowed?
* Does capturing a mutable local force a heap box?

## Captured mutable variables

A mutable variable captured by an escaping closure may require boxed storage so that the closure and original scope can share the same variable.

Example:

```swift
func makeCounter() -> () -> Int {
    var value = 0

    return {
        value += 1
        return value
    }
}
```

The `value` variable must outlive the function call, so the compiler needs storage that survives beyond the stack frame.

This is correct behavior, not automatically a problem. It becomes relevant when similar closure boxes are created repeatedly in hot paths.

## Existential storage

An existential value such as `any Protocol` is a runtime container for a value whose concrete type is not statically exposed at that point.

An existential may involve:

* inline storage for small values
* boxed storage for larger or otherwise unsuitable values
* type metadata
* witness tables for protocol requirements
* dynamic opening when the value is used

Review questions:

* Is the existential used in a hot loop?
* Is the concrete type actually homogeneous?
* Is runtime heterogeneity required?
* Is the value stored long-term or only passed through?
* Is boxing visible in Allocations or optimized SIL?
* Would a generic, opaque type, or concrete helper preserve the design?

Example:

```swift
protocol CellFormatter {
    func title(for model: CellModel) -> String
}

func renderHomogeneous<F: CellFormatter>(
    models: [CellModel],
    formatter: F
) -> [String] {
    models.map { formatter.title(for: $0) }
}

func renderMixed(
    models: [CellModel],
    formatters: [any CellFormatter]
) -> [String] {
    zip(models, formatters).map { model, formatter in
        formatter.title(for: model)
    }
}
```

The generic version fits a homogeneous formatter path. The existential version fits a runtime-mixed formatter path. Neither is universally better; the storage model should match the design.

## Arrays, dictionaries, strings, and reference-backed storage

Many standard value types use heap-backed storage internally.

Common examples:

* `Array`
* `String`
* `Dictionary`
* `Set`
* `Data`

Copying these values often copies a small value header and shares storage until mutation. This is usually efficient, but it still can involve reference counting, uniqueness checks, and occasional buffer copies.

Example:

```swift
func appendMarker(to ids: [MessageID], marker: MessageID) -> [MessageID] {
    var copy = ids
    copy.append(marker)
    return copy
}
```

This may be cheap when the buffer is uniquely referenced and has capacity. It may allocate and copy when the buffer is shared or needs growth.

Review questions:

* Is mutation performed after copying?
* Is this repeated for many elements?
* Is capacity reserved when growth is expected?
* Can mutation happen at the owning boundary instead?
* Would a lazy view or streaming approach avoid intermediate storage?

## Async frames and task storage

Async functions may need to preserve state across suspension points. Values that live across suspension points become part of the async function state. Depending on optimization and runtime behavior, that state may not behave like an ordinary synchronous stack frame.

Do not assume that every `await` creates a heap allocation. Also do not assume that async code has the same lifetime behavior as synchronous stack-only code. Inspect optimized output or Instruments before attributing a performance issue to async state allocation.

Example:

```swift
func loadProfile(id: UserID, service: ProfileService) async throws -> ProfileViewModel {
    let profile = try await service.profile(id: id)
    let settings = try await service.settings(for: id)

    return ProfileViewModel(profile: profile, settings: settings)
}
```

Values that must survive between suspension points may be stored as part of the async function state. This is usually the right model, but it matters when large temporary values are kept alive across `await` unnecessarily.

Review questions:

* Does a large value live across an `await` even though it is no longer needed?
* Can expensive temporary state be scoped more tightly?
* Is a task retaining a large object graph longer than intended?
* Is `Task {}` or `Task.detached {}` extending lifetime beyond the caller?

## Actor instances

Actors are reference types with identity and isolated mutable state. Their allocation model is closer to class instances than to plain values, while calls across actor isolation introduce concurrency runtime behavior that should be analyzed separately.

Use actors when:

* state must be protected across async access
* the API is naturally async
* the actor represents a consistency boundary
* serialization of access is part of the design

Be cautious when:

* tiny synchronous state would be better protected by a narrow lock
* CPU-heavy work is placed inside actor isolation
* callers repeatedly cross the actor boundary for small reads
* the actor becomes a global bottleneck

Example:

```swift
actor RecentSearches {
    private var terms: [String] = []

    func snapshot() -> [String] {
        terms
    }

    func replace(with newTerms: [String]) {
        terms = newTerms
    }
}
```

The actor protects access to its stored state, but the returned array is still a value with copy-on-write storage behavior outside the actor boundary.

For actor hop, executor, task, and contention analysis, use `references/concurrency-runtime.md`.

## Object layout: use cautiously

Avoid relying on exact object layout details unless you are working on runtime-level diagnostics or carefully verified low-level code.

It is safe to reason at this level:

* class instances are heap-managed reference objects
* classes have identity
* class references participate in ARC
* dynamic dispatch may use runtime metadata
* stored properties are part of the object representation
* subclasses extend the stored representation of their superclass

Avoid presenting implementation details as stable API:

* exact header size
* exact refcount location
* exact metadata fields
* exact object alignment
* exact class layout across compiler/runtime versions

For performance review, the actionable point is usually allocation frequency, object graph size, ARC traffic, or dispatch behavior, not the precise byte layout.

## Enum and optional layout

Enums may have compact representations when the compiler can use spare bits or encode cases efficiently. Optional references are commonly optimized using the null pointer representation.

Do not assume every enum case requires extra allocation.

Example:

```swift
enum LoadState {
    case idle
    case loading(TaskID)
    case loaded(ProfileSummary)
    case failed(any Error)
}
```

The payloads determine the representation. If a payload contains an existential such as `any Error`, the representation may involve existential storage, witness metadata, or reference-backed error values depending on the concrete error type.

Inspect hot paths rather than assuming all enums are cheap or expensive.

## Generics and allocation

Generic code does not automatically allocate. However, generic abstraction can hide layout details until specialization.

If the optimizer specializes the generic function for a concrete type, many abstractions can disappear. If specialization does not happen, the code may rely on metadata, witness tables, or indirect operations.

A generic function can still allocate if its arguments allocate. For example, a transform closure, type-erased wrapper, captured reference, or returned collection may dominate the cost even when the generic function itself specializes well.

Review questions:

* Is the generic function internal and visible to the optimizer?
* Is it public across a module boundary?
* Is it called with a small number of concrete types?
* Is it in a hot path?
* Is optimized SIL showing specialization or generic runtime operations?
* Do closure arguments allocate or block inlining?

Example:

```swift
func compactMapValid<Element, Output>(
    _ elements: [Element],
    transform: (Element) -> Output?
) -> [Output] {
    var result: [Output] = []
    result.reserveCapacity(elements.count)

    for element in elements {
        if let output = transform(element) {
            result.append(output)
        }
    }

    return result
}
```

This generic function may optimize well when the concrete types and closure body are visible. Across module boundaries or with escaping/type-erased transforms, the optimizer may have less room.

## Signs to look for in Instruments

In Allocations:

* repeated allocation of the same small class
* closure context allocation
* boxing around protocol or type-erased values
* temporary arrays or dictionaries
* unexpected `Data`, `String`, or collection growth
* allocation spikes during scrolling, parsing, or rendering

In Time Profiler:

* allocator functions near the hot path
* retain/release traffic around object-heavy code
* dynamic dispatch-heavy call stacks
* repeated small helper calls that did not inline
* collection copying or bridging overhead

In Swift Concurrency Instrument:

* tasks retaining large state
* actor contention around storage access
* MainActor work caused by allocation-heavy processing
* unbounded task creation that multiplies allocation

## Signs to look for in optimized SIL

Useful patterns:

* `alloc_ref`: class instance allocation
* `alloc_box`: boxed mutable state or captured variable
* `partial_apply`: closure creation
* `strong_retain` / `strong_release`: ARC traffic
* `copy_value` / `destroy_value`: ownership operations
* `init_existential_*`: existential initialization
* `open_existential_*`: existential opening
* `alloc_stack`: stack allocation
* `dealloc_stack`: stack cleanup
* `class_method`: class dispatch
* `witness_method`: protocol witness dispatch

Use SIL as evidence, not as a replacement for user-visible measurement. A SIL pattern is important when it explains a measured cost or a credible hot-path concern.

## Common review recommendations

Prefer:

* keeping hot temporary values local and scoped
* avoiding repeated allocation inside loops
* reserving collection capacity when growth is predictable
* narrowing closure captures
* using concrete or generic hot paths when runtime heterogeneity is unnecessary
* batching actor or storage accesses
* removing unnecessary type-erased wrappers in inner loops
* measuring optimized builds before and after changes

Avoid:

* replacing all classes with structs
* replacing all structs with classes to avoid copies
* removing existentials where heterogeneity is the actual requirement
* adding unsafe pointers before measuring
* relying on exact runtime object layout
* optimizing Debug-build behavior
* making public API less flexible for an unmeasured allocation concern

## Output guidance

When this reference is used, include:

```markdown
## Allocation/layout model

Explain where the relevant values are likely stored and why.

## Suspected allocation source

Identify class allocation, closure context, existential boxing, COW buffer, async state, or temporary collection.

## Why it matters

Tie the storage model to the measured or likely performance concern.

## Safer alternative

Suggest a minimal change that preserves semantics.

## Validation

Recommend Allocations, Time Profiler, optimized SIL, benchmark, or signposts.
```

If the issue is theoretical and not on a hot path, say so directly.
