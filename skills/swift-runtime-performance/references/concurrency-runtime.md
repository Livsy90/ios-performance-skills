# Concurrency Runtime

Use this reference when a review involves Swift Concurrency performance, actor isolation, task creation, structured concurrency, MainActor responsiveness, cancellation, priority, continuations, blocking calls, executor hops, or Swift 6.2+ isolation behavior.

The goal is not to rewrite every asynchronous API. The goal is to identify whether concurrency structure improves responsiveness, parallelism, safety, and resource usage, or whether it introduces avoidable scheduling, allocation, contention, or lifetime cost.

## Core model

Avoid simple rules such as:

* `async` means background.
* `await` means a new thread.
* actor means parallel execution.
* actor means no logical races.
* `Task {}` is the default way to run async work.
* `Task.detached` is the default way to leave the main actor.
* more tasks means more performance.
* `nonisolated` always means background execution.
* `MainActor.run` is the normal way to fix isolation.
* Swift Concurrency automatically makes blocking APIs safe.

Use a runtime-and-isolation model instead.

Swift Concurrency is built around:

* tasks as units of asynchronous work
* suspension points marked by `await`
* actors as isolation domains for mutable state
* executors as places where jobs are scheduled
* structured concurrency for scoped child work
* cooperative cancellation
* priority propagation through task relationships
* compiler-enforced data-race safety in Swift 6 language mode

A task is not the same thing as a thread. An actor hop is not the same thing as an OS context switch. An `await` is a suspension point where the task may give up execution and later resume.

## What to inspect first

Before proposing a concurrency refactor, identify:

1. Is the issue responsiveness, throughput, memory, battery, correctness, or readability?
2. Is the problem on `MainActor`, a custom actor, a task group, or an unstructured task?
3. Is work being blocked, suspended, serialized, or over-parallelized?
4. Is there evidence from Swift Concurrency Instrument, Time Profiler, Allocations, signposts, or logs?
5. Are cancellation, priority, and task lifetime preserved?
6. Is the code relying on actor state remaining unchanged across `await`?
7. Is blocking synchronous work running inside Swift concurrency tasks?
8. Is Swift 6.2+ caller isolation changing where nonisolated async work runs?
9. Is the proposed change improving performance without weakening isolation safety?

Do not treat concurrency as a performance feature by itself. Concurrency helps when it removes blocking, expresses independent work, or protects shared state with the right granularity.

## Measurement guidance

Use the right tool for the suspected problem:

* Swift Concurrency Instrument: task trees, task counts, actor timelines, actor contention, MainActor blocking, continuation misuse.
* Time Profiler: CPU-heavy work, blocking calls, expensive parsing/decoding, retain/release traffic around async code.
* Allocations: task/closure churn, captured object graphs, temporary buffers, continuation wrappers.
* Points of Interest / signposts: user-visible timing around screen loading, imports, exports, searches, or pipelines.
* Main Thread Checker and UI responsiveness traces: accidental UI-thread blocking.
* Network/file traces: synchronous APIs hidden inside async wrappers.
* Optimized builds: runtime behavior and optimizer effects differ significantly from Debug builds.

If evidence is missing, state what should be measured before recommending broad rewrites.

## Tasks are units of work, not threads

A task represents asynchronous work that can suspend and resume. It does not guarantee a dedicated thread and should not be used as a thread abstraction.

Review questions:

* Is the task doing useful independent work?
* Is the task created only to call one async function from sync code?
* Does the task have a clear owner and lifetime?
* Will cancellation reach this task?
* Does the task inherit actor isolation intentionally?
* Is the task retaining large state longer than expected?

Avoid creating tasks just to avoid designing ownership. Unstructured tasks can easily outlive the object or screen that created them.

## Structured concurrency

Prefer structured concurrency when child work belongs to the lifetime of the caller.

Use:

* `async let` for a small, fixed number of independent child operations.
* `withTaskGroup` for dynamic child operations.
* `withThrowingTaskGroup` when child work can fail.
* bounded task groups when input size can be large.
* ordinary `await` when work is sequential and not independent.

Structured child tasks are scoped. Their lifetime, cancellation, priority, and task-local behavior are tied to the parent task.

Example:

```swift
func loadDashboard(
    accountID: AccountID,
    service: DashboardService
) async throws -> DashboardModel {
    async let summary = service.summary(for: accountID)
    async let alerts = service.alerts(for: accountID)
    async let limits = service.limits(for: accountID)

    return try await DashboardModel(
        summary: summary,
        alerts: alerts,
        limits: limits
    )
}
```

This fits `async let` because the number of independent operations is small and fixed.

Do not use `async let` for a large or unknown number of operations. Use a task group with an explicit concurrency limit instead.

## Bounded task groups

Task groups make it easy to express dynamic parallelism, but adding one task per input can create too many tasks, too much memory pressure, too many network requests, or too much downstream contention.

Check every task-group loop:

* Can the input be large?
* Does each child allocate significant memory?
* Does each child perform network or file I/O?
* Is the backend or database rate-limited?
* Is ordering required?
* What happens on cancellation or first error?

Risky pattern:

```swift
func decodeAll(_ blobs: [DataBlob]) async throws -> [DecodedBlob] {
    try await withThrowingTaskGroup(of: DecodedBlob.self) { group in
        for blob in blobs {
            group.addTask {
                try await decode(blob)
            }
        }

        var decoded: [DecodedBlob] = []
        for try await value in group {
            decoded.append(value)
        }
        return decoded
    }
}
```

This may be fine for small inputs. For large inputs, prefer bounded concurrency.

Example:

```swift
func decodeAllBounded(
    _ blobs: [DataBlob],
    maxConcurrentTasks: Int
) async throws -> [DecodedBlob] {
    precondition(maxConcurrentTasks > 0)

    var iterator = blobs.enumerated().makeIterator()
    var results = Array<DecodedBlob?>(repeating: nil, count: blobs.count)

    try await withThrowingTaskGroup(of: (Int, DecodedBlob).self) { group in
        for _ in 0..<maxConcurrentTasks {
            guard let next = iterator.next() else { break }
            group.addTask {
                try Task.checkCancellation()
                return (next.offset, try await decode(next.element))
            }
        }

        while let (index, decoded) = try await group.next() {
            results[index] = decoded

            if let next = iterator.next() {
                group.addTask {
                    try Task.checkCancellation()
                    return (next.offset, try await decode(next.element))
                }
            }
        }
    }

    return results.compactMap { $0 }
}
```

Use this pattern when task count, memory, network pressure, or CPU saturation must be controlled.

## Unstructured tasks

`Task {}` creates an unstructured task. It can inherit useful context, such as actor isolation, priority, and task-local values, but it is not scoped like a child task in structured concurrency.

Use unstructured tasks when:

* bridging from synchronous UI/event code into async work
* starting work from a lifecycle boundary that cannot itself be async
* creating a task with explicit ownership and cancellation storage
* intentionally decoupling work from the immediate call stack, but not from logical ownership

Be cautious when:

* the task is fire-and-forget
* no one stores or cancels the task
* the task captures `self`
* the task updates UI after the screen is gone
* multiple tasks can overlap and race logically
* task lifetime exceeds the lifetime of the feature that created it

Example:

```swift
@MainActor
final class SearchViewModel {
    private var searchTask: Task<Void, Never>?

    func queryChanged(_ query: String) {
        searchTask?.cancel()

        searchTask = Task { [query] in
            try? await Task.sleep(for: .milliseconds(250))
            guard !Task.isCancelled else { return }

            let results = try? await SearchService.shared.results(for: query)
            guard !Task.isCancelled else { return }

            self.apply(results ?? [])
        }
    }

    private func apply(_ results: [SearchResult]) {
        // update UI-facing state
    }
}
```

The task is unstructured, but it has explicit ownership and cancellation.

## Detached tasks

`Task.detached` creates work that does not inherit the current actor isolation. It is also not linked to the parent task's structured lifetime.

Treat `Task.detached` as an escape hatch, not a default background-work primitive.

Use it only when:

* the work must not run on the current actor
* the work is intentionally independent from the caller's lifetime
* captured values are safe to send across isolation
* cancellation is handled explicitly
* priority is chosen intentionally
* task-local values are not required, or are propagated deliberately

Risky pattern:

```swift
@MainActor
final class ExportViewModel {
    func exportTapped(document: Document) {
        Task.detached {
            try await ExportService.shared.export(document)
        }
    }
}
```

This detaches lifetime, cancellation, and isolation from the feature that created the work. If the export belongs to the screen or user action, prefer an owned `Task` or a structured call from an async context.

Better:

```swift
@MainActor
final class ExportViewModel {
    private var exportTask: Task<Void, Never>?

    func exportTapped(document: Document) {
        exportTask?.cancel()

        exportTask = Task { [document] in
            do {
                try await ExportService.shared.export(document)
            } catch is CancellationError {
                return
            } catch {
                self.showExportError(error)
            }
        }
    }

    func cancelExport() {
        exportTask?.cancel()
    }

    private func showExportError(_ error: Error) {
        // update UI-facing state
    }
}
```

Use `Task.detached` only when independence is part of the design, not because "background" is desired.

## MainActor responsiveness

`MainActor` protects UI-facing state and coordinates work that must run on the main thread. It should not become the place where expensive non-UI work runs.

Look for:

* parsing on `MainActor`
* JSON decoding on `MainActor`
* image decoding or resizing on `MainActor`
* compression or encryption on `MainActor`
* large diff computation on `MainActor`
* database migrations on `MainActor`
* synchronous file I/O on `MainActor`
* long loops inside UI-isolated types

Review questions:

* Does this work actually need UI isolation?
* Can the UI state update remain on `MainActor` while the expensive work moves elsewhere?
* Is this function `@MainActor` because the whole type is UI-facing?
* Is a Swift 6.2+ default actor isolation setting causing more code to run on `MainActor`?
* Should the heavy work use `@concurrent`, a non-main actor service, or a dedicated async API?

Example:

```swift
@MainActor
final class ImportViewModel {
    private(set) var state: ImportState = .idle

    func importTapped(url: URL) {
        state = .running

        Task {
            do {
                let summary = try await ImportService().importFile(at: url)
                state = .finished(summary)
            } catch is CancellationError {
                state = .idle
            } catch {
                state = .failed(error)
            }
        }
    }
}
```

The view model stays UI-isolated. The expensive import should happen outside `MainActor` inside `ImportService`.

## Swift 6.2+ caller isolation, `nonisolated`, and `@concurrent`

In Swift 6.2+ configurations that adopt the new async isolation behavior, a plain `nonisolated async` function may run on the caller's actor by default instead of automatically switching away from it.

Do not assume:

* `async` means background
* `nonisolated async` means generic executor
* calling from `MainActor` automatically moves work off the main actor

Use this model:

* isolated functions run on their actor.
* synchronous `nonisolated` functions run in the caller's execution context.
* with Swift 6.2+ caller-isolation behavior, `nonisolated async` functions can also run on the caller's actor.
* `@concurrent` explicitly states that an async function should switch away from the caller's actor to run concurrently.

Review questions:

* Which Swift language mode and upcoming-feature settings are used?
* Is default actor isolation enabled for the module?
* Does the function need caller isolation for safety and usability?
* Does the function need to leave the caller's actor for responsiveness?
* Would `@concurrent` be clearer than relying on older `nonisolated async` behavior?
* Is the data crossing the boundary `Sendable`?

Example:

```swift
@MainActor
final class ReportViewModel {
    func refresh() async throws {
        let raw = try await ReportLoader().load()
        let model = try await ReportBuilder().build(from: raw)
        apply(model)
    }

    private func apply(_ model: ReportModel) {
        // update UI-facing state
    }
}
```

If `ReportBuilder.build(from:)` is CPU-heavy and the project uses Swift 6.2+ caller-isolation behavior, confirm whether it actually leaves `MainActor`. If it should always leave the caller's actor, make that explicit in the API design.

```swift
struct ReportBuilder {
    @concurrent
    func build(from raw: RawReport) async throws -> ReportModel {
        try Task.checkCancellation()
        return try buildReportModel(from: raw)
    }
}
```

Use `@concurrent` deliberately. It is not a decoration for every async function; it communicates an execution choice and may require values crossing isolation to be safe.

## Actor isolation

Actors protect their isolated mutable state. They do not make all logic inside them automatically efficient or logically atomic across suspension points.

Use actors for:

* async state shared across tasks
* consistency boundaries
* serialized access to mutable state
* long-lived services with async APIs
* state where compile-time isolation is valuable

Do not use actors for:

* tiny synchronous counters when a narrow lock is simpler
* pure CPU algorithms with no shared state
* avoiding API design decisions
* making every mutable object "safe" by default
* storing all app state behind one global bottleneck

Review questions:

* What invariant does the actor protect?
* Are actor-isolated methods short and state-focused?
* Is CPU-heavy work performed while isolated?
* Are callers repeatedly crossing the actor boundary for small operations?
* Can reads/writes be batched?
* Does the actor method suspend while relying on old state?

## Actor reentrancy

Swift actors are reentrant. When an actor-isolated function reaches an `await`, the actor may run other work before the original function resumes.

Do not assume actor state is unchanged after an `await`.

Risky pattern:

```swift
actor SessionStore {
    private var token: AuthToken?

    func refreshIfNeeded() async throws -> AuthToken {
        if let token, !token.isExpired {
            return token
        }

        let newToken = try await AuthAPI.refreshToken()
        token = newToken
        return newToken
    }
}
```

If two tasks call `refreshIfNeeded()` concurrently when the token is expired, both can observe the expired state, both can suspend during the network call, and both can perform refresh work.

A safer pattern is to track in-flight work:

```swift
actor SessionStore {
    private var token: AuthToken?
    private var refreshTask: Task<AuthToken, Error>?

    func refreshIfNeeded() async throws -> AuthToken {
        if let token, !token.isExpired {
            return token
        }

        if let refreshTask {
            return try await refreshTask.value
        }

        let task = Task<AuthToken, Error> {
            try await AuthAPI.refreshToken()
        }

        refreshTask = task

        do {
            let newToken = try await task.value
            token = newToken
            refreshTask = nil
            return newToken
        } catch {
            refreshTask = nil
            throw error
        }
    }
}
```

Review rule:

* Re-check actor state after `await` when the decision depends on state observed before suspension.
* Use in-flight task tracking when duplicate work must be coalesced.
* Avoid holding a logical invariant across `await` unless the design explicitly accounts for reentrancy.

## Actor hops and batching

A call across actor isolation may require scheduling work on that actor's executor. This is not necessarily expensive in isolation, but repeated hops in hot paths can add latency and serialization.

Risky pattern:

```swift
func render(ids: [ImageID], index: ThumbnailIndex) async -> [Thumbnail] {
    var thumbnails: [Thumbnail] = []

    for id in ids {
        if let thumbnail = await index.thumbnail(for: id) {
            thumbnails.append(thumbnail)
        }
    }

    return thumbnails
}
```

Better:

```swift
actor ThumbnailIndex {
    private var storage: [ImageID: Thumbnail] = [:]

    func thumbnails(for ids: [ImageID]) -> [Thumbnail] {
        ids.compactMap { storage[$0] }
    }
}

func render(ids: [ImageID], index: ThumbnailIndex) async -> [Thumbnail] {
    await index.thumbnails(for: ids)
}
```

Batch actor operations when:

* the caller repeatedly reads related state
* the caller needs a consistent snapshot
* the actor hop happens inside a loop
* the work is cheap but called many times
* the actor owns the data needed for the operation

Do not batch unrelated operations only to reduce hops. Preserve API clarity and actor invariants.

## CPU-heavy work inside actors

Actor isolation should protect state. It should not automatically become the place where expensive CPU work runs.

Risky pattern:

```swift
actor DocumentIndex {
    private var documents: [DocumentID: Document] = [:]

    func rebuildSearchIndex() {
        var newIndex = SearchIndex()

        for document in documents.values {
            newIndex.insert(document)
        }

        // store index
    }
}
```

If rebuilding is expensive, the actor cannot process other messages while this synchronous work runs.

Better pattern:

```swift
actor DocumentIndex {
    private var documents: [DocumentID: Document] = [:]
    private var index: SearchIndex = .empty

    func rebuildSearchIndex() async {
        let snapshot = Array(documents.values)
        let newIndex = await SearchIndexBuilder().build(from: snapshot)

        index = newIndex
    }
}
```

This snapshots actor state while isolated, performs expensive work outside the actor, then returns to update actor state.

In Swift 6.2+ projects, ensure that the builder actually leaves the actor if that is required for responsiveness. Consider `@concurrent` for APIs that intentionally run away from the caller's actor.

## Blocking and the cooperative pool

Swift Concurrency uses cooperative scheduling. Tasks should suspend instead of blocking threads.

Flag these patterns inside async code:

* `Thread.sleep`
* semaphore waits
* condition waits
* synchronous file I/O on important executors
* synchronous network I/O
* blocking database APIs
* blocking legacy SDK calls
* long-held locks around expensive work

Prefer:

* native async APIs
* `Task.sleep` instead of `Thread.sleep`
* short lock scopes
* moving unavoidable blocking work outside the Swift concurrency cooperative pool
* bridging legacy callback APIs with checked continuations
* explicit cancellation handling around blocking adapters

Risky pattern:

```swift
func waitBeforeRetry() async {
    Thread.sleep(forTimeInterval: 1)
}
```

Better:

```swift
func waitBeforeRetry() async throws {
    try await Task.sleep(for: .seconds(1))
}
```

If a legacy API is truly blocking, wrap it carefully:

```swift
func loadLegacyFile(at url: URL) async throws -> FilePayload {
    try await withCheckedThrowingContinuation { continuation in
        legacyQueue.async {
            do {
                let payload = try LegacyFileReader.readSynchronously(from: url)
                continuation.resume(returning: payload)
            } catch {
                continuation.resume(throwing: error)
            }
        }
    }
}
```

The key point is not "DispatchQueue is faster". The key point is to avoid blocking cooperative Swift concurrency threads with operations that cannot suspend.

## Continuations

Continuations are for bridging callback-based or legacy asynchronous APIs into async/await.

Use checked continuations by default.

Review every continuation for:

* exactly one resume on every path
* success path
* error path
* cancellation path
* timeout path
* early-return path
* callback called multiple times
* callback never called
* underlying operation cleanup

Risky pattern:

```swift
func fetchToken() async throws -> AuthToken {
    try await withCheckedThrowingContinuation { continuation in
        authClient.fetchToken { result in
            if case let .success(token) = result {
                continuation.resume(returning: token)
            }

            // failure path forgot to resume
        }
    }
}
```

Better:

```swift
func fetchToken() async throws -> AuthToken {
    try await withCheckedThrowingContinuation { continuation in
        authClient.fetchToken { result in
            switch result {
            case .success(let token):
                continuation.resume(returning: token)
            case .failure(let error):
                continuation.resume(throwing: error)
            }
        }
    }
}
```

Checked continuations help detect misuse, but they do not design cancellation for you. If the underlying API supports cancellation, connect it explicitly.

## Cancellation

Cancellation in Swift is cooperative. Cancelling a task marks it as cancelled; the running code must check and respond.

Review questions:

* Does long-running work check cancellation?
* Does a loop check cancellation periodically?
* Does the code check cancellation before expensive work?
* Does cancellation propagate to child tasks?
* Does cancellation clean up underlying operations?
* Are cancellation errors swallowed accidentally?
* Is UI state updated correctly when cancellation occurs?

Prefer:

```swift
func processRecords(_ records: [Record]) async throws -> [Output] {
    var output: [Output] = []
    output.reserveCapacity(records.count)

    for record in records {
        try Task.checkCancellation()
        output.append(try await process(record))
    }

    return output
}
```

Use `Task.isCancelled` when cancellation should produce a non-throwing early return. Use `Task.checkCancellation()` when throwing cancellation is appropriate.

Do not treat cancellation as failure unless the product experience requires it.

## Priority

Task priority is a scheduling hint, not a correctness mechanism.

Structured tasks help priority propagate through the task tree. Detached or unstructured tasks require more deliberate handling.

Review questions:

* Is user-initiated work accidentally running as background?
* Is background maintenance competing with interactive work?
* Is priority manually set everywhere instead of relying on structure?
* Does detached work need an explicit priority?
* Could priority inversion come from blocking or actor contention?

Prefer preserving structure over manually assigning priorities. Use explicit priority when work is intentionally independent and the default would be misleading.

## Task-local values

Task-local values propagate through structured task relationships. Detached tasks do not automatically preserve the same logical context.

Review questions:

* Does logging, tracing, request ID, locale, or tenant context rely on task-local values?
* Does unstructured or detached work lose that context?
* Should the context be passed explicitly instead?
* Is the task-local value being used as hidden global state?

Task-local values are useful for contextual metadata, not for core data flow.

## Sendable and isolation crossing

Values crossing concurrency boundaries may need to be `Sendable`. Do not silence Sendable errors by adding unchecked conformance unless the type is truly safe to share.

Review questions:

* Is the type immutable?
* Does it contain mutable reference state?
* Is shared state protected by actor isolation or a lock?
* Is `@unchecked Sendable` documented and justified?
* Is a UI-owned reference being sent away from `MainActor`?
* Would a value snapshot be safer?

Prefer value snapshots for data crossing isolation boundaries.

Example:

```swift
struct ExportSnapshot: Sendable {
    var title: String
    var itemIDs: [ItemID]
    var options: ExportOptions
}
```

If a class must be `Sendable`, make the synchronization strategy explicit.

## MainActor.run

`MainActor.run` is useful at narrow boundaries, but it should not be the default way to repair isolation after the fact.

Prefer declaring isolation in APIs:

* mark UI-facing types `@MainActor`
* isolate state where it is owned
* move non-UI work out of main-actor types
* return values from background work and apply them on `MainActor`

Use `MainActor.run` when:

* bridging from a non-isolated legacy callback
* making a small UI update from non-UI code
* isolating a narrow boundary is clearer than changing the API

Avoid code that repeatedly bounces between background work and `MainActor.run` inside loops.

Risky pattern:

```swift
func importItems(_ items: [Item]) async {
    for item in items {
        let result = await importer.import(item)

        await MainActor.run {
            progress.append(result)
        }
    }
}
```

Prefer batching UI updates when possible:

```swift
func importItems(_ items: [Item]) async {
    var imported: [ImportResult] = []

    for item in items {
        imported.append(await importer.import(item))
    }

    await MainActor.run {
        progress.append(contentsOf: imported)
    }
}
```

## AsyncSequence and streams

Async streams can hide lifetime, buffering, and cancellation problems.

Review questions:

* Who owns the stream?
* What happens when the consumer stops iterating?
* Is the producer cancelled?
* Is buffering bounded?
* Can events arrive faster than they are consumed?
* Does the stream retain `self`?
* Is the continuation finished exactly once?

For `AsyncStream` / `AsyncThrowingStream`, define:

* buffering policy
* termination behavior
* cancellation cleanup
* producer lifetime

Avoid unbounded buffering unless the event rate is naturally small or bounded elsewhere.

## Common review recommendations

Prefer:

* structured concurrency for scoped child work
* bounded task groups for large inputs
* owned tasks for UI-triggered work that must be cancellable
* `Task.detached` only for intentionally independent work
* short actor-isolated critical sections
* batching repeated actor reads/writes
* re-checking actor state after `await`
* moving CPU-heavy work away from `MainActor`
* `Task.sleep` instead of `Thread.sleep`
* checked continuations for callback bridging
* explicit cancellation handling
* value snapshots across isolation boundaries

Avoid:

* fire-and-forget tasks without ownership
* unbounded task creation
* blocking operations inside Swift tasks
* assuming `async` means background execution
* assuming actor state is unchanged after `await`
* using actors as a replacement for every lock
* using `Task.detached` to bypass isolation diagnostics
* adding `@unchecked Sendable` to silence the compiler
* using `MainActor.run` as a broad isolation patch
* designing APIs that require callers to perform many small actor hops

## Signs to look for in Instruments

In Swift Concurrency Instrument:

* a large number of short-lived tasks
* tasks that remain alive longer than expected
* MainActor blocking
* actor contention
* repeated actor hops
* continuations that never resume
* task groups with excessive fan-out
* detached work without clear parent context

In Time Profiler:

* CPU-heavy work running on `MainActor`
* blocking calls inside async functions
* repeated retain/release around task closures
* expensive work inside actor-isolated methods
* synchronization primitives around long work

In Allocations:

* task closure churn
* captured object graphs retained by tasks
* temporary buffers created by child tasks
* stream buffering growth
* repeated continuation wrapper allocation

## Common gotchas

* A task is not a thread.
* `await` is a suspension point, not a guarantee of background execution.
* Actor isolation prevents data races, not all logical races.
* Actor methods can be reentrant at suspension points.
* Actor hops are not OS thread context switches.
* `Task {}` is unstructured even if it inherits useful context.
* `Task.detached` does not preserve structured lifetime.
* Cancellation is cooperative.
* Priority is a scheduling hint, not a correctness guarantee.
* Blocking inside async code can harm the cooperative pool.
* `nonisolated async` behavior depends on Swift version and language settings.
* In Swift 6.2+ caller-isolation mode, use `@concurrent` when an async function should intentionally leave the caller's actor.
* `@unchecked Sendable` is a responsibility, not a fix.
* `MainActor.run` should be a narrow bridge, not the main architecture.

## Output guidance

When this reference is used, include:

```markdown
## Concurrency runtime model

Explain which tasks, actors, executors, or isolation boundaries are involved.

## Suspected runtime issue

Classify the issue:
MainActor blocking / actor contention / task explosion / blocking cooperative pool / continuation misuse / cancellation gap / priority issue / Sendable boundary / Swift 6.2 isolation behavior.

## Why it matters

Tie the concurrency structure to responsiveness, throughput, memory, correctness, or resource usage.

## Recommended change

Suggest the smallest safe change that preserves structured lifetime, cancellation, priority, and isolation.

## Validation

Recommend Swift Concurrency Instrument, Time Profiler, Allocations, signposts, or targeted tests.
```

If the concern is theoretical and not connected to responsiveness, throughput, or correctness, say so directly.
