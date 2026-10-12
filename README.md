# Engineering notes

Architecture write-ups from production mobile work.

Most of what I've built professionally isn't mine to publish — it's employer or
client IP. These notes are how I talk about it anyway: the problem, the design, the
trade-offs and what it cost. **Patterns and public APIs only. No employer source
code; every snippet here was written for the write-up.**

---

### [One service, one player](one-service-one-player.md)

A media app accumulates `ExoPlayer` instances by convenience, and four familiar bugs
turn out to be one bug reported four times — nothing owns the answer to "what is
playing?" On collapsing to a single `MediaLibraryService`, what that constraint gives
you for free (notification, audio focus, Android Auto, process death) and what it
genuinely costs (everything must be a `MediaItem`, previews become impossible,
controllers are async).

### [Porting a Kotlin Flow architecture to Swift Concurrency](kotlin-flow-to-swift-concurrency.md)

`Flow` → `AsyncStream` is the obvious mapping and it's wrong in four specific ways:
`AsyncStream` has one consumer where `SharedFlow` multicasts; `StateFlow`'s real
analogue is an `@Observable` property, not a stream at all; `flatMapLatest` is
cancellation that Swift makes you write out by hand; and `@Observable` notifies on
every set where `MutableStateFlow` deduplicates. Includes the full mapping table and
the rule that makes it tractable — port the *guarantee*, not the mechanism.

---

Mobile engineer, Kotlin and Swift. More at
[github.com/sankalp047](https://github.com/sankalp047).
