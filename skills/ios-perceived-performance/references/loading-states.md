# Loading States

Use this reference when reviewing what the app shows while content is unavailable, loading, refreshing, transitioning, empty, or failed.

This reference is about state communication. It covers placeholders, skeleton views, progress indicators, loading copy, empty states, error states, and transitions between loading, loaded, empty, and failed states.

For staged real-content rendering, read `references/progressive-rendering.md`. For optimistic UI and rollback, read `references/optimistic-updates.md`. For high-stakes operations that should not appear complete before confirmation, read `references/high-stakes-actions.md`. For runtime validation, read `references/validation-and-testing.md`.

## Core Idea

A loading state should tell the user that the app is working, what kind of content is expected, and what they can do next.

A blank or static screen can look broken even when the app is loading correctly. A good loading state reduces uncertainty without pretending that work completed earlier than it did.

Loading states do not usually reduce actual execution time. They improve perceived responsiveness by making progress, structure, and state transitions visible.

## What the Agent Can and Cannot Prove

The agent can inspect and improve:

* missing loading states
* screens that show a blank view while loading
* state models that collapse loading, empty, and error into one case
* refresh flows that hide existing content unnecessarily
* missing retry paths
* missing disabled/pending state for controls
* placeholders that cause layout jumps
* error states that do not explain what happened
* empty states that provide no next action
* loading copy that is vague or misleading

The agent cannot prove that a loading state feels better without runtime evidence.

Do not claim:

* “This makes the app faster.”
* “This guarantees better perceived performance.”
* “The loading experience is now correct.”

Prefer:

* “This makes loading visible.”
* “This separates loading from empty and error states.”
* “This should reduce uncertainty during the wait.”
* “Validate with a screen recording or state-transition test.”

## State Model First

Avoid representing too many states with one boolean.

Risky:

```swift
@MainActor
final class InboxModel {
    private(set) var isLoading = false
    private(set) var messages: [Message] = []

    func load() async {
        isLoading = true
        messages = (try? await service.loadMessages()) ?? []
        isLoading = false
    }
}
```

This cannot distinguish:

* loading
* loaded with content
* loaded but empty
* failed
* refreshing existing content
* failed refresh with stale content still visible

Prefer explicit states:

```swift
enum LoadState<Value> {
    case idle
    case loading
    case loaded(Value)
    case empty
    case failed(Error)
}
```

For refresh flows, keep existing content when safe:

```swift
enum InboxState {
    case loading
    case loaded(messages: [Message], isRefreshing: Bool)
    case empty
    case failed(Error)
}
```

The exact shape depends on the screen. The important part is that loading, empty, loaded, and failed are not accidentally collapsed into the same UI.

## Initial Loading

Initial loading is the first time the screen has no useful content to show.

Use one of these depending on context:

* progress indicator for generic waiting
* skeleton or placeholder when the content structure is predictable
* empty-state shell when the screen needs orientation before data arrives
* full-screen loading only when the entire screen is unavailable
* section-level loading when parts of the screen can appear independently

Risky:

```swift
var body: some View {
    if model.items.isEmpty {
        EmptyView()
    } else {
        ItemList(items: model.items)
    }
}
```

This makes loading and empty look the same.

Prefer:

```swift
var body: some View {
    switch model.state {
    case .loading:
        ItemListPlaceholder()
    case .loaded(let items):
        ItemList(items: items)
    case .empty:
        ContentUnavailableView(
            "No items yet",
            systemImage: "tray",
            description: Text("New items will appear here when they are available.")
        )
    case .failed:
        RetryView(message: "Could not load items") {
            Task { await model.reload() }
        }
    case .idle:
        EmptyView()
    }
}
```

Use system components when they fit. For example, SwiftUI `ContentUnavailableView` and UIKit content-unavailable configurations can provide standard empty-state presentation.

## Progress Indicators

Use progress indicators when the app is doing work and the user needs to know it has not stalled.

Use determinate progress when progress is measurable:

```swift
struct ImportState {
    var completedFiles: Int
    var totalFiles: Int

    var progress: Double {
        guard totalFiles > 0 else { return 0 }
        return Double(completedFiles) / Double(totalFiles)
    }
}
```

Use indeterminate progress when duration or progress cannot be estimated.

Do not fake precision. If the app does not know how much work remains, avoid a determinate progress bar that moves arbitrarily.

Good candidates for determinate progress:

* file import
* upload with known byte count
* export with known steps
* batch operation with known item count
* download with known content length

Good candidates for indeterminate progress:

* waiting for an unknown network response
* short server operation
* authentication check
* initial request with unknown duration

Avoid showing a spinner as the only UI for long waits when more informative progress or partial content is possible.

## Placeholders and Skeleton Views

Placeholders and skeleton views are useful when no real content is available yet but the structure is predictable.

Use them to:

* show the expected layout
* reserve stable space
* reduce the sense of a blank screen
* prevent layout jumps when real content arrives
* communicate that content is loading

Do not use skeletons when:

* real cached content can be shown instead
* the content shape is not predictable
* the skeleton would be misleading
* the operation is high-stakes and needs explicit progress
* the skeleton hides an error
* the skeleton remains indefinitely without explanation

Risky:

```swift
if let profile {
    ProfileHeader(profile)
}
```

If the profile header appears late, the layout may jump.

Prefer a stable placeholder when the header is expected:

```swift
switch state.profile {
case .loading:
    ProfileHeaderPlaceholder()
case .loaded(let profile):
    ProfileHeader(profile)
case .empty:
    ProfileUnavailableView()
case .failed:
    ProfileRetryView()
case .idle:
    ProfileHeaderPlaceholder()
}
```

Skeletons should visually read as temporary. They should not look like real content, and they should not be interactive.

## Empty States

An empty state means loading completed but there is no content to show.

Do not show an empty state while data is still loading. Do not show a loading state after the app already knows the result is empty.

A useful empty state usually explains:

* what is missing
* why it may be missing
* whether the user can do anything about it
* what action is available next

Good empty states:

* “No saved articles yet” with an action to browse articles
* “No transactions in this period” with a date filter action
* “No search results” with a suggestion to change the query
* “No downloads” with a call to start one

Risky:

```swift
Text("Nothing here")
```

Prefer context and recovery:

```swift
ContentUnavailableView(
    "No saved articles",
    systemImage: "bookmark",
    description: Text("Articles you save will appear here.")
)
```

If the empty state depends on filters, mention that context:

```swift
ContentUnavailableView(
    "No results for this filter",
    systemImage: "line.3.horizontal.decrease.circle",
    description: Text("Try changing the date range or clearing filters.")
)
```

## Error States

An error state should explain that loading failed and provide a realistic recovery path.

Avoid replacing all errors with generic copy.

Risky:

```swift
Text("Something went wrong")
```

Prefer actionable copy:

```swift
RetryView(
    title: "Could not load messages",
    message: "Check your connection and try again.",
    actionTitle: "Retry"
) {
    Task { await model.reload() }
}
```

For section-level content, prefer section-level errors when the rest of the screen remains useful:

```swift
switch state.recommendations {
case .loading:
    RecommendationPlaceholder()
case .loaded(let recommendations):
    RecommendationList(recommendations)
case .empty:
    EmptyRecommendationsView()
case .failed:
    InlineRetryView("Could not load recommendations") {
        Task { await model.reloadRecommendations() }
    }
case .idle:
    EmptyView()
}
```

Use a full-screen error only when the screen has no useful content without the failed data.

## Refreshing Existing Content

Refreshing is different from initial loading.

Initial loading may need a placeholder, skeleton, or full-screen loading state. Refresh often should preserve existing content and show a smaller indicator.

Risky:

```swift
func refresh() async {
    state = .loading

    do {
        let items = try await service.loadItems()
        state = items.isEmpty ? .empty : .loaded(items)
    } catch {
        state = .failed(error)
    }
}
```

This clears useful content during refresh.

Prefer preserving existing content when safe:

```swift
enum FeedState {
    case loading
    case loaded(items: [FeedItem], isRefreshing: Bool)
    case empty
    case failed(Error)
}

@MainActor
func refresh() async {
    guard case .loaded(let currentItems, _) = state else {
        await loadInitial()
        return
    }

    state = .loaded(items: currentItems, isRefreshing: true)

    do {
        let newItems = try await service.loadItems()
        state = newItems.isEmpty
            ? .empty
            : .loaded(items: newItems, isRefreshing: false)
    } catch {
        state = .loaded(items: currentItems, isRefreshing: false)
        errorPresenter.show("Could not refresh. Showing the latest available content.")
    }
}
```

Use stale content carefully when old content could mislead the user. For financial, legal, medical, security-sensitive, or server-authoritative data, read `references/high-stakes-actions.md`.

## Transitions Between States

Review transitions, not only final states.

Important transitions:

* idle → loading
* loading → loaded
* loading → empty
* loading → failed
* loaded → refreshing
* refreshing → loaded
* refreshing → failed while preserving old content
* failed → retrying
* empty → loading after user action
* loaded → empty after filters change

Common problems:

* flicker between loading and loaded
* empty state flashes before loading begins
* old content disappears during refresh
* error replaces useful stale content
* retry button starts work but gives no feedback
* controls remain enabled during submission
* multiple loading indicators compete for attention
* layout jumps when placeholders are replaced

Prefer transitions that keep the user oriented.

For very short operations, avoid UI flicker. It can be better to keep the current state visible or delay showing a transient loading indicator briefly, but do this carefully and consistently. Do not add artificial delays to hide real performance problems.

## Loading Copy

Loading copy should be clear, brief, and honest.

Prefer specific copy when the action is meaningful:

* “Loading messages…”
* “Uploading 3 files…”
* “Checking availability…”
* “Preparing preview…”
* “Refreshing feed…”

Avoid vague or misleading copy:

* “Please wait…”
* “Almost done…” when progress is unknown
* “Finishing up…” when the app has no estimate
* “Success” before the server confirms success

Do not over-explain routine loading. For short, obvious operations, a simple indicator may be enough.

For high-stakes actions, copy should make the status explicit:

* “Submitting transfer…”
* “Waiting for confirmation…”
* “Do not close this screen until confirmation appears.”

Use high-stakes language only when the product actually requires it.

## Accessibility

Loading and state changes should be understandable with assistive technologies.

Check:

* Does the loading indicator have an accessible label when needed?
* Does the empty state explain itself through accessible text?
* Does a retry control have a clear accessible label?
* Does focus jump unexpectedly when content appears?
* Are brief state changes announced when the UI change is not otherwise obvious?
* Are skeletons hidden from accessibility if they do not represent real content?
* Does preserved stale content communicate that refresh is in progress?

For brief or non-obvious updates, accessibility announcements can be appropriate. Do not overuse announcements for every minor loading update; too many announcements can make the experience noisy.

## Implementation Checklist

When reviewing loading states, check:

* [ ] Are loading, loaded, empty, and failed states distinct?
* [ ] Is initial loading different from refreshing existing content?
* [ ] Does the UI avoid blank or static screens during meaningful waits?
* [ ] Is a progress indicator shown when the app might otherwise look stalled?
* [ ] Is determinate progress used only when progress is actually measurable?
* [ ] Are placeholders or skeletons used only when structure is predictable?
* [ ] Do placeholders reserve stable space and avoid layout jumps?
* [ ] Are skeletons clearly temporary and non-interactive?
* [ ] Does the empty state explain why content is unavailable?
* [ ] Does the empty state provide a useful next action when possible?
* [ ] Does the error state include a realistic recovery path?
* [ ] Are section-level errors used when the rest of the screen remains useful?
* [ ] Does refresh preserve existing content when safe?
* [ ] Are high-stakes states explicit and server-confirmed?
* [ ] Does loading copy avoid false precision or premature success?
* [ ] Are accessibility labels and announcements considered?
* [ ] Is the perceived improvement validated with runtime evidence when claimed?

## Validation

Validate loading states with evidence that reflects what the user sees.

Useful validation:

* screen recording from action to visible feedback
* time to first visible loading state
* time to first meaningful content
* retry flow test
* slow network testing
* offline or failure simulation
* repeated refresh attempts
* accessibility review with VoiceOver when state changes are dynamic
* UI tests that verify state transitions

Do not claim that loading states improved performance unless there is evidence.

Correct phrasing:

* “This separates loading, empty, and error states.”
* “This should make the wait visible instead of looking stalled.”
* “Validate by recording the transition from tap to first feedback.”

Avoid:

* “This makes loading faster.”
* “This fixes the performance problem.”
* “Users will definitely perceive this as faster.”

For older-device testing, Low Power Mode, release builds, Instruments, production signals, and broader responsiveness validation, read `references/validation-and-testing.md`.

## Review Output Guidance

When using this reference, explain:

```markdown
## Finding

The screen does not clearly communicate loading, empty, or failed states.

## User impact

The user may see a blank screen, unclear wait, misleading empty state, or missing recovery path.

## Recommended change

Introduce explicit loading, loaded, empty, failed, and refreshing states. Use placeholders, progress indicators, empty states, or retry UI based on what the user needs to understand.

## Validation

Record the state transition and verify that the user receives visible feedback before the operation completes.
```
