# One service, one player

*Why a media app should own exactly one player instance, and what that constraint
buys you.*

---

## The symptom

A media app accumulates players. Not by design — by convenience.

Live radio ships first, with an `ExoPlayer` held in the radio screen's ViewModel.
Then podcasts arrive, and they need seeking, speed control and a position bar that
live radio doesn't have, so they get their own. Then video interviews, which need a
surface. Then a thirty-second preview in a search result, which seems too small to
justify ceremony. Four players, four lifecycles.

The bug reports are always the same four:

1. Two audio sources playing simultaneously.
2. The notification shows a podcast while the radio stream is audible.
3. Pausing from the lock screen stops one of them and the other keeps going.
4. Unplugging headphones pauses one and not the others.

These read as four bugs. They are one bug, reported four times: **nothing in the app
owns the answer to "what is playing right now?"**

Each fix makes it worse. A bus event telling other players to stop is a distributed
mutual-exclusion protocol written by hand, and it will be wrong in the race where
two players start within the same frame. Audio focus helps less than it looks —
`AudioManager` focus is granted per *app*, so your own four players never contend
for it, only other apps do. The platform cannot save you from yourself here.

## The actual shape

Media3's `MediaLibraryService` wants to be the only player. Taking that literally:

- **One service**, holding **one** `ExoPlayer` instance, for the entire app.
- **One** `MediaSession`, which is what the OS, the lock screen, Bluetooth controls,
  Wear and Android Auto all talk to.
- The UI layer holds a `MediaController` — a *client* of that session, not a player.
  No screen constructs a player. No ViewModel holds one.
- Every kind of content becomes a `MediaItem` carrying `MediaMetadata`.

The last point is the load-bearing one. Content type stops being a code path and
becomes data:

```kotlin
// Illustrative — not production source.
fun MediaItem.Builder.asLiveStream(station: Station) = this
    .setUri(station.streamUrl)
    .setMediaId("station:${station.id}")
    .setMediaMetadata(
        MediaMetadata.Builder()
            .setTitle(station.name)
            .setArtist(station.nowPlaying)
            .setArtworkUri(station.logoUri)
            .setIsPlayable(true)
            // live: no duration, seeking must be refused not just hidden
            .setExtras(bundleOf(KEY_IS_LIVE to true))
            .build()
    )
```

A podcast episode builds the same object with a duration and `KEY_IS_LIVE` absent.
The player doesn't branch. The *UI* branches, on metadata it reads from the
controller — which is the right place for it, because that's a presentation
question.

## What falls out for free

**The notification.** `MediaSessionService` publishes it from session state. There is
no notification code to keep in sync, so the class of bug where the notification
disagrees with reality stops existing — not "gets fixed," stops being
representable.

**Audio interruptions.** `setAudioAttributes(..., handleAudioFocus = true)` on one
player means call interruptions, other apps and headphone unplugs are handled once.
With four players you write that logic four times and get it subtly wrong three
times.

**Android Auto and Wear.** Both are clients of the browse tree, not of your UI. If
you've implemented `onGetLibraryRoot` and `onGetChildren` to serve your catalog
hierarchy, Auto largely works because it consumes the same tree the app does:

```kotlin
// Illustrative.
override fun onGetChildren(
    session: MediaLibrarySession,
    browser: ControllerInfo,
    parentId: String,
    page: Int, pageSize: Int,
    params: LibraryParams?
): ListenableFuture<LibraryResult<ImmutableList<MediaItem>>> =
    future {
        when (parentId) {
            ROOT -> LibraryResult.ofItemList(topLevelTabs(), params)
            STATIONS -> LibraryResult.ofItemList(stationItems(), params)
            else -> LibraryResult.ofError(SessionError.ERROR_BAD_VALUE)
        }
    }
```

This is the clearest payoff of the one-service design: a surface you didn't
specifically build for arrives nearly complete, because it was never asking your
screens anything.

**Process death.** One place restores state. Not four that each think they own it.

## What it costs

Honest accounting, because this isn't free.

**Everything must be expressible as a `MediaItem`.** Content that doesn't fit has to
be made to fit, and that forces modelling decisions you were avoiding. Live radio
has no duration and no meaningful seek. The temptation is to report a fake duration
so progress UI "works." Don't — report it as live and refuse seeking at the player
level, so that every client, including ones you didn't write, behaves correctly.
Hiding the scrubber in your own UI fixes your UI and leaves Android Auto broken.

**One player genuinely means one.** Preview-while-playing is now *impossible*, not
merely discouraged. That's a real product constraint and it surfaces at design time
as a decision: duck the main stream, replace it, or decline the preview. Previously
that decision was made accidentally, by whichever player happened to start last.

**A controller is asynchronous.** `MediaController` connects over IPC, so a screen
can mount before it exists. Every screen needs a connecting state. With a local
player, `player.play()` was synchronous and always available — this is a genuine
ergonomic regression, and worth it, but it isn't nothing.

**Testing moves.** You can no longer instantiate a player in a unit test and assert
on it. Playback logic belongs behind an interface the service consumes; what you test
is that interface, with the service as a thin shell.

## The rule that generalises

The specific lesson is about Media3. The general one isn't:

> When a resource is inherently singular — the audio output, the lock screen, the
> foreground service, the user's attention — make exactly one thing own it, and make
> everything else a client of that owner.

Four players was never a performance problem. It was an *authority* problem: four
components each believing they were in charge of something only one of them could
be. No amount of coordination fixes a design where the authority is ambiguous. You
have to put it somewhere.

---

*Describes architecture and public API only. No employer source code; all snippets
written for this write-up.*
