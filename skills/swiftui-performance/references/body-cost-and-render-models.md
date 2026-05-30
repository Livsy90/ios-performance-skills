# Observation and Dependencies

Use this reference when a SwiftUI performance task involves Observation, `ObservableObject`, `@Published`, broad model dependencies, computed properties that read many fields, environment dependencies, or splitting screens into dependency islands.

This reference is about dependency scope. It should not duplicate identity ownership rules, expensive `body` work rules, list pagination rules, or profiling workflows unless those topics are directly needed to explain dependency behavior.

## Core Rule

Review what the view reads, not only what the view stores.

A SwiftUI performance issue often starts when a large view reads more state than it visually needs. The result is not always a slow operation by itself. The problem is that unrelated changes can invalidate a larger part of the view tree than necessary.

When reviewing dependencies, ask:

- Which observable properties are read by this view?
- Are those reads happening in a large parent view or in a small child view?
- Does a computed property hide reads from several unrelated fields?
- Does an environment value get read higher than necessary?
- Are unrelated update frequencies mixed into one model?
- Is the model using object-level invalidation or property-level observation?

## `ObservableObject` and `@Published`

With `ObservableObject`, a view that observes the object through `@ObservedObject`, `@StateObject`, or `@EnvironmentObject` is commonly invalidated when the object publishes a change.

This means a broad model can make small updates expensive to reason about:

```swift
final class PortfolioModel: ObservableObject {
    @Published var userDisplayName = ""
    @Published var holdings: [HoldingRowModel] = []
    @Published var isRefreshing = false
    @Published var selectedFilter: Filter = .all
}

struct PortfolioScreen: View {
    @ObservedObject var model: PortfolioModel

    var body: some View {
        VStack {
            PortfolioHeader(name: model.userDisplayName)
            HoldingsList(rows: model.holdings)
            RefreshFooter(isRefreshing: model.isRefreshing)
        }
    }
}
```

If `isRefreshing` changes, the screen observes the same object that also owns `holdings` and `userDisplayName`. The exact amount of work still depends on the view structure, but the dependency boundary is broad.

Prefer narrower observable boundaries when the screen is large or frequently updated:

```swift
final class PortfolioHeaderModel: ObservableObject {
    @Published var userDisplayName = ""
}

final class HoldingsModel: ObservableObject {
    @Published var rows: [HoldingRowModel] = []
}

final class RefreshStateModel: ObservableObject {
    @Published var isRefreshing = false
}

struct PortfolioScreen: View {
    let headerModel: PortfolioHeaderModel
    let holdingsModel: HoldingsModel
    let refreshStateModel: RefreshStateModel

    var body: some View {
        VStack {
            PortfolioHeader(model: headerModel)
            HoldingsList(model: holdingsModel)
            RefreshFooter(model: refreshStateModel)
        }
    }
}
```

Do not split models mechanically. Split when the model mixes data with different update frequencies, different owners, or different UI regions.

## Observation and Property Reads

With Observation, SwiftUI can form dependencies from observable property reads. A view that reads one observable property does not need to become dependent on every property in the model.

Example:

```swift
@Observable
final class PortfolioModel {
    var userDisplayName = ""
    var holdings: [HoldingRowModel] = []
    var isRefreshing = false
}

struct PortfolioHeader: View {
    let model: PortfolioModel

    var body: some View {
        Text(model.userDisplayName)
    }
}
```

`PortfolioHeader` reads `userDisplayName`, so that is the dependency that matters for this view.

However, Observation does not make broad composition free. If a large parent view reads many properties, the parent becomes dependent on many properties:

```swift
struct PortfolioScreen: View {
    let model: PortfolioModel

    var body: some View {
        VStack {
            PortfolioHeader(name: model.userDisplayName)
            HoldingsList(rows: model.holdings)
            RefreshFooter(isRefreshing: model.isRefreshing)
        }
    }
}
```

Prefer moving property reads into the smallest subview that needs them:

```swift
struct PortfolioScreen: View {
    let model: PortfolioModel

    var body: some View {
        VStack {
            PortfolioHeader(model: model)
            HoldingsList(model: model)
            RefreshFooter(model: model)
        }
    }
}

struct PortfolioHeader: View {
    let model: PortfolioModel

    var body: some View {
        Text(model.userDisplayName)
    }
}

struct HoldingsList: View {
    let model: PortfolioModel

    var body: some View {
        List(model.holdings) { row in
            HoldingRow(row: row)
        }
    }
}

struct RefreshFooter: View {
    let model: PortfolioModel

    var body: some View {
        if model.isRefreshing {
            ProgressView()
        }
    }
}
```

This pattern is useful with Observation because each child reads a narrower slice. Do not apply the same pattern blindly to `ObservableObject`; passing one `ObservableObject` into many children often keeps object-level invalidation broad.

## Passing Values vs Passing Observable Models

There is no universal rule that says “always pass values” or “always pass the model.”

Prefer passing plain values when:

- the child is simple and purely visual
- the child only needs one or two immutable values
- the parent already has to read those values for other reasons
- the model is an `ObservableObject` and passing the object would make the child observe unrelated `@Published` changes

Example:

```swift
struct AccountBadge: View {
    let title: String
    let isPremium: Bool

    var body: some View {
        Label(title, systemImage: isPremium ? "star.fill" : "person")
    }
}
```

Prefer passing an observable model when:

- the child owns a coherent region of the screen
- the child should form its own Observation dependencies
- passing values would force a large parent to read too many properties
- the child needs to mutate observable properties through a focused editing surface

Example:

```swift
struct AccountBadge: View {
    let model: AccountModel

    var body: some View {
        Label(model.title, systemImage: model.isPremium ? "star.fill" : "person")
    }
}
```

With Observation, this can keep the property reads inside `AccountBadge`. With `ObservableObject`, consider a smaller submodel or value inputs instead.

## Computed Properties Can Widen Dependencies

Computed properties are easy to underestimate. With Observation, a computed property can still read several stored properties internally. A view that reads the computed property may become dependent on everything that computed property reads.

Risky:

```swift
@Observable
final class DashboardModel {
    var userName = ""
    var unreadCount = 0
    var isPremium = false
    var marketStatus: MarketStatus = .closed
    var featureFlags = DashboardFeatureFlags()

    var headerTitle: String {
        if featureFlags.showMarketStatus, marketStatus == .open {
            return "Welcome, \(userName) · Market open"
        }

        if isPremium {
            return "Welcome back, \(userName) · \(unreadCount) updates"
        }

        return "Welcome, \(userName)"
    }
}

struct DashboardHeader: View {
    let model: DashboardModel

    var body: some View {
        Text(model.headerTitle)
    }
}
```

`DashboardHeader` looks like it reads one value, but `headerTitle` reads `featureFlags`, `marketStatus`, `isPremium`, `userName`, and `unreadCount`.

Prefer computed properties whose dependency scope matches the UI region:

```swift
@Observable
final class DashboardModel {
    var userName = ""
    var unreadCount = 0
    var isPremium = false
    var marketStatus: MarketStatus = .closed
    var featureFlags = DashboardFeatureFlags()

    var greetingText: String {
        "Welcome, \(userName)"
    }

    var unreadText: String? {
        unreadCount > 0 ? "\(unreadCount) updates" : nil
    }

    var marketStatusText: String? {
        featureFlags.showMarketStatus && marketStatus == .open ? "Market open" : nil
    }
}
```

Then split the UI so each region reads only what it needs:

```swift
struct DashboardHeader: View {
    let model: DashboardModel

    var body: some View {
        VStack(alignment: .leading) {
            Text(model.greetingText)
            MarketStatusLine(model: model)
            UnreadUpdatesLine(model: model)
        }
    }
}

struct MarketStatusLine: View {
    let model: DashboardModel

    var body: some View {
        if let text = model.marketStatusText {
            Text(text)
        }
    }
}

struct UnreadUpdatesLine: View {
    let model: DashboardModel

    var body: some View {
        if let text = model.unreadText {
            Text(text)
        }
    }
}
```

This is not about avoiding computed properties. It is about avoiding convenient computed properties that hide broad, unrelated reads.

## Avoid “Screen State” Computed Properties That Hide Everything

A common performance smell is a single computed property that turns the whole model into one large enum or view state during rendering.

Risky:

```swift
@Observable
final class PaymentsModel {
    var accounts: [Account] = []
    var selectedAccountID: Account.ID?
    var recentPayments: [Payment] = []
    var limits: PaymentLimits?
    var isLoading = false
    var error: Error?

    var screenState: PaymentsScreenState {
        if isLoading { return .loading }
        if let error { return .failed(error) }
        return .content(
            accounts: accounts,
            selectedAccountID: selectedAccountID,
            recentPayments: recentPayments,
            limits: limits
        )
    }
}

struct PaymentsScreen: View {
    let model: PaymentsModel

    var body: some View {
        PaymentsContent(state: model.screenState)
    }
}
```

This can make every part of the screen depend on every field used to build `screenState`.

Prefer smaller state reads at region boundaries:

```swift
struct PaymentsScreen: View {
    let model: PaymentsModel

    var body: some View {
        VStack {
            LoadingOverlay(isVisible: model.isLoading)
            ErrorBanner(error: model.error)
            AccountPicker(model: model)
            RecentPaymentsList(model: model)
            LimitsFooter(model: model)
        }
    }
}
```

A full-screen enum state is still useful when the screen genuinely has mutually exclusive modes. It is risky when it becomes a convenience wrapper around many independent regions.

## Dependency Islands

A dependency island is a small view subtree that reads a focused slice of state and can update independently from unrelated regions.

Split screens by:

- update frequency
- state owner
- visual region
- interaction surface
- data source
- loading lifecycle
- animation behavior

Risky large body:

```swift
struct TradingDashboard: View {
    let model: TradingDashboardModel

    var body: some View {
        VStack {
            Text(model.accountName)
            Text(model.buyingPowerText)

            ForEach(model.watchlist) { quote in
                QuoteRow(row: quote)
            }

            if model.isRefreshing {
                ProgressView()
            }

            Button("Reload") {
                model.reload()
            }
        }
    }
}
```

Prefer dependency-focused subviews:

```swift
struct TradingDashboard: View {
    let model: TradingDashboardModel

    var body: some View {
        VStack {
            AccountSummary(model: model)
            WatchlistPanel(model: model)
            RefreshIndicator(model: model)
            ReloadButton(model: model)
        }
    }
}

struct AccountSummary: View {
    let model: TradingDashboardModel

    var body: some View {
        VStack(alignment: .leading) {
            Text(model.accountName)
            Text(model.buyingPowerText)
        }
    }
}

struct WatchlistPanel: View {
    let model: TradingDashboardModel

    var body: some View {
        ForEach(model.watchlist) { quote in
            QuoteRow(row: quote)
        }
    }
}

struct RefreshIndicator: View {
    let model: TradingDashboardModel

    var body: some View {
        if model.isRefreshing {
            ProgressView()
        }
    }
}
```

This is especially useful with Observation because property reads move into smaller `body` scopes.

## Extracted Computed Views Are Not Dependency Islands

Extracting a private computed view property can improve readability, but it does not necessarily create a separate dependency boundary.

Risky if the goal is dependency isolation:

```swift
struct TradingDashboard: View {
    let model: TradingDashboardModel

    var body: some View {
        VStack {
            accountSummary
            watchlist
        }
    }

    private var accountSummary: some View {
        VStack(alignment: .leading) {
            Text(model.accountName)
            Text(model.buyingPowerText)
        }
    }

    private var watchlist: some View {
        ForEach(model.watchlist) { quote in
            QuoteRow(row: quote)
        }
    }
}
```

Prefer separate `View` types when dependency isolation matters:

```swift
struct TradingDashboard: View {
    let model: TradingDashboardModel

    var body: some View {
        VStack {
            AccountSummary(model: model)
            WatchlistPanel(model: model)
        }
    }
}
```

Use computed view properties for readability. Use extracted view types for clearer dependency boundaries.

## Environment Dependencies

Environment values are dependencies too. Reading environment high in the tree can make a broad view depend on changes that only a small child actually needs.

Risky:

```swift
struct AppShell: View {
    @Environment(\.colorScheme) private var colorScheme
    @Environment(SessionModel.self) private var session

    var body: some View {
        VStack {
            Header(
                userName: session.userName,
                isPremium: session.isPremium,
                colorScheme: colorScheme
            )

            ContentArea()
            Footer(isSignedIn: session.isSignedIn)
        }
    }
}
```

Prefer reading environment values near the view that needs them:

```swift
struct AppShell: View {
    var body: some View {
        VStack {
            Header()
            ContentArea()
            Footer()
        }
    }
}

struct Header: View {
    @Environment(\.colorScheme) private var colorScheme
    @Environment(SessionModel.self) private var session

    var body: some View {
        HeaderContent(
            userName: session.userName,
            isPremium: session.isPremium,
            colorScheme: colorScheme
        )
    }
}

struct Footer: View {
    @Environment(SessionModel.self) private var session

    var body: some View {
        if session.isSignedIn {
            Text("Connected")
        }
    }
}
```

Do not move every environment read downward mechanically. Shared environment reads at a container boundary are fine when the container genuinely controls layout, theme, or routing for the whole subtree.

## EnvironmentObject vs Observation Environment

For legacy `ObservableObject` environment models, `@EnvironmentObject` commonly behaves like object-level observation. A change in one published property can update views observing the environment object, even if the view only uses a different property.

For Observation-based environment models, a view can read an observable model from the environment and form dependencies from the properties it reads.

The same design rule still applies:

- avoid reading a broad environment model in a large root view if only children need it
- avoid putting unrelated app-wide state into one environment object
- prefer focused environment models for focused domains
- keep frequently changing values away from huge root-level dependencies

Risky:

```swift
final class AppEnvironmentModel: ObservableObject {
    @Published var session: Session?
    @Published var unreadCount = 0
    @Published var currentTheme: Theme = .system
    @Published var activeExperimentIDs: Set<String> = []
    @Published var networkStatus: NetworkStatus = .online
}
```

Prefer smaller environment models when update frequency and ownership differ:

```swift
final class SessionStore: ObservableObject {
    @Published var session: Session?
}

final class NotificationBadgeStore: ObservableObject {
    @Published var unreadCount = 0
}

final class NetworkStatusStore: ObservableObject {
    @Published var status: NetworkStatus = .online
}
```

## `@Bindable` and Editing Dependencies

Use `@Bindable` when a view needs to create bindings to mutable properties of an Observation model.

Good focused editing surface:

```swift
@Observable
final class ProfileModel {
    var displayName = ""
    var isPublic = true
    var avatarURL: URL?
}

struct ProfileEditor: View {
    @Bindable var model: ProfileModel

    var body: some View {
        Form {
            TextField("Display name", text: $model.displayName)
            Toggle("Public profile", isOn: $model.isPublic)
        }
    }
}
```

Avoid making a large parent `@Bindable` just because one small child edits a field:

```swift
struct ProfileScreen: View {
    @Bindable var model: ProfileModel

    var body: some View {
        VStack {
            ProfileHeader(model: model)
            ProfileEditor(model: model)
            ProfileActivity(model: model)
        }
    }
}
```

Prefer keeping binding creation close to the editing region:

```swift
struct ProfileScreen: View {
    let model: ProfileModel

    var body: some View {
        VStack {
            ProfileHeader(model: model)
            ProfileEditor(model: model)
            ProfileActivity(model: model)
        }
    }
}
```

`@Bindable` is not a performance problem by itself. The review question is whether binding and mutation capability are introduced at the smallest view that needs them.

## `@ObservationIgnored`

For `@Observable` models, use `@ObservationIgnored` only for values that should not participate in UI observation.

Good candidates:

- cached helpers
- formatter caches
- task handles
- non-visual services
- loggers
- transient bookkeeping not meant to refresh UI

Example:

```swift
@Observable
final class SearchModel {
    var query = ""
    var results: [SearchResultRowModel] = []

    @ObservationIgnored
    private var currentSearchTask: Task<Void, Never>?
}
```

Do not use `@ObservationIgnored` to hide UI-relevant state changes. If the UI should update when a value changes, that value should remain observable or be represented by another observable property.

## Migration Guidance

Do not recommend migrating every `ObservableObject` to Observation automatically.

Consider Observation when:

- the deployment target and architecture support it
- broad `@Published` invalidation is causing unnecessary updates
- views can be split so property-level reads are useful
- the model does not depend heavily on Combine-specific observation behavior
- bindings can be localized with `@Bindable`

Keep or improve `ObservableObject` when:

- the app must support older OS versions without an Observation-based path
- the model is already integrated with Combine pipelines
- object-level invalidation is acceptable for the screen size
- the simpler fix is splitting a large object into focused smaller objects

When migrating, avoid keeping old assumptions:

- `@Published` is not needed for properties inside an `@Observable` model
- property-level observation helps only if views read properties at narrow boundaries
- broad computed properties can still create broad dependencies
- environment-based state can still be overused
- Observation reduces unnecessary updates, but it does not make expensive rendering, large lists, or poor identity free

## Review Checklist

When reviewing observation and dependency scope, check:

- Does a large parent view read many observable properties directly?
- Is the model using `ObservableObject` where every `@Published` change can affect a broad screen?
- Would Observation actually narrow reads in this structure, or would the parent still read everything?
- Are computed properties hiding reads from many unrelated fields?
- Is a full-screen enum state used for independent regions that could update separately?
- Are environment values read at the smallest useful boundary?
- Is one app-wide environment object mixing session, theme, network, badges, experiments, and routing state?
- Are extracted computed view properties being mistaken for dependency islands?
- Would separate `View` types create clearer dependency boundaries?
- Are `@Bindable` models introduced only where editing is needed?
- Is `@ObservationIgnored` used only for non-visual state?
- Are model splits based on ownership and update frequency rather than arbitrary file organization?

## Agent Response Pattern

When this reference is relevant, explain the finding in dependency terms:

```md
## Finding
This view reads `model.holdings`, `model.isRefreshing`, and `model.userDisplayName` in the same parent body.

## Why it matters
Those reads make the parent depend on several unrelated update sources. A refresh flag change can cause the parent body to be re-evaluated even though only the footer visually depends on that flag.

## Suggested refactor
Move the reads into smaller subviews: `PortfolioHeader`, `HoldingsList`, and `RefreshFooter`. With Observation, pass the model to those subviews so each child reads only the properties it needs. With `ObservableObject`, consider smaller submodels or value inputs to avoid broad object-level observation.

## Validation
If the screen is already suspected to be slow, temporarily log body updates or use the SwiftUI instrument to confirm that unrelated changes no longer update the large parent region.
```

Keep the recommendation concrete. Do not claim that Observation or splitting views improves performance unless the dependency path or measurement supports that conclusion.
