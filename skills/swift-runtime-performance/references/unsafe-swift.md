# Unsafe Swift

Use this reference when reviewing Swift code that uses unsafe pointers, raw memory, manual allocation, memory binding, `unsafeBitCast`, `Unmanaged`, C/C++ interop, `unowned(unsafe)`, `nonisolated(unsafe)`, or strict memory-safety annotations.

The goal is not to ban unsafe code. The goal is to keep unsafe code small, justified, documented, testable, and wrapped behind safe APIs whenever possible.

## Core principle

Unsafe Swift should be treated as a boundary, not as a style.

Use unsafe constructs only when:

* a safe API cannot express the required operation efficiently or correctly
* profiling shows the safe abstraction is a real bottleneck
* the code interacts with C, C++, Objective-C, Core Foundation, memory-mapped data, custom binary formats, or low-level buffers
* pointer lifetime, bounds, binding, alignment, initialization, aliasing, and thread-safety rules are documented
* the unsafe region is isolated behind a safe wrapper
* tests cover invalid input, short buffers, lifetime assumptions, mutation, and concurrency

Do not recommend unsafe code as the first performance optimization. Consider algorithmic changes, data layout changes, specialization, borrowing, `Span`/`RawSpan` when available, standard library APIs, and better ownership boundaries first.

## Unsafe does not automatically mean faster

Unsafe APIs can remove some dynamic checks or abstraction overhead, but they can also make code slower, harder to optimize, and easier to break.

Unsafe code may introduce:

* undefined behavior
* missed compiler optimizations
* stricter alignment requirements
* hidden copies around bridging
* accidental lifetime extension
* data races
* invalid pointer escape
* incorrect memory binding
* hard-to-reproduce crashes
* security vulnerabilities

Review rule:

* Require a measured reason for unsafe code when it is introduced for performance.
* Require a correctness reason when unsafe code is introduced for interop or low-level representation.
* Avoid speculative unsafe rewrites.

## Dimensions of memory safety

When reviewing unsafe code, classify the risk.

### Lifetime safety

Is every access performed while the pointed-to memory is still valid?

Risks:

* pointer escapes from a `withUnsafe...` closure
* pointer stored after the owner deallocates or reallocates
* buffer pointer used after collection mutation
* C API stores a pointer longer than Swift expects
* `unowned(unsafe)` reference used after object deallocation

### Bounds safety

Is every read and write inside the valid allocation?

Risks:

* trusting a count from untrusted input
* pointer arithmetic past the buffer end
* writing more elements than capacity
* interpreting byte count as element count
* assuming null termination without checking

### Type safety

Is memory accessed as a type it is actually bound to and initialized as?

Risks:

* using `assumingMemoryBound(to:)` on memory that is not bound to that type
* using typed pointers to reinterpret arbitrary bytes
* using `unsafeBitCast` for semantic conversion
* reading a value as the wrong type
* violating memory binding rules across C interop

### Initialization safety

Is memory initialized before it is read and deinitialized before it is released?

Risks:

* reading uninitialized memory
* initializing the same typed storage twice without deinitialization
* deallocating initialized storage without deinitializing when required
* using capacity as if it were initialized count
* returning partially initialized output after an error

### Thread safety

Is shared memory synchronized correctly?

Risks:

* sharing mutable raw pointers across tasks or threads
* using `nonisolated(unsafe)` to bypass actor isolation
* wrapping unsafe storage in `@unchecked Sendable` without synchronization
* mutating memory while another thread reads it
* relying on C APIs that are not thread-safe

## Strict memory safety mode

In Swift 6.2 and later, strict memory safety checking can be enabled with `-strict-memory-safety`.

When available, use it as an audit tool. It helps identify uses of unsafe constructs, but it does not prove that the unsafe code is correct.

Relevant markers:

* `@unsafe`: marks declarations that are unsafe to use.
* `@safe`: marks declarations whose signature may involve unsafe constructs but whose use is safe because the declaration enforces the required invariants.
* `unsafe` expression: acknowledges a use of unsafe constructs at the call site.

Review guidance:

* Do not silence strict memory-safety warnings mechanically.
* Treat every `unsafe` expression as an audit point.
* Prefer moving unsafe operations into small internal helpers.
* Use `@safe` only when the declaration truly wraps unsafe details behind safe invariants.
* Avoid exposing unsafe types in public APIs unless the unsafe contract is intentionally part of the API.

Example shape:

```swift
struct PacketHeader {
    var version: UInt8
    var flags: UInt8
    var length: UInt16
}

enum PacketParseError: Error {
    case tooShort
}

func parseHeader(from bytes: [UInt8]) throws -> PacketHeader {
    guard bytes.count >= 4 else {
        throw PacketParseError.tooShort
    }

    let length = UInt16(bytes[2]) << 8 | UInt16(bytes[3])

    return PacketHeader(
        version: bytes[0],
        flags: bytes[1],
        length: length
    )
}
```

This code uses safe byte access instead of pointer rebinding. It may be preferable unless measurement proves this path needs a lower-level representation.

## Pointer families

Swift has several unsafe pointer families. Review them by what they promise and what they do not promise.

### Typed pointers

Examples:

* `UnsafePointer<T>`
* `UnsafeMutablePointer<T>`
* `UnsafeBufferPointer<T>`
* `UnsafeMutableBufferPointer<T>`

They represent memory accessed as values of `T`.

Review questions:

* Is the memory bound to `T`?
* Are the elements initialized?
* Is the pointer valid for the whole access?
* Is mutation exclusive?
* Is capacity measured in elements, not bytes?
* Does the pointer escape a scope where it is valid?

### Raw pointers

Examples:

* `UnsafeRawPointer`
* `UnsafeMutableRawPointer`
* `UnsafeRawBufferPointer`
* `UnsafeMutableRawBufferPointer`

They represent raw bytes.

Review questions:

* Is the access byte-oriented or typed?
* If typed values are loaded, are alignment and layout requirements satisfied?
* Is memory binding handled correctly?
* Is the code parsing bytes safely rather than pretending bytes are already typed values?
* Is unaligned data possible?

### Buffer pointers

Buffer pointers pair a base address with a count.

Review questions:

* Is the count the number of elements or bytes?
* Can `baseAddress` be nil for an empty buffer?
* Is every index checked?
* Is the buffer pointer used only during its valid lifetime?
* Is the buffer mutated while another view exists?

## Lifetime of `withUnsafe...` APIs

Pointers created by `withUnsafePointer`, `withUnsafeMutablePointer`, `withUnsafeBytes`, `withUnsafeBufferPointer`, and similar APIs are generally valid only for the duration of the closure unless the API explicitly documents otherwise.

Bad pattern:

```swift
func leakedPointer(from data: Data) -> UnsafeRawPointer? {
    data.withUnsafeBytes { buffer in
        buffer.baseAddress
    }
}
```

The returned pointer may outlive the storage guarantee provided by `withUnsafeBytes`.

Prefer doing the work inside the closure:

```swift
enum ByteReadError: Error {
    case empty
}

func firstByte(in data: Data) throws -> UInt8 {
    try data.withUnsafeBytes { buffer in
        guard let baseAddress = buffer.baseAddress, buffer.count > 0 else {
            throw ByteReadError.empty
        }

        return baseAddress.load(as: UInt8.self)
    }
}
```

Review rule:

* Do not let unsafe pointers escape `withUnsafe...` closures unless the API explicitly transfers ownership or guarantees lifetime.
* Be especially careful with arrays, strings, and `Data`, because mutation or reallocation can invalidate previous storage assumptions.

## Memory binding

Memory binding tells Swift what type a region of memory may be accessed as.

Common operations:

* `bindMemory(to:capacity:)`: binds raw memory to a type.
* `assumingMemoryBound(to:)`: assumes memory is already bound to a type.
* `withMemoryRebound(to:capacity:_:)`: temporarily accesses memory through another compatible type for a limited scope.

Review rule:

* Do not use `assumingMemoryBound(to:)` to make memory become a type.
* Do not use `withMemoryRebound` as arbitrary type punning.
* Do not read structured values directly from arbitrary network/file bytes unless alignment, initialization, layout, and endianness are guaranteed.
* Prefer explicit parsing for external binary formats.

Suspicious pattern:

```swift
func parseRecord(_ raw: UnsafeRawPointer) -> Record {
    raw.assumingMemoryBound(to: Record.self).pointee
}
```

This is only valid if the memory is already bound to `Record`, initialized as `Record`, properly aligned, and valid for the access. For file, network, or IPC bytes, those assumptions are usually not safe.

Prefer explicit parsing unless the memory truly comes from typed Swift storage or a well-defined C API contract.

## Alignment

Typed access usually requires memory to be properly aligned for the target type.

Review questions:

* Is this memory coming from Swift typed storage, C typed storage, or arbitrary bytes?
* Could the data be packed or unaligned?
* Is the code using an API that supports unaligned loads when unaligned input is possible?
* Is the code parsing a portable binary format where byte order and alignment must be handled explicitly?

Avoid assuming that raw bytes from `Data`, files, sockets, compression libraries, or memory-mapped regions are aligned for arbitrary Swift structs.

## Endianness and binary formats

Unsafe pointer casts do not solve byte order.

For external binary formats, check:

* byte order
* integer size
* signedness
* padding
* alignment
* struct layout
* versioning
* validation of declared lengths
* bounds for nested offsets

Bad assumption:

```swift
let value = rawPointer.load(as: UInt32.self)
```

This may be wrong if the bytes are unaligned, in a different endianness, or not actually initialized as a `UInt32`.

Prefer explicit conversion for protocol/file formats:

```swift
func readBigEndianUInt32(_ bytes: ArraySlice<UInt8>) throws -> UInt32 {
    guard bytes.count >= 4 else {
        throw ByteReadError.empty
    }

    var iterator = bytes.makeIterator()

    let b0 = UInt32(iterator.next()!)
    let b1 = UInt32(iterator.next()!)
    let b2 = UInt32(iterator.next()!)
    let b3 = UInt32(iterator.next()!)

    return b0 << 24 | b1 << 16 | b2 << 8 | b3
}
```

Use lower-level loads only when the input representation and alignment are intentionally controlled.

## Manual allocation

When code manually allocates memory, review allocation, initialization, deinitialization, and deallocation as a single lifecycle.

Checklist:

* Is capacity measured correctly?
* Is alignment correct?
* Is memory initialized before typed reads?
* Is every initialized element deinitialized exactly once when needed?
* Is allocated memory deallocated exactly once?
* Are error paths handled?
* Is partial initialization handled safely?
* Is ownership transferred clearly?

Example shape:

```swift
final class TemporaryIntBuffer {
    private let pointer: UnsafeMutablePointer<Int>
    let count: Int

    init(count: Int, initialValue: Int) {
        self.count = count
        self.pointer = UnsafeMutablePointer<Int>.allocate(capacity: count)
        self.pointer.initialize(repeating: initialValue, count: count)
    }

    deinit {
        pointer.deinitialize(count: count)
        pointer.deallocate()
    }

    subscript(index: Int) -> Int {
        precondition(index >= 0 && index < count)
        return pointer[index]
    }
}
```

This wrapper controls lifecycle, bounds, and ownership. It is still low-level and should be justified, but it is safer than spreading pointer allocation across call sites.

## Uninitialized capacity APIs

Some APIs allow writing directly into uninitialized storage to avoid extra initialization or copying.

Review questions:

* Does the closure initialize exactly the number of elements it reports?
* Are all initialized elements valid values?
* What happens if the closure throws?
* Is the initialized count correct on every path?
* Are partially initialized values cleaned up correctly by the API contract?

Do not use uninitialized-capacity APIs unless the initialization protocol is clear and tested.

## `unsafeBitCast`

`unsafeBitCast` should be rare.

Use it only when:

* there is no safe conversion API
* the source and destination representations are layout-compatible
* size and alignment assumptions are known
* the result is valid for the destination type
* the code documents why the cast is correct
* tests cover the exact representation assumptions

Do not use it for:

* numeric conversion
* pointer lifetime extension
* converting arbitrary bytes into a Swift value
* bypassing generic or protocol design
* avoiding a safe initializer
* working around Sendable or actor-isolation diagnostics

Prefer safe APIs such as `init(bitPattern:)`, explicit byte parsing, typed initializers, or standard library conversions when available.

## `Unmanaged`

`Unmanaged` is for explicit ownership transfer across APIs that cannot express Swift ARC ownership, often C or Core Foundation callbacks.

Review questions:

* Who owns the object before the call?
* Does the callee retain, release, store, or only borrow the pointer?
* Is `passRetained` balanced with a consuming release path?
* Is `passUnretained` used only when the object is guaranteed to outlive the callback?
* Are callbacks invoked exactly once or potentially many times?
* Is there a cleanup callback?
* Does the code survive cancellation or early failure?

Do not use `Unmanaged` just to avoid ARC. Use it only when the ownership contract is explicit and external to Swift ARC.

## `unowned(unsafe)`

`unowned(unsafe)` disables the runtime safety check that normal `unowned` provides.

Use it only in exceptional low-level code where:

* the lifetime relationship is formally guaranteed
* the object graph makes dangling access impossible
* performance measurement proves normal `unowned` or `weak` is a problem
* the invariant is documented
* tests cover destruction order

Prefer:

* strong references when ownership is intended
* `weak` when nil is a valid state
* normal `unowned` when lifetime is guaranteed and a runtime check is acceptable

Do not use `unowned(unsafe)` as a casual performance shortcut.

## `nonisolated(unsafe)`

`nonisolated(unsafe)` bypasses normal actor-isolation checking.

Use extreme caution.

Review questions:

* What synchronization protects the accessed state?
* Can the value be read and written concurrently?
* Is the declaration immutable, thread-safe, or externally synchronized?
* Is this hiding a real data race?
* Could the design use `nonisolated`, `Sendable`, actor isolation, a lock, or a snapshot instead?
* Is the invariant documented at the declaration site?

Do not use `nonisolated(unsafe)` to silence concurrency diagnostics without a separate synchronization story.

## C and C++ interop

C and C++ APIs can import unsafe memory behavior into Swift.

Review questions:

* Does the C API borrow the pointer only for the duration of the call?
* Does it store the pointer?
* Does it require the memory to be mutable?
* Does it require null termination?
* Does it expect a byte count, element count, or sentinel?
* Does the API write into the buffer?
* Who owns returned memory?
* How is returned memory freed?
* Are callbacks invoked synchronously or asynchronously?
* Are C structs packed, aligned, or containing pointers?
* Are Swift objects passed through `void *` with correct retain/release behavior?

Prefer small adapter layers:

* keep C pointer handling in one file or module
* translate unsafe results into Swift values immediately
* document ownership and lifetime at the boundary
* validate counts and nullability
* avoid leaking C pointer types into high-level app code

## Safe wrappers

Unsafe code should usually be wrapped behind a safe API.

A safe wrapper should:

* validate input sizes
* avoid pointer escape
* preserve lifetime
* enforce bounds
* handle errors
* hide raw pointers from callers
* document representation assumptions
* keep unsafe expressions small
* expose value-oriented Swift results

Example:

```swift
enum PacketError: Error {
    case tooShort
    case invalidVersion
}

struct Packet {
    var version: UInt8
    var payloadLength: UInt16
}

func parsePacketHeader(from bytes: [UInt8]) throws -> Packet {
    guard bytes.count >= 3 else {
        throw PacketError.tooShort
    }

    let version = bytes[0]
    guard version == 1 else {
        throw PacketError.invalidVersion
    }

    let payloadLength = UInt16(bytes[1]) << 8 | UInt16(bytes[2])

    return Packet(version: version, payloadLength: payloadLength)
}
```

This wrapper exposes a safe Swift value and validates the input. If a lower-level implementation is later required for performance, keep the public contract safe.

## Prefer modern safe memory tools when available

When the project uses a Swift toolchain that supports them, consider safer memory-oriented APIs before raw pointers.

Examples:

* `Span`
* `RawSpan`
* `MutableSpan`
* `OutputSpan`
* `UTF8Span`
* borrowing APIs
* noncopyable types
* standard library APIs that avoid unnecessary copies

Use these when they express the same lifetime and performance goal without allowing pointer escape or invalid memory access.

Review guidance:

* Prefer non-escapable views over raw pointers when they fit.
* Prefer typed safe APIs over manual binding.
* Prefer explicit parsing over struct reinterpretation for external data.
* Prefer safe initialization helpers over manually initialized buffers.

## Unsafe and concurrency

Unsafe memory can bypass Swift's data-race protection.

Review questions:

* Is mutable memory shared across tasks or threads?
* Is access synchronized?
* Is the pointer wrapper marked `Sendable` or `@unchecked Sendable`?
* What invariant makes it safe to send?
* Does a task capture a pointer whose owner can deallocate?
* Does cancellation skip cleanup?
* Does an actor expose unsafe storage outside isolation?

Avoid:

* sharing mutable pointers across tasks without synchronization
* using unsafe pointers to avoid `Sendable`
* returning unsafe buffers from actors
* storing continuation state in raw memory without lifecycle control
* wrapping non-thread-safe C state in `@unchecked Sendable` without a lock or actor boundary

## Unsafe performance review workflow

When unsafe code appears in a performance review:

1. Identify the safe baseline.

   * What safe implementation exists?
   * Is it too slow, too allocating, or unable to express the operation?

2. Locate the measured cost.

   * Time Profiler, Allocations, benchmark, signposts, or optimized SIL.

3. Identify the unsafe assumption.

   * Lifetime, bounds, binding, alignment, initialization, aliasing, thread safety, ownership, or layout.

4. Check whether the assumption is locally enforceable.

   * Can the wrapper validate it?
   * Is the invariant documented?
   * Are invalid inputs rejected?

5. Prefer a safe wrapper.

   * Keep unsafe code small and internal.
   * Return safe Swift values.
   * Avoid exposing raw pointers.

6. Validate correctness and speed.

   * Unit tests, fuzz tests, sanitizers, benchmarks, release builds.

## Code review checklist

Use this checklist when reviewing unsafe Swift.

* [ ] Is unsafe code necessary?
* [ ] Is there a safe baseline implementation?
* [ ] Is there measurement if unsafe was introduced for performance?
* [ ] Is the unsafe region small?
* [ ] Is the unsafe code hidden behind a safe API?
* [ ] Are lifetime assumptions documented?
* [ ] Are bounds checked?
* [ ] Are alignment requirements satisfied?
* [ ] Is memory binding correct?
* [ ] Is memory initialized before reads?
* [ ] Is memory deinitialized and deallocated correctly?
* [ ] Is ownership transfer explicit?
* [ ] Is pointer escape prevented?
* [ ] Are C callbacks and cleanup paths handled?
* [ ] Is concurrency synchronized?
* [ ] Are `@unchecked Sendable`, `nonisolated(unsafe)`, and `unowned(unsafe)` justified?
* [ ] Is `unsafeBitCast` avoided or heavily justified?
* [ ] Are tests covering invalid, short, malformed, and boundary inputs?
* [ ] Is strict memory safety mode considered where available?

## Common gotchas

* A pointer from `withUnsafeBytes` must not escape the closure unless the API explicitly guarantees it.
* `baseAddress` can be nil for an empty buffer.
* Buffer count may be bytes or elements depending on the type.
* `capacity` is not initialized count.
* Raw memory is not automatically bound to the type you want.
* `assumingMemoryBound(to:)` does not bind memory.
* Pointer casts do not fix alignment.
* Pointer casts do not fix endianness.
* Swift struct layout is not a portable file/network format by default.
* `unsafeBitCast` is not a conversion API.
* `Unmanaged` requires an explicit ownership balance.
* `unowned(unsafe)` can create dangling references without a runtime trap.
* `nonisolated(unsafe)` can hide data races.
* `@unchecked Sendable` does not make unsafe storage thread-safe.
* Unsafe code can make compiler optimizations harder, not easier.
* Unsafe code should be easier to audit than the safe code it replaces.

## Testing guidance

For unsafe wrappers, recommend:

* unit tests for normal cases
* tests for empty and short buffers
* tests for boundary lengths
* tests for malformed input
* tests for large input
* tests for cancellation and cleanup paths
* tests for repeated allocation/deallocation
* tests for concurrent access if shared
* fuzz tests for parsers and binary formats
* Address Sanitizer or Thread Sanitizer where applicable
* release-build benchmarks for performance claims

Debug builds are useful for diagnostics, but do not use Debug performance as evidence for unsafe optimization.

## Output guidance

When this reference is used, include:

```markdown
## Unsafe boundary

Identify the unsafe construct and why it is being used.

## Safety assumptions

List lifetime, bounds, binding, alignment, initialization, ownership, aliasing, and thread-safety assumptions.

## Risk

Explain what can go wrong if the assumptions are false.

## Safer alternative

Suggest a safe API, safe wrapper, `Span`/`RawSpan` when available, explicit parsing, or a narrower unsafe region.

## Validation

Recommend tests, sanitizers, Instruments, optimized builds, and benchmarks.
```

If unsafe code is not justified by correctness or measurement, recommend removing it or isolating it behind a safe wrapper.
