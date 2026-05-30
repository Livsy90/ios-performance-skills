# Layout, Drawing, and Animation

Use this reference when a SwiftUI task involves modifier chains, `GeometryReader`, `PreferenceKey`, layout feedback, shadows, masks, blurs, `Canvas`, `TimelineView`, animation hitches, Core Animation, or drawing-heavy UI.

The goal is not to remove visual polish. The goal is to keep layout, drawing, and animation costs proportional to the UI surface, especially inside repeated or frequently updating content.

## Review Mindset

First classify the symptom or risk:

- layout work: geometry reads, custom measurement, preference propagation, nested layouts
- drawing work: many shapes, gradients, shadows, masks, blurs, charts, waveforms, decorative effects
- animation work: broad animated state changes, repeated updates during animation, expensive layout-affecting transitions
- compositing work: layered effects, clipping, transparency, materials, offscreen rendering candidates
- unrelated app work: main-thread CPU work that merely appears during a visual interaction

Do not assume every hitch is caused by SwiftUI diffing. For visible stutter, check whether the dominant cause is main-thread blocking, layout, drawing, compositing, state churn, image work, or animation scope.

Prefer precise language:

```md
This row performs a geometry read and applies a blur, mask, and shadow in repeated content. That may increase layout and compositing work during scrolling. Validate with Animation Hitches, Core Animation, Time Profiler, or the SwiftUI instrument if the issue is user-visible.
```

Avoid unsupported claims:

```md
This blur causes a 30 ms frame.
```

unless a trace, screen recording, metric, or benchmark shows it.

## Modifier Chains and View Hierarchy

Modifiers create additional view structure. Their order affects layout, drawing, hit testing, animation, and sometimes identity.

In hot paths, review long modifier chains carefully, especially when they include:

- `background`
- `overlay`
- `mask`
- `clipShape`
- `shadow`
- `blur`
- `drawingGroup`
- `compositingGroup`
- gestures
- animations
- geometry reads
- preference keys
- conditional wrappers

A long modifier chain is not automatically bad. It becomes suspicious when it is repeated many times, updates frequently, or hides layout/drawing work that should happen at a coarser boundary.

Risky in a large list:

```swift
struct PaymentRow: View {
    let row: PaymentRowModel

    var body: some View {
        HStack {
            Text(row.title)
            Spacer()
            Text(row.amountText)
        }
        .padding(16)
        .background(.ultraThinMaterial)
        .clipShape(RoundedRectangle(cornerRadius: 18))
        .overlay {
            RoundedRectangle(cornerRadius: 18)
                .stroke(.white.opacity(0.3))
        }
        .shadow(radius: row.isHighlighted ? 12 : 4)
        .blur(radius: row.isLocked ? 1.5 : 0)
        .animation(.easeInOut, value: row.isHighlighted)
    }
}
```

Prefer moving expensive decoration to a stable or coarser boundary when the visual design allows it:

```swift
struct PaymentRow: View {
    let row: PaymentRowModel

    var body: some View {
        HStack {
            Text(row.title)
            Spacer()
            Text(row.amountText)
        }
        .padding(16)
        .background(rowBackground)
    }

    private var rowBackground: some View {
        RoundedRectangle(cornerRadius: 18)
            .fill(row.isHighlighted ? Color.accentColor.opacity(0.12) : Color.secondary.opacity(0.08))
    }
}
```

If the visual effect is essential, keep it. The agent should suggest validating the cost rather than deleting design details blindly.

## Conditional Modifiers

Be careful with helper APIs that wrap views conditionally in repeated or frequently updating content.

Risky:

```swift
metricLabel
    .applyIf(row.isWarning) { view in
        view
            .padding(6)
            .background(.orange.opacity(0.2))
            .clipShape(Capsule())
    }
```

This can make the structure differ between states. That may be fine for genuinely different UI, but it is unnecessary for simple visual value changes.

Prefer value-based modifiers when the view remains conceptually the same:

```swift
metricLabel
    .padding(6)
    .background(row.isWarning ? Color.orange.opacity(0.2) : Color.clear)
    .clipShape(Capsule())
```

Use structural branching when the UI really has different structure, different children, or a different lifecycle.

## GeometryReader

`GeometryReader` is useful when a view genuinely needs container size, coordinate space, or measured geometry. It is not inherently wrong.

Use it carefully because geometry reads can couple layout to state updates and can make a repeated subtree more sensitive to size changes.

Common risks:

- placing a geometry reader inside every row of a long list
- storing frequently changing geometry values in broad observable state
- using geometry to drive layout that standard SwiftUI layout can express directly
- creating layout feedback loops where measurement changes state, which changes layout, which changes measurement
- using geometry as a generic workaround instead of narrowing the layout problem

Risky in repeated content:

```swift
List(cards) { card in
    GeometryReader { proxy in
        LoyaltyCardRow(
            card: card,
            availableWidth: proxy.size.width
        )
    }
}
```

Prefer reading geometry at a stable container boundary when every row needs the same container value:

```swift
struct LoyaltyCardList: View {
    let cards: [LoyaltyCardModel]

    var body: some View {
        GeometryReader { proxy in
            let availableWidth = proxy.size.width

            List(cards) { card in
                LoyaltyCardRow(
                    card: card,
                    availableWidth: availableWidth
                )
            }
        }
    }
}
```

Even better, prefer layout APIs that avoid explicit geometry when possible:

```swift
LoyaltyCardRow(card: card)
    .frame(maxWidth: .infinity, alignment: .leading)
```

For complex reusable layout logic, consider a custom `Layout` instead of scattering `GeometryReader` and preference keys across many views. Respect the app's deployment target before recommending this.

## PreferenceKey and Layout Feedback

Preference keys are useful for child-to-parent communication, such as reporting measured size, anchor positions, scroll-related values, or header offsets.

They can also introduce extra update cycles because child layout produces a value, the parent reacts, and that reaction may cause another layout pass.

Use them carefully in:

- large lists
- nested scroll views
- animated containers
- headers that resize while scrolling
- rows that update often
- layouts where the measured value is written back into state

Risky:

```swift
.onPreferenceChange(HeaderHeightKey.self) { height in
    headerHeight = height
}
```

Prefer guarding state updates so tiny measurement changes do not trigger unnecessary work:

```swift
.onPreferenceChange(HeaderHeightKey.self) { height in
    guard abs(headerHeight - height) > 0.5 else { return }
    headerHeight = height
}
```

Example preference key:

```swift
private struct HeaderHeightKey: PreferenceKey {
    static var defaultValue: CGFloat = 0

    static func reduce(value: inout CGFloat, nextValue: () -> CGFloat) {
        value = max(value, nextValue())
    }
}
```

Example measurement:

```swift
HeaderView()
    .background {
        GeometryReader { proxy in
            Color.clear.preference(
                key: HeaderHeightKey.self,
                value: proxy.size.height
            )
        }
    }
```

Agent guidance:

- Keep preference values small and stable.
- Reduce values intentionally.
- Avoid emitting per-row preferences when only a container-level value is needed.
- Do not write preference results into broad shared models unless multiple independent views truly need them.
- Guard against oscillation and tiny floating-point changes.

## Shadows, Masks, Blurs, Materials, and Clipping

Visual effects can increase drawing and compositing work, especially when repeated, animated, or combined.

Review repeated content for:

- large dynamic shadows
- animated shadow radius or opacity
- nested masks
- repeated `blur`
- translucent materials in many rows
- clipping plus shadows
- complex overlays and backgrounds
- animated gradients
- large images with masks or rounded clipping

Do not state that every shadow, blur, or mask is slow. The risk depends on size, frequency, device, content, and animation.

Prefer:

- applying expensive effects at a coarser container boundary when visually acceptable
- keeping row effects simple in large lists
- avoiding animated blur/shadow changes during scrolling
- pre-rendering decorative assets when the effect is static and complex
- using simpler shapes for repeated masks and clips
- validating with Instruments when the effect is suspected to cause hitches

Risky in repeated animated content:

```swift
AvatarView(user: user)
    .clipShape(Circle())
    .shadow(color: .black.opacity(0.25), radius: isActive ? 16 : 4)
    .blur(radius: isDisabled ? 1 : 0)
    .animation(.spring(), value: isActive)
```

Prefer a simpler hot-path version when the same design intent can be preserved:

```swift
AvatarView(user: user)
    .clipShape(Circle())
    .overlay {
        Circle().stroke(isActive ? Color.accentColor : Color.clear, lineWidth: 2)
    }
```

Use visual simplification as a candidate refactor, not as a universal rule.

## `drawingGroup()` and `compositingGroup()`

Use these modifiers deliberately.

`compositingGroup()` changes how SwiftUI composites a subtree. `drawingGroup()` can rasterize a subtree into an offscreen representation, which may help some complex vector drawing cases but can also increase memory use, offscreen rendering, and texture work.

Do not recommend `drawingGroup()` as a generic performance fix.

Potentially reasonable:

```swift
ComplexBadgeVector()
    .drawingGroup()
```

Risky if applied mechanically to every row:

```swift
ForEach(rows) { row in
    TransactionRow(row: row)
        .drawingGroup()
}
```

Agent guidance:

- Use only when the subtree is visually complex enough to justify testing it.
- Avoid for normal text, controls, and simple row layout.
- Validate before and after because it can move cost rather than remove cost.
- Watch memory and animation behavior, not only CPU samples.

## Canvas for Dense Drawing

Use `Canvas` when the UI represents many small visual elements that update together and do not need to be individual SwiftUI subviews.

Good candidates:

- sparklines
- mini charts
- timelines
- heat maps
- audio waveforms
- dense decorative particles
- compact financial graphs
- custom progress shapes

Risky approach:

```swift
HStack(spacing: 1) {
    ForEach(samples.indices, id: \.self) { index in
        RoundedRectangle(cornerRadius: 1)
            .frame(width: 2, height: samples[index] * 40)
    }
}
```

Prefer drawing dense primitives in one surface:

```swift
struct MiniVolumeChart: View {
    let samples: [Double]

    var body: some View {
        Canvas { context, size in
            guard !samples.isEmpty else { return }

            let barWidth = size.width / CGFloat(samples.count)

            for index in samples.indices {
                let normalized = min(max(samples[index], 0), 1)
                let height = size.height * CGFloat(normalized)
                let rect = CGRect(
                    x: CGFloat(index) * barWidth,
                    y: size.height - height,
                    width: max(1, barWidth - 1),
                    height: height
                )

                context.fill(Path(rect), with: .color(.primary))
            }
        }
        .frame(height: 44)
    }
}
```

Keep the renderer lightweight. Prepare normalized points, colors, and labels outside the drawing closure when they are expensive to compute.

Do not replace normal interactive UI controls with `Canvas`. Individual elements drawn inside a canvas are not automatically separate SwiftUI views with their own accessibility, hit testing, focus, or state lifecycle.

## TimelineView for Scheduled Updates

Use `TimelineView` when UI should redraw on a known schedule.

Good candidates:

- clocks
- countdowns
- market session timers
- live progress indicators
- time-based visualizations
- animated drawings paired with `Canvas`

Prefer scheduled redraws over manually driving repeated `@State` changes with `Timer` when the UI is naturally time-based.

Example:

```swift
struct AuctionCountdownView: View {
    let endDate: Date

    var body: some View {
        TimelineView(.periodic(from: .now, by: 1)) { context in
            Text(remainingText(now: context.date))
                .monospacedDigit()
        }
    }

    private func remainingText(now: Date) -> String {
        let seconds = max(0, Int(endDate.timeIntervalSince(now)))
        return "\(seconds)s"
    }
}
```

Do not use a high-frequency schedule when a slower cadence is enough.

Risky:

```swift
TimelineView(.periodic(from: .now, by: 1.0 / 60.0)) { context in
    CountdownText(endDate: endDate, now: context.date)
}
```

for a countdown that only displays whole seconds.

Agent guidance:

- Match the schedule to the visible precision.
- Avoid expensive formatting or data transformation inside every tick.
- Do not use `TimelineView` as a replacement for async loading or event subscriptions.
- For watchOS or reduced update environments, account for lower update cadence when relevant.

## Animation Scope

Animation hitches often come from too much work happening during an animated transaction.

Review:

- what state changes are animated
- how broad the affected subtree is
- whether the animation changes layout, drawing, or only transform/opacity
- whether repeated rows update during the animation
- whether expensive effects are animated
- whether high-frequency values are stored in broad state or environment

Prefer narrow animation scope:

```swift
withAnimation(.easeInOut) {
    expandedTransactionID = row.id
}
```

and make sure only the relevant subtree depends on `expandedTransactionID`.

Avoid applying broad implicit animation to a large container when many unrelated values change:

```swift
PortfolioScreen(model: model)
    .animation(.easeInOut, value: model)
```

Prefer animating specific values:

```swift
HoldingsSection(
    rows: model.rows,
    expandedID: model.expandedID
)
.animation(.easeInOut, value: model.expandedID)
```

Even this can be too broad if `HoldingsSection` is large. Consider moving the animation closer to the row or component that visually changes.

## Layout-Affecting vs Compositor-Friendly Animation

Animations that change layout can be more expensive than animations that affect transform or opacity.

Potentially heavier:

- animating text size
- animating dynamic content height across many rows
- animating layout constraints through state changes
- animating blur, mask, shadow, or gradient changes
- expanding many rows at once

Often cheaper:

- opacity
- scale
- translation
- simple rotation

Do not force every design into transform-only animation. Use this distinction to reason about risk and decide what to measure.

## Matched Geometry and Transitions

`matchedGeometryEffect` and custom transitions can produce polished UI, but they can also make layout and animation dependencies harder to reason about.

Use them carefully when:

- source and destination views live inside large, frequently updating containers
- list rows are inserted or removed during the same animation
- the matched views have heavy masks, materials, shadows, or dynamic text
- multiple matched elements animate at once

Agent guidance:

- Keep matched elements visually focused.
- Avoid pairing matched geometry with broad unrelated state updates.
- Validate on target devices if the transition is central to the experience.
- If the animation hitches, test a simpler transition before redesigning the entire screen.

## Animation Hitches and Core Animation Validation

Use Animation Hitches, Core Animation, Time Profiler, and the SwiftUI instrument based on the question being asked.

Use Animation Hitches when the symptom is:

- stutter during scrolling
- skipped frames during animation
- visible pauses during gestures
- jank during transitions

Use Core Animation-oriented inspection when the suspicion is:

- too many layers or effects
- expensive compositing
- repeated offscreen rendering candidates
- heavy masks, shadows, blurs, or transparency
- large animated surfaces

Use Time Profiler when the suspicion is:

- main-thread CPU work
- expensive formatting or image work during updates
- custom layout or drawing code consuming CPU
- synchronous work that overlaps animation

Use the SwiftUI instrument when available and the suspicion is:

- long view body updates
- unnecessary body updates
- broad SwiftUI update causes
- representable update cost

Do not claim that Instruments proved a cause unless the trace or user-provided artifact actually shows it.

## Common Red Flags

Flag these patterns during layout, drawing, and animation review:

```swift
GeometryReader { proxy in
    Row(width: proxy.size.width)
}
```

inside every row of a large collection.

```swift
.onPreferenceChange(RowOffsetKey.self) { value in
    model.scrollOffset = value
}
```

when `model` is broad state observed by a large screen.

```swift
LargeList(rows: rows)
    .animation(.default, value: screenModel)
```

when many unrelated properties can change.

```swift
ForEach(rows) { row in
    Row(row: row)
        .shadow(radius: row.isSelected ? 20 : 4)
        .blur(radius: row.isDisabled ? 2 : 0)
}
```

in a frequently scrolling list.

```swift
ForEach(points.indices, id: \.self) { index in
    Circle()
        .frame(width: 2, height: 2)
        .position(points[index])
}
```

for dense chart-like drawing that could be a single `Canvas`.

```swift
TimelineView(.periodic(from: .now, by: 1.0 / 60.0)) { context in
    Text(expensiveFormattedText(for: context.date))
}
```

when the visible content does not require 60 updates per second.

## Preferred Refactoring Order

For layout, drawing, and animation issues, prefer this order:

1. Narrow the state dependency that triggers the visual update.
2. Remove expensive data preparation from the visual update path.
3. Simplify repeated row modifier chains.
4. Move geometry reads to stable container boundaries.
5. Replace layout feedback loops with simpler layout APIs when possible.
6. Guard preference-driven state updates.
7. Apply expensive effects at coarser boundaries when visually acceptable.
8. Replace dense subview trees with `Canvas` when the content is drawing-like.
9. Match `TimelineView` cadence to visible precision.
10. Narrow animation scope to the component that actually changes.
11. Prefer transform/opacity animation when it preserves the design intent.
12. Validate suspected hitches with the smallest suitable profiling tool.

## Agent Output Guidance

When responding to a code review involving this reference, include:

1. The likely layout, drawing, or animation risk.
2. Why it matters in SwiftUI.
3. Whether the issue is a static risk or measured finding.
4. A focused refactor.
5. What to measure if the user needs confirmation.

Avoid generic advice such as:

```md
Use fewer modifiers.
```

Prefer concrete advice:

```md
This `GeometryReader` runs inside every row even though all rows need the same container width. Move the width read to the list container or use `.frame(maxWidth: .infinity)` if the row only needs flexible width.
```
