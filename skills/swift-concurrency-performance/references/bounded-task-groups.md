# Bounded Task Groups

Use this reference when reviewing `withTaskGroup`, `withThrowingTaskGroup`, large loops that create child tasks, batch processing, uploads, downloads, media processing, OCR, indexing, parsing, or any code that starts one task per input item.

## Core Rule

A task group gives structured concurrency for a dynamic number of child tasks. It does not automatically make the amount of concurrency safe.

Creating one child task per item is fine for small inputs. For large or untrusted input sizes, it can create excessive work:

* too many live tasks
* memory pressure
* scheduler overhead
* too many network requests
* service rate-limit failures
* CPU contention
* battery drain
* worse responsiveness

When input size can be large, use bounded concurrency.

## When Not to Parallelize

Do not replace sequential awaits mechanically.

Sequential awaits are correct when:

* each step depends on the previous result
* ordering is required
* the resource is intentionally serial
* parallel work would overload a service
* the work is small enough that task overhead dominates
* the caller needs predictable memory usage
* the operation is already internally concurrent

Risky rewrite:

```swift
async let session = createSession()
async let token = refreshToken()
async let data = fetchData(using: token)
```

This is wrong if `fetchData` needs the refreshed token.

Prefer:

```swift
let session = try await createSession()
let token = try await refreshToken(for: session)
let data = try await fetchData(using: token)
```

Concurrency is useful only when the operations are actually independent.

## `async let` vs Task Group

Use `async let` for a small fixed set of independent operations:

```swift
async let profile = profileService.load()
async let preferences = preferencesService.load()
async let permissions = permissionsService.load()

return try await UserContext(
    profile: profile,
    preferences: preferences,
    permissions: permissions
)
```

Use a task group when the number of child operations is dynamic:

```swift
try await withThrowingTaskGroup(of: ImportedAsset.self) { group in
    for descriptor in descriptors {
        group.addTask {
            try await importAsset(descriptor)
        }
    }

    var imported: [ImportedAsset] = []

    for try await asset in group {
        imported.append(asset)
    }

    return imported
}
```

This is acceptable only when `descriptors` is known to be small or externally bounded.

## Risky Unbounded Pattern

```swift
func transcodeAll(_ clips: [VideoClip]) async throws -> [EncodedClip] {
    try await withThrowingTaskGroup(of: EncodedClip.self) { group in
        for clip in clips {
            group.addTask {
                try await encoder.encode(clip)
            }
        }

        var output: [EncodedClip] = []

        for try await encoded in group {
            output.append(encoded)
        }

        return output
    }
}
```

If `clips` contains hundreds of items, this can start too much work.

## Bounded Concurrency Pattern

Start a limited number of child tasks. Each time one child completes, add one more.

```swift
func transcodeAll(
    _ clips: [VideoClip],
    maxConcurrentJobs: Int = 3
) async throws -> [EncodedClip] {
    precondition(maxConcurrentJobs > 0)

    var iterator = clips.makeIterator()
    var encodedClips: [EncodedClip] = []

    try await withThrowingTaskGroup(of: EncodedClip.self) { group in
        for _ in 0..<maxConcurrentJobs {
            guard let clip = iterator.next() else { break }

            group.addTask {
                try await encoder.encode(clip)
            }
        }

        while let encoded = try await group.next() {
            encodedClips.append(encoded)

            if let nextClip = iterator.next() {
                group.addTask {
                    try await encoder.encode(nextClip)
                }
            }
        }
    }

    return encodedClips
}
```

This keeps at most `maxConcurrentJobs` child tasks active at once.

## Preserving Input Order

Task group results arrive in completion order, not input order.

If output order matters, include the index in the child task result:

```swift
func renderPages(
    _ pages: [PDFPage],
    maxConcurrentJobs: Int = 4
) async throws -> [PageImage] {
    precondition(maxConcurrentJobs > 0)

    var nextIndex = 0
    var results = Array<PageImage?>(repeating: nil, count: pages.count)

    try await withThrowingTaskGroup(of: (Int, PageImage).self) { group in
        func addNextTaskIfNeeded() {
            guard nextIndex < pages.count else { return }

            let index = nextIndex
            let page = pages[index]
            nextIndex += 1

            group.addTask {
                let image = try await render(page)
                return (index, image)
            }
        }

        for _ in 0..<maxConcurrentJobs {
            addNextTaskIfNeeded()
        }

        while let (index, image) = try await group.next() {
            results[index] = image
            addNextTaskIfNeeded()
        }
    }

    return results.compactMap { $0 }
}
```

If missing results would be a bug, validate before returning:

```swift
guard results.allSatisfy({ $0 != nil }) else {
    throw RenderError.missingPage
}
```

## Cancellation

Task groups are structured, but child tasks still need cooperative cancellation inside expensive work.

Review child task bodies:

```swift
group.addTask {
    try Task.checkCancellation()
    let decoded = try await decoder.decode(file)
    try Task.checkCancellation()
    return try await optimizer.optimize(decoded)
}
```

If child work loops internally, cancellation checks should be inside the loop.

```swift
func extractFrames(from video: VideoFile) async throws -> [Frame] {
    var frames: [Frame] = []

    for timestamp in video.timestamps {
        try Task.checkCancellation()
        frames.append(try await video.frame(at: timestamp))
    }

    return frames
}
```

## Choosing the Limit

There is no universal concurrency limit.

Consider:

* CPU count
* memory footprint per task
* API rate limits
* network behavior
* server-side limits
* priority of the user action
* battery impact
* whether the task is CPU-bound or I/O-bound
* whether the downstream dependency already throttles internally

Typical starting points:

* CPU-heavy image/video work: small limit
* network requests to the same backend: respect API and server limits
* local parsing of many small files: measure
* user-visible work: prefer responsiveness over maximum throughput

Do not hard-code a magic number without a reason. A small default plus measurement is usually better than unbounded concurrency.

## Error Behavior

In a throwing task group, the first thrown error observed by the parent can cause remaining work to be cancelled when exiting the group.

Review whether that behavior is desired.

Use throwing groups when one failure should fail the whole operation.

Use non-throwing groups with per-item `Result` when partial success is valid:

```swift
await withTaskGroup(of: ImportResult.self) { group in
    for file in files {
        group.addTask {
            do {
                return .success(try await importFile(file))
            } catch {
                return .failure(file, error)
            }
        }
    }

    for await result in group {
        report.add(result)
    }
}
```

## Priority

Prefer structured child tasks so priority and cancellation follow the parent operation.

Avoid replacing task-group children with `Task.detached` unless the work is intentionally independent.

If child work is lower priority than the parent, make that explicit and explain why.

## Diagnostics

Use Instruments or logging when task-group behavior is unclear.

Look for:

* high number of live tasks
* memory spikes during batch processing
* slow UI while task group is active
* backend rate-limit errors
* child tasks that continue after the screen is gone
* no throughput improvement despite more tasks
* many tasks blocked on the same actor or dependency

Map symptoms to code:

* many live tasks → unbounded group
* memory spike → too many active child tasks or large captured values
* rate-limit failures → concurrency limit too high
* no speedup → bottleneck is serialized elsewhere
* UI stalls → child work or result aggregation hitting MainActor

## Review Checklist

* [ ] Is the input size bounded?
* [ ] Is each child operation independent?
* [ ] Is output order important?
* [ ] Does the code create one task per item?
* [ ] Should concurrency be limited?
* [ ] Is cancellation checked inside expensive child work?
* [ ] Are errors all-or-nothing or per-item?
* [ ] Are large captures avoided in child task closures?
* [ ] Does aggregation happen off MainActor when heavy?
* [ ] Is the selected concurrency limit justified?
* [ ] Is the performance benefit measured?
