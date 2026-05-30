# Optimistic Updates

Use this reference when reviewing actions that can update the UI before server confirmation because the expected result is predictable, low-risk, and reversible.

This reference is about perceived responsiveness for reversible user actions. For loading states, read `references/loading-states.md`. For staged real-content rendering, read `references/progressive-rendering.md`. For financial, legal, destructive, irreversible, or trust-sensitive flows, read `references/high-stakes-actions.md`. For runtime validation and production metrics, read `references/validation-and-testing.md`.

## Core Idea

An optimistic update applies the expected UI change immediately, sends the backend request afterward, and reconciles the UI when the backend responds.

It improves perceived latency because the user sees feedback without waiting for a network round trip. It does not make the backend operation faster.

Use optimistic updates only when the local prediction is safe enough and the app has a clear recovery path.

## What the Agent Can and Cannot Prove

The agent can inspect and improve:

* actions that wait for backend confirmation before changing simple local UI
* missing pending or syncing state
* missing rollback path
* missing failure presentation
* duplicate tap behavior
* local state that can become inconsistent with server state
* missing retry or reconciliation path
* optimistic updates used for unsafe actions
* state models that cannot represent pending, synced, and failed sync states

The agent cannot prove from code alone:

* that the failure rate is low enough
* that the backend conflict model is safe
* that users will understand the pending state
* that optimistic UI is acceptable for the product domain
* that rollback behavior feels good in real use

Do not claim:

* “This is always safe.”
* “This removes the need for server confirmation.”
* “This makes the operation complete instantly.”

Prefer:

* “This moves visual feedback out of the network round trip.”
* “This requires rollback and failure handling.”
* “This is appropriate only if the action is reversible and low-risk.”
* “Validate with failure simulation and repeated interaction tests.”

## Good Candidates

Optimistic updates are usually good candidates when the user action is:

* reversible
* local or user-specific
* low-risk
* easy to retry
* easy to roll back
* unlikely to fail
* unlikely to conflict with another user or device
* not legally, financially, medically, or security significant

Examples:

* save or unsave an article
* mute or unmute a topic
* mark an item as read
* dismiss a local tip
* follow or unfollow a non-critical feed
* change a local display preference
* reorder local draft content
* update a draft before sync
* archive a low-risk notification with undo

## Risky Candidates

Avoid optimistic success when the action is:

* financial
* legal
* destructive
* irreversible
* security-sensitive
* identity-related
* server-authoritative
* likely to fail validation
* dependent on inventory, balance, eligibility, or permissions
* difficult to roll back
* likely to conflict across devices or users

Examples:

* sending money
* deleting an account
* confirming a purchase
* accepting legal terms
* changing security settings
* submitting medical information
* verifying identity
* booking a scarce resource
* applying a server-calculated discount
* showing final eligibility before server confirmation

For these flows, prefer explicit progress and final confirmation. Read `references/high-stakes-actions.md`.

## Basic Optimistic Update Pattern

Risky non-optimistic interaction:

```swift id="b0d8nd"
@MainActor
func toggleSaved(id: Article.ID) async {
    do {
        let isSaved = try await articleAPI.toggleSaved(id: id)
        articles.update(id) { article in
            article.isSaved = isSaved
        }
    } catch {
        errorPresenter.show("Could not update article")
    }
}
```

The UI waits for the backend before reflecting the tap.

Prefer optimistic feedback when the action is reversible:

```swift id="85uj3c"
@MainActor
func toggleSaved(id: Article.ID) {
    let previous = articles[id].isSaved
    let next = !previous

    articles.update(id) { article in
        article.isSaved = next
        article.syncState = .syncing
    }

    syncTasks[id]?.cancel()
    syncTasks[id] = Task {
        do {
            try await articleAPI.setSaved(next, for: id)

            await MainActor.run {
                articles.update(id) { article in
                    article.syncState = .synced
                }
                syncTasks[id] = nil
            }
        } catch {
            await MainActor.run {
                articles.update(id) { article in
                    article.isSaved = previous
                    article.syncState = .failed
                }
                errorPresenter.show("Could not update article")
                syncTasks[id] = nil
            }
        }
    }
}
```

This gives immediate visual feedback and still reconciles with the backend.

Use this shape only when the task is intentionally owned by the model or screen. If the action already runs inside an async parent scope, prefer structured concurrency.

## State Model

Optimistic UI needs more than a final boolean.

Risky:

```swift id="158hi7"
struct Article {
    var isSaved: Bool
}
```

This cannot show whether the local value is synced, pending, or failed.

Prefer explicit sync state:

```swift id="1fo15n"
struct Article {
    var id: Article.ID
    var isSaved: Bool
    var syncState: SyncState
}

enum SyncState {
    case synced
    case syncing
    case failed
}
```

For retry behavior, include enough information to retry or revert:

```swift id="y91d89"
enum SyncState<Value> {
    case synced
    case syncing(previous: Value)
    case failed(previous: Value, error: Error)
}
```

Keep the model as simple as the product needs. Do not add complex sync state for trivial local-only actions.

## Pending State

An optimistic update should not pretend the server already confirmed the result.

Show a pending state when confirmation matters:

* subtle spinner near the changed control
* disabled repeated tap while syncing
* temporary “Saving…” label
* inline sync indicator
* local pending badge
* queued action state
* undo affordance when appropriate

Risky:

```swift id="x8220h"
article.isSaved.toggle()
```

The UI now looks final even though the backend may fail.

Prefer:

```swift id="n6g2gw"
articles.update(id) { article in
    article.isSaved = next
    article.syncState = .syncing
}
```

The UI can show that the value is local and not yet confirmed.

Do not overuse pending indicators for very low-risk actions where a visible pending state would create more noise than value. Make the trade-off explicit.

## Rollback

Every optimistic update needs a rollback or reconciliation strategy.

Rollback is usually appropriate when:

* the failed action has no local value without server confirmation
* the previous state is still valid
* the change is easy to reverse
* keeping the optimistic value would mislead the user

Example:

```swift id="rvbuh8"
func rollbackSavedState(id: Article.ID, previous: Bool) {
    articles.update(id) { article in
        article.isSaved = previous
        article.syncState = .failed
    }

    errorPresenter.show("Could not save. Restored the previous state.")
}
```

Rollback may be wrong when:

* the user has made additional changes after the original optimistic update
* the server accepted a different canonical value
* the same value changed on another device
* the app supports offline-first behavior
* the action should remain queued for later sync

In these cases, use reconciliation instead of simple rollback.

## Conflict Handling

Optimistic UI can conflict with server state.

Review what happens when:

* the server rejects the change
* the server returns a different canonical value
* another device changed the same item
* the user taps repeatedly before the first request completes
* the app goes offline after the optimistic update
* the app terminates before confirmation
* a later request completes before an earlier one

Risky:

```swift id="hixr9p"
articles.update(id) { article in
    article.isSaved = next
}

Task {
    try? await articleAPI.setSaved(next, for: id)
}
```

This ignores failure, ordering, and reconciliation.

Prefer tracking enough metadata to reconcile:

```swift id="4fsgvk"
struct PendingMutation<Value> {
    let localValue: Value
    let previousValue: Value
    let mutationID: UUID
    let createdAt: Date
}
```

Use a mutation identifier or version when requests can complete out of order.

```swift id="axb1dm"
let mutationID = UUID()

articles.update(id) { article in
    article.isSaved = next
    article.pendingMutationID = mutationID
    article.syncState = .syncing
}

syncTasks[id] = Task {
    do {
        let confirmed = try await articleAPI.setSaved(next, for: id)

        await MainActor.run {
            articles.update(id) { article in
                guard article.pendingMutationID == mutationID else {
                    return
                }

                article.isSaved = confirmed.isSaved
                article.pendingMutationID = nil
                article.syncState = .synced
            }
        }
    } catch {
        await MainActor.run {
            articles.update(id) { article in
                guard article.pendingMutationID == mutationID else {
                    return
                }

                article.isSaved = previous
                article.pendingMutationID = nil
                article.syncState = .failed
            }
        }
    }
}
```

This avoids an older request overwriting a newer local action.

## Repeated Taps

Repeated taps are common in optimistic UI.

Choose one policy:

* disable the control while syncing
* coalesce multiple taps into the latest desired value
* cancel the previous request and send the latest value
* queue mutations in order
* allow immediate toggles but reconcile with mutation IDs
* provide undo instead of repeated toggles

The right policy depends on the action.

For a simple toggle, latest-value-wins is often reasonable:

```swift id="qwdogr"
func toggleSaved(id: Article.ID) {
    let next = !articles[id].isSaved

    syncTasks[id]?.cancel()
    applyOptimisticSavedState(id: id, isSaved: next)
    syncTasks[id] = syncSavedState(id: id, isSaved: next)
}
```

For actions where every tap is meaningful, do not coalesce without product approval.

## Retry

Failure recovery should be explicit.

Options:

* automatic retry for transient network failures
* manual retry from an inline failed state
* undo to previous state
* keep queued change for offline sync
* show error and restore previous state
* fetch canonical server state

Example inline retry:

```swift id="h5gxv3"
func retrySavedState(id: Article.ID) {
    guard case .failed(let previous, _) = articles[id].syncState else {
        return
    }

    let desired = articles[id].isSaved

    articles.update(id) { article in
        article.syncState = .syncing(previous: previous)
    }

    syncTasks[id] = Task {
        await syncSavedState(id: id, desired: desired, previous: previous)
    }
}
```

Do not automatically retry forever. Use bounded retries and avoid creating hidden background work that users cannot understand or control.

## Offline and App Restart

If optimistic state can outlive the current screen, decide whether to persist it.

Ask:

* Should pending mutations survive app restart?
* Should failed mutations remain visible?
* Should the app retry when connectivity returns?
* Should local state be replaced by server state on next launch?
* Should the user be warned before leaving with unsynced changes?
* Does the backend provide idempotency keys or mutation IDs?

For local-only draft flows, persisting pending state can be correct.

For server-authoritative state, blindly persisting optimistic values can mislead users.

## Cancellation

Cancellation policy must be explicit.

When the screen disappears, should the sync request continue?

Possible answers:

* Cancel it because the action belongs only to the screen.
* Continue it because the user already changed account-level state.
* Queue it because the app supports offline-first sync.
* Revert it because the action is not meaningful without immediate confirmation.

Do not assume that screen disappearance always cancels optimistic work.

Example:

```swift id="l9cg01"
func close() {
    for task in syncTasks.values {
        task.cancel()
    }
    syncTasks.removeAll()
}
```

This is correct only when cancellation matches product semantics. For account-level preferences, cancelling on disappearance may be wrong.

## Local Reconciliation

When the server responds, prefer applying the server’s canonical result over assuming the local value is final.

```swift id="uz1o0p"
let response = try await articleAPI.setSaved(next, for: id)

articles.update(id) { article in
    article.isSaved = response.isSaved
    article.syncState = .synced
}
```

This matters when the server normalizes, rejects, deduplicates, or modifies the requested value.

If the server returns only success without canonical state, document that the client assumes the requested value was accepted.

## High-Stakes Boundary

Do not use optimistic final success for high-stakes actions.

Risky:

```swift id="b6w2e1"
func confirmPayment() {
    state = .paid

    Task {
        try await paymentService.confirm()
    }
}
```

This can mislead the user.

Prefer explicit progress and final confirmation after the server responds:

```swift id="gxcaez"
func confirmPayment() async {
    state = .submitting

    do {
        let receipt = try await paymentService.confirm()
        state = .confirmed(receipt)
    } catch {
        state = .failed(error)
    }
}
```

For financial, legal, destructive, irreversible, or security-sensitive operations, read `references/high-stakes-actions.md`.

## Implementation Checklist

When proposing optimistic updates, check:

* [ ] Is the action reversible?
* [ ] Is the action low-risk?
* [ ] Is the local result predictable?
* [ ] Is the previous value captured before mutation?
* [ ] Is a pending or syncing state represented when needed?
* [ ] Is rollback implemented?
* [ ] Is failure visible to the user?
* [ ] Is retry behavior defined?
* [ ] Are repeated taps handled intentionally?
* [ ] Are stale or out-of-order responses ignored or reconciled?
* [ ] Does the server return canonical state?
* [ ] Is conflict handling defined?
* [ ] Is offline behavior defined when relevant?
* [ ] Is app restart behavior defined when relevant?
* [ ] Is cancellation policy explicit?
* [ ] Are high-stakes actions excluded?
* [ ] Is the improvement validated with failure simulation and interaction tests?

## Validation

Validate optimistic updates with both success and failure paths.

Recommended validation:

* normal success path
* simulated network failure
* slow network response
* repeated tap during pending state
* screen disappearance during sync
* app restart with pending mutation if persistence is supported
* server rejection
* server returns a different canonical value
* offline mode if supported
* out-of-order response if multiple mutations can overlap

Do not claim that optimistic UI is safe without testing failure and conflict paths.

Correct phrasing:

* “This gives immediate feedback while the backend request continues.”
* “This requires rollback and pending-state handling.”
* “Validate success, failure, repeated taps, and out-of-order responses.”

Avoid:

* “This makes the operation instant.”
* “This removes the need for backend confirmation.”
* “This is safe because the UI can always roll back.”

For broader validation strategy, read `references/validation-and-testing.md`.

## Review Output Guidance

When using this reference, explain:

```markdown id="yep8kf"
## Finding

The UI waits for server confirmation before reflecting a reversible user action.

## User impact

The interaction feels slower because the user does not receive immediate feedback.

## Recommended change

Apply the expected local state immediately, mark it as pending, send the backend request, then reconcile with the server response. Roll back or show a failed sync state if the request fails.

## Safety checks

Confirm that the action is low-risk, reversible, and not financially, legally, destructively, or security sensitive.

## Validation

Test success, failure, repeated taps, cancellation or disappearance, and server conflict behavior.
```
