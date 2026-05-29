# SIL Inspection

Use this reference when source-level reasoning is not enough to explain a Swift runtime performance issue.

SIL inspection is useful for validating hypotheses about allocation, ARC traffic, closure creation, dispatch, existential storage, generic specialization, ownership operations, inlining, and module-boundary optimization.

Do not use SIL as a replacement for profiling. Use it to explain why a measured cost may exist, or to verify whether the optimizer can remove an abstraction.

## Core model

SIL is Swift Intermediate Language. It sits between the type-checked Swift AST and LLVM IR.

SIL preserves Swift-specific semantic information that is useful for performance analysis:

* ownership
* reference counting
* generics
* protocol witness calls
* class dispatch
* existential operations
* closure representation
* allocations
* value copies and destruction
* async and actor-related lowering details

SIL is not a stable user-facing API. Instruction names, ownership forms, and generated patterns may change between Swift compiler versions.

Use SIL as evidence, not as dogma.

## When to inspect SIL

Inspect optimized SIL when you need to answer questions such as:

* Did this generic function specialize?
* Did this protocol call remain a witness-table call?
* Did this class call devirtualize?
* Did this closure allocate?
* Is a mutable capture boxed?
* Are there repeated retain/release operations?
* Is an existential value being initialized, opened, or boxed?
* Did a helper function inline?
* Is a value copied, borrowed, or destroyed repeatedly?
* Is a public/module boundary preventing optimization?

Do not inspect SIL for every code review. It is most useful for hot paths, framework code, performance-sensitive libraries, and suspicious allocation/dispatch/ARC patterns.

## Guardrails

Follow these rules:

* Prefer optimized SIL for performance conclusions.
* Do not make performance conclusions from `-Onone` SIL unless the issue is specifically debug-build behavior.
* Compare SIL using the same compiler version and optimization settings as the target build.
* Treat SIL patterns as clues, not final proof of user-visible performance.
* Confirm important claims with Instruments, benchmarks, signposts, or assembly when needed.
* Do not rewrite clear code only because SIL contains a scary-looking instruction.
* Do not assume a SIL instruction always survives to final machine code.
* Do not assume absence of one SIL instruction means absence of all runtime cost.

The optimizer may inline, specialize, remove retains, promote storage, scalar-replace values, or lower operations further during later stages.

## Generating SIL

For a small standalone file:

```bash
swiftc -O -emit-sil Example.swift > optimized.sil
```

For raw SIL directly after SILGen:

```bash
swiftc -Onone -emit-silgen Example.swift > raw.sil
```

For canonical SIL without performance optimization:

```bash
swiftc -Onone -emit-sil Example.swift > canonical-onone.sil
```

For optimized SIL with a module name:

```bash
swiftc -O -emit-sil -module-name MyModule *.swift > optimized.sil
```

Use the target's real build settings when possible. Optimization level, whole-module compilation, library evolution, access control, and module boundaries can materially change the output.

If symbols are hard to read, demangle them:

```bash
swift-demangle '$s4Demo9makeValueSiyF'
```

## Which SIL form to use

Use the right form for the question.

### Raw SIL

Generated with:

```bash
swiftc -Onone -emit-silgen Example.swift
```

Use raw SIL to understand how Swift source is initially lowered.

Do not use raw SIL to judge final runtime performance. Raw SIL contains many intermediate constructs that later mandatory and optimization passes may remove.

### Canonical `-Onone` SIL

Generated with:

```bash
swiftc -Onone -emit-sil Example.swift
```

Use it to understand mandatory lowering, ownership, and diagnostics-related transformations.

Do not treat it as Release performance evidence.

### Optimized SIL

Generated with:

```bash
swiftc -O -emit-sil Example.swift
```

Use optimized SIL to inspect performance-relevant questions:

* specialization
* devirtualization
* ARC optimization
* inlining
* closure allocation
* box promotion
* existential handling
* ownership traffic

Optimized SIL is usually the best starting point for this skill.

## Reading SIL without overreading it

When reading SIL, focus on patterns around the function under review.

Useful approach:

1. Find the function.
2. Ignore unrelated declarations, thunks, and standard library noise.
3. Search for allocation instructions.
4. Search for closure creation.
5. Search for dispatch instructions.
6. Search for ARC and ownership operations.
7. Search for existential operations.
8. Check whether generic code specialized.
9. Compare before and after the proposed change.
10. Tie the pattern back to a measured or credible hot-path concern.

Do not quote large SIL dumps in normal review output. Summarize the relevant pattern.

## Common instruction patterns

Instruction names can vary across compiler versions and ownership modes. Use these as practical search targets, not as a complete SIL specification.

### Allocation and storage

Look for:

* `alloc_ref`
* `alloc_ref_dynamic`
* `alloc_box`
* `alloc_stack`
* `dealloc_stack`
* `project_box`
* `mark_uninitialized`
* `copy_addr`
* `destroy_addr`

Interpretation:

* `alloc_ref` usually indicates class instance allocation.
* `alloc_box` often indicates boxed storage, commonly for captured mutable variables or values requiring shared addressable storage.
* `alloc_stack` indicates stack storage in SIL, but it may still be promoted, eliminated, or transformed.
* `copy_addr` and `destroy_addr` may indicate address-based value copying and destruction, especially for non-trivial or address-only values.

Do not assume every `alloc_stack` is bad or every `alloc_ref` is removable. Look at frequency and hot-path relevance.

### Closure creation

Look for:

* `partial_apply`
* `function_ref`
* `thin_to_thick_function`
* `convert_function`
* `alloc_box`
* `strong_retain`
* `copy_value`

Interpretation:

* `function_ref` usually references a known function.
* `partial_apply` usually means a closure value is being formed by binding captured context to a function.
* `alloc_box` near closure creation may indicate boxed captured mutable state.
* Retains near closure creation may indicate captured references.

Review questions:

* Is the closure escaping?
* Is it created repeatedly?
* Does it capture `self` or a large object graph?
* Can captured dependencies be narrowed?
* Can the closure be non-escaping?
* Can a concrete helper type avoid repeated closure allocation?

Example source pattern:

```swift
final class AnalyticsRouter {
    private let sink: AnalyticsSink
    private let userProvider: UserProvider

    init(sink: AnalyticsSink, userProvider: UserProvider) {
        self.sink = sink
        self.userProvider = userProvider
    }

    func makeHandler() -> (Event) -> Void {
        { event in
            self.sink.send(event, user: self.userProvider.currentUserID)
        }
    }
}
```

If this handler is created frequently, inspect SIL for closure creation and captures. A narrower capture may reduce retained state:

```swift
func makeHandler() -> (Event) -> Void {
    let sink = sink
    let userProvider = userProvider

    return { event in
        sink.send(event, user: userProvider.currentUserID)
    }
}
```

This does not guarantee zero allocation. It only narrows ownership and may make the cost clearer.

### ARC and ownership traffic

Look for:

* `strong_retain`
* `strong_release`
* `retain_value`
* `release_value`
* `copy_value`
* `destroy_value`
* `begin_borrow`
* `end_borrow`
* `load_borrow`
* `store_borrow`
* `copy_addr`
* `destroy_addr`

Interpretation:

* `strong_retain` and `strong_release` indicate reference-counting operations around class references.
* `retain_value` and `release_value` can appear for non-trivial values.
* `copy_value` and `destroy_value` reflect ownership movement and lifetime.
* Borrow instructions may indicate the optimizer is avoiding unnecessary ownership copies.
* Address-based copy/destroy can appear for non-trivial values or values whose layout is not fully known.

Do not count retains manually and conclude performance from the count alone. The final code may differ after later lowering, and a few retains may not matter outside hot paths.

Review questions:

* Are retains/releases inside a frequently executed loop?
* Are they caused by closure captures?
* Are they caused by type erasure?
* Are they caused by reference-heavy values?
* Are they caused by cross-module abstraction?
* Can ownership be simplified without changing semantics?

### Class dispatch

Look for:

* `class_method`
* `super_method`
* `objc_method`
* `objc_super_method`
* `dynamic_method_br`
* `function_ref`
* `apply`

Interpretation:

* `class_method` indicates a class vtable-style method lookup.
* `objc_method` indicates Objective-C method dispatch.
* `function_ref` followed by `apply` often indicates a direct call.
* `super_method` is used for superclass method lookup.
* Devirtualization may replace an indirect class call with a direct function reference.

Review questions:

* Is dynamic dispatch intentional?
* Is the class or method a real subclassing point?
* Could the type be `final` as a semantic default?
* Is the call in a hot path?
* Can the concrete type be kept visible internally?
* Is Objective-C dynamic behavior required?

Do not recommend `final` only when a hot path is proven. For app-level classes not designed for inheritance, `final` is a good semantic default. Require hot-path evidence when presenting `final` as a performance optimization.

### Protocol witness dispatch

Look for:

* `witness_method`
* `apply`
* `init_existential_*`
* `open_existential_*`

Interpretation:

* `witness_method` indicates a call through a protocol witness table.
* This is expected when calling a protocol requirement through a generic constraint or existential.
* In optimized code, the compiler may specialize generic paths and replace witness calls with direct calls.
* Existential use may require opening the existential and then calling through witness tables.

Review questions:

* Is the protocol call in a hot path?
* Is runtime heterogeneity required?
* Is the concrete type actually homogeneous?
* Did generic specialization remove the witness call?
* Is this across a module boundary?
* Would a generic, opaque, or concrete path preserve design and improve optimizer visibility?

Do not automatically replace protocols with concrete types. Protocol dispatch is a design tool. It matters when it remains in a measured hot path.

### Existential operations

Look for:

* `init_existential_addr`
* `init_existential_ref`
* `init_existential_value`
* `open_existential_addr`
* `open_existential_ref`
* `open_existential_value`
* `alloc_existential_box`
* `project_existential_box`
* `deinit_existential_addr`
* `witness_method`

Interpretation:

* `init_existential_*` indicates construction of an existential container.
* `open_existential_*` indicates opening an existential to access the underlying dynamic type.
* `alloc_existential_box` can indicate boxed existential storage.
* `witness_method` near existential use indicates protocol requirement dispatch.
* The exact pattern depends on value size, loadability, ownership mode, and compiler version.

Review questions:

* Is the existential stored long-term or only passed through?
* Is the conforming type large or address-only?
* Is existential boxing visible in Allocations?
* Is the existential required for heterogeneous storage?
* Could the hot part use a generic helper while the boundary remains existential?

Example source pattern:

```swift
protocol EventFormatter {
    func title(for event: Event) -> String
}

func titles(
    events: [Event],
    formatter: any EventFormatter
) -> [String] {
    events.map { formatter.title(for: $0) }
}
```

If this is hot and the formatter is homogeneous, inspect whether existential operations and witness calls remain. A generic helper may be useful:

```swift
func titles<F: EventFormatter>(
    events: [Event],
    formatter: F
) -> [String] {
    events.map { formatter.title(for: $0) }
}
```

This is not automatically better. It is better when specialization happens and the abstraction cost matters.

### Generic specialization

Look for:

* specialized function names
* `function_ref` to a specialized variant
* absence or presence of `witness_method`
* generic signatures in SIL functions
* metadata or witness table parameters
* `apply` of generic functions

Interpretation:

* Specialized generic code often appears as separate specialized SIL functions or direct calls to specialized variants.
* Unspecialized generic code may carry metadata and witness table parameters.
* Generic specialization is more likely when the optimizer can see concrete call sites.
* Cross-module boundaries, resilience, public APIs, and type erasure can limit specialization.

Review questions:

* Is the generic function internal to the module?
* Is it public or used across module boundaries?
* Is the concrete type visible at the call site?
* Is the function small and hot?
* Is specialization actually visible in optimized SIL?
* Would `@inlinable` be justified, or would it overexpose implementation details?

Do not add `@inlinable` only because a function is generic. Treat it as an API and ABI commitment.

### Inlining

Look for:

* call still present as `apply`
* direct call through `function_ref`
* body folded into caller
* specialized/inlined helper no longer visible in the caller
* large helper still present as a separate call

Interpretation:

* Absence of a call can indicate inlining or optimization away.
* Presence of a call does not always mean a problem.
* Inlining can improve performance but can also increase code size.
* The compiler may avoid inlining large functions or functions across module boundaries.

Review questions:

* Is the function tiny and in a hot path?
* Did inlining enable further optimization?
* Is code size a concern?
* Is the function public across a module boundary?
* Would `@inline(__always)` be justified by measurement?

Avoid `@inline(__always)` as a default fix. Use it only when inspection and measurement show the compiler is not inlining a small hot function and the code-size trade-off is acceptable.

### Copying and destruction

Look for:

* `copy_value`
* `destroy_value`
* `copy_addr`
* `destroy_addr`
* `retain_value`
* `release_value`
* collection operations
* calls into array/string/dictionary helpers

Interpretation:

* Copies and destroys are expected for non-trivial values.
* Repeated copies may matter in loops, parsers, renderers, reducers, and data pipelines.
* Value types containing references or COW storage may carry ownership cost.
* Address-only values may require indirect operations.

Review questions:

* Is a large value copied repeatedly?
* Is mutation following a copy and triggering COW behavior?
* Is a value kept alive longer than necessary?
* Can scope be narrowed?
* Can mutation happen at the owning boundary?
* Can a borrowing or in-place API avoid repeated ownership traffic?

### Stack and box promotion

Look for:

* `alloc_box` in raw or unoptimized SIL
* `alloc_stack` in canonical or optimized SIL
* disappearance of both after optimization
* `project_box`
* captured variable patterns

Interpretation:

* Raw SIL may contain boxes that later get promoted.
* Mandatory memory promotion may convert some boxes to stack storage.
* Optimized SIL may eliminate stack storage entirely by scalar replacement.
* A remaining `alloc_box` in optimized SIL is more interesting than one in raw SIL.

Review questions:

* Is a box still present in optimized SIL?
* Is it caused by an escaping closure?
* Is it caused by captured mutable state?
* Is it created repeatedly?
* Can the variable be made immutable?
* Can the closure become non-escaping?
* Can state be moved into a small explicit type?

### Async and concurrency-related SIL

SIL for async functions and actors can be complex and compiler-version-sensitive.

Use SIL cautiously for concurrency analysis. Prefer the Swift Concurrency instrument for task, actor, and executor behavior.

SIL may help answer:

* Does a closure capture large state into a task?
* Does a value live across an `await`?
* Is a function actor-isolated or nonisolated in the lowered representation?
* Are there continuation or task-related allocations near the hot path?

Review questions:

* Is the performance issue scheduling, allocation, actor contention, or lifetime extension?
* Is the large value kept alive across suspension?
* Is `Task {}` extending lifetime beyond the caller?
* Is a closure passed to task creation capturing `self` or a large object graph?
* Would structured concurrency or narrower captures reduce lifetime?

Do not claim that every `await` creates heap allocation. Values crossing suspension become part of async state, but placement and optimization are compiler/runtime details.

## Before/after inspection workflow

Use this workflow when proposing a code change.

1. Capture the baseline.

   * optimized SIL
   * benchmark or Instruments evidence
   * source snippet
   * build settings

2. State the hypothesis.

   * "This existential remains in the hot loop."
   * "This closure is allocated repeatedly."
   * "This class call does not devirtualize."
   * "This generic function does not specialize across the module boundary."

3. Make the smallest change.

   * narrow a capture
   * make a class `final`
   * use a generic helper
   * move hot code inside the module
   * reduce type erasure in the inner loop
   * reserve collection capacity
   * avoid unnecessary mutation after copy

4. Inspect optimized SIL again.

5. Validate with measurement.

Do not stop at "SIL looks better" if user-visible performance matters.

## Common source patterns and what to check

### Type-erased hot loop

Source smell:

```swift
func processAll(_ processors: [any RecordProcessor], records: [Record]) {
    for record in records {
        for processor in processors {
            processor.process(record)
        }
    }
}
```

Check:

* existential opening
* witness dispatch
* boxing
* allocation of type-erased wrappers
* call frequency

Possible direction:

* keep existential at the boundary
* move homogeneous inner loop to a generic helper
* group records by processor type if design allows
* measure before changing architecture

### Escaping closure created repeatedly

Source smell:

```swift
func handlers(for items: [MenuItem]) -> [() -> Void] {
    items.map { item in
        { open(item) }
    }
}
```

Check:

* `partial_apply`
* closure context allocation
* captured object graph
* repeated allocation count in Instruments

Possible direction:

* create handlers lazily
* store IDs instead of whole objects
* use a concrete command type
* narrow captures

### Non-final class used as concrete implementation

Source smell:

```swift
class CurrencyFormatterService {
    func displayText(for amount: Money) -> String {
        amount.localizedDescription
    }
}
```

Check:

* whether subclassing is intentional
* whether calls remain `class_method`
* whether the class is app-level and concrete

Possible direction:

```swift
final class CurrencyFormatterService {
    func displayText(for amount: Money) -> String {
        amount.localizedDescription
    }
}
```

This is primarily a semantic default for non-inheritable classes. Treat performance as a secondary benefit unless the call path is hot.

### Generic function not specializing

Source smell:

```swift
public func transformAll<T: Transform>(
    _ values: [T.Input],
    using transform: T
) -> [T.Output] {
    values.map { transform.apply($0) }
}
```

Check:

* whether the function is public across a module boundary
* whether specialized variants appear in optimized SIL
* whether witness calls remain
* whether closure arguments inhibit inlining

Possible direction:

* keep hot generic implementation internal
* use a small internal generic helper
* consider `@inlinable` only for stable, performance-critical public API
* validate code size and ABI trade-offs

### Mutable captured variable

Source smell:

```swift
func makeAccumulator() -> (Int) -> Int {
    var total = 0

    return { value in
        total += value
        return total
    }
}
```

Check:

* `alloc_box`
* `project_box`
* closure capture
* repeated creation

Possible direction:

* make state explicit in a type
* avoid creating many accumulators in a hot path
* keep mutation local when possible

## Common false conclusions

Avoid these mistakes:

* "`alloc_stack` means this is optimized enough."
* "`alloc_ref` means this class must be replaced with a struct."
* "`witness_method` means protocols are bad."
* "`partial_apply` means closures are always too expensive."
* "`strong_retain` means ARC is the bottleneck."
* "`copy_value` means a deep copy happened."
* "No `witness_method` means no abstraction cost remains."
* "No SIL issue means the code is fast."
* "SIL from Debug build explains Release performance."
* "`@inline(__always)` is the next step whenever a call remains."
* "`@inlinable` is safe because it improves optimization."

SIL explains compiler lowering and optimization opportunities. It does not replace performance measurement.

## Practical search checklist

When inspecting a hot function, search for:

```text
alloc_ref
alloc_box
partial_apply
strong_retain
strong_release
retain_value
release_value
copy_value
destroy_value
copy_addr
destroy_addr
class_method
objc_method
witness_method
init_existential
open_existential
alloc_existential_box
function_ref
apply
```

Then answer:

1. Which of these are inside the hot path?
2. Which are expected and harmless?
3. Which explain the measured cost?
4. Which can be removed by a local semantic improvement?
5. Which require API, module, or architecture changes?
6. What will validate the improvement?

## Output guidance

When using this reference, respond with:

```markdown
## SIL question

State the exact question SIL is being used to answer.

## Relevant SIL pattern

Summarize the important instruction pattern without pasting a large dump.

## Interpretation

Explain what the pattern likely means and what it does not prove.

## Recommended change

Suggest the smallest safe source-level change.

## Trade-offs

Mention API clarity, code size, module boundaries, ABI, or maintainability.

## Validation

Recommend optimized SIL comparison plus Instruments, benchmark, signposts, or assembly when needed.
```

If the issue is not in a hot path, say that SIL inspection is optional and avoid low-level rewrites.
