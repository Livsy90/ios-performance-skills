# Async Lifecycle and MainActor

Use this reference when reviewing SwiftUI code that involves `.task(id:)`, `.task`, `.onAppear`, row lifecycle callbacks, duplicate loading, pagination triggers, cancellation, unstructured `Task`, `Task.detached`, `MainActor`, or heavy work during UI updates.

## Core Principle

Async work in SwiftUI should have a clear lifecycle boundary, a stable trigger, cooperative cancellation, and a narrow final UI update.

The agent should optimize for:

- no side effects during `body` evaluation
- predictable task start and restart behavior
- idempotent loading operations
- cancellation-aware state commits
- minimal heavy work on the main actor
- local, compact state mutations after async work completes

Do not describe async lifecycle issues as generic SwiftUI slowness. Explain which task starts, what causes it to restart, whether it can duplicate work, and where the main actor is doing too much.

## 1. Never Start Work From `body` Evaluation

`body` can be evaluated many times. Creating async work during `body` evaluation can start duplicate tasks, race with state updates, and make rendering produce side effects.

Risky:

```swift
struct AccountSummaryView: View {
    let model: AccountSummaryModel

    var body: some View {
        let _ = Task {
            await model.refresh()
        }

        AccountSummaryContent(model: model)
    }
}
```

Prefer a lifecycle modifier or an explicit user action:

```swift
struct AccountSummaryView: View {
    let accountID: Account.ID
    let model: AccountSummaryModel

    var body: some View {
        AccountSummaryContent(model: model)
            .task(id: accountID) {
                await model.load(accountID: accountID)
            }
    }
}
```

Use `.task(id:)` when the async work belongs to the view lifecycle and should restart when a meaningful input changes.

## 2. Use `.task(id:)` for Lifecycle-Bound Async Work

Use `.task(id:)` when work should:

- start with the view lifecycle
- cancel when the view disappears or changes identity
- restart when a specific semantic input changes

Good candidates:

- loading details for `accountID`
- applying a search query
- fetching content for a selected tab
- refreshing data when a filter changes
- starting a time-limited async subscription for the current entity

Example:

```swift
struct TransactionDetailsView: View {
    let transactionID: Transaction.ID
    let model: TransactionDetailsModel

    var body: some View {
        TransactionDetailsContent(state: model.state)
            .task(id: transactionID) {
                await model.load(transactionID: transactionID)
            }
    }
}
```

Avoid unstable IDs:

```swift
.task(id: UUID()) {
    await model.reload()
}
```

Avoid using an ID that changes as a side effect of the task itself:

```swift
.task(id: model.lastRefreshDate) {
    await model.refresh()
}
```

If `refresh()` updates `lastRefreshDate`, the trigger may cause unnecessary restarts.

Prefer a stable semantic trigger:

```swift
.task(id: filter) {
    await model.apply(filter: filter)
}
```

## 3. Use `.task` Without `id` Only for One Lifecycle Run

Use `.task` without `id` when the work should run for the current view identity and should not restart for changing inputs.

Example:

```swift
struct WelcomeView: View {
    let model: WelcomeModel

    var body: some View {
        WelcomeContent(state: model.state)
            .task {
                await model.loadInitialContentIfNeeded()
            }
    }
}
```

The model method should still be idempotent:

```swift
@MainActor
final class WelcomeModel {
    private var didLoad = false
    private(set) var state: WelcomeState = .idle

    func loadInitialContentIfNeeded() async {
        guard !didLoad else { return }
        didLoad = true

        state = .loading
        do {
            let content = try await service.loadWelcomeContent()
            try Task.checkCancellation()
            state = .loaded(content)
        } catch is CancellationError {
            state = .idle
        } catch {
            state = .failed(error)
        }
    }
}
```

Idempotency matters because SwiftUI view identity can change during navigation, conditional rendering, or parent restructuring.

## 4. Use `.onAppear` for Appearance Side Effects, Not Default Loading

`.onAppear` is useful for synchronous or appearance-related side effects:

- analytics exposure events
- starting an animation flag
- focusing a field after a view appears
- notifying a parent that a child became visible

Do not use `.onAppear` as the default async loading mechanism when `.task` or `.task(id:)` better expresses lifecycle and cancellation.

Risky:

```swift
.onAppear {
    Task {
        await model.load()
    }
}
```

This creates an unstructured task that is not automatically tied to the view's task modifier lifecycle.

Prefer:

```swift
.task {
    await model.loadIfNeeded()
}
```

If `.onAppear` must start async work, store and cancel the task when appropriate:

```swift
struct ExportStatusView: View {
    @State private var pollingTask: Task<Void, Never>?
    let model: ExportStatusModel

    var body: some View {
        ExportStatusContent(state: model.state)
            .onAppear {
                pollingTask = Task {
                    await model.pollStatus()
                }
            }
            .onDisappear {
                pollingTask?.cancel()
                pollingTask = nil
            }
    }
}
```

Prefer `.task` for this pattern when manual task ownership is not required.

## 5. Row Lifecycle Callbacks Can Fire Often

Rows in `List`, `LazyVStack`, and other lazy containers can appear more than once during scrolling, navigation, refreshes, filtering, and view identity changes.

Avoid assuming row `.onAppear` means "the row appeared for the first time".

Risky pagination trigger:

```swift
List(model.rows) { row in
    ActivityRow(row: row)
        .onAppear {
            if row.id == model.rows.last?.id {
                Task {
                    await model.loadNextPage()
                }
            }
        }
}
```

Problems:

- row appearance may happen multiple times
- the task is unstructured
- the model may start duplicate page loads
- the last row may disappear and reappear during layout changes
- a slow earlier request can race with a later one

Prefer an idempotent model method and a stable prefetch boundary:

```swift
List(model.rows) { row in
    ActivityRow(row: row)
        .task(id: row.id) {
            guard model.shouldPrefetch(after: row.id) else { return }
            await model.loadNextPageIfNeeded(trigger: row.id)
        }
}
```

Then guard the actual load in the model:

```swift
@MainActor
final class ActivityFeedModel {
    private var isLoadingNextPage = false
    private var loadedPageNumbers: Set<Int> = []

    var rows: [ActivityRowModel] = []
    var nextPage: Int? = 1

    func shouldPrefetch(after rowID: ActivityRowModel.ID) -> Bool {
        rows.suffix(5).contains { $0.id == rowID }
    }

    func loadNextPageIfNeeded(trigger rowID: ActivityRowModel.ID) async {
        guard shouldPrefetch(after: rowID) else { return }
        guard !isLoadingNextPage else { return }
        guard let page = nextPage else { return }
        guard !loadedPageNumbers.contains(page) else { return }

        isLoadingNextPage = true
        defer { isLoadingNextPage = false }

        do {
            let response = try await service.loadPage(page)
            try Task.checkCancellation()

            loadedPageNumbers.insert(page)
            nextPage = response.nextPage
            rows.append(contentsOf: response.items.map(ActivityRowModel.init))
        } catch is CancellationError {
            // Ignore view-lifecycle cancellation.
        } catch {
            // Store an error state if the UI needs to show retry.
        }
    }
}
```

For very large lists, consider moving the pagination trigger to a footer or sentinel view so every row does not create its own lifecycle task.

## 6. Pagination Loads Must Be Idempotent

The agent should flag pagination code that lacks guards for:

- `isLoading`
- `hasMore` or `nextPage != nil`
- already requested page keys
- stale trigger IDs
- cancellation before committing results
- duplicate refresh and pagination overlap

Risky:

```swift
@MainActor
func loadNextPage() async {
    let response = try? await service.loadPage(nextPage)
    rows.append(contentsOf: response?.items.map(ActivityRowModel.init) ?? [])
    nextPage += 1
}
```

Prefer:

```swift
@MainActor
func loadNextPageIfNeeded() async {
    guard !isLoadingNextPage else { return }
    guard let page = nextPage else { return }

    isLoadingNextPage = true
    defer { isLoadingNextPage = false }

    do {
        let response = try await service.loadPage(page)
        try Task.checkCancellation()

        rows.append(contentsOf: response.items.map(ActivityRowModel.init))
        nextPage = response.nextPage
    } catch is CancellationError {
        // Cancellation is expected when the view disappears or the trigger changes.
    } catch {
        pageLoadError = error
    }
}
```

For refresh plus pagination, prevent overlapping state commits:

```swift
@MainActor
func refresh() async {
    guard !isRefreshing else { return }

    isRefreshing = true
    nextPage = nil
    defer { isRefreshing = false }

    do {
        let response = try await service.loadFirstPage()
        try Task.checkCancellation()

        rows = response.items.map(ActivityRowModel.init)
        nextPage = response.nextPage
    } catch is CancellationError {
        // Leave existing content if cancellation should not clear the screen.
    } catch {
        refreshError = error
    }
}

@MainActor
func loadNextPageIfNeeded() async {
    guard !isRefreshing else { return }
    guard !isLoadingNextPage else { return }
    guard let page = nextPage else { return }

    // Continue page loading.
}
```

## 7. Cancellation Is Cooperative

Cancellation marks a task as canceled; it does not forcibly stop every operation immediately.

The agent should check whether long-running work:

- observes cancellation
- avoids committing stale results after cancellation
- handles `CancellationError` separately from real failures
- cancels child tasks or stored task handles when manual ownership is used

Risky:

```swift
@MainActor
func search(query: String) async {
    state = .loading

    do {
        let results = try await service.search(query)
        state = .loaded(results)
    } catch {
        state = .failed(error)
    }
}
```

Better:

```swift
@MainActor
func search(query: String) async {
    state = .loading

    do {
        let results = try await service.search(query)
        try Task.checkCancellation()
        state = .loaded(results)
    } catch is CancellationError {
        // Do not show cancellation as a user-facing error.
    } catch {
        state = .failed(error)
    }
}
```

For CPU-bound loops, check cancellation inside the loop:

```swift
func buildRows(from items: [Transaction]) async throws -> [TransactionRowModel] {
    var rows: [TransactionRowModel] = []
    rows.reserveCapacity(items.count)

    for item in items {
        try Task.checkCancellation()
        rows.append(TransactionRowModel(item))
    }

    return rows
}
```

## 8. Search and Filter Tasks Should Cancel Older Work

For async search, `.task(id:)` can express restart-on-query-change behavior.

Example with debounce:

```swift
struct SearchResultsView: View {
    @State private var query = ""
    let model: SearchResultsModel

    var body: some View {
        VStack {
            TextField("Search", text: $query)
            SearchResultsContent(state: model.state)
        }
        .task(id: query) {
            let trimmed = query.trimmingCharacters(in: .whitespacesAndNewlines)
            guard !trimmed.isEmpty else {
                await model.clearResults()
                return
            }

            do {
                try await Task.sleep(for: .milliseconds(300))
                try Task.checkCancellation()
                await model.search(query: trimmed)
            } catch is CancellationError {
                // A newer query replaced this one.
            } catch {
                await model.handleSearchError(error)
            }
        }
    }
}
```

Do not debounce by storing many independent `Task` values unless manual ownership is required.

## 9. `Task` Is Not a Background-Thread Escape Hatch

Creating `Task { ... }` from SwiftUI or from a `@MainActor` context does not automatically make heavy work safe for UI responsiveness.

Risky:

```swift
@MainActor
func applyFilter(_ filter: Filter) {
    Task {
        let rows = allItems
            .filter { $0.matches(filter) }
            .map(TransactionRowModel.init)

        self.rows = rows
    }
}
```

This may still run heavy transformation work in a main-actor context.

Prefer moving pure transformation work outside the main actor when the data is safe to send across concurrency boundaries:

```swift
@MainActor
func applyFilter(_ filter: Filter) async {
    let snapshot = allItems

    do {
        let rows = try await rowBuilder.buildRows(
            from: snapshot,
            filter: filter
        )

        try Task.checkCancellation()
        self.rows = rows
    } catch is CancellationError {
        // Ignore stale filter work.
    } catch {
        self.error = error
    }
}
```

Where the worker is not main-actor isolated:

```swift
struct TransactionRowBuilder: Sendable {
    func buildRows(
        from items: [Transaction],
        filter: Filter
    ) async throws -> [TransactionRowModel] {
        var rows: [TransactionRowModel] = []
        rows.reserveCapacity(items.count)

        for item in items where item.matches(filter) {
            try Task.checkCancellation()
            rows.append(TransactionRowModel(item))
        }

        return rows
    }
}
```

Use `Task.detached` sparingly. It does not inherit actor isolation in the same way as ordinary tasks and requires careful `Sendable` boundaries. It is not the default fix for slow UI.

## 10. Keep MainActor State Mutations Compact

A `@MainActor` view model is often a good fit for SwiftUI state, but it should not perform large synchronous transformations on the main actor during user interactions.

Risky:

```swift
@MainActor
final class StatementModel {
    var rows: [StatementRowModel] = []

    func apply(response: StatementResponse) {
        rows = response.entries
            .sorted { $0.date > $1.date }
            .map { entry in
                StatementRowModel(
                    id: entry.id,
                    title: entry.title,
                    amountText: CurrencyFormatter.shared.string(from: entry.amount),
                    dateText: DateFormatter.shared.string(from: entry.date)
                )
            }
    }
}
```

Prefer preparing render-ready data outside the main actor, then applying one compact state update:

```swift
@MainActor
final class StatementModel {
    var rows: [StatementRowModel] = []
    private let rowBuilder: StatementRowBuilder

    func load() async {
        do {
            let response = try await service.loadStatement()
            let preparedRows = try await rowBuilder.makeRows(from: response.entries)
            try Task.checkCancellation()
            rows = preparedRows
        } catch is CancellationError {
            // Ignore lifecycle cancellation.
        } catch {
            // Update error state if needed.
        }
    }
}
```

The exact boundary depends on the app architecture. The important rule is to avoid doing large CPU-bound work as part of UI update handling.

## 11. Prefer Type Isolation Over Repeated `MainActor.run`

Repeated `MainActor.run` calls can hide isolation design problems and fragment the update path.

Risky:

```swift
func load() async {
    await MainActor.run { state = .loading }

    do {
        let response = try await service.load()
        await MainActor.run { title = response.title }
        await MainActor.run { rows = response.items.map(RowModel.init) }
        await MainActor.run { state = .loaded }
    } catch {
        await MainActor.run { state = .failed(error) }
    }
}
```

Prefer isolating the UI-facing model and applying compact updates:

```swift
@MainActor
final class DashboardModel {
    var state: DashboardState = .idle

    func load() async {
        state = .loading

        do {
            let response = try await service.loadDashboard()
            let rows = try await rowBuilder.makeRows(from: response.items)
            try Task.checkCancellation()
            state = .loaded(rows)
        } catch is CancellationError {
            state = .idle
        } catch {
            state = .failed(error)
        }
    }
}
```

Use `MainActor.run` for explicit boundary hops when needed, not as a replacement for clear actor isolation.

## 12. Avoid Stale Result Commits

When a task depends on an input, an old task can complete after a newer one. `.task(id:)` helps by canceling the old task, but cancellation is cooperative, so stale result protection may still be needed.

Risky:

```swift
@MainActor
func load(accountID: Account.ID) async {
    let details = try? await service.loadDetails(accountID: accountID)
    state = details.map(AccountState.loaded) ?? .failed
}
```

Prefer checking cancellation and/or validating the active input before committing:

```swift
@MainActor
final class AccountDetailsModel {
    private var activeAccountID: Account.ID?
    var state: AccountDetailsState = .idle

    func load(accountID: Account.ID) async {
        activeAccountID = accountID
        state = .loading

        do {
            let details = try await service.loadDetails(accountID: accountID)
            try Task.checkCancellation()
            guard activeAccountID == accountID else { return }
            state = .loaded(details)
        } catch is CancellationError {
            // A new account or view disappearance canceled this load.
        } catch {
            guard activeAccountID == accountID else { return }
            state = .failed(error)
        }
    }
}
```

Use this especially for search, tab switching, navigation detail screens, and fast-changing filters.

## 13. Avoid Updating State Too Frequently During Async Work

Repeated main-actor state updates can cause unnecessary view invalidation.

Risky:

```swift
@MainActor
func importItems(_ items: [ImportItem]) async {
    rows.removeAll()

    for item in items {
        let row = await importer.importRow(item)
        rows.append(row)
    }
}
```

This can invalidate the UI once per item.

Prefer batching when the UI does not need per-item progress:

```swift
@MainActor
func importItems(_ items: [ImportItem]) async {
    do {
        let importedRows = try await importer.importRows(items)
        try Task.checkCancellation()
        rows = importedRows
    } catch is CancellationError {
        // Ignore cancellation or preserve existing rows.
    } catch {
        error = error
    }
}
```

If progress is needed, throttle updates or update a lightweight progress value instead of rebuilding the full visible collection on every step.

## 14. Async Streams and Long-Lived Subscriptions

When a view consumes an `AsyncSequence`, tie the loop to `.task` so it cancels with the view lifecycle.

Example:

```swift
struct PriceTickerView: View {
    let model: PriceTickerModel

    var body: some View {
        PriceTickerContent(prices: model.prices)
            .task {
                await model.observePrices()
            }
    }
}
```

Model:

```swift
@MainActor
final class PriceTickerModel {
    var prices: [PriceRowModel] = []
    private let priceStream: PriceStream

    func observePrices() async {
        do {
            for try await snapshot in priceStream.snapshots() {
                try Task.checkCancellation()
                prices = snapshot.map(PriceRowModel.init)
            }
        } catch is CancellationError {
            // Expected when the view disappears.
        } catch {
            // Store stream error if visible to the user.
        }
    }
}
```

For high-frequency streams, avoid updating full UI state for every event. Coalesce, throttle, diff, or update only the visible/changed values when possible.

## 15. Error Handling Should Treat Cancellation Separately

Cancellation is often a normal lifecycle event, not a user-facing failure.

Risky:

```swift
catch {
    state = .failed(error)
}
```

Prefer:

```swift
catch is CancellationError {
    // Expected lifecycle cancellation.
} catch {
    state = .failed(error)
}
```

Only show cancellation to the user when cancellation was explicitly user-initiated and the product wants visible feedback.

## 16. Review Checklist

When reviewing async lifecycle and main-actor code, check:

- Is async work started from `.task`, `.task(id:)`, `.refreshable`, an explicit user action, or a controlled model method rather than `body`?
- Does the task have a stable semantic trigger?
- Could the trigger restart because the task mutates the same value used as its ID?
- Is `.onAppear` used only where repeated appearances are acceptable or guarded?
- Are row lifecycle callbacks idempotent?
- Can pagination start duplicate requests?
- Are refresh and pagination prevented from committing conflicting state?
- Does the code handle cancellation separately from real errors?
- Is cancellation checked before committing final state?
- Could an older task overwrite newer state?
- Is heavy transformation work kept off the main actor when safe?
- Are main-actor mutations compact and batched?
- Is `Task {}` being used as an accidental background-thread workaround?
- Is `Task.detached` avoided unless there is a clear Sendable and isolation boundary?
- Are high-frequency async streams coalesced or throttled before updating UI?

## 17. Common Red Flags

Flag these patterns:

```swift
let _ = Task { await model.load() }
```

inside `body`.

```swift
.onAppear {
    Task { await model.load() }
}
```

when `.task` would express lifecycle and cancellation better.

```swift
.task(id: UUID()) { ... }
```

or any unstable task ID.

```swift
.task(id: model.lastUpdatedAt) {
    await model.refresh()
}
```

when the task changes the same value that triggers it.

```swift
.onAppear {
    if row.id == rows.last?.id {
        Task { await loadNextPage() }
    }
}
```

without duplicate-load guards.

```swift
Task {
    let rows = expensiveMainActorTransformation()
    self.rows = rows
}
```

when created from a main-actor context and assumed to be background work.

```swift
catch {
    state = .failed(error)
}
```

when cancellation should be ignored or handled separately.

```swift
for item in items {
    rows.append(makeRow(item))
}
```

when each append invalidates visible UI and batching would be sufficient.

## 18. Agent Guidance

When the agent reviews async SwiftUI code, it should answer with:

1. the lifecycle trigger that starts the work
2. whether the trigger is stable
3. whether duplicate work is possible
4. whether cancellation is handled correctly
5. whether stale results can commit
6. whether heavy work runs on the main actor
7. the smallest refactor that fixes the issue
8. how to validate the fix if confirmation is needed

Prefer precise statements:

```md
This `.onAppear` can run repeatedly as rows enter and leave the lazy list. The model should make pagination idempotent with `isLoadingNextPage`, `nextPage`, and cancellation checks before committing rows.
```

Avoid unsupported claims:

```md
This definitely causes a 300 ms hitch.
```

Unless a trace, signpost, XCTest result, or user-provided measurement proves it.

## Final Rule

Async lifecycle performance in SwiftUI is mostly about preventing accidental work: duplicate tasks, unstable restarts, stale commits, excessive main-actor transformation, and too many state updates. Make the trigger stable, make the operation idempotent, make cancellation explicit, and keep the final UI mutation small.
