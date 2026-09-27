# Gelato — speed-patch fork

This is a personal fork of [lostb1t/Gelato](https://github.com/lostb1t/Gelato)
(the on-demand Stremio-catalog plugin for Jellyfin), patched for one thing:
**opening a title you've never seen before shouldn't be slow to *look at*.**

Everything else is unmodified upstream Gelato. This fork exists to be
transparent about that one change, not to replace or compete with the
original project — please use [upstream](https://github.com/lostb1t/Gelato)
unless you specifically want this patch.

## What's patched

**Branch:** [`speed-patch`](https://github.com/corveck79/Gelato/tree/speed-patch),
based on upstream `main` at the time of writing (three commits ahead of the
`v0.26.20.0` release: two scheduled-task fixes and a housekeeping commit,
none of which touch the file below).

**File changed:** `Decorators/MediaSourceManagerDecorator.cs`, one method
(`GetStaticMediaSources`). Nothing else.

### The problem

Gelato materializes a title (creates the Jellyfin item, fetches its
metadata, syncs its available streams from your Stremio addon) the first
time something *opens* it — not when it's added to a catalog. That's the
whole point of the addon-backed model: nothing is imported until someone
actually looks at it.

The trouble is *what counts as "opens it"*. Gelato's own `IsInsertableAction`
check treats a plain detail-page view (`GET /Users/{id}/Items/{id}`) the
same as an actual playback request (`PlaybackInfo`). Both used to block on
`GelatoManager.SyncStreams` — a full round trip to your Stremio addon plus a
database write of every stream row it returns — before the response went
out. Measured on a real server (AMD system, SSD-backed SQLite, addon
answering in ~1–2s):

| Request | Before | After |
|---|---|---|
| Detail view of a **new** title | 2–4s | ~0.7s (once the sync has finished in the background) |
| `PlaybackInfo` (Play) | unchanged | unchanged — still waits for the sync |

So: reading a synopsis and looking at a poster cost the same network round
trip as actually starting a stream, before you'd even touched Play.

### The fix

In `GetStaticMediaSources`, the stream sync now only *blocks the response*
when the request is genuinely about to play something
(`ctx.IsPlaybackInfoAction()` — covers both the GET and POST `PlaybackInfo`
actions Jellyfin's various clients use). Every other insertable action —
the detail view chief among them — still *starts* the exact same
single-flighted sync (so a second near-simultaneous request doesn't
duplicate the work, same as before), it just doesn't wait for it. The sync
finishes in the background and is normally done well before anyone reaches
the play button; if they get there first, `PlaybackInfo` waits for it
exactly as the unpatched code always did.

Nothing about *what* gets synced, cached (`StreamTTL`), or written changes.
No config option, no new setting — same behavior for playback, faster
response for everything that isn't.

Full rationale is in the patch's inline comments and its
[commit message](https://github.com/corveck79/Gelato/commits/speed-patch).

### What this does *not* fix

This patch only addresses the stream-sync half of a first open. The other
half — the metadata fetch that creates the item in the first place
(`GelatoManager.InsertMeta`, including, for movies, an extra TMDB lookup for
digital release dates) — is unchanged and still happens synchronously.
Measured at roughly 0.5–1.5s depending on the addon's response time. A
brand-new title's detail view is therefore faster than before, not
instant.

## Building

```sh
git clone --branch speed-patch https://github.com/corveck79/Gelato.git
cd Gelato
dotnet publish -c Release
```

Targets the same Jellyfin ABI as upstream (`12.1.0.0`, see `build.yaml`).
Built and tested against Jellyfin 12.1.0 / Gelato's own dependency set with
the official `mcr.microsoft.com/dotnet/sdk:10.0` image.

## Installing

Drop the built `Gelato.dll` (and its unchanged dependency DLLs — MonoTorrent,
Mono.Nat, ReusableTasks) into your existing Gelato plugin folder, replacing
the upstream build. Everything else — config, catalogs, your library — is
untouched; this only changes the compiled code.

## License

Gelato is [GPLv3](LICENSE). This fork keeps that license, unmodified, per
section 5. The change described above is the modification required to be
disclosed under section 5(a); this README and the branch name are that
disclosure, and the patch commit carries the same notice inline.

All credit for Gelato itself goes to [lostb1t](https://github.com/lostb1t)
and its contributors — this fork adds one behavioral change on top of their
work, nothing more.
