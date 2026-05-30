# Progressive Rendering

Use this reference when reviewing screens or flows that wait for all data before showing anything, clear existing content during refresh, delay critical content behind secondary content, or update the UI in one large final step.

## Core Idea

Progressive rendering means presenting useful UI in stages instead of waiting for the entire screen to be ready.

It can improve perceived performance because users see structure, feedback, or meaningful content earlier. It does not necessarily reduce total execution time. Treat it as a user-experience and state-modeling technique, not as a low-level optimization.

The goal is to shorten the time until the user can understand or interact with the screen.

## What the Agent Can and Cannot Prove

The agent can inspect and improve:

* all-or-nothing loading states
* state models that block the whole screen behind one request
* refresh flows that clear useful content
* screens where primary content could appear before secondary content
* missing section-level loading or error states
* code that waits for unrelated async operations before updating UI
* UI models that cannot represent partial content

The agent cannot prove perceived improvement without runtime evidence.

Do not claim:

* “This will definitely feel faster.”
* “This eliminates the performance issue.”
* “The screen is now responsive.”

Prefer:

* “This should reduce the time to first meaningful content.”
* “Validate with a screen recording or tap-to-content timing.”
* “This changes perceived latency, not necessarily total load time.”

## When Progressive Rendering Helps

Progressive rendering is useful when:

* the screen has independent sections
* primary content is more important than secondary content
* some data is available earlier than the rest
* cached or stale content is useful during refresh
* a user can start reading or interacting before all sections are complete
* the current implementation shows a blank screen for too long
* slow secondary content blocks the whole screen

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
* business rules require server-confirmed completeness
* the screen is small and loads fast enough
* staged rendering would cause layout jumps
* partial errors would be harder to explain than a single error state
* ordering is essential
* the product requirement is an atomic result

Examples where caution is needed:

* financial confirmation screens
* legal or consent screens
* checkout totals
* medical results
* identity verification
* security-sensitive settings
* server-authoritative eligibility decisions

For these flows, prefer explicit progress and final confirmation over partial presentation.

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
        state.headline = .loading
        state.shortcuts = .loading
        state.suggestions = .loading
        state.activity = .loading

        await loadPrimaryContent()
        await loadSecondaryContent()
    }

    private func loadPrimaryContent() async {
        async let headline = loadHeadlineSection()
        async let shortcuts = loadShortcutsSection()

        _ = await (headline, shortcuts)
    }

    private func loadSecondaryContent() async {
        async let suggestions = loadSuggestionsSection()
        async let activity = loadActivitySection()

        _ = await (suggestions, activity)
    }
}
```

Each section loader should update only its own state.

```swift
@MainActor
private func loadHeadlineSection() async {
    do {
        let value = try await service.loadHeadline()
        state.headline = .loaded(value)
    } catch {
        state.headline = .failed(error)
    }
}
```

This lets critical UI appear earlier without blocking on every secondary dependency.

## Section-Level Errors

A progressive screen needs section-level failure handling.

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

Prefer explicit failure boundaries when partial content is acceptable:

```swift
@MainActor
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

@MainActor
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

## Partial UI Updates

Avoid delaying the whole screen behind slow secondary work.

Risky:

```swift
let header = try await service.loadHeader()
let badges = try await service.loadBadges()
let timeline = try await service.loadTimeline()
let ads = try await service.loadPromotions()

state = .loaded(
    DetailsState(
        header: header,
        badges: badges,
        timeline: timeline,
        promotions: ads
    )
)
```

If `promotions` is slow, the primary details are delayed too.

Prefer independent updates when product rules allow:

```swift
@MainActor
func loadDetails() async {
    state.header = .loading
    state.badges = .loading
    state.timeline = .loading
    state.promotions = .loading

    Task {
        await loadHeader()
    }

    Task {
        await loadBadges()
    }

    Task {
        await loadTimeline()
    }

    Task {
        await loadPromotions()
    }
}
```

If these tasks belong to the screen lifetime, store and cancel them when the screen disappears.

```swift
private var loadingTasks: [Task<Void, Never>] = []

@MainActor
func loadDetails() {
    cancelLoading()

    loadingTasks = [
        Task { await loadHeader() },
        Task { await loadBadges() },
        Task { await loadTimeline() },
        Task { await loadPromotions() }
    ]
}

@MainActor
func cancelLoading() {
    loadingTasks.forEach { $0.cancel() }
    loadingTasks.removeAll()
}
```

Use structured concurrency when all child work belongs to one async scope. Use stored tasks only when work is tied to an object or screen lifetime and must be cancelled externally.

## Avoid Layout Jumps

Progressive rendering can feel worse if every section changes size after loading.

Review whether placeholders reserve stable space.

Risky:

```swift
if let banner {
    BannerView(banner)
}
```

If the banner appears late and pushes content down, the screen may jump.

Prefer reserving a stable region when the section is expected:

```swift
BannerContainer {
    switch state.banner {
    case .loading:
        BannerPlaceholder()
    case .loaded(let banner):
        BannerView(banner)
    case .empty:
        EmptyView()
    case .failed:
        BannerRetryView()
    case .idle:
        BannerPlaceholder()
    }
}
```

The exact UI depends on design, but the principle is stable layout during staged updates.

## Progressive Rendering vs Skeletons

Progressive rendering and skeletons solve different problems.

Progressive rendering shows real content as soon as it is available.

Skeletons show structure while content is unavailable.

Prefer real content over skeletons when useful content already exists or can arrive early.

Use skeletons when:

* no meaningful content is available yet
* the screen structure is predictable
* the placeholder reduces uncertainty
* it does not create misleading expectations
* it does not cause layout shifts

Do not use skeletons to hide poor state modeling. If content can be loaded in independent sections, section-level rendering may be better.

## Accessibility and State Communication

Progressive rendering should not make the screen confusing.

Check:

* Does VoiceOver receive meaningful state changes?
* Are loading states announced when appropriate?
* Does focus jump unexpectedly as sections appear?
* Are retry controls reachable?
* Are placeholders distinguishable from real content?
* Are error states clear at section level?
* Does preserving stale content make freshness ambiguous?

When content is stale or refreshing, communicate that state visually and, when appropriate, accessibly.

## Implementation Checklist

When proposing progressive rendering, check:

* [ ] Which content is critical?
* [ ] Which content can arrive later?
* [ ] Are sections independent enough to load separately?
* [ ] Does the state model support section-level loading?
* [ ] Can existing content remain visible during refresh?
* [ ] Are partial errors handled without collapsing the whole screen?
* [ ] Are empty states distinct from loading states?
* [ ] Is stale content safe to show?
* [ ] Is layout stable while sections update?
* [ ] Are async tasks cancelled when the screen disappears?
* [ ] Are duplicate loads avoided?
* [ ] Is accessibility considered?
* [ ] Is the perceived improvement validated with runtime evidence?

## Validation

Use validation that reflects user perception.

Recommended validation:

* screen recording from tap to first visible feedback
* time to first meaningful content
* time until primary action becomes available
* repeated refresh attempts
* slow network testing
* release build testing
* older device testing when available
* Instruments when UI stalls or hitches are suspected
* production metrics when the issue appears in the wild

Do not claim that progressive rendering improved performance unless there is evidence.

Correct phrasing:

* “This should reduce blank-screen time.”
* “This lets primary content appear before secondary content.”
* “Validate by measuring tap-to-first-content and checking for layout jumps.”

Avoid:

* “This makes the screen faster.”
* “This fixes performance.”
* “This guarantees better UX.”

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
