# Porting a Kotlin Flow architecture to Swift Concurrency

*The 1:1 mapping is wrong in four specific ways. Here they are.*

---

## The premise

You have a working Android app: `StateFlow`-backed ViewModels, `SharedFlow` for
events, cold `Flow`s from the repository layer, `flatMapLatest` for
search-as-you-type, `combine` for derived state. Now build the same product on iOS,
and you'd like the architecture to survive the trip — not because consistency is
pretty, but because two codebases that reason differently about the same product
drift, and then the same bug has to be found twice.

The obvious move is to find Swift's equivalent of each Kotlin piece. Swift has
`AsyncStream`, which is a sequence of values over time, so `Flow` → `AsyncStream`.

That single substitution is the source of almost every bug in the port.

## Trap 1: AsyncStream has exactly one consumer

`Flow` is cold: each `collect` starts an independent execution. `SharedFlow` is hot
and genuinely multicast — N collectors all receive every emission.

`AsyncStream` is neither. It's hot, and it has **one** consumer. Two `for await`
loops over the same `AsyncStream` don't both get every value; they *race*, and each
value goes to whichever is currently suspended. Nothing warns you.

```swift
// Illustrative. This is broken.
let events = AsyncStream<PlaybackEvent> { ... }

Task { for await e in events { analytics.record(e) } }   // gets ~half
Task { for await e in events { updateNowPlaying(e) } }   // gets ~the other half
```

On Android the equivalent `SharedFlow` is correct and both collectors work. In Swift
this ships, passes a casual test with one subscriber, and then analytics are
mysteriously missing half their events.

If you need real multicast, you build it: one consumer task fanning out to
registered handlers, or an `@Observable` object that both call sites read. There is
no drop-in `SharedFlow`.

## Trap 2: StateFlow's analogue isn't a stream at all

This is the mapping most ports get backwards.

`StateFlow` is not primarily a stream — it's **a value you can observe**, with a
current value always available. Its stream-ness is the mechanism, not the point.
Swift already has that mechanism, and it isn't `AsyncStream`:

```kotlin
// Kotlin
private val _state = MutableStateFlow(PlayerState.idle)
val state: StateFlow<PlayerState> = _state.asStateFlow()
```

```swift
// Swift — the actual equivalent
@Observable @MainActor
final class PlayerModel {
    private(set) var state: PlayerState = .idle
}
```

No stream, no subscription, no bridging. SwiftUI views that read `model.state`
re-render when it changes, tracked per-property. Porting a `StateFlow` to an
`AsyncStream` and then driving `@State` from a `.task { for await ... }` loop
reimplements, by hand and worse, what `@Observable` does for free — and introduces a
window where the view has mounted but the first value hasn't arrived, which
`StateFlow`'s always-present current value specifically prevents.

The working mapping table:

| Kotlin | Swift | Note |
|---|---|---|
| `StateFlow<T>` | `@Observable` property | not a stream |
| `MutableStateFlow` | `private(set) var` + setter | |
| `combine(a, b) { }` | computed property | recomputed on read, tracked automatically |
| `map` on `StateFlow` | computed property | |
| `distinctUntilChanged` | **nothing** — see trap 4 | |
| `SharedFlow` (replay 0) | `AsyncStream` *if single consumer* | else hand-rolled fan-out |
| cold `Flow<T>` | `async throws -> T`, or `AsyncThrowingStream` | one-shot work is just a function |
| `flatMapLatest` | task cancellation — see trap 3 | |
| `stateIn(scope, Eagerly)` | unnecessary | |
| `viewModelScope` | `@MainActor` + stored `Task` | lifetime is manual |
| `Dispatchers.Main` | `@MainActor` | compiler-enforced |
| `withContext(IO)` | `nonisolated` + `await` | no dispatcher to name |

Note how many rows collapse to "you don't need this." Roughly a third of a Flow
architecture is machinery for *observability over time* that `@Observable` provides
structurally. The port gets smaller, which is disorienting if you're measuring
fidelity by how recognisable the translation is.

## Trap 3: flatMapLatest is cancellation, and Swift makes you say so

Search-as-you-type is the canonical case: each keystroke should cancel the in-flight
request.

```kotlin
// Kotlin — cancellation is in the operator
val results = query
    .debounce(300)
    .flatMapLatest { repo.search(it) }   // previous search cancelled here
    .stateIn(viewModelScope, Eagerly, emptyList())
```

`flatMapLatest` is doing something important and nearly invisible: cancelling the
previous inner flow. Swift has no operator that does this, so the cancellation
becomes explicit — which is more code, and also the first time anyone on the team
sees clearly what was happening:

```swift
// Illustrative.
@Observable @MainActor
final class SearchModel {
    private(set) var results: [Track] = []
    private var searchTask: Task<Void, Never>?

    func query(_ text: String) {
        searchTask?.cancel()                 // ← was flatMapLatest
        searchTask = Task {
            try? await Task.sleep(for: .milliseconds(300))   // ← was debounce
            guard !Task.isCancelled else { return }
            let found = (try? await repo.search(text)) ?? []
            guard !Task.isCancelled else { return }          // ← and again, after await
            results = found
        }
    }
}
```

Two details that bite:

**Cancellation is cooperative.** `Task.cancel()` sets a flag. It does not unwind
anything. Without the `isCancelled` checks *after every await*, a cancelled search
still assigns its stale results, and you get the exact out-of-order-results bug
`flatMapLatest` existed to prevent. Kotlin's coroutine cancellation is also
cooperative, but the suspension points inside `repo.search` are usually already
cancellation-aware, so it tends to work by accident.

**`viewModelScope` has no equivalent.** Flows collected in `viewModelScope` die with
the ViewModel. A Swift `Task` captured by an object outlives that object's
observable usefulness unless you cancel it — `deinit` on an `@Observable` is the
wrong place for `@MainActor` work, so cancellation is usually tied to the view's
`.task` modifier or an explicit teardown. This is the single most common leak in a
Flow→Swift port.

## Trap 4: @Observable fires on every set, not on change

`MutableStateFlow` deduplicates by value: assigning an equal value emits nothing.
`StateFlow` collectors are already `distinctUntilChanged`.

`@Observable` has no such behaviour. Every `set` notifies, equal or not. So a
poll-and-assign that was free on Android becomes a render per tick:

```swift
// Was invisible on Android. Here it re-renders every 500 ms.
for await tick in clock {
    state = computeState()          // identical value, still notifies
}
```

The fix is trivial and must be remembered everywhere:

```swift
let next = computeState()
if next != state { state = next }   // requires Equatable
```

Worth making a habit at every assignment boundary, because the symptom — a slightly
hot app, scroll jank under a progress timer — never points at the cause.

A related asymmetry in the other direction: `@MainActor` is **compiler-enforced**,
where `Dispatchers.Main` is a convention. Porting surfaced two places where the
Kotlin code was touching UI state off the main thread and had simply never crashed
under test. Swift refused to compile them. That's the port paying for itself.

## What I'd tell someone starting one

**Don't port the mechanism. Port the guarantee.**

For each piece of the Kotlin architecture, write down what it *promises* — "a current
value always exists," "only the newest request wins," "equal values don't
propagate," "this dies with its owner" — and then satisfy that promise the way the
target platform prefers. A faithful translation of `StateFlow` into `AsyncStream`
preserves the shape and loses the guarantee. A computed property on an `@Observable`
looks nothing like the original and keeps it.

The useful side effect is that writing those guarantees down is an audit of the
Android code. Two of ours turned out not to hold there either.

---

*Describes architecture and public API only. No employer source code; all snippets
written for this write-up.*
