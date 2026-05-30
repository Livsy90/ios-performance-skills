---

name: swiftui-performance
description: Use this skill when reviewing or refactoring SwiftUI screens for unnecessary updates, slow scrolling, heavy rows, broad state dependencies, unstable identity, animation hitches, expensive body work, list pagination problems, closure-heavy row views, custom bindings, Observation or ObservableObject granularity, async lifecycle work, or profiling validation with Instruments, xctrace, signposts, XCTest, and MetricKit.
---

# SwiftUI Performance

## Purpose

Use this skill to review, generate, and refactor SwiftUI code with a performance-first mental model focused on change locality.

This skill is primarily about SwiftUI code composition: identity, lifetime, dependency scope, state ownership, row complexity, list structure, layout cost, drawing cost, and async lifecycle work.

Profiling is a validation layer. Use profiling tools only when the environment supports them or when the user provides profiling artifacts.

## Non-Goals

Do not turn every SwiftUI review into a profiling session.

Do not recommend broad architectural rewrites for small static views unless there is a clear performance risk.

Do not claim that a performance issue was measured unless there is actual evidence from Instruments, `xctrace`, XCTest performance tests, signpost logs, MetricKit payloads, user-provided traces, screenshots, logs, or benchmark results.

Do not describe SwiftUI as generically slow. Explain the concrete update, dependency, identity, layout, drawing, or scheduling issue.

## Core Mental Model

Reason about SwiftUI performance through three concepts:

* Identity: whether SwiftUI treats a view as the same view across updates.
* Lifetime: how long state associated with that identity is preserved.
* Dependencies: which data reads cause a view to update.

A SwiftUI view is a value description of UI. Do not treat it as a long-lived UIKit-style object.

When reviewing code, explain performance issues in terms of:

* what changed
* which view depends on that change
* how much of the view tree becomes affected
* whether identity is stable
* whether rendering work is cheap
* whether the structure keeps local updates local

Prefer precise wording:

* "This parent view reads the whole model, so changes to this model can invalidate a large part of the screen."
* "This row receives non-visual closure inputs, which can make row equality and update behavior harder to reason about."
* "This list rebuilds derived row data during rendering, so appending or filtering can create avoidable work on the update path."

## Review Workflow

1. Identify the symptom, request, or risk:

   * unnecessary updates
   * slow scrolling
   * heavy rows
   * broad state dependencies
   * unstable identity
   * expensive `body` work
   * animation hitches
   * layout or drawing cost
   * pagination update bursts
   * async lifecycle problems
   * profiling validation

2. Classify the issue:

   * identity and lifetime
   * dependency scope
   * state ownership
   * `body` cost
   * render model preparation
   * list, row, or pagination structure
   * closure, binding, or action handler stability
   * layout, drawing, or animation cost
   * async lifecycle or main actor work
   * profiling evidence or measurement plan

3. Inspect the smallest relevant code path before suggesting broad changes.

4. Prefer targeted refactors over architectural rewrites.

5. Separate static code review findings from hypotheses and measured results.

6. Suggest profiling, debug probes, or instrumentation only when confirmation is useful or requested.

## Refactoring Order

When fixing SwiftUI performance risks, prefer this order:

1. Stabilize identity.
2. Remove expensive work from `body`.
3. Prepare render-ready models outside rendering.
4. Narrow view dependencies.
5. Move local state closer to the component that owns it.
6. Split large views into dependency-focused subviews.
7. Improve list, row, and pagination structure.
8. Simplify row inputs and repeated view structure.
9. Remove unnecessary type erasure from hot paths.
10. Reduce closure-heavy stored row inputs.
11. Prefer key-path bindings over custom closure bindings when no transformation is needed.
12. Use `Equatable` only when visual equality is clear and useful.
13. Simplify layout, geometry, drawing, and animation work.
14. Add profiling or temporary debug probes to validate the hypothesis when needed.
15. Consider UIKit only for genuinely hot paths that need lower-level control.

## Evidence Rule

Always distinguish between:

* static code review findings
* likely risks
* hypotheses
* measured results
* user-provided evidence
* tool-generated evidence

Do not invent numbers or claim profiling results unless actual evidence is available.

Avoid unsupported claims such as:

```md
This costs 500 ms.
```

Prefer:

```md
This can cause broad invalidation. To confirm it, profile the append interaction with signposts around page insertion and inspect body invocation frequency or Time Profiler samples.
```

## Profiling Capability Rules

The agent may use profiling tools only when the environment supports them.

Before attempting profiling, check whether the task has:

* a macOS host
* a full Xcode installation
* a selected Xcode path
* a buildable project
* a runnable target
* an available simulator or device
* a reproducible user scenario
* permission to run shell commands
* permission to create trace or log artifacts

If tools are unavailable, provide a local profiling plan and, when useful, suggest instrumentation code.

Never pretend to have run Instruments, `xctrace`, XCTest, or MetricKit analysis when no command was run and no artifact was provided.

## Reference Routing

Use the references only when the task needs that level of detail.

Read `references/identity-and-state.md` when the task involves structural identity, explicit `.id(...)`, `ForEach` identity, local state lifetime, `@State`, `@StateObject`, `@ObservedObject`, or owned observable models.

Read `references/observation-and-dependencies.md` when the task involves Observation, `ObservableObject`, `@Published`, broad model dependencies, computed properties that read many fields, environment dependencies, or splitting screens into dependency islands.

Read `references/body-cost-and-render-models.md` when the task involves expensive work in `body`, sorting, filtering, grouping, formatting, computed properties, render model preparation, or main-actor data transformation.

Read `references/lists-pagination-and-rows.md` when the task involves `List`, `ScrollView`, `LazyVStack`, `ForEach`, large collections, row complexity, filtering inside repeated content, pagination, section/page models, swipe actions, menus, or scrolling performance.

Read `references/closures-bindings-and-equatable.md` when the task involves stored closures in views, capture lists, action handlers, gestures, custom `Binding(get:set:)`, key-path bindings, `.equatable()`, or Equatable view inputs.

Read `references/layout-drawing-and-animation.md` when the task involves modifier chains, `GeometryReader`, `PreferenceKey`, layout feedback, shadows, masks, blurs, `Canvas`, `TimelineView`, animation hitches, Core Animation, or drawing-heavy UI.

Read `references/async-lifecycle-and-mainactor.md` when the task involves `.task(id:)`, `.onAppear`, row lifecycle callbacks, duplicate loading, pagination triggers, cancellation, `Task`, `MainActor`, or heavy work during UI updates.

Read `references/profiling-validation.md` when the user asks for profiling help or provides Instruments traces, `xctrace` output, signpost logs, XCTest benchmark results, MetricKit payloads, screen recordings, memory graphs, or other performance artifacts.

## Code Review Checklist

When reviewing SwiftUI code, check:

* Does the view read only the state it needs?
* Is state owned by the smallest stable component that needs it?
* Is identity stable and intentional?
* Is `body` free from expensive transformation work?
* Are render models prepared outside rendering when rows are non-trivial?
* Are large lists using stable IDs and an appropriate lazy container?
* Are pagination updates local and not rebuilding old render data unnecessarily?
* Are rows visually lightweight and structurally predictable?
* Are closure-heavy row inputs avoided or intentionally designed?
* Are custom bindings necessary, or would key-path bindings be enough?
* Are layout, drawing, and animation costs appropriate for repeated content?
* Is async work started from lifecycle modifiers or explicit actions, not from `body`?
* Is profiling evidence separated from static review hypotheses?

## Common Red Flags

Flag these patterns during review:

* unstable IDs such as `.id(UUID())`
* computed identifiers such as `var id: UUID { UUID() }`
* index-based identity for mutable collections
* sorting, filtering, grouping, formatting, parsing, decoding, or database reads inside `body`
* filtering inside `ForEach` instead of preparing visible rows first
* passing one giant model into every child view
* broad computed properties that hide many state reads
* `AnyView` inside large repeated collections
* custom conditional modifier helpers that create structural instability in hot paths
* custom `Binding(get:set:)` when a key-path binding would work
* row views that store many non-visual action closures
* `GeometryReader` inside every row of a large collection
* heavy shadows, masks, blurs, overlays, or preference keys in repeated views
* async work started directly from `body`
* unguarded `.onAppear` pagination triggers inside rows

## Agent Response Format for Code Reviews

When reviewing code, use this structure:

1. Main issue
2. Why it matters in SwiftUI
3. Risk level
4. Suggested refactor
5. Before/after code when useful
6. What to measure if confirmation is needed
7. Expected effect

Keep explanations concrete. Avoid generic statements like "SwiftUI is slow."

## Agent Response Format for Profiling Requests

When the user asks for profiling help, use this structure:

1. Reproducible scenario
2. Hypothesis
3. Tool choice
4. Exact command or local steps when possible
5. What to look for
6. How to interpret findings
7. Refactor candidates
8. How to compare before and after

Never claim measurements that were not actually taken.

## Final Principle

SwiftUI performance improves when code makes change locality obvious.

Optimize for:

* stable identity
* narrow dependencies
* cheap rendering
* predictable structure
* local state ownership
* lightweight row inputs
* explicit update boundaries
* clear async lifecycle
* measured validation when needed
