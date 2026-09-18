# Fodaxman Solo Point Five — race splits

A single-page tool for working with checkpoint times from the
**FODAXMAN XTRI Solo Point Five** (Serra do Rio do Rastro, SC — 2 km swim,
87 km bike, 22 km run). It serves two jobs:

- **Simulation.** The athlete changes a split and watches the rest of the day move.
- **Race day.** Staff read clock times per checkpoint and open its map link.

No build step, no dependencies, no server. Open `index.html`.

```
git clone <this repo>
cd fodaxman-splits
open index.html          # or: python3 -m http.server 8000
```

The only external request is the Archivo webfont from Google Fonts. Offline,
it falls back to the system sans and everything else still works.

---

## Data model

One flat, ordered array of sections. A section is a stretch of course ending
at a checkpoint — not a point in time.

```js
{
  start: 21600,                  // gun time, seconds since midnight
  savedAt: 1758193200000,        // ms epoch of the last write
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

The paste dialog takes three things, previewing each before it commits:

1. **Raw race notes** — leg headings, `6:44 - fim natação (44')`,
   `(T1 = 11')`. Parenthetical durations are ignored as redundant; a
   `largada da bike` line following an explicit `T1` is recognised as the same
   moment rather than a duplicate checkpoint.
2. **Three columns** — clock, name, elapsed (tab or comma separated).
3. **A JSON backup** produced by *Copy backup*, which restores map links and
   notes too.

Clock times win over parenthetical elapsed times where the two disagree.

Imported bike checkpoints get the 87 km leg split evenly and flagged `est`, so
the leg pace is right while the per-section numbers are honestly marked as
guesses. Typing a real distance clears the flag.

## Persistence

`localStorage`, written on every mutation, validated field by field on read,
with a migration path from the previous storage key. The header shows the
clock time of the last write, and turns red if the browser refuses to store
(private window, blocked site data) instead of failing silently.

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
1. Model            SPORTS table, the seeded race, storage
2. Time             parsing and formatting (h:mm:ss, 11', 1h15')
3. Derived          cumulative, totals, pace, leg grouping
4. Mutations        the two editing rules, move, add, remove
5. Render           timeline strip, leg tables, rows
6. Map links        coordinate detection and URL validation
7. Importer         raw-notes parser
8. Wiring           dialogs, toasts, undo, keyboard
```

Only `http:` and `https:` URLs are accepted as map links; bare `lat, lng` is
converted to a Google Maps search URL.

## Not done yet

- Elevation per checkpoint, feeding an elevation profile under the timeline.
- Cut-off times per checkpoint, with the margin shown against each.
- A live "where is he now" marker for the staff view.
- Real bike checkpoint distances — the six in the seeded race are still even
  splits of 87 km, so per-section km/h is not yet meaningful.
