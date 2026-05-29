---

name: swift-concurrency-performance
description: Use this skill when reviewing or diagnosing Swift Concurrency code for performance, responsiveness, MainActor overuse, actor contention, task lifecycle bugs, unbounded child tasks, blocking work inside async contexts, cancellation propagation, AsyncSequence resource cleanup, continuation safety, Swift 6.2 isolation behavior, or Swift Concurrency Instruments findings.

---

# Swift Concurrency Performance

Use this skill to review Swift Concurrency code with a focus on responsiveness, resource usage, task lifetime, actor isolation, cancellation, and measurable performance behavior.

This skill is not a general async/await tutorial. Apply it when the user asks about concurrency-related performance, UI hangs, actor bottlenecks, task creation patterns, cancellation behavior, async streams, continuations, or Swift Concurrency diagnostics.

## Review Principles

Start from the smallest useful diagnosis.

Do not add concurrency blindly. Async tasks, actors, task groups, and detached work are not performance improvements by themselves. Recommend them only when there is a clear reason: latency hiding, parallelism, isolation, responsiveness, cancellation control, or lifetime management.

Prefer simple synchronous code when the work is local, fast, already on the correct isolation domain, and does not block a critical executor.

Before proposing a rewrite, classify the issue:

* UI responsiveness problem
* too much work on `MainActor`
* blocking call inside async code
* actor contention or excessive actor hops
* duplicate work caused by actor reentrancy
* too many child tasks
* missing cancellation checks
* unstructured task lifetime leak
* unsafe continuation bridge
* async stream that does not release resources
* Swift 6.2 isolation behavior keeping work on the caller’s actor

When possible, ask for evidence: Instruments trace, logs, reproducible scenario, cancellation path, affected screen, or the specific async call chain.

## References

Read these only when the task requires deeper guidance:

* `references/actor-reentrancy.md` — actor methods with `await`, cache coordination, duplicate work, state validation after suspension, in-flight tasks, or actor contention.
* `references/bounded-task-groups.md` — task groups over large collections, batch processing, dynamic child tasks, ordering, throttling, cancellation, partial failure, or concurrency limit selection.
* `references/swift-6-2-isolation.md` — Swift 6.2+, `@MainActor`, default actor isolation, `nonisolated`, `@concurrent`, Sendable boundary checks, snapshots, or CPU work that may stay on the caller’s actor.

## Runtime Rule: Suspend, Do Not Block

Swift tasks are expected to suspend at suspension points. A suspended task lets the executor make progress on other work. Blocking a thread used by Swift Concurrency can reduce throughput for unrelated tasks and can make hangs harder to diagnose.

Flag blocking patterns inside async contexts:

```swift
Thread.sleep(forTimeInterval: 0.5)
gate.wait()
queue.sync { performWork() }
let bytes = try Data(contentsOf: fileURL)
legacyClient.fetchSynchronously()
```

Prefer async or suspending alternatives:

```swift
try await Task.sleep(for: .milliseconds(500))
let bytes = try await fileLoader.read(fileURL)
let response = try await client.fetch()
```

Do not hold a lock across an `await`.

A short lock-protected critical section can be reasonable for tiny synchronous state, but long-held or highly contended locks inside async tasks are a performance smell. Do not replace every lock with an actor by default; choose the primitive that matches the lifetime and API shape.

## MainActor Responsiveness

Use `MainActor` for UI state, presentation state, and short coordination. Keep heavy computation away from it.

Watch for:

* parsing large payloads on `MainActor`
* formatting large collections on `MainActor`
* image processing on `MainActor`
* sorting/filtering large arrays on `MainActor`
* building large view models synchronously on `MainActor`
* running database or file operations from `MainActor`

Risky:

```swift
@MainActor
final class PortfolioScreenModel {
    private(set) var rows: [PortfolioRow] = []

    func reload() async throws {
        let positions = try await positionsAPI.loadPositions()
        rows = positions
            .map { PortfolioRow(position: $0) }
            .sorted { $0.displayName < $1.displayName }
    }
}
```

The network request suspends, but the mapping and sorting still happen on the main actor.

Prefer moving heavy work out of the main actor boundary:

```swift
@MainActor
final class PortfolioScreenModel {
    private(set) var rows: [PortfolioRow] = []

    func reload() async throws {
        let positions = try await positionsAPI.loadPositions()
        rows = await makeRows(from: positions)
    }
}

@concurrent
func makeRows(from positions: [Position]) async -> [PortfolioRow] {
    positions
        .map { PortfolioRow(position: $0) }
        .sorted { $0.displayName < $1.displayName }
}
```

When recommending `@concurrent`, check that captured values, parameters, and return types can safely cross isolation boundaries. Prefer `Sendable` value types. Do not use `@concurrent` to move non-Sendable UI-owned reference models away from `MainActor`.

For long CPU-heavy work, `@concurrent` only moves execution away from the caller’s actor. It does not automatically make the computation cooperative. Long computations may still need chunking, cancellation checks, yielding, or a dedicated execution mechanism.

Avoid `DispatchQueue.main.async` from code that is already isolated to `MainActor`. After suspension, a `@MainActor` function resumes on the main actor.

Use `MainActor.run` when only a small section of otherwise non-main work must update UI state.

For deeper Swift 6.2 isolation behavior, read `references/swift-6-2-isolation.md`.

## Structured Concurrency by Default

Prefer structured concurrency when the child work belongs to the caller’s scope.

Use `async let` for a small fixed set of independent operations:

```swift
async let account = accountService.currentAccount()
async let limits = limitsService.dailyLimits()
async let alerts = alertsService.activeAlerts()

let summary = try await AccountSummary(
    account: account,
    limits: limits,
    alerts: alerts
)
```

Use a task group when the amount of child work is dynamic.

Do not mechanically parallelize sequential `await`s. Sequential awaits are correct when the next operation depends on the previous result, when ordering matters, when a resource must be used serially, or when parallelism would overload a service.

Risky unstructured task:

```swift
func userDidPullToRefresh() {
    Task {
        await refreshContent()
    }
}
```

If this task belongs to the lifetime of an object, store it and cancel it when it is no longer needed:

```swift
private var refreshJob: Task<Void, Never>?

func userDidPullToRefresh() {
    refreshJob?.cancel()
    refreshJob = Task {
        await refreshContent()
    }
}

func close() {
    refreshJob?.cancel()
    refreshJob = nil
}
```

Use `Task.detached` only when the work is intentionally independent from the caller’s actor, cancellation, priority, and task-local context. Treat it as an escape hatch, not as a default way to “move work to background”.

## Bounded Child Tasks

A task group can still create too much work.

Risky pattern:

```swift
try await withThrowingTaskGroup(of: ReceiptText.self) { group in
    for scan in scans {
        group.addTask {
            try await recognizeText(in: scan)
        }
    }

    for try await text in group {
        recognizedTexts.append(text)
    }
}
```

This may create excessive memory pressure, scheduling overhead, or service load when `scans` is large.

For large input collections, recommend bounded concurrency: start only a limited number of child tasks, then add a new task each time one completes.

Use bounded concurrency for batch uploads, downloads, image processing, OCR, parsing, indexing, compression, and other large collections.

For full bounded-concurrency patterns, ordering, cancellation, partial failure, and limit selection, read `references/bounded-task-groups.md`.

## Cancellation

Cancellation is cooperative state. The task must observe it.

Add cancellation checks before expensive work and inside long loops:

```swift
func indexDocuments(_ documents: [Document]) async throws -> SearchIndex {
    var builder = SearchIndexBuilder()

    for document in documents {
        try Task.checkCancellation()
        try await builder.add(document)
    }

    return builder.build()
}
```

Do not hide cancellation accidentally.

Risky:

```swift
func cachedPreview() async -> Preview? {
    try? await previewService.makePreview()
}
```

This collapses cancellation, failure, and absence of value into the same result.

Prefer preserving cancellation when the caller needs to react:

```swift
func cachedPreview() async throws -> Preview {
    try await previewService.makePreview()
}
```

Use `withTaskCancellationHandler` when bridging an API that exposes a specific cancellation handle. Cancel that operation, not a global shared service.

If an operation is expensive and cannot be cancelled internally, make that limitation explicit in the review.

## Actor Isolation, Reentrancy, and Duplicate Work

Actors protect isolated state from data races. They do not make an entire async method atomic.

Every `await` inside an actor-isolated method is a reentrancy boundary. While the method is suspended, another call can enter the actor and observe or mutate actor state.

Watch for:

* cache miss followed by `await`
* state validation before `await`
* mutation after `await`
* duplicate network, decoding, parsing, or indexing work
* actor methods that combine coordination and expensive work
* hot actor methods called once per item in a large loop

Prefer:

* committing actor state before suspension when correct
* re-validating actor state after suspension when needed
* tracking in-flight work when deduplication matters
* batching hot actor calls
* moving pure computation to `nonisolated` code
* reducing the isolated section instead of making the whole workflow actor-isolated

Use `nonisolated` only when a method does not read or mutate actor-isolated state.

Do not split state into many actors just to increase parallelism unless the split matches the consistency model. Too many actors can create excessive hops and make correctness harder to reason about.

For detailed reentrancy examples, in-flight task patterns, state validation, and actor contention guidance, read `references/actor-reentrancy.md`.

## Swift 6.2 and Explicit Isolation

In Swift 6.2 and later, do not assume that `async` means “background”.

Check the isolation context:

* Is the caller on `MainActor`?
* Is the enclosing type actor-isolated?
* Does the function inherit the caller’s actor?
* Does the heavy work need to explicitly switch off the caller’s actor?
* Are values crossing the boundary `Sendable`?

Use `nonisolated` for code that does not need actor-isolated state.

Use `@concurrent` when an async function should explicitly switch off the caller’s actor so that actor can continue making progress.

Before recommending `@concurrent`, verify that parameters, captures, and return values can safely cross isolation boundaries. Prefer value types that conform to `Sendable`.

Do not add `@concurrent` everywhere. Use it when there is a concrete isolation or responsiveness problem.

For detailed Swift 6.2 behavior, Sendable boundary checks, snapshots, and refactoring patterns, read `references/swift-6-2-isolation.md`.

## Blocking Legacy APIs

Do not run meaningful blocking work directly inside async tasks.

Risky:

```swift
func loadArchive() async throws -> Archive {
    try archiveReader.readSynchronously()
}
```

Prefer a native async API when one exists.

If the API is inherently blocking, run it outside the Swift Concurrency cooperative pool and bridge the result back:

```swift
func loadArchive() async throws -> Archive {
    try await withCheckedThrowingContinuation { continuation in
        archiveQueue.async {
            do {
                let archive = try archiveReader.readSynchronously()
                continuation.resume(returning: archive)
            } catch {
                continuation.resume(throwing: error)
            }
        }
    }
}
```

If the legacy API provides a cancellation token, handle, or operation object, pair the bridge with cancellation handling. Cancel the specific operation, not a shared service.

Use this pattern only for truly blocking APIs. Do not wrap already-async APIs in queues.

## Continuation Safety

Prefer checked continuations by default.

For `withCheckedContinuation` and `withCheckedThrowingContinuation`, verify that every possible path resumes exactly once.

Review:

* success path
* failure path
* timeout path
* cancellation path
* early return path
* delegate callback path
* invalid input path

Risky:

```swift
func exportReport() async throws -> ExportedReport {
    try await withCheckedThrowingContinuation { continuation in
        exporter.start { output, failure in
            if let output {
                continuation.resume(returning: output)
            }

            if let failure {
                continuation.resume(throwing: failure)
            }
        }
    }
}
```

If both values are present or both are nil, this may resume incorrectly.

Prefer a single explicit result path:

```swift
func exportReport() async throws -> ExportedReport {
    try await withCheckedThrowingContinuation { continuation in
        exporter.start { result in
            switch result {
            case .finished(let report):
                continuation.resume(returning: report)
            case .failed(let error):
                continuation.resume(throwing: error)
            }
        }
    }
}
```

Use unsafe continuations only when profiling shows checked continuation overhead matters.

## AsyncSequence and AsyncStream

When reviewing async streams, check lifetime, cancellation, buffering, and cleanup.

Risky:

```swift
func paymentEvents() -> AsyncStream<PaymentEvent> {
    AsyncStream { continuation in
        paymentMonitor.onEvent = { event in
            continuation.yield(event)
        }

        paymentMonitor.start()
    }
}
```

The producer may continue running after the consumer stops.

Prefer explicit termination cleanup:

```swift
func paymentEvents() -> AsyncStream<PaymentEvent> {
    AsyncStream { continuation in
        paymentMonitor.onEvent = { event in
            continuation.yield(event)
        }

        continuation.onTermination = { _ in
            paymentMonitor.stop()
        }

        paymentMonitor.start()
    }
}
```

If the producer object is created inside the stream builder, ensure it remains alive for the lifetime of the stream and is released on termination.

For high-frequency streams, check buffering behavior. Avoid unbounded memory growth when the producer is faster than the consumer.

Inside long-running `for await` loops, check cancellation when doing expensive per-element work:

```swift
for await update in updates {
    try Task.checkCancellation()
    await apply(update)
}
```

## Diagnostics

Use Instruments when code review alone is insufficient.

Look for:

* long work on `MainActor`
* actor queues with growing wait time
* high number of created or alive tasks
* tasks that survive longer than their owner
* blocked cooperative threads
* continuations that never resume
* async streams that keep producers alive
* excessive actor hops on hot paths

Map symptoms to likely causes:

* UI stall → heavy work on `MainActor` or blocking call on the main path
* actor queue buildup → hot actor, long isolated section, or chatty actor API
* duplicate network work → actor reentrancy without in-flight tracking
* memory spike → unbounded task group or unbounded stream buffering
* ignored navigation cancellation → task not stored, not cancelled, or cancellation swallowed
* stuck task → continuation not resumed or stream never terminates
* low throughput with busy threads → blocking legacy API, semaphore wait, sync file I/O, or lock contention

When the user provides a trace, connect every recommendation to an observable symptom.

## Code Review Checklist

Before finalizing a recommendation, check:

* [ ] Is concurrency being added for a clear reason?
* [ ] Is `MainActor` limited to UI state and short coordination?
* [ ] Is CPU-heavy work outside main-actor isolation?
* [ ] Are values crossing isolation boundaries safe to send?
* [ ] Are blocking calls absent from async contexts?
* [ ] Are large task groups bounded?
* [ ] Are sequential awaits only parallelized when independence is clear?
* [ ] Are unstructured tasks stored and cancelled when their lifetime is owned?
* [ ] Is `Task.detached` used only as an explicit escape hatch?
* [ ] Does cancellation propagate instead of being swallowed?
* [ ] Are long loops and expensive pipelines cancellation-aware?
* [ ] Are actor methods free from unnecessary long isolated work?
* [ ] Are actor hops batched on hot paths?
* [ ] Is actor reentrancy considered after every `await`?
* [ ] Is duplicate work prevented when actor methods suspend during cache misses?
* [ ] Are continuations resumed exactly once?
* [ ] Do async streams clean up producers on termination?
* [ ] Is Swift 6.2 isolation behavior considered?
* [ ] Is the recommendation validated by measurement when performance impact is uncertain?

## Output Format

When reviewing code, respond with:

```markdown
## Finding

Describe the likely concurrency performance or lifecycle issue.

## Why it matters

Explain the impact on responsiveness, throughput, memory, cancellation, actor contention, or correctness.

## Evidence

Point to the code pattern, trace symptom, missing cancellation path, actor hop pattern, or lifecycle mismatch.

## Recommended change

Give the smallest safe change first. Avoid broad rewrites unless the design itself causes the issue.

## Validation

Explain how to verify the result with Instruments, cancellation tests, logs, UI behavior, memory usage, or production metrics.
```

Prefer practical fixes over theoretical rewrites. Do not recommend actors, detached tasks, task groups, or `@concurrent` by default. Explain why the chosen primitive matches the lifetime, isolation, cancellation, and performance requirements.
