# Actor Reentrancy

Use this reference when reviewing actor methods that contain `await`, coordinate cached work, mutate state around suspension points, or show duplicate async work under load.

## Core Rule

Actor isolation prevents data races on actor-isolated state. It does not make an entire `async` actor method atomic.

Every `await` inside an actor-isolated method is a reentrancy boundary. While the method is suspended, the actor may process other work. When the original method resumes, actor state may no longer match the assumptions made before suspension.

Review actor methods as sequences of synchronous isolated regions separated by suspension points.

## What to Look For

Flag actor methods where code:

* reads actor state
* reaches an `await`
* later assumes the previously read state is still valid

Common symptoms:

* duplicate network requests
* duplicate decoding or parsing
* stale validation
* quota or budget going negative
* state machine transitions happening out of order
* cache writes overwriting newer state
* inconsistent metrics or counters

## Cache Miss Duplicate Work

Risky:

```swift
protocol ResourceLoading: Sendable {
    func loadResource(for key: ResourceKey) async throws -> Resource
}

actor ResourceCatalog {
    private var cache: [ResourceKey: Resource] = [:]
    private let loader: any ResourceLoading

    init(loader: any ResourceLoading) {
        self.loader = loader
    }

    func resource(for key: ResourceKey) async throws -> Resource {
        if let cached = cache[key] {
            return cached
        }

        let resource = try await loader.loadResource(for: key)
        cache[key] = resource
        return resource
    }
}
```

This has no data race, but it has a reentrancy window.

If two tasks request the same missing key:

1. Task A enters the actor and sees a cache miss.
2. Task A suspends while loading.
3. Task B enters the actor and sees the same cache miss.
4. Task B starts a second load.
5. Both tasks eventually store a value.

Actor isolation protected the dictionary, but it did not deduplicate the async operation.

## In-Flight Task Pattern

Use an in-flight table when the actor must deduplicate work across suspension points.

```swift
protocol ResourceLoading: Sendable {
    func loadResource(for key: ResourceKey) async throws -> Resource
}

actor ResourceCatalog {
    private var cache: [ResourceKey: Resource] = [:]
    private var inFlight: [ResourceKey: Task<Resource, Error>] = [:]
    private let loader: any ResourceLoading

    init(loader: any ResourceLoading) {
        self.loader = loader
    }

    func resource(for key: ResourceKey) async throws -> Resource {
        if let cached = cache[key] {
            return cached
        }

        if let existing = inFlight[key] {
            return try await existing.value
        }

        let loader = self.loader
        let task = Task<Resource, Error> {
            try await loader.loadResource(for: key)
        }

        inFlight[key] = task

        do {
            let resource = try await task.value
            cache[key] = resource
            inFlight[key] = nil
            return resource
        } catch {
            inFlight[key] = nil
            throw error
        }
    }
}
```

This still suspends, but the important coordination state is written before suspension. Later callers find `inFlight[key]` and await the same task instead of starting duplicate work.

## Cleanup and Cancellation Notes

When reviewing an in-flight task pattern, check:

* Is the in-flight entry removed on success?
* Is it removed on failure?
* What happens if all waiters are cancelled?
* Should cancellation of one waiter cancel the shared operation?
* Is the shared task intentionally allowed to complete for future callers?
* Is the task priority appropriate for the work?

For shared cache loads, cancelling the underlying operation when one caller cancels is often wrong, because other callers may still need the result. For owner-scoped work, cancellation may need to cancel the underlying task.

Do not assume one cancellation policy fits every actor.

## State Validation Across Await

Risky:

```swift
actor DownloadQuota {
    private var remainingMegabytes: Int

    init(remainingMegabytes: Int) {
        self.remainingMegabytes = remainingMegabytes
    }

    func approveDownload(size: Int) async throws {
        guard remainingMegabytes >= size else {
            throw QuotaError.notEnoughCapacity
        }

        await audit.logApproval(size)

        remainingMegabytes -= size
    }
}
```

The validation happens before suspension. Another task may consume quota while `audit.logApproval` is running.

Prefer committing state before suspension when the business rule allows it:

```swift
actor DownloadQuota {
    private var remainingMegabytes: Int

    init(remainingMegabytes: Int) {
        self.remainingMegabytes = remainingMegabytes
    }

    func approveDownload(size: Int) async throws {
        guard remainingMegabytes >= size else {
            throw QuotaError.notEnoughCapacity
        }

        remainingMegabytes -= size
        await audit.logApproval(size)
    }
}
```

If the external operation must happen before the state mutation, re-check after suspension:

```swift
actor DownloadQuota {
    private var remainingMegabytes: Int

    init(remainingMegabytes: Int) {
        self.remainingMegabytes = remainingMegabytes
    }

    func approveAfterVerification(size: Int) async throws {
        guard remainingMegabytes >= size else {
            throw QuotaError.notEnoughCapacity
        }

        try await verifier.verifyDownload(size)

        guard remainingMegabytes >= size else {
            throw QuotaError.notEnoughCapacity
        }

        remainingMegabytes -= size
    }
}
```

## Transaction-Like Actor Methods

If the code needs transaction-like behavior, avoid placing suspension points inside the critical state transition.

Prefer this shape:

```swift
actor SyncStateMachine {
    private var state: SyncState = .idle

    func beginSync() throws -> SyncToken {
        guard state == .idle else {
            throw SyncError.alreadyRunning
        }

        let token = SyncToken()
        state = .running(token)
        return token
    }

    func finishSync(token: SyncToken, result: Result<Void, Error>) {
        guard state == .running(token) else {
            return
        }

        state = .finished(result)
    }
}
```

Then perform the async work outside the actor transaction:

```swift
let token = try await stateMachine.beginSync()

do {
    try await syncEngine.run()
    await stateMachine.finishSync(token: token, result: .success(()))
} catch {
    await stateMachine.finishSync(token: token, result: .failure(error))
}
```

This makes the actor responsible for state transitions, not for the whole async operation.

## Actor Contention

Reentrancy prevents one suspended method from blocking the actor forever, but actors can still become bottlenecks.

Look for:

* many tasks awaiting the same actor
* hot actor methods called once per item
* expensive synchronous work inside actor methods
* repeated actor hops in tight loops
* actor used as a logging or metrics funnel
* actor protecting unrelated pieces of state

Risky:

```swift
for event in events {
    await analyticsStore.append(event)
}
```

Prefer batching:

```swift
await analyticsStore.append(contentsOf: events)
```

Risky:

```swift
actor DocumentStore {
    private var documents: [DocumentID: Document] = [:]

    func tokenize(_ document: Document) -> [Token] {
        Tokenizer.tokenize(document.text)
    }
}
```

If `tokenize` does not need actor state, it should not be actor-isolated:

```swift
actor DocumentStore {
    private var documents: [DocumentID: Document] = [:]

    nonisolated func tokenize(_ document: Document) -> [Token] {
        Tokenizer.tokenize(document.text)
    }
}
```

Use `nonisolated` only when the method does not read or mutate actor-isolated state.

## Review Checklist

Use this checklist for actor methods:

* [ ] Does the method contain `await`?
* [ ] Is actor state read before `await` and trusted after `await`?
* [ ] Can another caller enter during suspension and change the state?
* [ ] Could two callers start the same expensive work?
* [ ] Should in-flight work be tracked?
* [ ] Is cleanup handled on success and failure?
* [ ] Is cancellation policy explicit?
* [ ] Can state be committed before suspension?
* [ ] If not, is state re-validated after suspension?
* [ ] Is the actor doing expensive work that does not need isolation?
* [ ] Are hot actor calls batched?
* [ ] Would a smaller isolated section or a synchronous primitive fit better?
