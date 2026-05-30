# Closures, Bindings, and Equatable Views

Use this reference when reviewing SwiftUI code that involves stored closures in views, capture lists, action handlers, gestures, custom `Binding(get:set:)`, key-path bindings, `.equatable()`, or `Equatable` view inputs.

The goal is not to remove every closure or make every view equatable. The goal is to keep visual inputs, action routing, and equality boundaries easy to reason about, especially inside large or frequently updating view trees.

## Core Principle

Separate visual data from non-visual behavior when a view is repeated many times or updated frequently.

A SwiftUI view value may contain both:

- visual inputs, such as text, flags, numbers, colors, and render models
- behavioral inputs, such as closures, gesture actions, menu actions, and custom binding closures

Visual inputs are usually easy to compare and reason about. Behavioral inputs are often harder to compare, easier to over-capture, and more likely to hide dependencies. This matters most in hot paths such as `List`, `LazyVStack`, complex rows, swipe actions, menus, and gesture-heavy components.

## Review Procedure

When reviewing closure-heavy SwiftUI code, check:

1. Is the closure part of a small static view or a hot repeated view?
2. Is the closure stored as a property of a visual row?
3. Does the closure capture the whole parent view implicitly?
4. Does the closure capture a large domain value when only an ID is needed?
5. Are multiple non-visual actions mixed into a row that otherwise has simple visual input?
6. Would a key-path binding express the same state relationship more clearly?
7. Is `.equatable()` used only for cheap, complete, visual equality?
8. Could the refactor make update locality clearer without adding unnecessary architecture?

Do not flag every closure. Closures are normal in SwiftUI. Focus on repeated views, broad captures, unstable behavior inputs, and cases where code structure makes updates harder to reason about.

## Stored Closures in Repeated Views

A view that stores many closures can become harder to reason about because the row value contains non-visual behavior in addition to visual data.

Risky in a large or frequently updating collection:

```swift
struct PaymentRow: View {
    let row: PaymentRowModel
    let onOpen: () -> Void
    let onRetry: () -> Void
    let onCancel: () -> Void

    var body: some View {
        HStack {
            VStack(alignment: .leading) {
                Text(row.title)
                Text(row.subtitle)
            }

            Spacer()

            Text(row.amountText)
        }
        .contentShape(Rectangle())
        .onTapGesture(perform: onOpen)
        .swipeActions {
            Button("Retry", action: onRetry)
            Button("Cancel", role: .destructive, action: onCancel)
        }
    }
}
```

And the parent creates new closures while building every row:

```swift
ForEach(model.payments) { payment in
    PaymentRow(
        row: payment,
        onOpen: {
            model.openPayment(payment.id)
        },
        onRetry: {
            model.retryPayment(payment.id)
        },
        onCancel: {
            model.cancelPayment(payment.id)
        }
    )
}
```

This is not automatically wrong. It becomes suspicious when the collection is large, the parent updates often, rows are already expensive, or unnecessary row updates are suspected.

Prefer keeping the visual row focused on visual input, and route actions at a stable boundary when practical:

```swift
struct PaymentRow: View {
    let row: PaymentRowModel

    var body: some View {
        HStack {
            VStack(alignment: .leading) {
                Text(row.title)
                Text(row.subtitle)
            }

            Spacer()

            Text(row.amountText)
        }
    }
}
```

```swift
ForEach(model.payments) { payment in
    PaymentRow(row: payment)
        .contentShape(Rectangle())
        .onTapGesture { [model, id = payment.id] in
            model.openPayment(id)
        }
        .swipeActions {
            Button("Retry") { [model, id = payment.id] in
                model.retryPayment(id)
            }

            Button("Cancel", role: .destructive) { [model, id = payment.id] in
                model.cancelPayment(id)
            }
        }
}
```

This does not remove closures from the SwiftUI tree. The narrower goal is to keep the row's stored input visual, make row values easier to compare, and avoid accidentally capturing more state than the action needs.

## Capture Lists

Closures created inside `body` can accidentally capture the whole parent view value. In repeated content, prefer capture lists that make dependencies explicit and small.

Risky:

```swift
ForEach(model.transfers) { transfer in
    TransferRow(
        row: transfer,
        onSelect: {
            model.selectTransfer(transfer.id)
        }
    )
}
```

Prefer:

```swift
ForEach(model.transfers) { transfer in
    TransferRow(
        row: transfer,
        onSelect: { [model, id = transfer.id] in
            model.selectTransfer(id)
        }
    )
}
```

Prefer capturing:

- a stable model reference
- a stable service or action handler
- a stable row ID
- a small immutable value needed by the action

Avoid capturing:

- the whole parent view implicitly
- a large domain model when only an ID is needed
- a mutable row object when identity is enough
- a broad handler container when a narrower dependency is available
- values recomputed during every render

Capture lists do not make closures automatically diffable. They reduce accidental captures and make action dependencies easier to audit.

## Passing IDs Instead of Full Models

For row actions, prefer passing a stable ID when the action only needs identity.

Risky:

```swift
Button("Open") {
    model.open(invoice)
}
```

Prefer:

```swift
Button("Open") { [model, id = invoice.id] in
    model.openInvoice(id)
}
```

Passing the full model is fine when the action genuinely needs the full immutable value. Do not mechanically replace all values with IDs if doing so makes the code less correct or forces extra lookups.

## Single Action Router Pattern

When a child must report interactions to a parent, prefer a compact action surface over many independent closure properties.

Useful for medium-complex components:

```swift
enum CardAction {
    case open(Card.ID)
    case favorite(Card.ID)
    case dismiss(Card.ID)
}
```

```swift
struct CardRow: View {
    let row: CardRowModel
    let send: (CardAction) -> Void

    var body: some View {
        HStack {
            Text(row.title)
            Spacer()
            Button("Favorite") {
                send(.favorite(row.id))
            }
        }
        .contentShape(Rectangle())
        .onTapGesture {
            send(.open(row.id))
        }
    }
}
```

This still stores a closure, so it is not a magic performance fix. It can reduce API noise and make behavior easier to inspect. For very hot rows, also consider whether actions can be attached outside the purely visual row.

## Gestures, Menus, and Swipe Actions

Gestures, menus, context menus, and swipe actions are interaction surfaces. In large collections, treat them as part of row complexity.

Review whether:

- every row really needs the interaction surface
- actions capture only stable IDs or narrow dependencies
- menu content is lightweight
- destructive actions have clear confirmation or state handling
- repeated action builders hide expensive work
- gesture state is owned locally where possible

Risky:

```swift
FeedRow(row: row)
    .contextMenu {
        ForEach(model.availableActions(for: row)) { action in
            Button(action.title) {
                model.perform(action, on: row)
            }
        }
    }
```

Prefer preparing lightweight action models outside the hot row builder when action computation is non-trivial:

```swift
struct FeedRowModel: Identifiable, Equatable {
    let id: FeedItem.ID
    let title: String
    let subtitle: String
    let availableActions: [FeedActionModel]
}
```

```swift
FeedRow(row: row)
    .contextMenu {
        ForEach(row.availableActions) { action in
            Button(action.title) { [model, itemID = row.id, actionID = action.id] in
                model.perform(actionID, on: itemID)
            }
        }
    }
```

Do not add indirection just because a row has one tap gesture. Apply this guidance when repeated interaction builders become heavy or hard to reason about.

## Custom Bindings

Prefer key-path bindings when no transformation is needed.

Good:

```swift
Toggle("Push notifications", isOn: $settings.pushNotificationsEnabled)
```

Risky as a default style:

```swift
Toggle(
    "Push notifications",
    isOn: Binding(
        get: { settings.pushNotificationsEnabled },
        set: { settings.pushNotificationsEnabled = $0 }
    )
)
```

A custom `Binding(get:set:)` is not automatically wrong. It often introduces fresh closures, broader captures, and hidden transformation logic inside rendering code. Use it when the binding really needs transformation, validation, optional handling, clamping, logging, or routing.

Valid use case:

```swift
TextField(
    "Limit",
    value: Binding(
        get: { draft.dailyLimit ?? 0 },
        set: { newValue in
            draft.dailyLimit = newValue == 0 ? nil : newValue
        }
    ),
    format: .number
)
```

When custom binding logic grows, move the behavior to a named helper or model method so the view does not hide business rules inside `body`.

## Key-Path Bindings with Observation

For Observation-based models, prefer `@Bindable` when a view needs editable bindings into an `@Observable` model.

```swift
@Observable
final class NotificationSettings {
    var pushEnabled = false
    var weeklySummaryEnabled = true
}
```

```swift
struct NotificationSettingsForm: View {
    @Bindable var settings: NotificationSettings

    var body: some View {
        Form {
            Toggle("Push", isOn: $settings.pushEnabled)
            Toggle("Weekly summary", isOn: $settings.weeklySummaryEnabled)
        }
    }
}
```

This keeps the binding relationship explicit and avoids replacing simple key-path access with custom closure bindings.

## Binding Red Flags

Flag these patterns in performance-sensitive SwiftUI code:

```swift
Binding(
    get: { model.expensiveDerivedValue },
    set: { model.apply($0) }
)
```

```swift
Binding(
    get: { formatter.string(from: value) },
    set: { text in model.update(text) }
)
```

```swift
Binding(
    get: { largeParentModel.items[index].isEnabled },
    set: { largeParentModel.items[index].isEnabled = $0 }
)
```

The issue is not the `Binding` type itself. The issue is hidden work, broad captures, index-based mutation risk, or transformation logic running on a hot rendering path.

## Equatable Views

Use `Equatable` only when a view has clear, cheap, visual inputs and its body is expensive enough to justify the equality check.

Good candidate:

```swift
struct RateBadge: View, Equatable {
    let currencyCode: String
    let valueText: String
    let direction: Direction

    static func == (lhs: Self, rhs: Self) -> Bool {
        lhs.currencyCode == rhs.currencyCode &&
        lhs.valueText == rhs.valueText &&
        lhs.direction == rhs.direction
    }

    var body: some View {
        HStack(spacing: 6) {
            Text(currencyCode)
            Text(valueText)
            DirectionIcon(direction: direction)
        }
    }
}
```

```swift
RateBadge(
    currencyCode: row.currencyCode,
    valueText: row.rateText,
    direction: row.direction
)
.equatable()
```

Do not use `.equatable()` as a bandage for broad invalidation. First try to narrow dependencies, stabilize identity, and remove expensive work from `body`.

## Equatable Input Rules

Include in equality:

- all visual values that affect rendering
- style flags that change visible output
- values that change layout, text, color, images, accessibility labels, or visibility

Avoid including:

- non-visual closures
- services
- model objects whose internal changes are not represented by the compared values
- large arrays when equality is more expensive than recomputing the view
- volatile values that change every update

Do not exclude a value from equality if changing it should visibly update the view.

Risky:

```swift
struct AccountSummaryCard: View, Equatable {
    let title: String
    let balanceText: String
    let isLoading: Bool

    static func == (lhs: Self, rhs: Self) -> Bool {
        lhs.title == rhs.title &&
        lhs.balanceText == rhs.balanceText
        // isLoading is missing even though it affects visible UI.
    }

    var body: some View {
        VStack {
            Text(title)
            Text(balanceText)

            if isLoading {
                ProgressView()
            }
        }
    }
}
```

Prefer complete visual equality:

```swift
static func == (lhs: Self, rhs: Self) -> Bool {
    lhs.title == rhs.title &&
    lhs.balanceText == rhs.balanceText &&
    lhs.isLoading == rhs.isLoading
}
```

## Equatable and Closures

Do not make action closures part of `Equatable` comparison. Swift closures generally do not have meaningful value equality.

Also avoid using `.equatable()` on a view whose behavior depends on changing closure values. If equality says the view is unchanged, but the action closure changed, the code becomes harder to reason about.

Prefer a purely visual equatable row plus actions attached outside that row:

```swift
struct ContactRow: View, Equatable {
    let row: ContactRowModel

    var body: some View {
        HStack {
            Text(row.name)
            Spacer()
            Text(row.statusText)
        }
    }
}
```

```swift
ContactRow(row: row)
    .equatable()
    .contentShape(Rectangle())
    .onTapGesture { [model, id = row.id] in
        model.openContact(id)
    }
```

This keeps equality focused on visible input and keeps behavior separate from the equality boundary.

## Prefer Equatable Render Models

For complex rows, it is often better to make the render model `Equatable` than to manually compare many view properties.

```swift
struct TransactionRowModel: Identifiable, Equatable {
    let id: Transaction.ID
    let title: String
    let subtitle: String
    let amountText: String
    let isPending: Bool
}
```

```swift
struct TransactionRow: View, Equatable {
    let row: TransactionRowModel

    var body: some View {
        HStack {
            VStack(alignment: .leading) {
                Text(row.title)
                Text(row.subtitle)
            }

            Spacer()

            Text(row.amountText)
        }
        .opacity(row.isPending ? 0.6 : 1)
    }
}
```

This works best when the render model is already prepared outside rendering and contains only display-ready values.

## When Not to Use `.equatable()`

Avoid `.equatable()` when:

- the view body is cheap
- equality is expensive
- inputs are large collections
- the view reads broad external state internally
- equality would omit values that affect visible output
- the view has hidden dependencies through environment, model references, or custom bindings
- the performance issue is actually identity, layout, drawing, or main-actor work

`Equatable` is a local optimization boundary, not a substitute for good dependency design.

## Risk Levels

Use low risk when:

- a closure appears in a small static view
- a custom binding performs a tiny necessary transformation
- `.equatable()` is used with complete and cheap visual equality

Use medium risk when:

- repeated rows store several action closures
- custom bindings capture a parent model in a large form or list
- closure capture lists are missing in a frequently updating parent
- `.equatable()` compares multiple values manually and could become incomplete over time

Use high risk when:

- row closures capture large mutable models or whole parent views in a large collection
- custom bindings perform expensive derived reads or index-based mutations in repeated content
- `.equatable()` omits visible state
- `.equatable()` is used to hide broad invalidation instead of fixing dependencies
- action builders do non-trivial work for every row during rendering

## Suggested Refactoring Order

For closure, binding, and equatable issues, prefer this order:

1. Keep visual row data separate from action behavior.
2. Capture stable IDs and narrow dependencies in closures.
3. Replace unnecessary custom bindings with key-path bindings.
4. Move binding transformation logic out of `body` when it grows.
5. Reduce multiple closure properties to a smaller action surface when useful.
6. Use `Equatable` only after visual inputs are clear and equality is cheap.
7. Validate with profiling or temporary probes only when unnecessary updates remain suspected.

## Agent Wording

Prefer:

```md
This row stores several non-visual action closures. In a large list, that can make row updates harder to reason about. Keep the row's stored input visual, attach actions at the list boundary, and capture only the model reference plus the row ID.
```

Prefer:

```md
This custom binding is not automatically wrong, but it hides transformation logic inside the render path. If no transformation is needed, use a key-path binding. If transformation is required, keep the closure small and avoid expensive derived reads.
```

Prefer:

```md
`.equatable()` is useful only if equality covers all visible inputs and is cheaper than recomputing the body. Do not use it to mask broad invalidation or omit state that changes the UI.
```

Avoid:

```md
Closures always make SwiftUI rows redraw.
```

Avoid:

```md
Custom bindings are bad for performance.
```

Avoid:

```md
Add `.equatable()` to fix this list.
```
