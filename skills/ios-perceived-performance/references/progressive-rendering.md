# Progressive Rendering

Use this reference when reviewing screens or flows that wait for all data before showing anything, delay primary content behind secondary content, clear existing content during refresh, or update the UI in one large final step.

This reference is about revealing real content in stages. For placeholders, skeletons, progress indicators, loading copy, and loading/error transitions, read `references/loading-states.md`. For optimistic UI, read `references/optimistic-updates.md`. For high-stakes or irreversible flows, read `references/high-stakes-actions.md`. For broader runtime validation, read `references/validation-and-testing.md`.

## Core Idea

Progressive rendering means presenting useful UI in stages instead of waiting for the entire screen to be ready.

It can improve perceived performance because users see structure, feedback, or meaningful content earlier. It does not necessarily reduce total execution time. Treat it as a product and state-modeling technique, not as a low-level optimization.

The goal is to reduce the time until the user can understand the screen or start interacting with the most important content.

## What the Agent Can and Cannot Prove

The agent can inspect and improve:

* all-or-nothing loading states
* screens where primary content could appear before secondary content
* code that waits for unrelated async operations before updating UI
* refresh flows that clear useful existing content
* state models that cannot represent partial content
* missing section-level loading, empty, or error states
* UI updates that happen only after every dependency completes

The agent cannot prove that the screen feels faster without runtime evidence.

Do not claim:

* “This will definitely feel faster.”
* “This fixes the performance issue.”
* “The screen is now responsive.”

Prefer:

* “This should reduce blank-screen time.”
* “This lets primary content appear before secondary content.”
* “Validate with a screen recording or tap-to-first-content timing.”
* “This changes perceived latency, not necessarily total load time.”

## When Progressive Rendering Helps

Progressive rendering is useful when:

* the screen has independent sections
* primary content is more important than secondary content
* some data is available earlier than the rest
* cached or stale content is useful during refresh
* the user can start reading or interacting before all sections are complete
* the current implementation shows a blank screen for too long
* slow secondary content blocks the whole screen
* partial failure can be shown inline without collapsing the whole flow

Examples:

* profile header before recommendations
* account summary before transaction history
* article title and body before comments
* search results before filters or suggestions
* cached feed while new items refresh
* product details before related products
* local draft content before sync status

## When Progressive Rendering Is Risky

Do not recommend progressive rendering blindly.

It may be wrong when:

* partial content would mislead the user
* all data must be consistent at the same point in time
* the product requires an atomic result
* the screen is small and already loads quickly
* staged rendering would cause layout jumps
* partial errors would be harder to explain than a single error state
* ordering is essential
* the result must be server-confirmed before it is meaningful

For financial, legal, destructive, irreversible, security-sensitive, or trust-sensitive flows, read `references/high-stakes-actions.md`.

## Review Workflow

Before proposing progressive rendering, answer:

1. What is the primary content?
2. What can safely appear later?
3. Which sections are independent?
4. Which section failures should block the whole screen?
5. Which section failures can be shown inline?
6. Can existing content remain visible during refresh?
7. Is stale content safe enough to show?
8. Would staged updates cause layout jumps?
9. Does the state model support partial content?
10. Can the improvement be validated with tap-to-first-content timing or screen recording?

If these questions cannot be answered from code, mark the missing product or design decision explicitly.

## All-or-Nothing Loading Smell

Risky:

```swift
@MainActor
final class HomeModel {
    private(set) var state: HomeState = .idle

    func load() async {
        state = .loading

        do {
            async let headline = service.loadHeadline()
            async let shortcuts = service.loadShortcuts()
            async let suggestions = service.loadSuggestions()
            async let activity = service.loadRecentActivity()

            let content = try await HomeContent(
                headline: headline,
                shortcuts: shortcuts,
                suggestions: suggestions,
                activity: activity
            )

            state = .loaded(content)
        } catch {
            state = .failed(error)
        }
    }
}
```

This may be technically concurrent, but the UI still waits for every section before showing the screen.

A better design depends on product requirements. If sections are independent, use section-level state.

```swift
struct HomeScreenState {
    var headline: Loadable<Headline>
    var shortcuts: Loadable<[Shortcut]>
    var suggestions: Loadable<[Suggestion]>
    var activity: Loadable<[ActivityItem]>
}

enum Loadable<Value> {
    case idle
    case loading
    case loaded(Value)
    case empty
    case failed(Error)
}
```

The UI can render the screen shell and update sections independently.

For deeper loading-state modeling, empty states, placeholders, and loading copy, read `references/loading-states.md`.

## Critical Content First

Identify what the user needs first.

Ask:

* What makes the screen meaningful?
* What can the user act on first?
* Which sections are secondary?
* Which data can arrive later without confusing the user?
* Which failures should block the whole screen?
* Which failures can be shown inline?

Example:

```swift
@MainActor
final class HomeModel {
    private(set) var state = HomeScreenState(
        headline: .idle,
        shortcuts: .idle,
        suggestions: .idle,
        activity: .idle
    )

    func load() async {
        markAllSectionsAsLoading()

        await loadPrimaryContent()
        await loadSecondaryContent()
    }

    private func markAllSectionsAsLoading() {
        state.headline = .loading
        state.shortcuts = .loading
        state.suggestions = .loading
        state.activity = .loading
    }

    private func loadPrimaryContent() async {
        async let headline: Void = loadHeadlineSection()
        async let shortcuts: Void = loadShortcutsSection()

        _ = await (headline, shortcuts)
    }

    private func loadSecondaryContent() async {
        async let suggestions: Void = loadSuggestionsSection()
        async let activity: Void = loadActivitySection()

        _ = await (suggestions, activity)
    }

    private func loadHeadlineSection() async {
        do {
            let value = try await service.loadHeadline()
            state.headline = .loaded(value)
        } catch {
            state.headline = .failed(error)
        }
    }
}
```

This lets critical UI appear earlier without blocking on every secondary dependency.

Use this pattern only when primary and secondary sections are independent enough to load and fail separately.

## Section-Level Errors

A progressive screen needs section-level failure boundaries.

Risky:

```swift
do {
    let content = try await loadEverything()
    state = .loaded(content)
} catch {
    state = .failed(error)
}
```

One secondary failure can collapse the whole screen.

Prefer explicit section failure when partial content is acceptable:

```swift
private func loadSuggestionsSection() async {
    do {
        let suggestions = try await service.loadSuggestions()
        state.suggestions = suggestions.isEmpty ? .empty : .loaded(suggestions)
    } catch {
        state.suggestions = .failed(error)
    }
}
```

Use full-screen failure only when the primary content cannot load or the screen has no useful partial state.

## Stale While Refreshing

Refreshing should not always clear existing content.

Risky:

```swift
func refresh() async {
    state = .loading

    do {
        let items = try await service.loadItems()
        state = .loaded(items)
    } catch {
        state = .failed(error)
    }
}
```

If the user already had useful content, this creates a blank or unstable experience during refresh.

Prefer preserving current content when it is safe:

```swift
enum ListState {
    case loading
    case loaded(items: [ListItem], isRefreshing: Bool)
    case empty
    case failed(Error)
}

func refresh() async {
    guard case .loaded(let items, _) = state else {
        await loadInitial()
        return
    }

    state = .loaded(items: items, isRefreshing: true)

    do {
        let refreshed = try await service.loadItems()
        state = refreshed.isEmpty
            ? .empty
            : .loaded(items: refreshed, isRefreshing: false)
    } catch {
        state = .loaded(items: items, isRefreshing: false)
        errorPresenter.showRefreshFailed()
    }
}
```

This keeps the screen useful while communicating that refresh is happening.

Use stale content carefully when:

* old content may be dangerous or misleading
* data is financial, medical, legal, or security-sensitive
* the user needs server-confirmed freshness
* stale content should be visually marked

For broader loading, refreshing, and error-state transitions, read `references/loading-states.md`.

## Partial UI Updates

Avoid delaying the whole screen behind slow secondary work.

Risky:

```swift
let header = try await service.loadHeader()
let badges = try await service.loadBadges()
let timeline = try await service.loadTimeline()
let promotions = try await service.loadPromotions()

state = .loaded(
    DetailsState(
        header: header,
        badges: badges,
        timeline: timeline,
        promotions: promotions
    )
)
```

If `promotions` is slow, the primary details are delayed too. The same problem can happen even with parallel requests if the UI waits for all results before updating.

Prefer section-level updates when product rules allow partial content.

```swift
@MainActor
final class DetailsModel {
    private var state: DetailsState

    func loadDetails() async {
        state.header = .loading
        state.badges = .loading
        state.timeline = .loading
        state.promotions = .loading

        async let header: Void = loadHeader()
        async let badges: Void = loadBadges()
        async let timeline: Void = loadTimeline()
        async let promotions: Void = loadPromotions()

        _ = await (header, badges, timeline, promotions)
    }

    private func loadHeader() async {
        do {
            let header = try await service.loadHeader()
            state.header = .loaded(header)
        } catch {
            state.header = .failed(error)
        }
    }

    private func loadPromotions() async {
        do {
            let promotions = try await service.loadPromotions()
            state.promotions = promotions.isEmpty
                ? .empty
                : .loaded(promotions)
        } catch {
            state.promotions = .failed(error)
        }
    }
}
```

Use `async let` here because the set of sections is small and fixed, and the section loads belong to the lifetime of `loadDetails()`.

Use a task group when the number of sections is dynamic.

Use stored `Task {}` only when the work is intentionally owned by the screen or model lifetime and must be cancelled externally, such as when loading is started from a synchronous method or must be cancelled on disappearance.

Do not use unstructured `Task {}` merely to make section loading parallel. Prefer structured concurrency when all child work belongs to one async scope.

## Stable Layout During Staged Updates

Progressive rendering can feel worse if every section changes size after loading.

Review whether the UI reserves stable space for expected sections.

Risky:

```swift
if let banner {
    BannerView(banner)
}
```

If the banner appears late and pushes content down, the screen may jump.

Prefer a stable region when the section is expected:

```swift
BannerContainer {
    switch state.banner {
    case .loading:
        BannerLoadingView()
    case .loaded(let banner):
        BannerView(banner)
    case .empty:
        EmptyView()
    case .failed:
        BannerRetryView()
    case .idle:
        BannerLoadingView()
    }
}
```

The exact UI depends on design, but the principle is stable layout during staged updates.

For skeletons, placeholders, loading copy, and empty-state presentation, read `references/loading-states.md`.

## Duplicate Loads and Cancellation

Progressive rendering often creates more independent loading paths. Make sure this does not create duplicate work or abandoned tasks.

Check:

* Can the same section load be started twice?
* What happens when the screen disappears?
* What happens when refresh starts while initial loading is still running?
* Are owner-scoped tasks stored and cancelled?
* Are structured child tasks preferred when work belongs to one async call?
* Are section-level loaders idempotent or protected from duplicate requests?

Keep detailed task-lifetime and structured-concurrency guidance in the concurrency skill or related references. In this reference, focus only on how task lifetime affects staged UI updates.

## Implementation Checklist

When proposing progressive rendering, check:

* [ ] Which content is critical?
* [ ] Which content can arrive later?
* [ ] Are sections independent enough to load separately?
* [ ] Does the state model support section-level loading?
* [ ] Can existing content remain visible during refresh?
* [ ] Are partial errors handled without collapsing the whole screen?
* [ ] Is stale content safe enough to show during refresh?
* [ ] Is layout stable while sections update?
* [ ] Are fixed section loads structured with `async let`?
* [ ] Is `TaskGroup` used when the number of sections is dynamic?
* [ ] Are owner-scoped tasks stored and cancelled if unstructured tasks are necessary?
* [ ] Are duplicate loads avoided?
* [ ] Is the perceived improvement validated with runtime evidence?

## Validation

Validate progressive rendering with evidence that reflects user perception.

Recommended validation for this reference:

* screen recording from tap to first meaningful content
* time to first meaningful content
* time until primary action becomes available
* repeated refresh attempts
* slow network testing
* inspection for layout jumps during staged updates

Do not claim that progressive rendering improved performance unless there is evidence.

Correct phrasing:

* “This should reduce blank-screen time.”
* “This lets primary content appear before secondary content.”
* “Validate by measuring tap-to-first-content and checking for layout jumps.”

Avoid:

* “This makes the screen faster.”
* “This fixes performance.”
* “This guarantees better UX.”

For older-device testing, Low Power Mode, release builds, Instruments, production signals, and broader responsiveness validation, read `references/validation-and-testing.md`.

## Review Output Guidance

When using this reference, explain:

```markdown
## Finding

The screen waits for too much work before showing useful UI.

## User impact

The user sees a blank or static state even though some content could appear earlier.

## Recommended change

Introduce section-level state, render critical content first, preserve stale content during refresh when safe, and show inline loading/error states for secondary sections.

## Validation

Measure time to first meaningful content and inspect screen recording for layout jumps or confusing transitions.
```
