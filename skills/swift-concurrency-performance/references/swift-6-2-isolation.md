# Swift 6.2 Isolation and `@concurrent`

Use this reference when reviewing Swift 6.2+ code that involves `@MainActor`, default actor isolation, `nonisolated`, `@concurrent`, CPU-heavy async functions, Sendable diagnostics, or code that unexpectedly remains on the caller’s actor.

## Core Rule

Do not assume that `async` means “background”.

In modern Swift, the execution context of an async function depends on actor isolation, compiler settings, and annotations. A function can be async and still run on the caller’s actor unless it explicitly switches away or calls something that does.

For performance review, always ask:

* What actor is the caller isolated to?
* Is the enclosing type `@MainActor`?
* Is default actor isolation enabled?
* Is this function `nonisolated`?
* Is this function `@concurrent`?
* Is the heavy work accidentally staying on `MainActor`?
* Are values crossing an isolation boundary safe to send?

## Important Terms

### Actor-isolated

A declaration isolated to an actor or global actor.

Example:

```swift
@MainActor
final class DashboardModel {
    var sections: [DashboardSection] = []
}
```

Methods and mutable state on this type are main-actor isolated unless marked otherwise.

### `nonisolated`

Use `nonisolated` when a member does not access actor-isolated state.

```swift
actor ExchangeRateStore {
    private var latestRates: [CurrencyPair: Rate] = [:]

    nonisolated func normalize(_ pair: CurrencyPair) -> CurrencyPair {
        CurrencyPair(
            base: pair.base.uppercased(),
            quote: pair.quote.uppercased()
        )
    }
}
```

A `nonisolated` member cannot read or mutate `latestRates`.

### `@concurrent`

Use `@concurrent` when an async function should explicitly switch off the caller’s actor so that actor can continue making progress.

This is useful when a caller is isolated to `MainActor` and the function performs meaningful CPU work that should not run on the main actor.

## Risk: Heavy Work Staying on MainActor

Risky:

```swift
@MainActor
final class SearchScreenModel {
    private(set) var results: [SearchResultRow] = []

    func applySnapshot(_ snapshot: SearchSnapshot) async {
        results = SearchResultBuilder.buildRows(from: snapshot)
    }
}
```

The method is async, but the heavy row-building work is still inside a main-actor-isolated method.

Prefer moving the work to a function that explicitly switches away when appropriate:

```swift
@MainActor
final class SearchScreenModel {
    private(set) var results: [SearchResultRow] = []

    func applySnapshot(_ snapshot: SearchSnapshot) async {
        results = await buildSearchRows(from: snapshot)
    }
}

@concurrent
func buildSearchRows(from snapshot: SearchSnapshot) async -> [SearchResultRow] {
    SearchResultBuilder.buildRows(from: snapshot)
}
```

Before recommending this, verify that `SearchSnapshot` and `SearchResultRow` are safe to send across the boundary.

Prefer value types:

```swift
struct SearchSnapshot: Sendable {
    let items: [SearchItem]
}

struct SearchResultRow: Sendable {
    let title: String
    let subtitle: String
}
```

Do not move UI-owned reference models away from `MainActor` just to silence performance concerns.

## `@concurrent` Is Not a Magic Optimizer

`@concurrent` can move work away from the caller’s actor. It does not automatically make long synchronous computation cooperative.

Risky:

```swift
@concurrent
func buildHugeIndex(from records: [Record]) async -> SearchIndex {
    SearchIndexBuilder.build(from: records)
}
```

This may avoid blocking `MainActor`, but the computation can still monopolize a worker thread for a long time.

For long work, consider chunking and cancellation:

```swift
@concurrent
func buildHugeIndex(from records: [Record]) async throws -> SearchIndex {
    var builder = SearchIndexBuilder()

    for batch in records.chunked(into: 500) {
        try Task.checkCancellation()
        builder.add(batch)
        await Task.yield()
    }

    return builder.build()
}
```

Use `Task.yield()` selectively. It can improve responsiveness in long cooperative loops, but it is not a substitute for good algorithmic design or measurement.

## Sendable Boundary Review

When recommending `@concurrent`, check what crosses the boundary:

* parameters
* return values
* captured values
* stored dependencies
* closures
* reference types
* mutable shared state

Risky:

```swift
@MainActor
final class ExportScreenModel {
    private let draft: MutableExportDraft

    func preview() async -> ExportPreview {
        await makePreview(from: draft)
    }
}

@concurrent
func makePreview(from draft: MutableExportDraft) async -> ExportPreview {
    ExportPreview(draft: draft)
}
```

A mutable UI-owned reference object should usually stay on `MainActor`.

Prefer extracting a Sendable snapshot:

```swift
struct ExportDraftSnapshot: Sendable {
    let title: String
    let pages: [ExportPage]
    let options: ExportOptions
}

@MainActor
final class ExportScreenModel {
    private let draft: MutableExportDraft

    func preview() async -> ExportPreview {
        let snapshot = draft.snapshot()
        return await makePreview(from: snapshot)
    }
}

@concurrent
func makePreview(from snapshot: ExportDraftSnapshot) async -> ExportPreview {
    ExportPreview(snapshot: snapshot)
}
```

The UI model remains isolated to `MainActor`; the background work receives immutable data.

## `nonisolated` vs `@concurrent`

Use `nonisolated` when a member does not need actor state.

Use `@concurrent` when an async function should explicitly leave the caller’s actor.

They solve different problems.

### Good `nonisolated` Use

```swift
actor SymbolStore {
    private var symbols: [String: Symbol] = [:]

    nonisolated func canonicalSymbol(_ raw: String) -> String {
        raw.trimmingCharacters(in: .whitespacesAndNewlines).uppercased()
    }
}
```

The method is pure and does not need actor state.

### Good `@concurrent` Use

```swift
@MainActor
final class ReportModel {
    private(set) var report: Report?

    func reload() async throws {
        let data = try await reportService.load()
        report = try await compileReport(data)
    }
}

@concurrent
func compileReport(_ data: ReportData) async throws -> Report {
    try ReportCompiler.compile(data)
}
```

The compile step is meaningful work that should not run on the main actor.

## Do Not Add `@concurrent` Everywhere

Avoid `@concurrent` when:

* the function is cheap
* the function already runs off the main actor
* the function must remain on the caller’s actor for correctness
* the values are not safe to send
* the operation is mostly awaiting another async API
* there is no measured or plausible responsiveness issue

Risky overuse:

```swift
@concurrent
func title(for item: MenuItem) async -> String {
    item.title
}
```

This adds async and isolation complexity without a performance reason.

Prefer simple synchronous code:

```swift
func title(for item: MenuItem) -> String {
    item.title
}
```

## Default Actor Isolation

With default actor isolation settings, especially in app targets, code may be more main-actor-oriented than expected.

When reviewing new Swift code, check whether the project uses default MainActor isolation. If it does, many declarations may be main-actor isolated unless explicitly written otherwise.

This can be helpful for UI safety, but it can hide performance issues when heavy work is added to types that are implicitly main-actor isolated.

Review questions:

* Is this type UI-facing?
* Is it intentionally `@MainActor`?
* Is heavy work mixed into the UI-facing type?
* Should the heavy work be moved to a separate service or free function?
* Should the service remain nonisolated?
* Should CPU work use `@concurrent`?
* Are Sendable snapshots used instead of mutable UI references?

## MainActor Refactoring Pattern

Before:

```swift
@MainActor
final class InsightsModel {
    private(set) var insights: [InsightRow] = []

    func refresh() async throws {
        let events = try await eventService.loadEvents()
        insights = InsightEngine.computeRows(from: events)
    }
}
```

After:

```swift
@MainActor
final class InsightsModel {
    private(set) var insights: [InsightRow] = []

    func refresh() async throws {
        let events = try await eventService.loadEvents()
        insights = try await computeInsightRows(from: events)
    }
}

@concurrent
func computeInsightRows(from events: [Event]) async throws -> [InsightRow] {
    try Task.checkCancellation()
    return InsightEngine.computeRows(from: events)
}
```

Better for larger work:

```swift
@concurrent
func computeInsightRows(from events: [Event]) async throws -> [InsightRow] {
    var rows: [InsightRow] = []
    rows.reserveCapacity(events.count)

    for chunk in events.chunked(into: 250) {
        try Task.checkCancellation()
        rows.append(contentsOf: InsightEngine.computeRows(from: chunk))
        await Task.yield()
    }

    return rows
}
```

## Avoid Mixing UI State and Work Engines

A common design smell:

```swift
@MainActor
final class TimelineViewModel {
    var rows: [TimelineRow] = []

    func load() async throws {
        let raw = try await api.timeline()
        rows = TimelineProcessor.process(raw)
    }

    private func normalize(_ event: TimelineEvent) -> TimelineEvent {
        TimelineNormalizer.normalize(event)
    }
}
```

The type owns UI state and also contains processing logic. With main-actor isolation, processing can accidentally stay on the main actor.

Prefer separating UI coordination from processing:

```swift
@MainActor
final class TimelineViewModel {
    private let processor: TimelineProcessing
    private(set) var rows: [TimelineRow] = []

    init(processor: TimelineProcessing) {
        self.processor = processor
    }

    func load() async throws {
        let raw = try await api.timeline()
        rows = try await processor.rows(from: raw)
    }
}

struct TimelineProcessor: TimelineProcessing {
    func rows(from raw: TimelinePayload) async throws -> [TimelineRow] {
        try await buildTimelineRows(from: raw)
    }
}

@concurrent
func buildTimelineRows(from raw: TimelinePayload) async throws -> [TimelineRow] {
    TimelineRowBuilder.build(from: raw)
}
```

## Diagnostics

Use Instruments when isolation behavior is uncertain.

Symptoms:

* UI freezes while an async method is running
* `@MainActor` task has a long running section after an `await`
* CPU-heavy work appears on main actor
* adding `async` did not improve responsiveness
* code behaves differently after enabling Swift 6.2 settings
* concurrency diagnostics complain about sending non-Sendable values
* task appears concurrent but throughput does not improve

Likely causes:

* heavy work remains actor-isolated
* function is async but not switching off the caller’s actor
* non-Sendable reference state cannot safely cross isolation
* work is serialized through a main-actor type
* the real bottleneck is another actor or service

## Review Checklist

* [ ] Is the caller isolated to `MainActor` or another actor?
* [ ] Is default actor isolation enabled?
* [ ] Is the function merely async, or does it explicitly switch away?
* [ ] Is `@concurrent` justified by meaningful work?
* [ ] Are parameters, captures, and return values safe to send?
* [ ] Can mutable UI state be converted to a Sendable snapshot?
* [ ] Would `nonisolated` be more appropriate than `@concurrent`?
* [ ] Is the heavy work cancellable?
* [ ] Does long CPU work need chunking or yielding?
* [ ] Is the suggested change measured or tied to a concrete responsiveness issue?
* [ ] Can the design be improved by separating UI coordination from processing?
