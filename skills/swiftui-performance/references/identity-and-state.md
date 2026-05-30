# Identity and State

Use this reference when reviewing SwiftUI code that involves structural identity, explicit `.id(...)`, `ForEach` identity, local state lifetime, `@State`, `@StateObject`, `@ObservedObject`, or owned observable models.

This reference is about identity and state lifetime. Keep Observation dependency granularity, broad model reads, list pagination strategy, closure-heavy rows, custom bindings, and profiling details in their dedicated references.

## Agent Goal

Help the user understand whether SwiftUI can preserve the right view identity and state across updates.

When reviewing code, answer these questions:

- Does this view keep the same identity across normal updates?
- Is local state attached to a stable identity?
- Are collection rows identified by stable data identity rather than position or temporary values?
- Is the model owned by the view, injected into the view, or owned elsewhere?
- Would a structural branch, explicit `.id(...)`, or unstable `ForEach` ID accidentally reset state?

Avoid vague claims such as "SwiftUI redraws everything." Prefer precise language about identity, lifetime, and dependency boundaries.

## 1. Core Mental Model

SwiftUI view values are temporary descriptions. State is not stored inside the view value itself. SwiftUI preserves state by associating it with identity in the view hierarchy.

A view's identity usually comes from:

- its structural position in the view tree
- the concrete view type at that position
- explicit identity provided by APIs such as `ForEach` or `.id(...)`

When identity changes, SwiftUI may treat the view as a different view. State attached to the previous identity can be discarded and new state can be created.

Review identity issues before suggesting lower-level optimization. Many SwiftUI performance and correctness bugs come from state being attached to the wrong lifetime boundary.

## 2. Structural Identity

SwiftUI normally derives identity from structure. Two branches that look visually similar can still represent different structural identities if they produce different view types or different positions in the hierarchy.

Risky when the same conceptual view appears in different branches:

```swift
struct PaymentStatusView: View {
    let isPending: Bool

    var body: some View {
        if isPending {
            StatusCard(title: "Processing")
                .transition(.opacity)
        } else {
            StatusCard(title: "Complete")
                .transition(.opacity)
        }
    }
}
```

This can be fine when the UI really has two separate states. But if the view should keep the same local state and only change values, prefer one stable structure:

```swift
struct PaymentStatusView: View {
    let isPending: Bool

    var body: some View {
        StatusCard(title: isPending ? "Processing" : "Complete")
            .transition(.opacity)
    }
}
```

Do not mechanically remove all branches. Branches are normal SwiftUI. Flag them only when they accidentally reset state, create unstable layout, or make a hot repeated view structurally unpredictable.

## 3. Explicit `.id(...)`

Use `.id(...)` only when creating a deliberate identity boundary.

Good reasons:

- reset local state intentionally
- make a detail/editor view start fresh for a different entity
- define a scroll target
- force a known lifecycle boundary after a meaningful identity change

Risky:

```swift
InvoiceEditor(invoice: invoice)
    .id(UUID())
```

This creates a new identity every time the body is evaluated. It can reset local state, restart lifecycle work, and make update behavior difficult to reason about.

Prefer stable domain identity when a reset is intentional:

```swift
InvoiceEditor(invoice: invoice)
    .id(invoice.id)
```

Only use this if changing `invoice.id` should discard editor-local state such as draft fields, focus, validation state, or temporary selections.

Avoid using `.id(...)` as a generic refresh workaround. First check whether the real issue is stale derived state, wrong ownership, missing dependency updates, or an async lifecycle bug.

## 4. `ForEach` Identity

Dynamic collections need stable, unique, cheap identifiers. Row identity should come from the underlying data, not from the current array position or a temporary value.

Risky for mutable or reorderable collections:

```swift
ForEach(Array(messages.enumerated()), id: \.offset) { _, message in
    MessageRow(message: message)
}
```

If items are inserted, removed, filtered, or reordered, offsets can change for existing rows. SwiftUI may associate old row state with the wrong item or recreate rows unnecessarily.

Prefer identity from the data:

```swift
ForEach(messages) { message in
    MessageRow(message: message)
}
```

or:

```swift
ForEach(messages, id: \.messageID) { message in
    MessageRow(message: message)
}
```

Avoid IDs that allocate or change on every access:

```swift
struct MessageRowModel: Identifiable {
    var id: UUID { UUID() }
    let title: String
}
```

Prefer stored identity:

```swift
struct MessageRowModel: Identifiable {
    let id: Message.ID
    let title: String
}
```

Use `id: \.self` only when the value is truly unique and stable for the lifetime of the collection. It is usually fine for a fixed list of unique strings or enum values. It is risky for duplicate values, mutable values, or values whose equality changes when visible content changes.

## 5. Local State Lifetime

Use `@State` for local value state owned by a view identity.

Good candidates:

- expansion state inside a component
- local draft text
- focus-related UI flags
- temporary selection inside a picker-like component
- small animation state
- presentation toggles owned by the current view

Example:

```swift
struct TransferNoteField: View {
    @State private var note = ""

    var body: some View {
        TextField("Note", text: $note)
    }
}
```

Keep `@State` private whenever possible. External code should not depend on a child view's private state storage.

Do not use `@State` as an accidental cache of parent input:

```swift
struct UserNameView: View {
    let userName: String
    @State private var displayedName: String

    init(userName: String) {
        self.userName = userName
        _displayedName = State(initialValue: userName)
    }

    var body: some View {
        Text(displayedName)
    }
}
```

This captures the initial value. Later changes to `userName` do not automatically update `displayedName`.

Prefer deriving directly when no local editing is needed:

```swift
struct UserNameView: View {
    let userName: String

    var body: some View {
        Text(userName)
    }
}
```

If local editing is needed, make the ownership explicit:

```swift
struct EditableUserNameView: View {
    let initialName: String
    @State private var draftName: String

    init(initialName: String) {
        self.initialName = initialName
        _draftName = State(initialValue: initialName)
    }

    var body: some View {
        TextField("Name", text: $draftName)
    }
}
```

In this case, the snapshot is intentional because `draftName` is local draft state.

## 6. Put State at the Smallest Stable Owner

State should live at the smallest stable boundary that owns it.

Risky: a parent owns unrelated local flags for many child regions.

```swift
struct WalletScreen: View {
    @State private var isSearchFocused = false
    @State private var isCardExpanded = false
    @State private var selectedFilter = Filter.all
    @State private var draftNickname = ""

    var body: some View {
        WalletContent(
            isSearchFocused: $isSearchFocused,
            isCardExpanded: $isCardExpanded,
            selectedFilter: $selectedFilter,
            draftNickname: $draftNickname
        )
    }
}
```

Prefer local ownership when the state belongs to a specific component:

```swift
struct WalletScreen: View {
    @State private var selectedFilter = Filter.all

    var body: some View {
        VStack {
            WalletSearchField()
            BalanceCard()
            TransactionFilter(selection: $selectedFilter)
            NicknameEditor()
        }
    }
}
```

This does not guarantee fewer updates by itself, but it makes lifetime and dependency boundaries easier to reason about.

Lift state up when multiple components need the same source of truth or when the parent coordinates the behavior. Do not push state down so far that synchronization becomes unclear.

## 7. `@StateObject` vs `@ObservedObject`

Use `@StateObject` when a SwiftUI view creates and owns an `ObservableObject`.

Risky:

```swift
struct RatesView: View {
    @ObservedObject private var model = RatesModel()

    var body: some View {
        RatesContent(model: model)
    }
}
```

The view declares ownership but uses a wrapper meant for externally owned observable objects.

Prefer:

```swift
struct RatesView: View {
    @StateObject private var model = RatesModel()

    var body: some View {
        RatesContent(model: model)
    }
}
```

Use `@ObservedObject` when the object is owned elsewhere and injected:

```swift
struct RatesContent: View {
    @ObservedObject var model: RatesModel

    var body: some View {
        List(model.rows) { row in
            RateRow(row: row)
        }
    }
}
```

Do not create an owned model as a plain stored property in a view:

```swift
struct RatesView: View {
    private let model = RatesModel()

    var body: some View {
        RatesContent(model: model)
    }
}
```

SwiftUI view values can be recreated. A plain stored reference does not express SwiftUI-managed lifetime.

## 8. Owned Observable Models with Observation

For iOS 17 and later, an `@Observable` model owned by a view can be stored with `@State`.

```swift
@Observable
final class RatesModel {
    var rows: [RateRowModel] = []
    var isRefreshing = false
}

struct RatesView: View {
    @State private var model = RatesModel()

    var body: some View {
        RatesContent(model: model)
    }
}
```

Use this when the view owns the model's lifetime.

If a parent owns the model, inject it into the child instead of creating it again:

```swift
struct RatesContent: View {
    let model: RatesModel

    var body: some View {
        List(model.rows) { row in
            RateRow(row: row)
        }
    }
}
```

This reference covers ownership and lifetime. Detailed Observation dependency behavior belongs in `observation-and-dependencies.md`.

## 9. State Reset Diagnostics

Suspect accidental identity changes when the user reports:

- text fields lose input unexpectedly
- focus disappears during updates
- scroll position resets without intent
- rows lose expansion or selection state after filtering
- async `.task` work restarts repeatedly
- animations restart during unrelated updates
- row-local state appears attached to the wrong item after insertion or deletion

Check:

- Is `.id(UUID())` or another unstable ID used?
- Does a `ForEach` use offsets, indices, or mutable values as identity?
- Does a row model compute a new ID on every access?
- Does conditional structure replace one stateful view with another?
- Is local state initialized from parent input and then expected to track later changes?
- Is an owned model created with `@ObservedObject` or a plain stored property?
- Is state owned too high or too low in the tree?

## 10. Safer Review Language

Use precise wording:

```md
This `ForEach` identifies rows by offset. If the array is filtered, reordered, or prepended, existing row state can become associated with a different item. Use a stable domain ID instead.
```

```md
This `.id(UUID())` creates a fresh identity on every update. That can reset local state and restart lifecycle-bound work. Use a stable domain ID only if changing that ID should intentionally reset the subtree.
```

```md
This view creates its own observable model, so `@StateObject` is the correct ownership wrapper for `ObservableObject`. Use `@ObservedObject` only when the model is injected from an external owner.
```

```md
This `@State` value is initialized from a parent input, so it behaves like an initial snapshot. If the view should always show the latest parent value, derive it directly from the input instead.
```

## 11. Do Not Overcorrect

Avoid these overcorrections:

- Do not add `.id(...)` everywhere. Explicit identity is a tool, not a default requirement.
- Do not replace every branch with value-based modifiers. Structural branching is fine when it expresses genuinely different UI.
- Do not move all state to the parent. Local state is often better for local UI behavior.
- Do not move all state to children. Shared source of truth still belongs at the coordinating owner.
- Do not claim `@StateObject` is faster than `@ObservedObject`. The main distinction is ownership and lifetime.
- Do not claim Observation removes the need to think about identity. Observation changes dependency tracking, not identity rules.

## 12. Minimal Checklist

Before finishing an identity/state review, verify:

- IDs are stable, unique, and cheap.
- `.id(...)` is intentional and not used as a refresh hack.
- Mutable collections do not use offset or index identity unless the order and membership are fixed.
- Local `@State` is attached to a stable structural position.
- `@State` initialized from input is either an intentional snapshot or replaced with derived input.
- `ObservableObject` created by the view uses `@StateObject`.
- `ObservableObject` owned elsewhere uses `@ObservedObject` or is passed through without claiming ownership.
- Owned `@Observable` models use `@State` when the deployment target and architecture support Observation.
- State is neither lifted too high nor pushed too low for the behavior being modeled.
