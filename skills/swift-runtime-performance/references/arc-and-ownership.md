# ARC and Ownership

Use this reference when a review involves ARC traffic, object lifetime, reference cycles, closure captures, weak/unowned references, copy-on-write storage, Objective-C bridging, or ownership-sensitive performance issues.

The goal is not to remove ARC from Swift code. The goal is to identify when reference ownership is contributing to a measured cost or correctness issue, then choose the smallest change that preserves the intended lifetime model.

## Core model

ARC manages the lifetime of reference-counted objects.

Common ARC-managed or ARC-related cases:

* class instances
* actor instances
* closure capture contexts
* reference-backed standard library storage
* Objective-C and Core Foundation bridged objects
* type-erased wrappers that store references
* values that contain references or copy-on-write buffers

Value types are not reference-counted as values, but they can contain fields that participate in ARC.

Avoid simplistic rules such as:

* ARC is always the bottleneck.
* value types have no ARC cost.
* `weak` is faster or safer by default.
* `unowned` is a performance optimization.
* `[weak self]` should be used in every closure.
* deinitialization timing is a stable correctness mechanism.
* replacing classes with structs automatically removes ownership cost.

ARC cost matters when retain/release traffic is frequent, appears in a hot path, causes memory pressure, extends object lifetime unexpectedly, or creates retained object graphs that outlive their intended scope.

## What to inspect first

Before proposing ownership changes, identify:

1. Is there a correctness issue, a memory growth issue, or a runtime performance issue?
2. Is the object graph retained longer than intended?
3. Is there a strong reference cycle?
4. Is ARC traffic visible in Time Profiler or optimized SIL?
5. Are closures capturing more state than necessary?
6. Is a value type hiding reference-backed storage?
7. Is copy-on-write storage being copied or retained repeatedly?
8. Is Objective-C bridging or autorelease behavior involved?
9. Would changing ownership semantics alter program behavior?
10. How will the change be validated?

Do not change ownership only because source code looks reference-heavy. First determine whether lifetime or ARC traffic is relevant to the problem.

## Strong ownership

A strong reference keeps an object alive.

Use strong references when:

* the owner is responsible for keeping the object alive
* the lifetime dependency is direct and intentional
* the reference is part of the object's required state
* the object should not disappear while it is being used

Example:

```swift
final class PlaybackController {
    private let engine: AudioEngine
    private let library: MediaLibrary

    init(engine: AudioEngine, library: MediaLibrary) {
        self.engine = engine
        self.library = library
    }

    func start(track: TrackID) throws {
        let file = try library.file(for: track)
        try engine.play(file)
    }
}
```

`PlaybackController` strongly owns the dependencies it needs to function. Replacing these references with `weak` would make the object graph less reliable and could introduce unexpected nil states.

Review rule:

* Use strong ownership when the current object depends on another object for correct operation.
* Do not weaken ownership just to reduce retain counts.

## Weak references

A weak reference does not keep an object alive and becomes `nil` when the referenced object is deallocated.

Use `weak` when:

* the referenced object may disappear independently
* `nil` is a valid state
* the relationship would otherwise create a cycle
* the reference is naturally back-pointing or observational
* the owner should not extend the lifetime of the referenced object

Common examples:

* delegates
* parent pointers
* view-to-coordinator back references
* observer relationships
* callbacks where the owner may disappear before the callback fires

Example:

```swift
protocol DownloadCellDelegate: AnyObject {
    func downloadCellDidTapRetry(_ cell: DownloadCell)
}

final class DownloadCell {
    weak var delegate: DownloadCellDelegate?

    func retryButtonTapped() {
        delegate?.downloadCellDidTapRetry(self)
    }
}
```

The cell should not own its delegate. The delegate may disappear, so `weak` and optional access model the relationship correctly.

Review rule:

* Recommend `weak` for lifetime semantics, not for speed.
* Do not use `weak` when the referenced object must remain alive for the current object to work.

## Unowned references

An unowned reference does not keep an object alive and is expected to always refer to a valid object while it is accessed.

Use `unowned` only when:

* the referenced object is guaranteed to outlive the reference
* `nil` is not a meaningful state
* a weak optional would misrepresent the invariant
* the ownership relationship is simple and easy to prove

Example:

```swift
final class Document {
    private(set) var pages: [Page] = []

    func addPage(number: Int) {
        pages.append(Page(number: number, document: self))
    }
}

final class Page {
    let number: Int
    unowned let document: Document

    init(number: Int, document: Document) {
        self.number = number
        self.document = document
    }
}
```

This is valid only if a `Page` can never outlive its `Document`. If pages can be detached, cached, moved elsewhere, or used after the document is gone, `unowned` is unsafe.

Review rule:

* Use `unowned` for proven lifetime invariants.
* Do not use `unowned` as a performance substitute for `weak`.
* If the lifetime relationship is not obvious, prefer `weak` or redesign ownership.

Avoid `unowned(unsafe)` unless the code is a carefully isolated low-level boundary with documented lifetime guarantees.

## Reference cycles

A strong reference cycle occurs when objects keep each other alive indefinitely.

Common patterns:

* parent strongly owns child, child strongly owns parent
* object stores a closure that captures the object strongly
* timer/display link/notification/token retains a callback that captures owner
* coordinator retains screen, screen retains coordinator through closure
* task handle retained by owner, task closure captures owner

Example:

```swift
final class GalleryPresenter {
    private var onSelection: ((ImageID) -> Void)?

    func configure() {
        onSelection = { id in
            self.openImage(id)
        }
    }

    private func openImage(_ id: ImageID) {
        // ...
    }
}
```

`GalleryPresenter` stores the closure and the closure captures `self`. This can create a cycle.

A safer version:

```swift
final class GalleryPresenter {
    private var onSelection: ((ImageID) -> Void)?

    func configure() {
        onSelection = { [weak self] id in
            self?.openImage(id)
        }
    }

    private func openImage(_ id: ImageID) {
        // ...
    }
}
```

Use `[weak self]` when the closure may outlive `self` and `self` should not be kept alive by the closure.

Do not use `[weak self]` reflexively when:

* the closure is non-escaping
* the closure is executed immediately
* the closure should keep the object alive for the operation
* losing `self` would silently drop required work
* there is no ownership cycle or lifetime extension problem

## Narrowing closure captures

Closure capture lists can be used to capture specific dependencies instead of the whole owner.

Example:

```swift
final class ReceiptExporter {
    private let encoder: ReceiptEncoder
    private let store: ReceiptStore
    private let logger: Logger

    init(encoder: ReceiptEncoder, store: ReceiptStore, logger: Logger) {
        self.encoder = encoder
        self.store = store
        self.logger = logger
    }

    func makeExportAction() -> (Receipt) throws -> URL {
        let encoder = encoder
        let store = store

        return { receipt in
            let data = try encoder.encode(receipt)
            return try store.save(data)
        }
    }
}
```

The closure does not retain `ReceiptExporter` or unrelated dependencies such as `logger`. It retains only the objects required by the action.

Review questions:

* Does the closure need `self`?
* Would capturing a specific dependency better express the lifetime?
* Would `[weak self]` silently skip required work?
* Is the closure stored by `self`, creating a cycle?
* Is the closure created repeatedly in a hot path?
* Is the closure crossing an async or actor boundary?

## Escaping vs non-escaping closures

Non-escaping closures usually do not create long-lived ownership problems because they cannot outlive the call.

Escaping closures can extend the lifetime of captured objects.

Look for escaping closures in:

* stored callbacks
* async completion handlers
* task closures
* notification handlers
* timers and display links
* Combine/Rx subscriptions
* UI event handlers stored by controls
* animation completion handlers
* operation queues and dispatch queues

Example:

```swift
final class SearchController {
    private let service: SearchService
    private var latestRequest: Task<Void, Never>?

    init(service: SearchService) {
        self.service = service
    }

    func search(query: String) {
        latestRequest?.cancel()

        latestRequest = Task { [service] in
            let results = try? await service.results(for: query)
            await MainActor.run {
                // update UI-facing state elsewhere
                _ = results
            }
        }
    }
}
```

Capturing `service` instead of `self` prevents the task from retaining the whole controller. This is useful when the task does not need the controller's identity.

Review rule:

* A task retains the closure and its captures while it is running.
* This is not automatically a cycle.
* It becomes a cycle when the owner stores the task and the task strongly captures the owner, or when the task is long-lived and unintentionally extends the owner lifetime.

## ARC traffic in hot paths

ARC traffic becomes important when retain/release operations are frequent enough to show up in a hot path.

Common sources:

* arrays of class instances processed in tight loops
* repeated creation of short-lived reference wrappers
* closure creation inside loops
* type-erased wrappers around reference-heavy objects
* storing and removing callbacks frequently
* bridging Swift values to Objective-C repeatedly
* reference-backed value types copied across many layers
* passing class references through generic or existential abstractions that the optimizer cannot simplify

Example:

```swift
final class ScoreBox {
    let value: Int

    init(value: Int) {
        self.value = value
    }
}

func totalScore(_ scores: [ScoreBox]) -> Int {
    var total = 0

    for score in scores {
        total += score.value
    }

    return total
}
```

This may be perfectly fine. If this loop is extremely hot, an array of reference objects may show ARC and pointer-chasing cost. A compact value representation could improve locality and reduce ownership traffic.

A value-oriented representation:

```swift
struct Score {
    var value: Int
}

func totalScore(_ scores: [Score]) -> Int {
    scores.reduce(0) { $0 + $1.value }
}
```

Do not make this rewrite unless the representation still matches the domain and the cost matters.

## Value types with reference-backed storage

A value type can carry ARC cost through its fields.

Example:

```swift
struct RenderCommand {
    var title: String
    var payload: Data
    var metadata: [String: String]
}
```

Copying `RenderCommand` copies value containers, but `String`, `Data`, and `Dictionary` may share reference-backed storage. Passing many such values across layers may involve retain/release traffic even without explicit classes.

Review questions:

* Does the struct contain COW storage?
* Is it copied frequently?
* Is mutation performed after copying?
* Are large buffers retained longer than needed?
* Would borrowing, scoping, or streaming reduce ownership pressure?

## Copy-on-write and ARC

Copy-on-write storage uses reference-backed buffers. Copies are often cheap while storage is shared, but mutation may require uniqueness checks and buffer copying.

Example:

```swift
func normalizedTags(_ tags: [String]) -> [String] {
    var result = tags
    for index in result.indices {
        result[index] = result[index].lowercased()
    }
    return result
}
```

This may allocate new strings and may trigger array buffer behavior depending on uniqueness and capacity.

Review rule:

* Do not assume COW means free copying.
* Do not assume COW means expensive copying.
* Inspect whether copies are followed by mutation and whether the path is hot.

## Objective-C bridging and autorelease behavior

Swift code that crosses Objective-C or Foundation boundaries may introduce ownership behavior that is not obvious from pure Swift source.

Watch for:

* repeated bridging between `String` and `NSString`
* repeated bridging between `Array` and `NSArray`
* repeated bridging between `Dictionary` and `NSDictionary`
* Foundation APIs returning autoreleased objects
* Objective-C callbacks that retain blocks
* Core Foundation APIs with create/copy/get ownership conventions
* `Unmanaged` usage

Example:

```swift
func containsKeyword(_ keyword: String, in text: NSString) -> Bool {
    text.range(of: keyword).location != NSNotFound
}
```

This may be fine at a boundary. If called repeatedly in a Swift hot path, check whether bridging or Foundation dispatch is contributing to the cost.

Review rule:

* Keep bridging at boundaries when possible.
* Avoid repeated back-and-forth conversion in inner loops.
* Use `autoreleasepool {}` around large Objective-C-heavy loops when memory spikes are caused by autoreleased temporary objects.
* Be careful with Core Foundation ownership rules.

## Observed object lifetime

Do not rely on the exact moment when an object is destroyed unless the language or API explicitly guarantees the lifetime.

The compiler may shorten or extend object lifetimes as long as program semantics are preserved. Optimization level and code shape can change when `deinit` runs.

Avoid patterns where correctness depends on incidental deinitialization timing.

Risky pattern:

```swift
final class TemporaryFile {
    let url: URL

    init(url: URL) {
        self.url = url
    }

    deinit {
        try? FileManager.default.removeItem(at: url)
    }
}

func uploadTemporaryFile(_ data: Data, uploader: Uploader) async throws {
    let file = TemporaryFile(url: FileManager.default.temporaryDirectory.appendingPathComponent(UUID().uuidString))
    try data.write(to: file.url)

    try await uploader.upload(file.url)
}
```

If resource cleanup timing is important, make it explicit.

Better:

```swift
func withTemporaryFile<Result>(
    data: Data,
    operation: (URL) async throws -> Result
) async throws -> Result {
    let url = FileManager.default.temporaryDirectory.appendingPathComponent(UUID().uuidString)
    try data.write(to: url)

    do {
        let result = try await operation(url)
        try? FileManager.default.removeItem(at: url)
        return result
    } catch {
        try? FileManager.default.removeItem(at: url)
        throw error
    }
}
```

Use explicit lifetime scopes for resources such as files, locks, observations, subscriptions, and unsafe buffers.

## `withExtendedLifetime`

Use `withExtendedLifetime` when code must guarantee that a value remains alive through a specific operation and the compiler may otherwise be allowed to end its lifetime earlier.

Typical cases:

* unsafe pointer interop
* C/Objective-C APIs with lifetime assumptions not visible to Swift
* side effects that depend on an object staying alive
* low-level code where lifetime is part of the API contract

Example:

```swift
final class BufferOwner {
    let bytes: UnsafeMutableRawPointer
    let count: Int

    init(count: Int) {
        self.count = count
        self.bytes = UnsafeMutableRawPointer.allocate(byteCount: count, alignment: 1)
    }

    deinit {
        bytes.deallocate()
    }
}

func callLegacyAPI(owner: BufferOwner) {
    legacy_consume(owner.bytes, owner.count)

    withExtendedLifetime(owner) {
        // Ensures owner is kept alive through the legacy call boundary.
    }
}
```

Prefer structuring code so lifetime is clear without this function. Use `withExtendedLifetime` as an explicit bridge when lifetime is otherwise invisible to Swift.

## SIL signs of ARC traffic

In optimized SIL, look for:

* `strong_retain`
* `strong_release`
* `retain_value`
* `release_value`
* `copy_value`
* `destroy_value`
* `partial_apply`
* `alloc_ref`
* `alloc_box`
* `load_weak`
* `store_weak`
* `strong_copy_unowned_value`
* `ref_to_unowned`
* `unowned_to_ref`

Use SIL as supporting evidence. Do not assume every visible retain/release is a problem. The important question is whether ARC traffic remains in a measured hot path after optimization.

## Instruments signs

In Time Profiler, look for:

* retain/release functions near the hot path
* closure allocation or destruction around repeated operations
* Objective-C retain/release or autorelease pool activity
* type-erased wrapper churn
* object-heavy loops with poor locality

In Allocations, look for:

* repeated short-lived objects
* closure context allocation
* retained object graphs growing over time
* unexpected Foundation objects
* temporary arrays, dictionaries, strings, or data buffers
* spikes during scrolling, parsing, rendering, or search

In Leaks / Memory Graph, look for:

* object cycles through closures
* delegate cycles
* long-lived tasks retaining owners
* subscription/token cycles
* caches without eviction
* observers not removed or invalidated

## Common review recommendations

Prefer:

* explicit ownership relationships
* strong references for required dependencies
* weak references for optional back references
* unowned references only for proven lifetime invariants
* narrow closure captures
* explicit resource cleanup
* scoped temporary values
* value representations in hot data paths when they match the domain
* avoiding repeated bridging in inner loops
* measuring optimized builds before changing ownership design

Avoid:

* using `weak` everywhere
* using `unowned` to avoid optionals
* using `[weak self]` reflexively
* relying on `deinit` timing for important side effects
* replacing all classes with structs
* rewriting clear ownership for theoretical ARC savings
* ignoring COW storage inside value types
* ignoring Objective-C bridging at Foundation-heavy boundaries
* treating every retain/release in SIL as a bug

## Decision rules

### If you see `[weak self]`

Ask:

* Is the closure escaping?
* Is the closure stored by `self` or by something retained by `self`?
* Can the closure outlive `self`?
* Is silently skipping the work correct if `self` is gone?
* Would capturing a specific dependency be better?

Use `[weak self]` when it prevents an unwanted lifetime extension or reference cycle. Do not use it as a universal closure style.

### If you see `[unowned self]`

Ask:

* Can the closure ever outlive `self`?
* Is the lifetime invariant obvious and documented?
* Would a crash be acceptable if the invariant is violated?
* Is `weak` more honest about the relationship?

Use `[unowned self]` only when the lifetime guarantee is strong.

### If you see a stored closure

Ask:

* Who owns the closure?
* What does the closure capture?
* Can the closure capture its owner?
* Is there an explicit invalidation path?
* Does the closure need the whole owner or only a dependency?

### If you see a reference-heavy value type

Ask:

* Which fields are reference-backed?
* Is the value copied often?
* Is it mutated after copying?
* Is the value crossing async or actor boundaries?
* Is it retained by closures or tasks?

### If you see a memory leak

Ask:

* Is there a strong cycle?
* Is a task/subscription/observer keeping the object alive?
* Is a cache retaining objects without eviction?
* Is a closure capturing `self` strongly?
* Is cleanup tied to unreliable `deinit` timing?

## Output guidance

When this reference is used, include:

```markdown
## Ownership model

Describe who owns what and which lifetimes are intentional.

## Suspected ARC/lifetime issue

Identify retain cycle, lifetime extension, ARC traffic, weak/unowned misuse, COW storage, bridging, or closure capture.

## Why it matters

Tie the issue to correctness, memory growth, CPU overhead, or user-visible performance.

## Recommended change

Suggest the smallest ownership change that preserves semantics.

## Validation

Recommend Memory Graph, Leaks, Allocations, Time Profiler, optimized SIL, or a targeted lifetime test.
```

If ARC traffic is only theoretical and there is no hot path or memory issue, say so directly.
