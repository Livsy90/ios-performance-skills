---

name: swiftui-performance
description: SwiftUI performance review and refactoring skill focused on code composition, invalidation scope, view identity, state ownership, Observation and ObservableObject granularity, list and pagination performance, closure-heavy views, bindings, layout cost, drawing cost, async lifecycle work, and profiling validation with Instruments, xctrace, signposts, XCTest, and MetricKit. Use when reviewing SwiftUI screens for unnecessary updates, slow scrolling, heavy rows, broad state dependencies, unstable identity, animation hitches, or performance-sensitive UI architecture.
---

# SwiftUI Performance: Code Composition and Profiling

## 1. Purpose

This skill helps the agent review, generate, and refactor SwiftUI code with a performance-first mental model.

The primary focus is code composition:

* view identity
* view lifetime
* dependency scope
* state ownership
* row complexity
* list and pagination structure
* Observation and ObservableObject granularity
* closure-heavy views
* custom bindings
* layout and drawing cost
* async work timing

Profiling is used as a validation layer. The agent should use profiling tools only when the environment supports them or when the user provides profiling artifacts.

## 2. Non-Goals

Do not turn every SwiftUI review into a profiling session.

Do not recommend complex architectural changes for small static views unless there is a clear performance risk.

Do not claim that a performance issue was measured unless there is actual evidence from:

* Instruments
* `xctrace`
* XCTest performance tests
* signpost logs
* MetricKit payloads
* user-provided traces, screenshots, logs, or benchmark results

## 3. Core Mental Model

Reason about SwiftUI through three concepts:

* Identity: whether SwiftUI treats a view as the same view across updates.
* Lifetime: how long state associated with that identity is preserved.
* Dependencies: which data reads cause a view to be updated.

A SwiftUI view is a value description of UI. It should not be treated as a long-lived object with UIKit-style lifecycle semantics.

When reviewing code, explain performance issues in terms of:

* what changed
* which view depends on that change
* how much of the view tree becomes affected
* whether identity is stable
* whether rendering work is cheap
* whether the structure makes local updates easy or difficult

## 4. Rendering Cycle Mental Model

When state changes, SwiftUI invalidates dependent parts of the view graph, recomputes affected view descriptions, and reconciles them with the existing graph.

Avoid absolute claims like “the whole app redraws” unless the code structure really creates a broad dependency.

Prefer precise wording:

* “This parent view reads the whole model, so changes to this model can invalidate a large part of the screen.”
* “This row receives a newly created closure on every update, which may make it harder for SwiftUI to treat the row as unchanged.”
* “This list is modeled as one expanding flat collection, so appending a page may still require SwiftUI to reconcile a large collection structure.”

## 5. Keep `body` Cheap

`body` can be evaluated frequently. It should describe UI, not prepare data.

Avoid inside `body`:

* sorting
* filtering
* grouping
* date parsing
* currency formatting
* number formatter creation
* image decoding
* JSON parsing
* database reads
* synchronous file access
* network calls
* expensive computed properties
* allocation-heavy mapping
* business logic

Risky:

```swift
struct StatementScreen: View {
    let entries: [StatementEntry]

    var body: some View {
        List {
            ForEach(entries.sorted(by: { $0.date > $1.date })) { entry in
                Text(entry.amount.formatted(.currency(code: entry.currencyCode)))
            }
        }
    }
}
```

Prefer preparing render-ready state before rendering:

```swift
struct StatementRowModel: Identifiable, Equatable {
    let id: StatementEntry.ID
    let title: String
    let amountText: String
    let dateText: String
}

struct StatementScreen: View {
    let rows: [StatementRowModel]

    var body: some View {
        List(rows) { row in
            StatementRow(row: row)
        }
    }
}
```

## 6. Structural Identity vs Explicit Identity

SwiftUI normally derives identity from the structure of the view tree: position, branch, and concrete view type.

Use explicit identity with `.id(...)` only when creating a deliberate identity boundary.

Good reasons to use `.id(...)`:

* resetting local state intentionally
* forcing a detail view to start fresh for a different entity
* defining a scroll target
* creating a known lifecycle boundary

Avoid using `.id(...)` as a generic refresh workaround.

Risky:

```swift
TransactionForm(model: model)
    .id(UUID())
```

Better:

```swift
TransactionForm(model: model)
    .id(model.transactionID)
```

Only do this when changing `transactionID` should really discard local form state.

## 7. Stable `ForEach` Identity

Large dynamic collections need stable, cheap identifiers.

Avoid IDs that change when the array changes:

```swift
ForEach(Array(rows.enumerated()), id: \.offset) { _, row in
    LedgerRow(row: row)
}
```

Prefer identity from the data:

```swift
ForEach(rows) { row in
    LedgerRow(row: row)
}
```

or:

```swift
ForEach(rows, id: \.transactionID) { row in
    LedgerRow(row: row)
}
```

Avoid computed IDs that create a new value on every access:

```swift
struct LedgerRowModel: Identifiable {
    var id: UUID { UUID() }
}
```

Prefer stored identity:

```swift
struct LedgerRowModel: Identifiable {
    let id: UUID
    let title: String
    let amountText: String
}
```

## 8. State Ownership and Lifetime

State should be owned at the smallest stable boundary that needs it.

Use local state for local UI concerns:

* expanded/collapsed row state
* local input text
* focus
* small animation flags
* temporary selection inside a component

Avoid putting every small UI flag into one large parent model when only a small child view needs it.

Risky:

```swift
struct DashboardView: View {
    @State private var isSearchFocused = false
    @State private var isCardExpanded = false
    @State private var draftName = ""
    @State private var selectedTab = 0

    var body: some View {
        DashboardContent(
            isSearchFocused: $isSearchFocused,
            isCardExpanded: $isCardExpanded,
            draftName: $draftName,
            selectedTab: $selectedTab
        )
    }
}
```

Prefer local ownership when possible:

```swift
struct DashboardView: View {
    @State private var selectedTab = 0

    var body: some View {
        VStack {
            SearchPanel()
            BalanceCard()
            DashboardTabs(selection: $selectedTab)
        }
    }
}
```

## 9. `@StateObject`, `@ObservedObject`, and Observation Ownership

Use `@StateObject` when a SwiftUI view creates and owns an `ObservableObject`.

Use `@ObservedObject` when the object is owned elsewhere and injected into the view.

Avoid creating an externally observed model inside a view declaration.

Risky:

```swift
struct QuotesView: View {
    @ObservedObject private var model = QuotesModel()

    var body: some View {
        QuotesContent(model: model)
    }
}
```

Prefer:

```swift
struct QuotesView: View {
    @StateObject private var model = QuotesModel()

    var body: some View {
        QuotesContent(model: model)
    }
}
```

For iOS 17 and later, prefer Observation where it fits the deployment target and architecture:

```swift
@Observable
final class QuotesModel {
    var prices: [QuoteRowModel] = []
    var isRefreshing = false
}
```

For owned observable models:

```swift
struct QuotesView: View {
    @State private var model = QuotesModel()

    var body: some View {
        QuotesContent(model: model)
    }
}
```

## 10. ObservableObject vs Observation Granularity

`ObservableObject` with `@Published` commonly creates object-level invalidation for views observing that object.

Observation can track property reads more precisely: a view that reads one property does not need to become dependent on every property in the model.

However, Observation does not make expensive UI composition free.

If a view reads a collection:

```swift
ForEach(model.entries) { entry in
    EntryRow(entry: entry)
}
```

then changing `model.entries` still invalidates the view that depends on that collection.

Use Observation to narrow dependencies, but still design lists, rows, and pagination carefully.

## 11. Avoid One Giant Observable Model

Do not pass a large screen model into every child view by default.

Risky:

```swift
struct PortfolioHeader: View {
    let model: PortfolioModel

    var body: some View {
        Text(model.userDisplayName)
    }
}
```

If `PortfolioHeader` only needs a string, pass the string:

```swift
struct PortfolioHeader: View {
    let userDisplayName: String

    var body: some View {
        Text(userDisplayName)
    }
}
```

Use direct model access when it genuinely improves clarity and dependency tracking, especially with Observation. Avoid broad computed properties that read many fields and make the dependency wider than it appears.

## 12. Split Screens into Dependency Islands

Large screens should be split by update behavior.

Prefer subviews that read different slices of state:

```swift
struct PortfolioScreen: View {
    let model: PortfolioModel

    var body: some View {
        VStack {
            PortfolioHeader(name: model.displayName)
            HoldingsList(pages: model.holdingPages)
            RefreshFooter(isRefreshing: model.isRefreshing)
        }
    }
}
```

Avoid one large `body` that reads every property and builds every region inline. It makes invalidation harder to reason about and refactor.

## 13. Render Models for Complex Rows

For non-trivial rows, use lightweight render models.

A render model should contain values already prepared for display:

```swift
struct HoldingRowModel: Identifiable, Equatable {
    let id: Holding.ID
    let symbol: String
    let name: String
    let priceText: String
    let changeText: String
    let isPositive: Bool
}
```

The row should mostly render:

```swift
struct HoldingRow: View {
    let row: HoldingRowModel

    var body: some View {
        HStack {
            VStack(alignment: .leading) {
                Text(row.symbol)
                Text(row.name)
            }

            Spacer()

            VStack(alignment: .trailing) {
                Text(row.priceText)
                Text(row.changeText)
                    .foregroundStyle(row.isPositive ? .green : .red)
            }
        }
    }
}
```

Avoid turning each row body into a data transformation pipeline.

## 14. Large Dynamic Collections

Treat any growing, scrolling, frequently updating collection as performance-sensitive.

Review:

* stable identity
* row body cost
* row initializer cost
* filtering and sorting location
* use of lazy containers
* `AnyView` usage
* custom bindings
* closures and action handlers
* swipe actions
* geometry reads
* conditional modifiers
* pagination structure
* state dependencies around the list

## 15. Lazy Containers

Use lazy containers for dynamic or potentially large collections.

Prefer:

* `List` for platform-native long lists, swipe actions, editing, accessibility, and system behavior.
* `LazyVStack` or `LazyHStack` for custom scrolling layouts.
* `VStack` for small, fixed-size content.

Avoid eager stacks for long dynamic collections:

```swift
ScrollView {
    VStack {
        ForEach(feedRows) { row in
            FeedCard(row: row)
        }
    }
}
```

Prefer:

```swift
ScrollView {
    LazyVStack(spacing: 12) {
        ForEach(feedRows) { row in
            FeedCard(row: row)
        }
    }
}
```

Do not use a fixed item-count rule mechanically. Consider row complexity, update frequency, scrolling behavior, and target devices.

## 16. Paginated Lists Should Have Stable Page Boundaries

For paginated data with non-trivial rows, avoid modeling UI as one endlessly growing flat array when appends cause expensive updates.

Risky:

```swift
List {
    ForEach(model.allActivities) { activity in
        ActivityRow(row: activity)
    }
}
```

Prefer explicit page or section models:

```swift
struct ActivityPage: Identifiable, Equatable {
    let id: Int
    let rows: [ActivityRowModel]
}
```

Render:

```swift
List {
    ForEach(model.pages) { page in
        Section {
            ForEach(page.rows) { row in
                ActivityRow(row: row)
            }
        }
    }
}
```

When loading more data, append a new page instead of rebuilding all previous rows:

```swift
@MainActor
func appendPage(_ response: ActivityPageResponse) {
    let page = ActivityPage(
        id: response.pageNumber,
        rows: response.items.map(ActivityRowModel.init)
    )

    pages.append(page)
}
```

Stable page boundaries make change locality more obvious to SwiftUI.

## 17. Do Not Chunk Pages Inside `body`

Avoid deriving pages from a flat array during rendering.

Risky:

```swift
ForEach(model.activities.chunked(into: 25)) { page in
    Section {
        ForEach(page) { activity in
            ActivityRow(activity: activity)
        }
    }
}
```

This can allocate new page values during rendering and may create unstable page identity.

Prefer storing pages as part of state:

```swift
@Observable
final class ActivityModel {
    var pages: [ActivityPage] = []
}
```

## 18. Keep Row Structure Predictable

Large repeated content should have stable structure.

Avoid filtering inside `ForEach`:

```swift
ForEach(rows) { row in
    if row.isVisible {
        FeedCard(row: row)
    }
}
```

Prefer preparing visible rows before rendering:

```swift
ForEach(visibleRows) { row in
    FeedCard(row: row)
}
```

Avoid returning different numbers of child views for different items unless the UI genuinely requires that structure.

## 19. Avoid `AnyView` in Hot Paths

`AnyView` hides concrete view type information and can reduce SwiftUI’s ability to reason about structure.

It is especially risky inside large repeated collections.

Risky:

```swift
ForEach(modules) { module in
    erasedModuleView(module)
}
```

where:

```swift
func erasedModuleView(_ module: ModuleRowModel) -> AnyView {
    switch module.kind {
    case .chart:
        return AnyView(ChartModule(row: module))
    case .news:
        return AnyView(NewsModule(row: module))
    }
}
```

Prefer concrete branching with `@ViewBuilder`:

```swift
@ViewBuilder
func moduleView(_ module: ModuleRowModel) -> some View {
    switch module.kind {
    case .chart:
        ChartModule(row: module)
    case .news:
        NewsModule(row: module)
    }
}
```

Using type erasure at a coarse boundary is less risky than type-erasing every row in a large list.

## 20. Conditional Modifiers

Be careful with custom conditional modifier helpers in repeated views.

Risky:

```swift
priceLabel
    .applyIf(row.isHighlighted) { view in
        view.background(.yellow.opacity(0.2))
    }
```

This may create structural differences between rows or across updates.

Prefer value-based modifiers when the view remains conceptually the same:

```swift
priceLabel
    .background(row.isHighlighted ? .yellow.opacity(0.2) : .clear)
```

Use structural branching when the UI really has different structure, not merely different values.

## 21. Modifier Order and View Hierarchy

Modifiers create nested view structure. Order affects layout, drawing, hit testing, and identity.

In hot rows, review long modifier chains carefully, especially when they include:

* overlays
* backgrounds
* masks
* shadows
* gestures
* preference keys
* geometry reads
* conditional wrappers
* animations

Prefer small extracted subviews when modifier chains obscure dependencies or repeat expensive structure across many rows.

## 22. Custom Bindings

Prefer key-path bindings over closure-created bindings when possible.

Good:

```swift
Toggle("Notifications", isOn: $settings.notificationsEnabled)
```

Risky:

```swift
Toggle(
    "Notifications",
    isOn: Binding(
        get: { settings.notificationsEnabled },
        set: { settings.notificationsEnabled = $0 }
    )
)
```

Custom `Binding(get:set:)` creates closures and can make child inputs less stable.

Use custom bindings when they express necessary transformation or validation, not as the default style.

## 23. Closures, Action Handlers, and Capture Lists

Closures created inside `body` can become part of a SwiftUI view value. SwiftUI usually cannot compare closures in a meaningful way, so closure-heavy child views may appear changed even when their visual input is unchanged.

This is most important in large repeated collections, rows with swipe actions, menus, gestures, or frequently updated parent views.

Avoid designing visual rows around multiple stored action closures unless the row genuinely owns that interaction surface.

Risky:

```swift
struct AssetRow: View {
    let row: AssetRowModel
    let onSelect: () -> Void
    let onFavorite: () -> Void
    let onHide: () -> Void

    var body: some View {
        HStack {
            Text(row.symbol)
            Spacer()
            Text(row.priceText)
        }
        .contentShape(Rectangle())
        .onTapGesture(perform: onSelect)
        .swipeActions {
            Button("Favorite", action: onFavorite)
            Button("Hide", role: .destructive, action: onHide)
        }
    }
}
```

And the parent creates new closures for every row during rendering:

```swift
ForEach(portfolioModel.rows) { row in
    AssetRow(
        row: row,
        onSelect: {
            portfolioModel.selectAsset(row.id)
        },
        onFavorite: {
            portfolioModel.toggleFavorite(row.id)
        },
        onHide: {
            portfolioModel.hideAsset(row.id)
        }
    )
}
```

Prefer keeping the row focused on visual data and routing actions at a stable boundary when possible:

```swift
ForEach(portfolioModel.rows) { row in
    AssetRow(row: row)
        .contentShape(Rectangle())
        .onTapGesture { [portfolioModel, id = row.id] in
            portfolioModel.selectAsset(id)
        }
        .swipeActions {
            Button("Favorite") { [portfolioModel, id = row.id] in
                portfolioModel.toggleFavorite(id)
            }

            Button("Hide", role: .destructive) { [portfolioModel, id = row.id] in
                portfolioModel.hideAsset(id)
            }
        }
}
```

This does not remove closures, but it keeps action handlers out of the row’s stored visual input. It also makes captured dependencies explicit and small.

When a child view must receive a closure, use a capture list to avoid accidentally capturing the whole parent view value:

```swift
ForEach(portfolioModel.rows) { row in
    AssetRow(
        row: row,
        onSelect: { [portfolioModel, id = row.id] in
            portfolioModel.selectAsset(id)
        }
    )
}
```

Prefer capturing:

* a stable model reference
* a stable service or action handler
* a row ID
* a small immutable value

Avoid capturing:

* the whole parent view implicitly
* large domain models when only an ID is needed
* mutable row objects when identity is enough
* broad handler containers with many unrelated actions
* values that are recomputed during every render

Prefer passing stable IDs into actions:

```swift
portfolioModel.selectAsset(id)
```

rather than passing the full row model:

```swift
portfolioModel.selectAsset(row)
```

This rule does not make closures automatically diffable. It reduces accidental captures, keeps visual row inputs cleaner, and makes update dependencies easier to reason about.

Use this rule mainly when:

* the closure is created inside `body`
* the closure is passed into a child view
* the child is inside a large `ForEach`
* the parent view has many stored properties
* the row has swipe actions, menus, buttons, or gestures
* unnecessary row updates are already suspected

Treat row-level interactions as part of row complexity. In large lists, keep action builders lightweight, capture stable IDs, and avoid constructing unnecessary menus or swipe actions for rows that do not need them.

## 24. Equatable Views

Use `Equatable` when a view has clear, visual, equatable inputs and its body is expensive enough to justify the equality check.

```swift
struct ExchangeRateBadge: View, Equatable {
    let code: String
    let valueText: String
    let trend: Trend

    static func == (lhs: Self, rhs: Self) -> Bool {
        lhs.code == rhs.code &&
        lhs.valueText == rhs.valueText &&
        lhs.trend == rhs.trend
    }

    var body: some View {
        HStack {
            Text(code)
            Text(valueText)
            TrendIcon(trend: trend)
        }
    }
}
```

Use:

```swift
ExchangeRateBadge(
    code: row.currencyCode,
    valueText: row.rateText,
    trend: row.trend
)
.equatable()
```

Do not include non-visual closures in equality.

Do not use `Equatable` to hide real visual changes.

## 25. GeometryReader and Layout Dependencies

`GeometryReader` can widen layout dependencies and increase layout work.

Avoid placing geometry readers inside every row of a large collection.

Risky:

```swift
List(cards) { card in
    GeometryReader { proxy in
        LoyaltyCardView(card: card, width: proxy.size.width)
    }
}
```

Prefer reading geometry at a stable container boundary:

```swift
GeometryReader { proxy in
    List(cards) { card in
        LoyaltyCardView(card: card, availableWidth: proxy.size.width)
    }
}
```

Even better, use layout APIs that do not require explicit geometry when possible:

```swift
LoyaltyCardView(card: card)
    .frame(maxWidth: .infinity, alignment: .leading)
```

## 26. PreferenceKey and Layout Feedback

Preference keys are useful for child-to-parent layout communication, but they can introduce additional update cycles.

Use them carefully in:

* large lists
* frequently resizing layouts
* animated containers
* nested scroll views
* rows that update often

Prefer simpler layout structures when possible.

## 27. Canvas for Dense Drawing

When a UI represents many small visual elements that update together, consider drawing instead of building a large view tree.

Good candidates:

* sparklines
* mini charts
* timelines
* heat maps
* audio waveforms
* dense decorative particles
* compact financial graphs

Example:

```swift
struct SparklineView: View {
    let points: [Double]

    var body: some View {
        Canvas { context, size in
            guard points.count > 1 else { return }

            var path = Path()

            for index in points.indices {
                let x = size.width * CGFloat(index) / CGFloat(points.count - 1)
                let y = size.height * (1 - CGFloat(points[index]))

                if index == points.startIndex {
                    path.move(to: CGPoint(x: x, y: y))
                } else {
                    path.addLine(to: CGPoint(x: x, y: y))
                }
            }

            context.stroke(path, with: .foreground, lineWidth: 2)
        }
        .frame(height: 44)
    }
}
```

Do not replace normal UI controls with `Canvas`. Use it for drawing-heavy content where individual subviews would be excessive.

## 28. TimelineView for Scheduled Updates

Use `TimelineView` when UI should update on a known schedule.

Good candidates:

* clocks
* countdowns
* market session timers
* live progress indicators
* time-based visualizations

Example:

```swift
struct MarketCountdownView: View {
    let closeDate: Date

    var body: some View {
        TimelineView(.periodic(from: .now, by: 1)) { timeline in
            Text(remainingText(now: timeline.date))
        }
    }

    private func remainingText(now: Date) -> String {
        let seconds = max(0, Int(closeDate.timeIntervalSince(now)))
        return "\(seconds)s"
    }
}
```

Prefer scheduled updates over manually driving repeated state changes with `Timer` when the UI is naturally time-based.

## 29. Async Work: `.task(id:)`, `.onAppear`, and `body`

Never start async work directly from `body`.

Risky:

```swift
var body: some View {
    Task {
        await model.refresh()
    }

    return ContentView()
}
```

Use `.task(id:)` for lifecycle-bound async work that should cancel and restart when a meaningful input changes:

```swift
struct AccountDetailsView: View {
    let accountID: Account.ID
    let model: AccountDetailsModel

    var body: some View {
        AccountDetailsContent(model: model)
            .task(id: accountID) {
                await model.load(accountID: accountID)
            }
    }
}
```

Use `.onAppear` for appearance-related side effects, not as the default data-loading mechanism.

Be extra careful with `.onAppear` inside rows. It can fire many times during scrolling and navigation. Guard duplicate pagination triggers explicitly.

## 30. MainActor Work

SwiftUI rendering and UI updates happen on the main actor.

Avoid doing heavy data preparation on the main actor during view updates.

Prefer:

* preparing render models after data loading
* moving pure transformations off the main actor when safe
* applying compact final state updates on the main actor
* avoiding repeated sorting/filtering during render

## 31. UIKit Fallback for Hot Paths

SwiftUI is a good default for many screens, but not every performance-sensitive surface must be pure SwiftUI.

Consider UIKit for extremely hot paths that need precise control over:

* cell reuse behavior
* batch updates
* prefetching
* advanced collection layouts
* very large datasets
* strict frame-time budgets
* complex editing or swipe behavior

A hybrid approach can use SwiftUI for screen composition and UIKit for the hottest scrolling component.

## 32. Profiling Capability Rules

The agent may use profiling tools only when the environment supports them.

Before attempting profiling, check:

* macOS host
* full Xcode installation
* selected Xcode path
* buildable project
* runnable target
* available simulator or device
* reproducible user scenario
* permission to run shell commands
* permission to create trace/log artifacts

If tools are unavailable, provide a local profiling plan and optionally add instrumentation code.

Do not pretend to have run Instruments or measured performance when no profiling command or artifact was used.

## 33. Profiling Workflow

Use profiling to validate a SwiftUI hypothesis.

Recommended flow:

1. Describe the suspected composition problem.
2. Define the exact interaction to test.
3. Make the scenario reproducible.
4. Use a Release-like build when possible.
5. Capture evidence with the smallest suitable tool.
6. Identify the dominant cost.
7. Apply a targeted refactor.
8. Repeat the same scenario.
9. Compare before and after.

Example hypothesis:

```md
Appending a new feed page invalidates a large flat list and causes old rows to be recomputed. Validate by signposting the append action and checking body invocation frequency or Time Profiler samples during the interaction.
```

## 34. `xctrace` and Command-Line Profiling

When running locally on macOS with Xcode, the agent may use command-line profiling workflows.

Potential tools:

```bash
xcodebuild
xcrun simctl
xcrun xctrace
```

Use command-line profiling for repeatable scenarios when possible.

The agent should not assume GUI Instruments automation is available. If GUI-only inspection is needed, provide clear instructions for the user to run locally and upload screenshots or exports.

## 35. Time Profiler

Use Time Profiler to answer:

* Is the main thread blocked?
* Which interaction creates the CPU spike?
* Is app code doing expensive work during rendering?
* Are formatters, sorters, mappers, or computed properties visible in the hot path?
* Are row builders or view initializers expensive?
* Is pagination append causing a large update burst?

Look for app-specific symbols first. SwiftUI framework symbols are useful context, but the actionable fix is often in app code called during rendering.

Common findings:

* sorting inside `body`
* date or currency formatting per row
* rebuilding all render models on every append
* expensive computed properties read by views
* image preparation on the main thread
* large closure/action builders in rows

## 36. SwiftUI Instrument

When available, use the SwiftUI instrument to inspect:

* body invocation count
* body invocation duration
* views updating more often than expected
* expensive view bodies
* broad invalidation after unrelated state changes
* list rows being recomputed during pagination or filtering

Use this to verify whether a state ownership or dependency narrowing refactor actually reduced view work.

## 37. Animation Hitches and Core Animation

Use Animation Hitches or Core Animation tools when the symptom is visible stutter, dropped frames, delayed gestures, or animation jank.

Check whether the issue is caused by:

* main-thread blocking work
* excessive layout
* expensive drawing
* too many layers/effects
* heavy shadows, masks, or blurs
* repeated state updates during animation
* list updates during scrolling

Do not assume every frame drop is caused by SwiftUI diffing.

## 38. Allocations

Use Allocations when the symptom suggests memory churn or repeated construction.

Look for:

* repeated formatter creation
* rebuilding large arrays of render models
* excessive temporary strings
* per-row closure-heavy objects
* image decoding or resizing
* repeated type-erased wrappers
* large copy-on-write structures copied during rendering

## 39. Signposts

Use `os_signpost` to mark important user actions and update phases.

Good signpost boundaries:

* load next page tapped
* network response received
* render models built
* page appended to state
* filter changed
* sort changed
* search text applied
* animation started
* expensive cache lookup started/finished

Example:

```swift
import os

private let performanceLog = OSLog(
    subsystem: "com.example.app",
    category: "PortfolioPerformance"
)

func appendNextPage(_ page: HoldingPage) {
    os_signpost(.begin, log: performanceLog, name: "AppendHoldingPage")
    pages.append(page)
    os_signpost(.end, log: performanceLog, name: "AppendHoldingPage")
}
```

Use signposts to align app-level events with profiler timelines.

## 40. `_printChanges()` and Temporary Debug Probes

For local debugging, the agent may suggest temporary probes.

Examples:

```swift
var body: some View {
    let _ = Self._printChanges()

    return content
}
```

Use this to understand why a view updates during development.

Other temporary probes:

* count row body invocations
* log page append counts
* log render model rebuilds
* log filtering/sorting duration
* log duplicate `.onAppear` pagination triggers

Remove debug probes before production.

## 41. XCTest Performance Tests

Use XCTest performance tests for repeatable local benchmarks.

Good candidates:

* render model generation
* filtering
* sorting
* grouping
* formatting pipelines
* pagination state updates
* cache lookups

Do not use XCTest microbenchmarks as the only proof of UI smoothness. They are useful for isolating app code costs, not for fully replacing Instruments.

## 42. MetricKit

Use MetricKit for production-level performance signals.

MetricKit can help identify trends in:

* hangs
* launch time
* memory
* CPU
* disk writes
* animation responsiveness
* crash diagnostics
* energy usage

MetricKit is not a replacement for local profiling. Use it to find production patterns and prioritize investigation.

## 43. Profiling Artifacts

When the user provides artifacts, analyze them directly.

Supported artifacts may include:

* `.trace` exports
* screenshots from Instruments
* `xctrace` exports
* signpost logs
* console logs
* MetricKit payloads
* XCTest benchmark output
* screen recordings
* memory graphs

Clearly distinguish what the artifact proves from what remains a hypothesis.

## 44. Evidence Rule

Always separate:

* static code review findings
* likely risks
* hypotheses
* measured results
* user-provided evidence
* tool-generated evidence

Do not write:

```md
This costs 500 ms.
```

unless a trace, benchmark, signpost, log, or user-provided measurement shows that number.

Prefer:

```md
This can cause broad invalidation. To confirm it, profile the append interaction with signposts around page insertion and inspect body invocation frequency.
```

## 45. Code Review Checklist

When reviewing SwiftUI code, check:

* Does the view read only the state it needs?
* Is state owned by the smallest stable view that needs it?
* Are observable dependencies narrow?
* Is `body` free from expensive work?
* Are render models prepared outside view rendering?
* Are list IDs stable and cheap?
* Is `.id(...)` used intentionally?
* Is a large list using an appropriate lazy container?
* Are paginated lists represented with stable page or section models?
* Are rows visually lightweight?
* Is filtering/sorting outside `ForEach`?
* Is `AnyView` absent from repeated hot paths?
* Are conditional modifiers structurally stable?
* Are custom bindings necessary?
* Are row closures capturing too much?
* Are swipe actions and menus lightweight?
* Is `GeometryReader` avoided per row?
* Are drawing-heavy elements using `Canvas` where appropriate?
* Are scheduled updates using `TimelineView` where appropriate?
* Is async work started through `.task(id:)` or explicit actions, not `body`?
* Is profiling evidence separated from hypotheses?

## 46. Red Flags

Flag these patterns during review:

```swift
ForEach(items.indices, id: \.self) { index in
    Row(item: items[index])
}
```

```swift
ForEach(items.sorted(by: sortRule)) { item in
    Row(item: item)
}
```

```swift
ForEach(items) { item in
    if item.matchesFilter {
        Row(item: item)
    }
}
```

```swift
RowView(
    model: largeScreenModel,
    item: item,
    onTap: {
        largeScreenModel.open(item)
    }
)
```

```swift
.id(UUID())
```

```swift
var id: UUID { UUID() }
```

```swift
AnyView(rowView)
```

```swift
Binding(
    get: { model.value },
    set: { model.value = $0 }
)
```

```swift
Text(formatter.string(from: value))
```

```swift
GeometryReader { proxy in
    Row(width: proxy.size.width)
}
```

inside a large repeated collection.

## 47. Preferred Refactoring Order

When fixing SwiftUI performance risks, prefer this order:

1. Stabilize identity.
2. Remove expensive work from `body`.
3. Prepare render models outside rendering.
4. Narrow view dependencies.
5. Move local state closer to the component that owns it.
6. Split large views into dependency-focused subviews.
7. Replace flat pagination with page or section models.
8. Simplify row structure.
9. Remove per-row type erasure.
10. Reduce closure-heavy row inputs.
11. Add explicit capture lists where closures are necessary.
12. Replace unnecessary custom bindings with key-path bindings.
13. Use `Equatable` only when visual equality is clear.
14. Add profiling or debug probes to validate the hypothesis.
15. Consider UIKit only for genuinely hot paths.

## 48. Agent Response Format for Code Reviews

When reviewing code, use this structure:

1. Main issue
2. Why it matters in SwiftUI
3. Risk level
4. Suggested refactor
5. Before/after code
6. What to measure if confirmation is needed
7. Expected effect

Keep explanations concrete. Avoid generic statements like “SwiftUI is slow.”

## 49. Agent Response Format for Profiling Requests

When the user asks for profiling help, use this structure:

1. Reproducible scenario
2. Hypothesis
3. Tool choice
4. Exact command or local steps when possible
5. What to look for
6. How to interpret findings
7. Refactor candidates
8. How to compare before/after

Never claim measurements that were not actually taken.

## 50. Final Principle

SwiftUI performance improves when code makes change locality obvious.

Optimize for:

* stable identity
* narrow dependencies
* cheap rendering
* predictable structure
* local state ownership
* lightweight row inputs
* explicit page boundaries
* clear async lifecycle
* measured validation when needed
