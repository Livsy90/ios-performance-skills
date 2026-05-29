# Existentials, Generics, and Opaque Types

Use this reference when reviewing Swift code that uses protocols as types, `any Protocol`, `some Protocol`, generic constraints, protocol witness dispatch, type erasure, heterogeneous collections, or APIs whose performance depends on specialization.

The goal is not to ban existentials or force generics everywhere. The goal is to choose the abstraction that matches the design while understanding its runtime and optimization trade-offs.

## Core model

Swift has several ways to express abstraction over types:

* concrete types
* generic parameters
* opaque types with `some`
* existential types with `any`
* manual type erasure wrappers
* class inheritance or Objective-C dynamic dispatch

Each option answers a different question:

* Concrete type: "This exact type is part of the implementation or API."
* Generic parameter: "The caller chooses one concrete type that satisfies constraints."
* Opaque result type: "The implementation chooses one concrete type, but exposes only its capabilities."
* Existential type: "The concrete type is erased and can vary at runtime."
* Type erasure wrapper: "A concrete wrapper hides another value behind a stable API."
* Class hierarchy: "Behavior varies through inheritance and reference identity."

Do not treat these as interchangeable syntax choices. They express different ownership, dispatch, storage, and API-evolution models.

## Quick decision table

| Situation                                                             | Prefer                                          |
| --------------------------------------------------------------------- | ----------------------------------------------- |
| Internal hot path with one known implementation                       | Concrete type                                   |
| Homogeneous algorithm over one conforming type                        | Generic parameter                               |
| Return a hidden but stable concrete type                              | `some Protocol`                                 |
| Store different conforming types together                             | `any Protocol`                                  |
| Public API needs stable concrete wrapper while hiding implementation  | Type erasure wrapper                            |
| Reference identity and overridable behavior are central to the design | Class hierarchy                                 |
| SwiftUI-style composed return type                                    | `some View`                                     |
| Dependency boundary with runtime replacement                          | `any Protocol` or type erasure                  |
| Plugin-style architecture                                             | `any Protocol`, type erasure, or class boundary |
| Measured hot loop using an existential unnecessarily                  | Generic or concrete type                        |

## Concrete types

Concrete types give the compiler the most direct information.

Prefer concrete types when:

* the implementation is local
* no runtime substitution is required
* the code is performance-sensitive
* the exact type is not a public API burden
* the type improves readability rather than leaking unnecessary details

Example:

```swift
struct JSONMessageDecoder {
    func decode(_ data: Data) throws -> Message {
        try JSONDecoder().decode(Message.self, from: data)
    }
}

func loadMessages(
    data: [Data],
    decoder: JSONMessageDecoder
) throws -> [Message] {
    try data.map { try decoder.decode($0) }
}
```

This is clear when there is only one decoding strategy. Introducing a protocol here may add abstraction without solving a real design problem.

Review questions:

* Is a protocol used only because "it feels flexible"?
* Is the code internal enough that a concrete type is simpler?
* Is the type part of a measured hot path?
* Would a protocol boundary help testing, architecture, or replacement?
* Is the exact type too verbose or too unstable to expose?

## Generic parameters

A generic parameter preserves the concrete type selected by the caller while expressing constraints through protocols.

Example:

```swift
protocol MessageDecoder {
    func decode(_ data: Data) throws -> Message
}

func decodeBatch<D: MessageDecoder>(
    _ batch: [Data],
    using decoder: D
) throws -> [Message] {
    try batch.map { try decoder.decode($0) }
}
```

Here `D` is one concrete type for this call. The compiler may specialize the function for the concrete decoder when the implementation is visible enough.

Prefer generics when:

* the call path is homogeneous
* one concrete conforming type is used per call
* performance depends on specialization or inlining
* the algorithm should work for many concrete types
* the caller should choose the concrete type
* type relationships must be preserved

Generics are not automatically free. They help most when the optimizer can specialize or inline through the abstraction.

Review questions:

* Is the generic function in the same module as the call site?
* Is it public across a module boundary?
* Is it called with many concrete types, increasing code size?
* Is the generic abstraction preserving useful type relationships?
* Is optimized SIL showing specialization?
* Would a concrete implementation be simpler for an internal hot path?

## Opaque types with `some`

An opaque type hides the concrete type from the API surface while preserving a specific underlying concrete type for the compiler.

In return position, `some Protocol` means the implementation chooses the concrete type.

Example:

```swift
protocol TimelineRenderer {
    func render(_ item: TimelineItem) -> RenderedRow
}

struct CompactTimelineRenderer: TimelineRenderer {
    func render(_ item: TimelineItem) -> RenderedRow {
        RenderedRow(title: item.title, subtitle: item.subtitle)
    }
}

func makeTimelineRenderer() -> some TimelineRenderer {
    CompactTimelineRenderer()
}
```

The caller does not know the exact return type from the function signature, but the function still returns one concrete underlying type.

Prefer opaque result types when:

* callers only need protocol capabilities
* the implementation should hide a verbose concrete type
* the concrete type should remain changeable without exposing it directly
* the result is homogeneous for that declaration
* preserving type identity matters
* existential storage is unnecessary

Do not use `some` when:

* the function must return unrelated concrete types from ordinary runtime branches
* the value must be stored with other different conforming types
* the API needs runtime heterogeneity
* callers need to choose the concrete type

Example where `some` is not the right model:

```swift
protocol ExportFormatter {
    func format(_ report: Report) -> Data
}

struct PDFFormatter: ExportFormatter {
    func format(_ report: Report) -> Data { Data() }
}

struct CSVFormatter: ExportFormatter {
    func format(_ report: Report) -> Data { Data() }
}

func makeFormatter(kind: ExportKind) -> any ExportFormatter {
    switch kind {
    case .pdf:
        PDFFormatter()
    case .csv:
        CSVFormatter()
    }
}
```

The formatter varies by runtime input, so an existential is appropriate. A single `-> some ExportFormatter` result would not express this ordinary runtime heterogeneity.

## `some` in parameter position

In parameter position, `some Protocol` is usually a concise way to express an implicit generic parameter.

Example:

```swift
func persist(_ event: some AnalyticsEvent, into store: EventStore) throws {
    try store.write(event)
}
```

This is similar in spirit to writing a generic function where the caller supplies the concrete event type.

Use parameter `some` when:

* the concrete type should be chosen by the caller
* the function does not need to name the generic type parameter
* the generic relationship is simple
* readability improves

Prefer an explicit generic parameter when:

* the same type parameter appears multiple times
* associated types or same-type relationships need to be named
* constraints are complex
* the generic relationship is important to the API

Example:

```swift
func merge<S: Sequence>(
    primary: S,
    secondary: S
) -> [S.Element] {
    Array(primary) + Array(secondary)
}
```

An explicit `S` makes the same-type relationship between `primary` and `secondary` visible.

## Existential types with `any`

An existential value stores a value whose concrete type is erased at that point.

Example:

```swift
protocol NotificationChannel {
    func send(_ message: NotificationMessage) async throws
}

struct EmailChannel: NotificationChannel {
    func send(_ message: NotificationMessage) async throws {}
}

struct PushChannel: NotificationChannel {
    func send(_ message: NotificationMessage) async throws {}
}

let channels: [any NotificationChannel] = [
    EmailChannel(),
    PushChannel()
]
```

The array intentionally stores different concrete types. This is a good use of `any`.

Existentials may involve:

* erased concrete type identity
* witness table dispatch for protocol requirements
* existential storage
* possible boxing for values that do not fit inline or otherwise need indirect storage
* pointer indirection
* less optimizer visibility than concrete or specialized generic code

Do not say "existentials are bad." Say what runtime flexibility they provide and what cost may come with that flexibility.

Prefer `any Protocol` when:

* runtime heterogeneity is required
* values of different concrete types must share one storage location
* the value crosses a dependency boundary
* a plugin-like architecture needs late binding
* the API intentionally erases implementation details
* the code is not a measured hot path
* the existential makes the design simpler and more stable

Avoid unnecessary existentials when:

* the concrete type is homogeneous
* the value is used in a tight loop
* the abstraction is local and not needed
* type relationships are lost and later reconstructed manually
* optimized SIL or Instruments shows boxing, witness calls, or allocation as relevant cost

## Existentials and protocols with associated types

Modern Swift can use existentials for many protocols that include `Self` or associated type requirements, but the type-erasing semantics still matter.

An existential erases the concrete conforming type. If an associated type relationship is important to the algorithm, a generic parameter may be a better fit.

Example:

```swift
protocol RecordSource {
    associatedtype Record

    func next() async throws -> Record?
}

func drain<S: RecordSource>(_ source: S) async throws -> [S.Record] {
    var source = source
    var records: [S.Record] = []

    while let record = try await source.next() {
        records.append(record)
    }

    return records
}
```

The generic version preserves the relationship between the source and its `Record` type.

Use constrained existentials when you need existential storage but can still preserve important constraints.

Example:

```swift
func describeStrings(_ values: any Sequence<String>) -> [String] {
    values.map { "Value: \($0)" }
}
```

This keeps the existential model while constraining the element type. Use this when runtime erasure is needed but not all type information should be discarded.

## Type erasure wrappers

Manual type erasure wraps an underlying concrete value in a stable concrete type.

Example:

```swift
struct AnyMessageHandler<Message> {
    private let _handle: (Message) async throws -> Void

    init<H: MessageHandler>(_ handler: H) where H.Message == Message {
        self._handle = handler.handle
    }

    func handle(_ message: Message) async throws {
        try await _handle(message)
    }
}
```

Type erasure can be useful when:

* the public API should expose one concrete wrapper
* stored properties need a stable type
* associated type relationships must be preserved through generic parameters
* the implementation needs to hide many concrete handler types
* a framework wants to provide a familiar wrapper type

Type erasure is not automatically faster than `any`. It often uses closures, boxes, references, or witness-table-like forwarding. Treat it as an API design tool, not a guaranteed performance optimization.

Review questions:

* Does the wrapper allocate?
* Does it store escaping closures?
* Does it preserve important associated type relationships?
* Does it hide too much from the optimizer?
* Is it needed for public API stability?
* Would `any Protocol` now be simpler?
* Would a generic type be better in the hot path?

## `any` vs type erasure

Prefer `any Protocol` when:

* the protocol can be used directly as an existential
* the API does not need a custom wrapper
* the concrete type can vary at runtime
* the existential is simple and local
* the code benefits from language-level clarity

Prefer a type erasure wrapper when:

* the protocol has associated type relationships that must be exposed through the wrapper's generic parameters
* the API needs a nominal concrete type
* you need custom forwarding, caching, cancellation, or lifecycle behavior
* you are preserving source compatibility with an existing wrapper
* you need to add conformance or behavior to the erased container itself

Do not add an `Any...` wrapper only because existentials feel "slow." Measure first.

## Protocol witness dispatch

When calling a protocol requirement through an existential, Swift dispatches through protocol conformance information rather than a direct concrete function call.

Example:

```swift
protocol ScoreRule {
    func score(_ candidate: Candidate) -> Int
}

func rank(
    candidates: [Candidate],
    rule: any ScoreRule
) -> [Candidate] {
    candidates.sorted {
        rule.score($0) > rule.score($1)
    }
}
```

If ranking is a hot path and `rule` is usually one concrete type, a generic version may give the optimizer more room:

```swift
func rank<R: ScoreRule>(
    candidates: [Candidate],
    rule: R
) -> [Candidate] {
    candidates.sorted {
        rule.score($0) > rule.score($1)
    }
}
```

Do not rewrite every existential call. Only consider this when the call is frequent enough and the abstraction is not required.

## Existentials in collections

A collection of existentials is appropriate when elements are intentionally heterogeneous.

Example:

```swift
protocol FeedModule {
    func makeRows() -> [FeedRow]
}

let modules: [any FeedModule] = [
    NewsModule(),
    AdsModule(),
    RecommendationsModule()
]
```

This is a natural existential use case.

Be cautious when a collection is existential only because the API lost concrete type information too early.

Example:

```swift
func renderModules(_ modules: [any FeedModule]) -> [FeedRow] {
    modules.flatMap { $0.makeRows() }
}
```

This is fine for a screen-level composition boundary. It may be a problem if called deeply inside a high-frequency rendering loop where all modules are actually the same concrete type.

Review questions:

* Is the collection truly heterogeneous?
* Is this a composition boundary or an inner loop?
* Are elements long-lived or created repeatedly?
* Is boxing visible in allocation traces?
* Can the hot part be moved behind a concrete or generic implementation?

## Opaque return types and branches

A function returning `some Protocol` must have a consistent underlying concrete type for that declaration, aside from specific language-supported availability cases.

Good:

```swift
func makeDefaultStore() -> some EventStore {
    SQLiteEventStore(path: "events.sqlite")
}
```

Not appropriate for ordinary runtime branching over unrelated concrete types:

```swift
func makeStore(kind: StoreKind) -> any EventStore {
    switch kind {
    case .memory:
        InMemoryEventStore()
    case .sqlite:
        SQLiteEventStore(path: "events.sqlite")
    }
}
```

Use `any` or a type erasure wrapper when different concrete implementations must be selected at runtime and returned through one API.

## `some` is not always a performance fix

`some Protocol` avoids existential storage in many API designs, but it is not a universal optimization.

Do not recommend `some` when:

* the caller needs to store multiple concrete types in one variable or collection
* the implementation really returns different concrete types
* the code needs dynamic replacement
* the abstraction boundary is architectural rather than local
* the protocol methods are not the measured cost
* the opaque type makes the API harder to use

Review rule:

* `some` is best when the underlying type is stable but should remain hidden.
* `any` is best when the underlying type is intentionally dynamic.
* generics are best when the caller chooses and the algorithm should preserve type identity.

## `any` is not always a problem

Existentials are a good design when dynamic behavior is the point.

Good existential use cases:

* heterogeneous plugin arrays
* dependency registries
* app composition roots
* UI module lists
* logging, analytics, routing, and notification fan-out
* public API boundaries where concrete types should remain private
* long-lived dependencies that are not in inner loops

The performance question is not "does this code use `any`?" The performance question is "does this existential sit in a measured hot path, allocate repeatedly, or hide information needed by the optimizer?"

## Associated type relationships

Generics preserve relationships that existentials may erase.

Example:

```swift
protocol SnapshotProvider {
    associatedtype Snapshot

    func snapshot() -> Snapshot
    func restore(_ snapshot: Snapshot)
}

func roundTrip<P: SnapshotProvider>(_ provider: P) {
    let snapshot = provider.snapshot()
    provider.restore(snapshot)
}
```

The generic version guarantees that `restore` receives exactly the snapshot type produced by the same provider.

With an unconstrained existential, this relationship may be unavailable or harder to express.

Review questions:

* Does the algorithm rely on an associated type relationship?
* Does the existential erase information that the next call needs?
* Is the code compensating with casts?
* Would a generic parameter express the relationship directly?
* Would a constrained existential be sufficient?

## API boundary guidance

For internal implementation:

* prefer concrete types when there is no need for abstraction
* use generics for reusable homogeneous algorithms
* use `some` to hide verbose concrete return types without erasing identity
* use `any` where dynamic heterogeneity is required

For public API:

* avoid exposing overly specific concrete types when they are implementation details
* use `some` when the implementation chooses one hidden concrete type
* use `any` or type erasure when callers need runtime polymorphism
* consider source and binary compatibility before changing abstraction style
* be careful with `@inlinable` as a performance workaround across module boundaries

For hot paths:

* keep the concrete type visible when possible
* avoid unnecessary existential storage in loops
* check whether generic specialization happens
* avoid type-erased closure wrappers around tiny repeated operations
* validate with optimized SIL or Instruments

## Cross-module specialization

Generics often optimize best when the compiler can see both the generic implementation and the concrete type.

Specialization may be limited when:

* a generic function is public and called from another module
* the implementation body is not visible to the client optimizer
* resilience prevents assumptions about layout or implementation
* the abstraction is wrapped in type erasure
* the call goes through an existential or class boundary

Do not add `@inlinable` automatically. Treat it as an API and ABI commitment.

Review questions:

* Is this hot generic code internal or public?
* Is the body visible to the optimizer?
* Is specialization visible in optimized SIL?
* Would moving the hot implementation internal help?
* Is `@inlinable` justified by a small, stable, performance-critical API?
* Would code size increase from many specializations?

## SIL patterns to inspect

When source-level reasoning is not enough, inspect optimized SIL.

Look for:

* `init_existential_*`: existential container creation
* `open_existential_*`: opening an existential
* `witness_method`: protocol witness dispatch
* `apply`: function call
* `partial_apply`: closure creation or type-erased forwarding
* `alloc_box`: boxed captured state
* `alloc_ref`: class or box allocation
* `strong_retain` / `strong_release`: ARC traffic
* specialized function names or specialization attributes
* generic code that remains unspecialized in a hot path

Use SIL to explain measured behavior, not to justify speculative rewrites by itself.

## Instruments signs

In Allocations:

* repeated existential boxes
* type-erasure wrapper objects
* closure contexts from `Any...` wrappers
* temporary arrays of `any Protocol`
* allocation spikes from converting concrete collections to existential collections

In Time Profiler:

* protocol witness dispatch in hot call stacks
* forwarding through type erasure wrappers
* repeated small methods that did not inline
* retain/release traffic around erased values
* sorting, mapping, rendering, or decoding loops using existential calls

In app-level traces:

* screen composition using existentials is usually fine
* row-level or pixel-level loops using existentials deserve closer inspection
* dependency injection boundaries are usually not the bottleneck unless called repeatedly

## Common review recommendations

Prefer:

* concrete types for local implementation when abstraction is unnecessary
* generics for homogeneous hot algorithms
* `some` for hidden but stable result types
* `any` for runtime heterogeneity and storage of mixed conformers
* constrained existentials when only part of the type information should be erased
* type erasure wrappers when a nominal public wrapper is useful
* moving dynamic dispatch outside the inner loop when possible
* measuring before changing API shape

Avoid:

* replacing every `any` with generics
* replacing every generic with `some`
* using `some` for runtime-varying return types
* adding `Any...` wrappers without a reason
* exposing verbose concrete types only for hypothetical performance
* assuming generics specialize across every module boundary
* assuming type erasure is faster than language existentials
* erasing associated type relationships and then recovering them with casts
* making public APIs harder to use for an unmeasured hot-path concern

## Decision rules

### If you see `any Protocol`

Ask:

* Is runtime heterogeneity required?
* Is this value stored, passed briefly, or used inside a loop?
* Is the concrete type actually homogeneous?
* Does the code rely on associated type relationships?
* Would a generic, opaque type, or concrete type preserve the design?
* Is boxing or witness dispatch visible in Instruments or optimized SIL?

Recommendation style:

* Keep `any` when it expresses a real dynamic boundary.
* Replace with generics or concrete types only when the path is homogeneous and performance-sensitive.
* Consider constrained existentials when some type information should remain visible.

### If you see `some Protocol`

Ask:

* Is the underlying type stable for this declaration?
* Is the implementation trying to return unrelated types from runtime branches?
* Is `some` hiding useful details or making the API clearer?
* Would callers need to store heterogeneous values?
* Is this a return-position use or a parameter-position shorthand for generics?

Recommendation style:

* Keep `some` when the implementation chooses one hidden concrete type.
* Use `any` or type erasure when the concrete type must vary at runtime.
* Use explicit generics when type relationships need names.

### If you see a generic function

Ask:

* Does the generic parameter preserve an important type relationship?
* Is the function internal or public across a module boundary?
* Does specialization happen in optimized SIL?
* Is code size a concern due to many concrete instantiations?
* Is a closure argument or returned collection dominating allocation cost?

Recommendation style:

* Keep generics when they express reusable homogeneous behavior.
* Use concrete types for local hot paths where abstraction adds no value.
* Use `@inlinable` only after identifying a real cross-module optimization issue.

### If you see type erasure

Ask:

* Is the wrapper needed for API stability or storage?
* Does it preserve associated type relationships?
* Does it store escaping closures?
* Does it allocate or retain unexpectedly?
* Would `any Protocol` be simpler in modern Swift?
* Is the erased wrapper used in a hot path?

Recommendation style:

* Keep type erasure when it provides a meaningful public wrapper or behavior.
* Prefer language existentials when the wrapper only forwards protocol calls.
* Prefer generics or concrete types inside hot implementation paths.

## Common gotchas

* `any Protocol` is not a generic constraint; it is an existential type.
* `some Protocol` is not the same as `any Protocol`.
* `some` preserves a concrete underlying type; `any` erases it.
* Generics help most when specialization happens.
* Existentials may allocate, but do not always allocate.
* Small existential values may fit inline; larger values may need indirect storage.
* Type erasure wrappers can allocate or capture closures.
* `some` cannot express ordinary runtime choice between unrelated return types.
* `any` is often correct at dependency and composition boundaries.
* `some` in parameter position is usually an implicit generic, not existential storage.
* Associated type relationships are often clearer with generics.
* Constrained existentials can be better than erasing all type information.
* Cross-module boundaries can limit specialization.
* Changing abstraction style can affect API stability, not only performance.

## Output guidance

When this reference is used, include:

```markdown
## Abstraction model

State whether the code uses concrete types, generics, `some`, `any`, type erasure, or inheritance.

## Runtime cost

Explain possible costs: witness dispatch, existential storage, boxing, closure capture, ARC, missed specialization, or code size.

## Design reason

Explain what flexibility or type relationship the current abstraction provides.

## Recommendation

Suggest whether to keep the abstraction or change it to concrete, generic, opaque, existential, or type-erased form.

## Validation

Recommend optimized SIL, Time Profiler, Allocations, benchmark, or code-size inspection.
```

If the abstraction is not in a hot path and expresses the design well, say so and avoid rewriting it only for theoretical performance.
