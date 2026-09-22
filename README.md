# Race Planner

A single-page tool for working with checkpoint times across several races. It
began as a one-off for the **FODAXMAN XTRI Solo Point Five** (Serra do Rio do
Rastro, SC) and now keeps a library of races in the browser, built around
watching one live: on race day, anyone with a link can see where the athlete
should be and when to expect them, and anyone close enough to the course can
correct the picture by typing in times as checkpoints go by. It serves three
jobs:

- **Library.** A home list of saved races. Each card carries **Watch Live**
  and an outlined **Edit** beside it, with rename / duplicate / delete behind
  the cog; new races come from *New race* or a template.
- **Simulation (edit).** The athlete changes a split and watches the rest of
  the day move.
- **Watch.** A live view with a running elapsed counter, two timeline pins —
  where the plan says the athlete should be, and where entered times say they
  are — and a resumable session in which typing in an actual time revises the
  projected finish while the original estimate stays on screen next to it.

No build step, no dependencies, no server. Open `index.html`.

```
git clone <this repo>
cd race-splits
open index.html          # or: python3 -m http.server 8000
```

The only external request is the Archivo webfont from Google Fonts. Offline,
it falls back to the system sans and everything else still works.

One exception, off by default: open the app with a `shorturl` query parameter
(`index.html?shorturl#/race/…`) and pressing Share calls TinyURL's
`api-create.php` to turn the race's ~270-character self-contained link into a
~28-character one. Without the flag nothing leaves the browser at share time.
The flag is per-session and does not travel — the link you share carries no
query string, so it cannot switch the feature on for whoever opens it. If
TinyURL is unavailable, or you are offline, Share falls back to the full
self-contained link, which works exactly as it always has.

---

## Views & routing

A race's URL carries the race itself, not just a reference to it — see
**Content-addressed links** below. Five route shapes, all bookmarkable and
back-navigable:

| Hash | View |
|---|---|
| `#/` | Home — the race library |
| `#/race/:token/edit` | Edit one race (the full simulation surface) |
| `#/race/:token/watch/:watchId` | Watch a named, resumable session for one race |
| `#/race/:token/watch` | Watch — resumes this browser's most recent session for that plan, or starts one |
| `#/race/:token` (bare) | Edit — the shared-link landing. The plan is filed in this browser's library on arrival (see **Content-addressed links**) and Watch Live in the masthead is one tap away |

`#/race/:id/edit` and `#/race/:id/follow` from before this rework still
resolve: an `:id` that matches a race already in this browser's library is
treated as that stable internal id, loaded, and the address bar is rewritten
to the token form in place. `follow` mounts the same read-only timeline watch
now uses, without a session — it has no actuals, no resumability, and stays
only as a landing spot for old bookmarks; nothing in the app links to it
anymore.

A link hands the reader the plan itself, so the bare form lands them in
**edit**, on a race that is now theirs: arriving on a plan this browser has
never seen saves it to the library before mounting (toasting that it did),
rather than the old link-only *transient* mount that vanished the moment they
navigated home. Opening the same link twice matches by digest and files
nothing new.

A token that fails to decode, or a `:watchId` this browser has no record of,
toasts and falls back to home or to a fresh session for that plan — it never
throws. Each view mounts against one race (or, in watch mode, a session's
frozen copy of one) loaded from storage; the module-level `active` race is
the mount's only mutable context.

## Data model

A **race** is a name, a gun time, and one flat, ordered array of sections — no
location, date, or per-race leg distances. A section is a stretch of course
ending at a checkpoint, not a point in time.

```js
{
  id:       "h84paun",           // stable per-race id
  name:     "Fodaxman Solo Point Five 2025",
  start:    21600,                // gun time, seconds since midnight
  savedAt:  1758193200000,        // ms epoch of the last write
  sections: [
    {
      id:    "k3f9a2p",           // stable, used for DOM keying and drag
      name:  "Mirante Serra",
      sport: "swim" | "bike" | "run" | "trans",
      dist:  14500,               // metres, or null if not yet known
      dur:   4800,                // seconds — THE source of truth
      note:  "",
      map:   ""                   // https URL
    }
  ]
}
```

**`dur` is primary; everything else is derived.** Elapsed time is a prefix sum
over `dur`; clock time is `start + elapsed`; pace is `dist / dur`. Legs are
computed as runs of consecutive sections sharing a `sport`, so nothing stores
leg membership and inserting a section can never desynchronise it. A leg
reports the distance it has even when a section is missing one, but withholds
its pace figure until every distance-bearing section in it carries a distance
— a wrong number is worse than none.

## The editing rules

All three are live, and which one applies depends on the column you type in.

| You edit | What happens | Invariant held |
|---|---|---|
| **Split** | `dur[i] = v`. Every later checkpoint shifts. | All other splits |
| **Pace** | `dur[i]` = what that pace implies over the row's distance, then the split rule above. | All other splits |
| **Elapsed** or **Clock** | `dur[i]` absorbs the delta and `dur[i+1]` gives it back. | The finish time |

The Pace column is an input in edit mode wherever the arithmetic is defined —
a sport with a pace unit *and* a distance on the row; without a distance it
stays the derived dash it always was. It is a third way of saying the split,
never a stored field: `dur` remains the only source of truth, and neither the
storage shape nor the canonical text format carries a pace. `m:ss` fields
(swim `/100 m`, run `/km`) mask as you type — `430` shows as `4:30` — and the
bike's `km/h` field masks to digits with a single separator, comma read as a
dot.

Editing a split is how you ask *"what if the Serra climb had taken 10 minutes
less?"*. Editing an elapsed time is how you correct a mis-recorded checkpoint
without moving the finish. Pressing Enter in any editable field commits it and
leaves the field, the same as blurring it; Escape reverts a time field without
saving.

## The timeline strip

In watch mode the bar is drawn against the **revised** plan, not the frozen
one: every section up to the latest actual is as wide as it really took
(underlined, with its own measured split as the label, and the plan it
replaced in the tooltip), and the rest of the day keeps its frozen durations
stacked on top of the last recorded time. The bar's right edge and the
revised finish are therefore the same instant by construction. A run of
checkpoints between two actuals shares the measured span in the frozen plan's
own proportion. Outside watch mode, and before the first actual lands, the
drawn axis is the frozen plan and every mapping below is the identity.

Below ~760px every segment is held to a 44px floor rather than crushed to a
sliver, so the bar is wider than the screen and scrolls sideways. On a phone
that scroll is: swipe-only (the scrollbar is hidden on a coarse pointer), it
never chains to the page or the browser's back-gesture
(`overscroll-behavior-x: contain`), the drag is claimed for panning up front
(`touch-action: pan-x pan-y`), and whichever edge the bar continues past is
faded — a mask toggled from the measured scroll position, so you can see it
is cut off rather than read it as the end of the race.

Because the floor makes the bar no longer a straight time→x mapping, pins
and hour ticks are placed off measured segment boxes; an hour can therefore
compress to a few pixels, so a tick is drawn only where it clears the
previous label. The live position is recentred (smoothly, unless the reader
asked for reduced motion) whenever the pin drifts out of the middle half of
the visible strip and nobody has panned for 30s. Tapping a segment jumps to
its checkpoint row, the same as tapping its course card.

## The course panel

One card per contiguous leg, in race order. A card is a button: tapping it
jumps to that leg's first checkpoint — the row is selected, scrolled clear of
the sticky panel, and given focus (the row itself, not a field in it, so a
phone keyboard is never raised uninvited). It works the same in edit and in
watch.

Scrolling past the panel brings the same legs back as a pinned rail of
pills at the top of the viewport — a **fixed overlay**, not the panel
collapsing in place, and the panel itself is left alone. Collapsing it took
~330px out of the flow mid-scroll, and the document changing height under
the reader is what caused both earlier bugs: the browser's scroll anchoring
pulled the viewport back up, which put the sentinel back on screen, which
expanded the panel — a scroll-down that bounced; and reserving the lost
height instead stopped the bounce but left a hole under the rail on a phone.
Out of the flow, the rail costs the layout nothing. Its pills are clones of
the panel's own chips, so label, sport hue and live state have one source.

## Reordering

Sections drag by their handle — pointer events rather than HTML5
drag-and-drop, so it works under a finger. Keyboard: focus a handle, arrow up
or down.

A move **clears `dur` and `dist`**: both were measured between the section's
old neighbours and mean nothing in a new slot. `name`, `note` and `map`
describe the place rather than its position, so they travel with it. Every
move is undoable (toast, or Cmd/Ctrl+Z).

Dropping a section inside another leg adopts that leg's sport. Transitions are
the exception both ways: they keep their own type wherever they land, and
dropping onto one never converts a checkpoint into a transition.

## Importing

*Copy data* and the paste dialog share one canonical text format: a name
line, a gun-time line, then one line per checkpoint — `S|B|R distance split
name` (`T split name` for a transition, which never carries a distance).
Distances are kilometres for every sport, up to three decimals; `-` marks a
checkpoint whose distance isn't known. The preview shows what will import; a
malformed line surfaces a warning naming it and commits nothing. This is the
only accepted input format — the old raw-race-notes parser (leg headings,
`6:44 - fim natação (44')` lines) is gone; a stray older-format paste is just
a parse error now, not a silently-misread one.

The paste dialog also accepts **a JSON backup** produced by *Copy backup*,
which restores map links and notes too — those are per-device decoration,
excluded from the canonical text format and from share links (see below), and
never restored by a plain-text import.

Importing replaces a race's `name`, `start` and `sections`, keeping its place
in the library — a restore never orphans the race.

## Content-addressed links

A race's URL carries the race itself: the canonical text (above) is
`deflate-raw`-compressed and base64url-encoded into the fragment behind a
one-byte prefix distinguishing that from an uncompressed fallback used when
`CompressionStream`/`DecompressionStream` aren't available — a link made on
an older browser still opens correctly everywhere, it's just longer. A short
digest of the canonical text is the race's link-matching key: it addresses a
race in a URL but never keys it in storage, so renaming, duplicating, or
editing a race never breaks a link to it under its *old* text — reopening
that exact old link finds no match and is filed as a **new** race from the
link's own text (never overwriting an existing one).

Editing rewrites the address bar in place (`history.replaceState`, no history
entry per keystroke) every time the canonical text changes, so the current
URL is always a live, shareable pointer to what's on screen. *Share link*
copies the bare `#/race/:token` form — the plan-carrying landing that opens
in edit and files the race in the receiver's library —
and warns instead of silently handing back a broken link if a race is too
large for a practical URL, offering the canonical text to copy instead.

## Watch sessions

Watching a race happens inside a **session**, its own stored record separate
from the race library: a library race id (nullable — a session can start from
a link to a race not yet saved in this browser), a *frozen* copy of the
canonical text taken the moment the session starts, and a sparse map of
entered actual times keyed by checkpoint. A session is addressed by
`#/race/:token/watch/:watchId`; the bare `#/race/:token` form resumes this
browser's most recent session for that plan, or starts one.

**A session is immune to later edits.** It always renders from its own frozen
text, never from the live race, so simulating a change to the plan in edit
mode leaves a running session's timeline and finish estimate untouched until
it's explicitly **reset** — which clears its entered times and re-freezes
against the race's current plan. Nothing deletes a session on its own; each
watched plan adds one, and there is no retention rule yet (see *Not done
yet*).

Within a session, any checkpoint can be given an actual time, typed as a
clock time, whether or not the plan's own clock has reached it yet — an
athlete running ahead of plan is recorded as they pass. Entering one revises
the projected time of every remaining checkpoint, the finish included; the
frozen plan's original estimate stays visible next to the revision, never
overwritten. An actual earlier than an already-recorded later checkpoint (or
later than an already-recorded earlier one) is rejected with a reason rather
than silently accepted. A checkpoint carrying an actual is visually
emphasised, as is a leg whose checkpoints all do; a checkpoint the wall clock
has passed with no actual yet is de-emphasised instead.

The timeline carries two pins on top of the same piecewise wall-clock
placement `follow` always used (so a race crossing midnight still places
correctly): a hollow **plan** pin at the frozen plan's position for right now,
and — once at least one actual exists — a filled **actual** pin, projected
forward from the last recorded checkpoint at the wall clock's own rate. Since
the drawn axis is measured time once actuals exist, the actual pin sits at the
athlete's own elapsed time directly, while the plan pin is mapped through the
frozen→revised conversion so it keeps pointing at the checkpoint the frozen
plan says is due now. The
gap between them is read from the unclamped numbers, not the drawn (and
therefore end-clamped) pin positions, so a late-running athlete keeps reading
correctly behind long after the frozen finish time has passed rather than the
number decaying toward zero. A running elapsed counter (countdown before the
gun, elapsed during, final time after) shares the same tick. Nothing in a
watch session ever writes to the race blob — entering actuals, like editing
splits in a session's own frozen copy, only ever touches the session's own
stored record.

## Persistence

`localStorage`, written on every mutation, validated field by field on read.
The **race library** is a light index key (`race-splits:index`) listing
`{id, name, savedAt, digest, finish, start}` — the digest, finish time and gun
time are cached here so the home library can render every card, including its
Watch Live link's target, without loading a single race blob — plus one blob
per race under `race-splits:race:<id>`; editing rewrites only the active race
blob and the index. **Watch sessions** mirror that same split under their own
prefix: a light `race-splits:watch-index` (`{id, raceId, digest, createdAt}`)
plus one blob per session under `race-splits:watch:<id>`. The header shows
the clock time of the last write, and turns red if the browser refuses to
store (private window, blocked site data) instead of failing silently.

The single race from the version before the multi-race rework migrates in
once, non-destructively, gated on a `race-splits:migrated` sentinel (not an
empty index, which the user can reach by deleting every race). Two further
one-time migrations run after it: one strips the removed `location`/`date`/
per-race `dist` fields (and the retired `est` flag) from every stored race,
and one backfills the library index's cached `digest`/`finish`/`start` for
any entry saved before those fields existed. Each is gated on its own
sentinel and touches only what it says it touches.

It is per-device by design — a shared link carries the *plan*, never the
times a viewer has entered; each viewer's watch session lives only on the
device that typed into it. *Copy backup* → paste elsewhere is how the full-
fidelity data (notes and map links included) moves between devices. The
published-artifact runtime also offers a server-backed `db` capability, which
would sync across devices and viewers, but declaring it makes the artifact
organisation-internal and kills the public link sharing this is built around.

## Layout of the source

`index.html` is deliberately one file: it is deployed as a published Claude
artifact, whose hosting requires a self-contained document. The script is
sectioned with banner comments in dependency order:

```
1. Model                    SPORTS table, race + watch-session stores, migrations
2. Time                     parsing and formatting (h:mm:ss, 11', 1h15')
3. Derived + watch actuals  cumulative, totals, pace, leg grouping, revised projection
4. Mutations                the two editing rules, move, add, remove
5. Render                   timeline strip, leg tables, rows (readOnly- and watch-aware)
6. Map links                coordinate detection and URL validation
7. Importer                 canonical text format: writer and parser
8. Content-addressed links  token encode/decode, digest
9. Wiring                   dialogs, toasts, undo, keyboard
10. Router + views          hash routing, home library, edit/watch/follow mounts, live pins
```

Only `http:` and `https:` URLs are accepted as map links; bare `lat, lng` is
converted to a Google Maps search URL.

## Versioning and release notes

The app is one file, so it carries one version number: `APP_VERSION` in
`index.html`, `0.2.0` at the time of writing. The footer under every view
prints it and opens **What's new** — the `RELEASES` table in `index.html`,
rendered newest first in the reader's language.

[`CHANGELOG.md`](CHANGELOG.md) holds the same entries in prose. The two are
kept in step by hand: when you bump `APP_VERSION`, add the matching `RELEASES`
entry (it must sit first, and its `v` must equal `APP_VERSION`) and the
matching changelog section.

The minor number moves when the app gains behaviour, the patch number when it
only gets fixes, and the major number when a stored race or an existing link
would stop resolving — which nothing has done yet: every version so far reads
what the one before it wrote.

## Not done yet

- Elevation per checkpoint, feeding an elevation profile under the timeline.
- Cut-off times per checkpoint, with the margin shown against each.
- Real bike checkpoint distances for the sample race — its six bike
  checkpoints are still even splits of the leg total, so per-section km/h is
  not yet meaningful.
- A retention rule for watch sessions — nothing prunes them yet, and each
  watched plan adds one.
- A share link scoped to one specific watch session rather than only the
  plan — sessions and their entered times are local to the device that made
  them, by design (see *Watch sessions*), and sharing one would mean sharing
  state, which is out of scope for this tool.
