# Race splits

A single-page tool for working with checkpoint times across several races. It
began as a one-off for the **FODAXMAN XTRI Solo Point Five** (Serra do Rio do
Rastro, SC — 2 km swim, 87 km bike, 22 km run) and now keeps a library of races
in the browser. It serves three jobs:

- **Library.** A home list of saved races; create, rename, duplicate, delete.
- **Simulation (edit).** The athlete changes a split and watches the rest of the day move.
- **Race day (follow).** A read-only view with a live wall-clock marker showing
  where the athlete is on the timeline right now.

No build step, no dependencies, no server. Open `index.html`.

```
git clone <this repo>
cd race-splits
open index.html          # or: python3 -m http.server 8000
```

The only external request is the Archivo webfont from Google Fonts. Offline,
it falls back to the system sans and everything else still works.

---

## Views & routing

Three views addressed by URL hash, so each is bookmarkable and back-navigable:

| Hash | View |
|---|---|
| `#/` | Home — the race library |
| `#/race/:id/edit` | Edit one race (the full simulation surface) |
| `#/race/:id/follow` | Follow one race (read-only + live marker) |

An unknown hash, or a well-formed route whose `id` is no longer stored (deleted
race, stale bookmark), redirects to home with a toast. Each view mounts against
one race loaded from storage; the module-level `active` race is the mount's only
mutable context.

## Data model

A **race** wraps identity and metadata around one flat, ordered array of
sections. A section is a stretch of course ending at a checkpoint — not a point
in time.

```js
{
  id:       "h84paun",           // stable per-race id
  name:     "Fodaxman Solo Point Five 2025",
  location: "Serra do Rio do Rastro, SC",
  date:     "2025-09-27",        // ISO, or ""
  dist:     { swim: 2000, bike: 87000, run: 22000 },  // per-race leg metres
  start:    21600,               // gun time, seconds since midnight
  savedAt:  1758193200000,       // ms epoch of the last write
  sections: [
    {
      id:    "k3f9a2p",          // stable, used for DOM keying and drag
      name:  "Mirante Serra",
      sport: "swim" | "bike" | "run" | "trans",
      dist:  14500,              // metres, or null
      dur:   4800,               // seconds — THE source of truth
      est:   true,               // dist is a placeholder, not measured
      note:  "",
      map:   ""                  // https URL
    }
  ]
}
```

**`dur` is primary; everything else is derived.** Elapsed time is a prefix sum
over `dur`; clock time is `start + elapsed`; pace is `dist / dur`. Legs are
computed as runs of consecutive sections sharing a `sport`, so nothing stores
leg membership and inserting a section can never desynchronise it.

The per-race `dist` seeds the even-split importer (below) for that race only;
new races seed it from the per-sport defaults (2 km / 87 km / 22 km).

## The two editing rules

Both are live, and which one applies depends on the column you type in.

| You edit | What happens | Invariant held |
|---|---|---|
| **Split** | `dur[i] = v`. Every later checkpoint shifts. | All other splits |
| **Elapsed** or **Clock** | `dur[i]` absorbs the delta and `dur[i+1]` gives it back. | The finish time |

Editing a split is how you ask *"what if the Serra climb had taken 10 minutes
less?"*. Editing an elapsed time is how you correct a mis-recorded checkpoint
without moving the finish.

## Reordering

Sections drag by their handle — pointer events rather than HTML5
drag-and-drop, so it works under a finger. Keyboard: focus a handle, arrow up
or down.

A move **clears `dur`, `dist` and `est`**: both were measured between the
section's old neighbours and mean nothing in a new slot. `name`, `note` and
`map` describe the place rather than its position, so they travel with it.
Every move is undoable (toast, or Cmd/Ctrl+Z).

Dropping a section inside another leg adopts that leg's sport. Transitions are
the exception both ways: they keep their own type wherever they land, and
dropping onto one never converts a checkpoint into a transition.

## Importing

*Copy data* and the paste dialog share one canonical text format: a name
line, a gun-time line, then one line per checkpoint — `S|B|R distance split
name` (`T split name` for a transition, which never carries a distance).
Distances are kilometres for every sport, up to three decimals; `-` marks a
checkpoint whose distance isn't known. The preview shows what will import;
a malformed line surfaces a warning naming it and commits nothing.

The paste dialog also accepts **a JSON backup** produced by *Copy backup*,
which restores map links and notes too — those are per-device decoration and
never appear in the canonical text format.

Importing replaces a race's `name`, `start` and `sections`, keeping its place
in the library — a restore never orphans the race.

## Follow mode

`#/race/:id/follow` renders the same timeline read-only — no inputs, drag
handles, add/remove, import, or gun-time editing — and overlays a live position
marker driven by the real wall clock.

Because `start` is stored as seconds-of-day only (no date), before-gun and
after-finish fall in the same clock region once the elapsed window wraps past
midnight, and a single `elapsed / total` ratio can't tell them apart. The marker
is placed piecewise instead: let `d = (now − start) mod 86400`; if `d ≤ total`
the marker sits at `d / total`, otherwise it snaps to the nearest boundary —
not-started (0) or finished (`total`). A splitless race shows no marker. The
marker is `aria-hidden` and its position is announced on a polite live region
only when the segment changes, not on every tick; the 1 s tick is cleared on
navigation away and skipped under `prefers-reduced-motion` (one static paint).

## Persistence

`localStorage`, written on every mutation, validated field by field on read.
The library is a light **index** key (`race-splits:index`) listing
`{id, name, location, date, savedAt}`, plus one blob per race under
`race-splits:race:<id>`; editing rewrites only the active race blob and the
index. The header shows the clock time of the last write, and turns red if the
browser refuses to store (private window, blocked site data) instead of failing
silently.

The single race from the previous version migrates in once, non-destructively:
gated on a `race-splits:migrated` sentinel (not an empty index, which the user
can reach by deleting every race), it copies the legacy blob into the new store
and leaves the old key untouched.

It is per-device by design. *Copy backup* → paste elsewhere is how the data
moves. The published-artifact runtime also offers a server-backed `db`
capability, which would sync across devices and viewers, but declaring it makes
the artifact organisation-internal and kills public link sharing — not viable
while race staff need to open it.

## Layout of the source

`index.html` is deliberately one file: it is deployed as a published Claude
artifact, whose hosting requires a self-contained document. The script is
sectioned with banner comments in dependency order:

```
1. Model            SPORTS table, per-race store, migration
2. Time             parsing and formatting (h:mm:ss, 11', 1h15')
3. Derived          cumulative, totals, pace, leg grouping
4. Mutations        the two editing rules, move, add, remove
5. Render           timeline strip, leg tables, rows (readOnly-aware)
6. Map links        coordinate detection and URL validation
7. Importer         raw-notes parser (per-race leg distances)
8. Wiring           dialogs, toasts, undo, keyboard
9. Router + views   hash routing, home library, edit/follow mounts, live marker
```

Only `http:` and `https:` URLs are accepted as map links; bare `lat, lng` is
converted to a Google Maps search URL.

## Not done yet

- Elevation per checkpoint, feeding an elevation profile under the timeline.
- Cut-off times per checkpoint, with the margin shown against each.
- A projected/target-pace overlay in follow mode (the marker shows *position*,
  not a projection).
- Real bike checkpoint distances for the sample race — its six bike checkpoints
  are still even splits of the leg total, so per-section km/h is not yet
  meaningful.
