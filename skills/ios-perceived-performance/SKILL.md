---

name: ios-perceived-performance
description: Use this skill when reviewing iOS screens, user flows, loading states, perceived latency, time to first feedback, progressive rendering, skeleton views, placeholders, optimistic updates, rollback behavior, high-stakes actions, interaction feedback, UI continuity, responsiveness validation, or product-level performance trade-offs.

---

# iOS Perceived Performance

Use this skill to review whether an iOS feature feels responsive to users, even when raw execution time does not change.

This skill is not a low-level profiling guide. It is not focused on CPU samples, allocations, ARC traffic, Swift Concurrency internals, or rendering diagnostics. Use it when the task is about user-perceived speed, feedback, loading behavior, optimistic UI, progressive content, or validation of responsiveness from a product perspective.

## Core Principle

Performance is not only how long work takes. It is also how the interface communicates progress, responds to actions, and keeps the user oriented while work is happening.

Users do not see CPU usage, network traces, or database timings. They notice:

* whether anything happens after they tap
* when meaningful content first appears
* when they can start interacting
* whether updates feel continuous or jumpy
* whether loading states are clear
* whether timing is predictable
* whether the app feels stuck, empty, or inconsistent

Start by asking what delay the user actually experiences.

## Agent Capability Boundaries

The agent can usually inspect and improve:

* state models for loading, loaded, empty, error, refreshing, pending, and failed states
* whether UI updates happen only after the whole operation completes
* whether a user action has immediate visual feedback
* whether optimistic updates have rollback and pending states
* whether high-stakes actions wait for server confirmation
* whether duplicate submissions are prevented
* whether stale content can remain visible during refresh
* whether code structure allows progressive rendering

The agent can suggest, but cannot directly prove without runtime evidence:

* whether a screen feels fast enough
* whether a skeleton improves perception
* whether a deliberate delay is appropriate
* whether Low Power Mode exposes frame drops
* whether older devices perform acceptably
* whether a product flow feels trustworthy to users

Do not claim that a manual or device-based validation step was performed unless the user provided evidence such as a recording, trace, test result, or profiling output.

## References

Read these only when the task requires deeper guidance:

* `references/progressive-rendering.md` — staged loading, critical content first, partial UI updates, stale-while-refreshing screens, section-level loading, and avoiding all-or-nothing rendering.
* `references/loading-states.md` — placeholders, skeleton views, empty states, progress indicators, loading copy, and transitions between loading, loaded, empty, and error states.
* `references/optimistic-updates.md` — optimistic UI, pending state, rollback strategy, conflict handling, retries, local reconciliation, and failure recovery.
* `references/high-stakes-actions.md` — financial, legal, destructive, irreversible, or trust-sensitive actions where optimistic UI may be unsafe.
* `references/validation-and-testing.md` — older-device testing, Low Power Mode as a manual stress signal, release builds, scrolling tests, animation tests, screen recordings, Instruments, and production responsiveness signals.

## Review Workflow

When reviewing a screen or flow, classify the perceived performance problem first:

1. Is there no feedback after a user action?
2. Does the screen stay blank while data loads?
3. Does the UI wait for all data before showing anything?
4. Can critical content appear earlier?
5. Does refresh remove useful existing content?
6. Are loading, empty, error, and retry states distinct?
7. Can the action be updated optimistically?
8. Is optimistic UI unsafe because the action is high-stakes?
9. Is the interaction vulnerable to duplicate submission?
10. Is the perceived delay caused by inconsistent timing or abrupt updates?
11. Is validation possible from code alone, or only by running the app?

Prefer product-level improvements before low-level optimization when the user-visible issue is lack of feedback, blank UI, or poor state communication.

## Time to First Feedback

After a user action, the interface should acknowledge that something started.

Look for actions that do this:

```swift
func submit() async throws {
    let result = try await service.submit()
    state = .finished(result)
}
```

The user may see no immediate response until the request completes.

Prefer an explicit intermediate state:

```swift
func submit() async {
    state = .submitting

    do {
        let result = try await service.submit()
        state = .finished(result)
    } catch {
        state = .failed(error)
    }
}
```

The agent can usually implement this when the state model and UI code are available.

Review questions:

* Does the UI change immediately after the user action?
* Is the tapped control disabled or marked as in progress?
* Is there a visible loading, pending, or progress state?
* Can duplicate submissions happen during the request?
* Does failure return the interface to a usable state?

## Progressive Rendering

Avoid all-or-nothing rendering when a screen can display useful content in stages.

Risky:

```swift
func loadScreen() async throws {
    state = .loading

    async let profile = api.loadProfile()
    async let recommendations = api.loadRecommendations()
    async let history = api.loadHistory()

    state = .loaded(
        try await ScreenData(
            profile: profile,
            recommendations: recommendations,
            history: history
        )
    )
}
```

This may delay the entire screen until every section is ready.

Consider section-level state when content has different importance or latency:

```swift
struct ScreenState {
    var profile: SectionState<Profile>
    var recommendations: SectionState<[Recommendation]>
    var history: SectionState<[HistoryItem]>
}
```

Then the UI can show primary content first and fill secondary content later.

Agent action:

* If the screen currently waits for all data, suggest staged loading.
* If code is available, propose a state model that supports partial content.
* If product requirements are unclear, do not assume every section should load independently.
* If ordering matters, consistency matters, or partial content would confuse users, prefer an explicit loading state.

For detailed staged-loading patterns, read `references/progressive-rendering.md`.

## Loading States, Skeletons, and Placeholders

Use loading states to reduce uncertainty. A blank or static screen can look broken even if work is happening.

Prefer distinct states:

```swift
enum ContentState<Value> {
    case idle
    case loading
    case loaded(Value)
    case empty
    case failed(Error)
}
```

For refresh flows, consider preserving existing content:

```swift
enum FeedState {
    case loading
    case loaded(items: [FeedItem], isRefreshing: Bool)
    case empty
    case failed(Error)
}
```

This lets the UI keep useful content visible while showing that refresh is in progress.

Skeleton views and placeholders are useful when they show structure before content arrives. They are less useful when they hide errors, create layout shifts, or remain on screen without progress.

Agent action:

* Can suggest skeletons or placeholders when a screen is blank during load.
* Can add loading states when the code has a clear state model.
* Should not claim that skeletons reduce actual execution time.
* Should not recommend skeletons when simple cached content, inline progress, or immediate partial content would be clearer.

For deeper loading-state guidance, read `references/loading-states.md`.

## Continuity and Predictability

A screen can feel slow when updates happen in abrupt jumps, even if total loading time is acceptable.

Watch for:

* clearing the screen during every refresh
* replacing all rows when only a small part changed
* jumping from blank state to dense content
* multiple unrelated spinners
* inconsistent loading behavior across repeated attempts
* controls that sometimes react instantly and sometimes do nothing
* layout shifts caused by late-arriving content

Prefer stable transitions:

* keep old content visible during refresh when safe
* mark sections as refreshing instead of clearing them
* preserve scroll position when possible
* avoid unnecessary layout shifts
* update rows or sections incrementally
* show consistent pending and failure states

Agent action:

* Can identify state transitions that clear useful content.
* Can suggest preserving existing data during refresh.
* Can recommend stable placeholders with fixed dimensions.
* Can suggest validation with screen recording or UI tests when perceived continuity cannot be proven from code.

## Optimistic Updates

Optimistic updates improve perceived latency by updating the UI before backend confirmation when the result is predictable and reversible.

Good candidates:

* mute or unmute a topic
* save or unsave an item
* mark as read
* dismiss a local tip
* update a local preference
* edit a draft
* reorder local content before sync

Risky candidates:

* payments
* transfers
* irreversible deletion
* legal consent
* medical or identity-related actions
* actions with complex server-side validation
* actions with high failure or conflict probability

Example pattern:

```swift
func toggleTopicMute(id: Topic.ID) {
    let previous = store.topic(id).isMuted
    let next = !previous

    store.updateTopic(id) { topic in
        topic.isMuted = next
        topic.syncState = .syncing
    }

    syncTask = Task {
        do {
            try await notificationAPI.setMuted(next, for: id)
            store.updateTopic(id) { topic in
                topic.syncState = .synced
            }
        } catch {
            store.updateTopic(id) { topic in
                topic.isMuted = previous
                topic.syncState = .failed
            }

            errorPresenter.showSyncFailure()
        }
    }
}
```

Review optimistic updates for:

* previous value captured before mutation
* pending/syncing state
* rollback path
* failure message
* retry behavior
* duplicate tap behavior
* conflict handling
* cancellation behavior
* consistency with server state after app restart

Agent action:

* Can propose optimistic UI when action is low-risk and reversible.
* Can implement rollback if code structure is available.
* Should not recommend optimistic success for high-stakes or irreversible operations.
* Should explicitly call out consistency trade-offs.

For detailed optimistic update patterns, read `references/optimistic-updates.md`.

## High-Stakes and Irreversible Actions

Some operations should not look complete until the server confirms them.

Use explicit progress and confirmation for:

* financial transactions
* destructive operations
* irreversible account changes
* legal consent
* security-sensitive changes
* server-authoritative decisions
* actions where local prediction can mislead the user

Prefer:

* visible progress state
* disabled repeated submission
* idempotency or request tracking when applicable
* clear success state only after confirmation
* clear failure recovery
* no optimistic final state before server acknowledgement

Example:

```swift
func confirmTransfer() async {
    state = .submitting
    isSubmitDisabled = true

    do {
        let receipt = try await transferService.confirm()
        state = .confirmed(receipt)
    } catch {
        state = .failed(error)
        isSubmitDisabled = false
    }
}
```

Agent action:

* Can identify unsafe optimistic UI.
* Can suggest explicit progress and duplicate-submission prevention.
* Should not suggest artificial trust-building delays as a default solution.
* Should not hide uncertainty when the operation outcome depends on the server.

For deeper high-stakes action guidance, read `references/high-stakes-actions.md`.

## Deliberate Delays

Deliberate delays are rarely a performance fix. Treat them as a product or trust-design decision, not a default optimization.

They may be considered only when:

* users expect visible processing
* instant completion would look suspicious or untrustworthy
* the operation is already complete but the flow requires acknowledgement
* product/design has a clear reason
* the delay is short, consistent, and tested

Do not recommend deliberate delays to hide poor performance, mask backend uncertainty, or make an app feel artificially busy.

Agent action:

* Can mention deliberate delay as a product hypothesis in narrow cases.
* Should not implement it unless explicitly requested or clearly required by the product flow.
* Should prefer real progress, immediate feedback, or clear state transitions over artificial waiting.

## Constrained Performance Validation

The agent cannot enable Low Power Mode, run the app on a device, or prove responsiveness unless the environment provides that capability or the user supplies evidence.

For render-heavy screens, suggest validation under constrained conditions:

* real older devices when available
* Low Power Mode as a cheap manual stress signal
* release configuration, not only debug
* long scrolling sessions
* repeated navigation in and out of the screen
* animation-heavy interactions
* screen recordings to inspect first feedback and visual stability
* Instruments for hangs, hitches, layout, rendering, and main-thread work

Low Power Mode does not simulate an older device. It does not reproduce older memory bandwidth, GPU architecture, refresh behavior, OS version, or thermal state. Use it only as a signal that a screen may have low performance headroom.

Agent action:

* Can recommend Low Power Mode testing for render-heavy screens.
* Must not claim Low Power Mode proves older-device performance.
* Must not claim it performed the test.
* Should explain what to observe: frame drops, delayed taps, animation hitches, image decoding, layout cost, compositing, transparency, or view invalidation.

For detailed validation strategies, read `references/validation-and-testing.md`.

## Evidence and Diagnostics

Prefer evidence that reflects user experience.

Useful evidence:

* video or screen recording
* tap-to-first-feedback timing
* time to first meaningful content
* time until interaction is possible
* scroll hitch observations
* animation hitch observations
* Xcode Organizer hangs and hitches
* Instruments traces
* MetricKit responsiveness data
* production logs around loading stages
* UI tests that capture state transitions
* user reports describing stuck, blank, or jumpy screens

Map symptoms to likely causes:

* blank screen → missing loading, placeholder, cached, or progressive state
* delayed tap response → no immediate state update or main-thread work
* everything appears at once after a long delay → all-or-nothing loading
* jumpy content → unstable layout or late-arriving sections
* repeated submission → missing pending state or disabled control
* incorrect optimistic result → missing rollback or conflict handling
* user distrust → unclear progress, premature success, or inconsistent timing

## Code Review Checklist

Before finalizing a recommendation, check:

* [ ] Is the user-visible delay clearly identified?
* [ ] Does the UI provide feedback immediately after user action?
* [ ] Is the screen blank when it could show structure, cached content, or partial content?
* [ ] Can critical content render before secondary content?
* [ ] Are loading, empty, error, refreshing, pending, and failed states distinct?
* [ ] Does refresh preserve useful existing content when safe?
* [ ] Are transitions stable enough to avoid visual jumps?
* [ ] Is optimistic UI appropriate for this action?
* [ ] Is rollback implemented for optimistic updates?
* [ ] Are pending and failed synchronization states visible?
* [ ] Is optimistic UI avoided for high-stakes or irreversible actions?
* [ ] Are duplicate submissions prevented?
* [ ] Is deliberate delay avoided unless there is a clear product reason?
* [ ] Does the recommendation distinguish what the agent can implement from what needs manual validation?
* [ ] Is validation realistic: device testing, Low Power Mode signal, Instruments, screen recording, UI tests, or production metrics?
* [ ] Does the answer avoid claiming unmeasured performance improvements?

## Output Format

When reviewing a screen or flow, respond with:

```markdown
## Finding

Describe the perceived performance issue.

## User impact

Explain what the user sees or feels: no feedback, blank screen, jumpy updates, delayed interaction, unclear progress, or premature success.

## Evidence

Point to code, state model, flow behavior, trace, recording, or missing UI state.

## Recommended change

Suggest the smallest useful product/UI change first: feedback, loading state, progressive rendering, optimistic update, rollback, explicit confirmation, or validation.

## Implementation notes

Explain what can be changed in code and what depends on product/design decisions.

## Validation

State how to verify the improvement. Do not claim manual/device validation was performed unless evidence is provided.
```

Prefer user-visible improvements over low-level optimization when the bottleneck is missing feedback, all-or-nothing rendering, unclear loading, or unsafe optimistic behavior.
