# Copy-on-Write and Large Values

Use this reference when reviewing Swift code that uses large value types, standard library collections, custom copy-on-write storage, value-semantic wrappers around references, or mutation-heavy data pipelines.

The goal is not to avoid value types. The goal is to understand when value semantics are cheap, when they hide reference-backed storage, when mutation triggers copying, and when a different ownership boundary would make performance and correctness clearer.

## Core model

Copy-on-write, or COW, is a storage strategy that lets a value type share backing storage until mutation.

The value behaves as if each copy were independent. Internally, the storage may be shared while values are only read. Before mutation, the implementation checks whether the storage is uniquely referenced. If it is not unique, the storage is copied first.

This gives Swift value types a useful combination:

* value semantics at the API level
* cheap copies for read-only paths
* deferred copying until mutation
* predictable mutation semantics when implemented correctly

Do not confuse the semantic model with the storage model:

* Value semantics means independent observable values.
* COW means storage may be shared as an optimization.
* Shared storage must not leak as shared mutable state.
* Copying a value does not always copy its entire storage immediately.
* Mutating a copied value may allocate and copy storage.

## Common COW-backed types

Common Swift and Apple-platform value types often use reference-backed or COW-like storage internally.

Examples include:

* `Array`
* `Dictionary`
* `Set`
* `String`
* `Data`
* custom structs with private reference storage

Do not assume every struct is COW-backed. Most ordinary structs store their fields directly. COW appears when the type is intentionally implemented with shared reference storage.

## First questions for review

Before recommending changes, ask:

1. Is the value actually large?
2. Is the value copied often?
3. Is mutation performed after copying?
4. Is the code in a measured hot path?
5. Is allocation, buffer growth, or ARC traffic visible?
6. Is storage shared intentionally or accidentally?
7. Is the value passed through many architectural layers?
8. Is the value stored in closures, tasks, actors, or type-erased wrappers?
9. Is the proposed optimization preserving value semantics?

Do not replace value semantics with shared mutable reference semantics unless the model truly requires identity.

## Copying is not always deep copying

A copy of a COW value often copies a small header and shares backing storage.

Example:

```swift
func keepOriginal(_ source: [ArticleID]) -> ([ArticleID], [ArticleID]) {
    var edited = source
    edited.append(.draft)

    return (source, edited)
}
```

The assignment to `edited` may be cheap. The `append` may still allocate or copy if the buffer is shared or if capacity must grow.

Review questions:

* Is the copy followed by mutation?
* Is mutation repeated many times?
* Is capacity sufficient?
* Is the original value still needed?
* Can mutation happen before sharing?
* Can the algorithm avoid creating intermediate collections?

## Mutation is the important boundary

COW cost often appears at mutation points, not at assignment points.

Look for patterns like:

```swift
func renameAll(_ folders: [Folder]) -> [Folder] {
    var result = folders

    for index in result.indices {
        result[index].title = result[index].title.uppercased()
    }

    return result
}
```

This may be fine. It becomes suspicious when:

* `folders` is large
* the function runs frequently
* each element is itself large or reference-backed
* mutation triggers repeated storage checks
* intermediate arrays are created before or after this function
* the same transformation could happen at the owning boundary

Prefer reviewing the whole pipeline rather than this function in isolation.

## Copy-then-mutate loops

A common performance issue is repeated copy-then-mutate inside a loop.

Risky shape:

```swift
func buildSections(from groups: [MessageGroup]) -> [Section] {
    var sections: [Section] = []

    for group in groups {
        var rows = sections.last?.rows ?? []
        rows.append(contentsOf: group.messages.map(Row.init))
        sections.append(Section(title: group.title, rows: rows))
    }

    return sections
}
```

The problem is not that arrays are bad. The problem is that the algorithm repeatedly copies and extends values in a way that may create many intermediate buffers.

Prefer building each owned value directly:

```swift
func buildSections(from groups: [MessageGroup]) -> [Section] {
    var sections: [Section] = []
    sections.reserveCapacity(groups.count)

    for group in groups {
        var rows: [Row] = []
        rows.reserveCapacity(group.messages.count)

        for message in group.messages {
            rows.append(Row(message))
        }

        sections.append(Section(title: group.title, rows: rows))
    }

    return sections
}
```

Review rule:

* Avoid repeatedly deriving a mutable copy from an existing COW value.
* Prefer constructing the final value at the ownership boundary.
* Reserve capacity when growth is predictable.

## Large structs

Large structs are not automatically wrong. They can be excellent when they model immutable snapshots or tightly grouped data.

Be cautious when large structs are:

* copied frequently
* mutated after copying
* captured by escaping closures
* stored in arrays and repeatedly transformed
* passed through existentials
* sent across actor or task boundaries
* composed of many COW-backed fields
* used as UI state that changes in small pieces

Example:

```swift
struct PortfolioSnapshot {
    var accounts: [AccountSummary]
    var positions: [PositionSummary]
    var alerts: [RiskAlert]
    var generatedAt: Date
}
```

This is a reasonable value model for an immutable snapshot. It may become expensive if every small alert change rebuilds and republishes the entire snapshot many times per second.

Review questions:

* Is the value intended to be an immutable snapshot?
* Is only a small part changing?
* Could the update be represented as a delta?
* Can ownership be moved closer to the mutation site?
* Is the whole value being captured or retained longer than needed?

## Large values in UI state

Large value types are common in UI state. They are useful because snapshots are easy to reason about, but broad invalidation and repeated copying can become expensive.

Risky shape:

```swift
struct DashboardState {
    var header: HeaderModel
    var cards: [CardModel]
    var filters: [FilterModel]
    var timeline: [TimelineEvent]
}

func applyingFilter(_ state: DashboardState, filter: FilterModel) -> DashboardState {
    var next = state
    next.filters.append(filter)
    next.timeline = recomputeTimeline(cards: next.cards, filters: next.filters)
    return next
}
```

This may be clear and correct. It becomes suspicious when used for high-frequency updates, scrolling, search-as-you-type, or animation-driven state.

Possible improvements:

* split high-frequency and low-frequency state
* store derived data separately when recomputation dominates
* apply diffs instead of replacing whole snapshots
* keep mutation inside a single owner
* avoid publishing a large state object for tiny changes

Do not split state blindly. Split state when it reduces real copying, recomputation, or invalidation.

## COW and ARC traffic

COW storage is usually reference-backed. That means copies may still involve ARC traffic, even when the underlying buffer is not deep-copied.

Look for:

* repeated assignment of COW values in hot paths
* COW values captured by escaping closures
* COW values passed through type-erased wrappers
* arrays of class instances
* structs containing many reference-backed fields

Example:

```swift
struct SearchPage {
    var query: String
    var results: [SearchResult]
    var diagnostics: [DiagnosticEvent]
}
```

Copying `SearchPage` may retain or release several backing storages. This is usually fine, but in a hot path it may show up as ARC traffic.

Review rule:

* COW reduces deep copy cost.
* COW does not mean zero cost.
* Check ARC and allocation evidence before changing the design.

## Arrays of value types vs arrays of reference types

An array's storage model is separate from the element model.

```swift
struct RenderCommand {
    var rect: Rect
    var opacity: Double
}

final class RenderNode {
    var command: RenderCommand

    init(command: RenderCommand) {
        self.command = command
    }
}
```

An `[RenderCommand]` stores values in the array buffer. An `[RenderNode]` stores references in the array buffer, while nodes live as separate heap objects.

Prefer value elements when:

* identity is not required
* mutation is controlled by the owning value
* cache locality matters
* snapshots are easier to reason about

Prefer reference elements when:

* identity is part of the model
* shared mutable state is intentional
* objects are large and stable
* independent mutation is required
* reference lifetime is already part of the design

Do not choose based on "struct good, class bad." Choose based on semantics, mutation pattern, and evidence.

## Avoid unnecessary intermediate collections

Chained transformations can create intermediate arrays unless optimized away or expressed lazily.

Potentially allocation-heavy shape:

```swift
func visibleTitles(from items: [FeedItem]) -> [String] {
    items
        .filter { $0.isVisible }
        .map { $0.title }
        .sorted()
}
```

This may be perfectly acceptable for small data. For large or frequent pipelines, consider whether intermediate storage matters.

Possible alternatives:

```swift
func visibleTitles(from items: [FeedItem]) -> [String] {
    var titles: [String] = []
    titles.reserveCapacity(items.count)

    for item in items where item.isVisible {
        titles.append(item.title)
    }

    titles.sort()
    return titles
}
```

Review rule:

* Do not rewrite every `map` / `filter` chain.
* Rewrite only when allocation or CPU cost matters.
* Prefer clarity unless the pipeline is large, frequent, or measured as expensive.

## Lazy sequences

Lazy sequences can avoid intermediate collections, but they are not always faster.

They can help when:

* the pipeline may stop early
* intermediate arrays are large
* only part of the result is consumed
* the lazy abstraction remains visible to the optimizer

They can hurt when:

* the result is consumed multiple times
* abstraction prevents optimization
* the pipeline becomes harder to understand
* repeated traversal performs work again

Example:

```swift
func firstMatchingTitle(in items: [FeedItem]) -> String? {
    items.lazy
        .filter { $0.isVisible }
        .map { $0.title }
        .first { !$0.isEmpty }
}
```

This can avoid building intermediate arrays because only the first matching title is needed.

Review rule:

* Use lazy pipelines when they match consumption.
* Do not use lazy as a default performance decoration.

## Reserving capacity

When building arrays or dictionaries incrementally, reserve capacity when the final size is predictable.

Example:

```swift
func makeRows(from models: [RowModel]) -> [RowViewModel] {
    var rows: [RowViewModel] = []
    rows.reserveCapacity(models.count)

    for model in models {
        rows.append(RowViewModel(model))
    }

    return rows
}
```

This can reduce buffer reallocations during growth.

Guardrails:

* Do not reserve wildly inflated capacity.
* Do not add capacity calculations that are more expensive than the savings.
* Do not use `reserveCapacity` to hide a poor algorithm.
* Measure when memory usage matters.

## `inout` and mutation at the ownership boundary

`inout` can express exclusive mutation of an existing value. It can help keep mutation at the owner boundary and avoid return-copy style APIs in some cases.

Example:

```swift
func appendVisibleRows(
    from items: [FeedItem],
    into rows: inout [FeedRow]
) {
    for item in items where item.isVisible {
        rows.append(FeedRow(item))
    }
}
```

This is useful when the caller owns the buffer and wants to build into it.

Guardrails:

* `inout` is not automatically faster.
* The optimizer may already remove many temporary copies.
* `inout` changes API shape and exclusivity constraints.
* Use it when it clarifies ownership and mutation, not as a blanket optimization.

## Custom COW types

Custom COW is useful when a value type needs to manage large mutable storage while preserving value semantics.

Typical shape:

```swift
struct DocumentBuffer {
    private var storage: Storage

    init(lines: [Line]) {
        self.storage = Storage(lines: lines)
    }

    var lines: [Line] {
        storage.lines
    }

    mutating func replaceLine(at index: Int, with line: Line) {
        makeUniqueStorage()
        storage.lines[index] = line
    }

    private mutating func makeUniqueStorage() {
        if !isKnownUniquelyReferenced(&storage) {
            storage = storage.copy()
        }
    }
}

private final class Storage {
    var lines: [Line]

    init(lines: [Line]) {
        self.lines = lines
    }

    func copy() -> Storage {
        Storage(lines: lines)
    }
}
```

Important properties:

* storage is private
* mutation goes through mutating methods
* uniqueness is checked before mutation
* copying storage preserves value semantics
* reference storage does not leak through the public API

## `isKnownUniquelyReferenced`

Use `isKnownUniquelyReferenced` to check whether reference storage is uniquely owned before mutating it.

Correct mental model:

* It works with class instances.
* It is used through `inout` storage.
* It answers whether the object is known to have a single strong reference.
* A false result means the implementation should copy before mutation.
* It does not make the storage thread-safe.
* It must be used under normal Swift exclusivity and synchronization rules.

Guardrails:

* Do not use it on public shared mutable storage as a synchronization mechanism.
* Do not expose the storage object and still expect value semantics.
* Do not skip deep copy of nested mutable references.
* Do not assume weak or unowned references provide the ownership model you want.
* Do not mutate storage before uniqueness is ensured.

## Deep copy requirements

A custom COW type must copy all mutable storage that participates in the value.

This copy is not always a trivial initializer call.

Risky shape:

```swift
private final class GraphStorage {
    var nodes: [Node]
    var index: NodeIndex

    func copy() -> GraphStorage {
        GraphStorage(nodes: nodes, index: index)
    }
}
```

This is safe only if `Node` and `NodeIndex` preserve the intended value semantics. If they hide mutable references, this copy may still share nested mutable state.

Review questions:

* Does storage contain reference types?
* Are nested references immutable?
* If mutable, are they copied too?
* Can callers observe shared mutation through nested objects?
* Are cached indexes or derived data copied consistently?

## Do not leak storage

Custom COW breaks if callers can directly mutate shared reference storage.

Risky shape:

```swift
struct ShapeList {
    private var storage: Storage

    var rawStorage: Storage {
        storage
    }
}
```

This exposes the reference object and lets external code bypass uniqueness checks.

Prefer safe accessors:

```swift
struct ShapeList {
    private var storage: Storage

    var shapes: [Shape] {
        storage.shapes
    }

    mutating func append(_ shape: Shape) {
        makeUniqueStorage()
        storage.shapes.append(shape)
    }
}
```

Review rule:

* Keep storage private.
* Expose values, not mutable storage identity.
* Route all mutations through uniqueness checks.

## Thread safety and COW

COW is not a synchronization mechanism.

It protects value semantics under normal exclusive access rules. It does not make concurrent mutation of the same variable safe.

Risky shape:

```swift
final class SharedStore {
    var snapshot: [Record] = []
}
```

If multiple threads mutate `snapshot` through the same `SharedStore` instance without synchronization, COW does not make it safe.

Review rule:

* Protect shared mutable owners with actor, lock, queue, or another synchronization strategy.
* Treat COW as a storage optimization, not a data-race solution.
* Ensure custom COW storage is mutated only through exclusive access.

## COW across actor boundaries

Returning a COW value from an actor gives the caller a value snapshot, but the storage may initially be shared internally until mutation.

Example:

```swift
actor MessageArchive {
    private var messages: [Message] = []

    func snapshot() -> [Message] {
        messages
    }
}
```

This is usually fine. The caller cannot mutate the actor's `messages` property directly. If the caller mutates the returned array and storage is shared, COW should create separate storage.

Potential costs:

* large snapshots returned frequently
* ARC traffic from repeated snapshots
* buffer copy on caller mutation
* broad UI invalidation from replacing whole snapshots

Review questions:

* Is a full snapshot necessary?
* Would a page, diff, or query API reduce copying?
* Is snapshot frequency causing allocation or ARC traffic?
* Is caller mutation expected?

## Large values and Sendable

Large value types are often convenient to send across concurrency boundaries, but the cost model still matters.

Check:

* Is the value actually `Sendable`?
* Does it contain reference-backed storage?
* Does it contain non-Sendable reference state?
* Is a large snapshot being copied or retained across tasks?
* Does the receiving task mutate the value and trigger a deep copy?

Do not mark reference-backed types as `Sendable` only to silence diagnostics. Preserve the actual ownership and thread-safety model.

## Modern alternatives for specialized cases

For specialized performance-sensitive code, consider newer ownership and storage tools only after measuring.

Possible tools:

* borrowing APIs to avoid unnecessary ownership traffic
* consuming APIs when transfer of ownership is the model
* noncopyable types when copying must be prevented
* `Span` for temporary non-owning access to contiguous memory
* `InlineArray` for fixed-size inline storage
* `ManagedBuffer` for custom storage layouts

Guardrails:

* These tools can make APIs harder to use.
* They are not replacements for ordinary value types.
* They should be isolated behind clear abstractions when possible.
* Prefer standard library containers unless measurement shows they are the bottleneck.

## Signs to look for in Instruments

In Allocations:

* repeated array or dictionary growth
* temporary collection creation
* large buffer copies
* custom storage copies
* type-erased wrappers allocating around value storage
* snapshot creation during scrolling or high-frequency updates

In Time Profiler:

* retain/release traffic around collection-heavy code
* sorting or transformation dominating the pipeline
* repeated uniqueness checks near mutation-heavy code
* bridging or conversion between collection representations
* expensive element copying or destruction

In Swift Concurrency Instrument:

* tasks retaining large snapshots
* actor methods returning large values frequently
* unbounded task groups multiplying value copies
* MainActor work caused by copying or transforming large state

## Signs to look for in optimized SIL

Useful patterns:

* `copy_value`: value copy
* `destroy_value`: value destruction
* `strong_retain` / `strong_release`: ARC traffic around storage
* `alloc_ref`: custom COW storage allocation
* `partial_apply`: closure capture that may retain large values
* `apply`: calls that remain abstract or unspecialized
* calls related to array/dictionary operations near the hot path
* missed specialization around generic collection pipelines

Use SIL to explain likely cost, not as the only proof. Prefer user-visible measurements when possible.

## Common recommendations

Prefer:

* preserving value semantics when they model the domain well
* building large values once at the ownership boundary
* reserving capacity for predictable growth
* mutating in place when the caller clearly owns the value
* avoiding repeated copy-then-mutate loops
* using generics or concrete types in homogeneous hot paths
* keeping custom COW storage private
* checking uniqueness before mutation
* copying nested mutable reference state when needed
* measuring before replacing COW with reference identity

Avoid:

* assuming every value copy is expensive
* assuming COW is free
* replacing value types with classes only to avoid hypothetical copies
* exposing custom COW storage
* using COW as thread synchronization
* adding custom COW for small simple structs
* adding unsafe buffers before measuring
* rewriting clear collection pipelines without evidence

## Output guidance

When this reference is used, include:

```markdown
## COW / large-value model

Explain whether the relevant value is plain inline storage, COW-backed storage, custom reference-backed storage, or a large snapshot.

## Suspected cost

Identify copy-after-mutation, buffer growth, ARC traffic, custom storage copy, intermediate collections, actor snapshotting, or broad state replacement.

## Why it matters

Tie the value behavior to the measured or likely performance problem.

## Safer alternative

Suggest the smallest change that preserves value semantics and ownership clarity.

## Validation

Recommend Allocations, Time Profiler, optimized SIL, signposts, XCTest performance tests, or a project benchmark.
```

If the concern is theoretical and the value is not in a hot path, say so directly.
