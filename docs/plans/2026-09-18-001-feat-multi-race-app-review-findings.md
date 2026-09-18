# Multi-Race Splits App — Plan Review Findings

Review of `2026-09-18-001-feat-multi-race-app-plan.md` by the ce-doc-review persona team (coherence, feasibility, design-lens, scope-guardian, adversarial). Two safe mechanical fixes were already applied to the plan (`real UI in U4` → `U3`; `Gate 2 U11` → `R11`). The items below are proposed fixes that touch plan meaning and warrant sign-off before implementation.

> **Status (2026-09-18): all findings F1–F16 folded into `2026-09-18-001-feat-multi-race-app-plan.md`.** P1 (F1–F4) and P2 (F5–F11) applied as concrete plan changes; F12 as the one-word edit; advisory F13–F16 applied (F13 terminology; F15 watch-live link; F14/F16 folded into the U2/U5 mount-lifecycle and follow-view rework).

Each item: **the problem**, **where it lands in the plan**, **why it matters**, **proposed solution**. Severity: P1 = correctness bug an implementer hits on first test; P2 = real gap; P3 = minor.

---

## P1 — correctness

### F1. Day-wrap marker shows "finished" before the gun
- **Where:** KTD5 / U5 (Planning Contract).
- **Source:** feasibility + adversarial (cross-corroborated).
- **Problem:** The specified formula — `elapsedNow = (nowClockSeconds - race.start)` normalized for day-wrap, then `clamp[0, total]`, position `elapsedNow/total` — cannot produce the "not started" state R9 requires. `race.start` is stored only as seconds-of-day (no date). Before the gun (e.g. now 5:00, start 6:00) the delta is negative; day-wrap turns it into ~23h positive, which clamps to `total` and renders the marker at 100% "finished" — the opposite of the required "not started". Before-gun and after-finish occupy the same clock region once you day-wrap, so a single subtraction can't tell them apart.
- **Why it matters:** Defeats a headline behavior of the follow view; directly contradicts the U5 not-started test.
- **Proposed solution:** Replace the single formula with piecewise logic. Let `d = ((now - start) % 86400 + 86400) % 86400`. If `d <= total` → mid-race, marker `d/total`. Otherwise disambiguate by nearest boundary: if `(86400 - d) < (d - total)` → not-started (marker 0); else finished (marker at total). Document this nearest-boundary convention in KTD5 as the intended behavior, since no race date is stored.

### F2. New empty race is rejected by `normalize()`
- **Where:** U1 (`createRace`), depended on by U2 mount.
- **Source:** feasibility.
- **Problem:** R2 requires "New race" to create an empty race and open it in edit. But `normalize()` (which U1 lists as a pattern to follow) returns null for any object with an empty sections array (`index.html:351` and `:364`). If `createRace` stores `sections: []` and `loadRace` runs the blob through `normalize()`, `loadRace` returns null and the U2 mount has nothing to render — the New-race flow is dead on the first manual test. "Seeded blank sections" is ambiguous on this point.
- **Why it matters:** The primary create action fails immediately.
- **Proposed solution:** Specify that `createRace` seeds exactly one placeholder checkpoint (mirror `removeSection`'s fallback `sec("", "swim", null, 0)`) so the blob survives `normalize()`. Keep `normalize()`'s empty-sections guard intact for backup/import validation.

### F3. Stale/deleted race id in a bookmarked hash → null mount
- **Where:** R5 / U2 (hash router).
- **Source:** design-lens.
- **Problem:** The plan sells per-race URLs as bookmarkable but only handles an unknown/malformed hash *shape*, not a well-formed route (`#/race/:id/edit|follow`) whose id no longer resolves — a deleted race, stale bookmark, a tab left open across a delete, or a hand-edited hash. `loadRace` returns null in exactly this case (U1), so mounting against null is undefined and can throw or paint a blank screen.
- **Why it matters:** Crash/blank on a route the plan explicitly markets as durable.
- **Proposed solution:** Specify that `#/race/:id/edit` and `#/race/:id/follow` whose id is not in the index redirect to `#/` and surface a toast ("That race is no longer here"), treating an unresolvable id the same as an unknown hash.

### F4. Migration resurrects deleted races
- **Where:** U1 migration / Assumptions.
- **Source:** feasibility + adversarial (cross-corroborated).
- **Problem:** The "runs once" guarantee is inferred from an empty index, but R4/U3 let the user delete races and U3 supports an all-deleted empty state. Delete the migrated legacy race (plausible — it's a sample) and the index is empty again; next load sees empty-index + legacy-key-still-present and re-imports it, resurrecting the deleted race as a zombie. Empty-index is not a durable once-only signal.
- **Why it matters:** Delete doesn't stick; the sample race keeps coming back.
- **Proposed solution:** Gate migration on a persistent sentinel key (e.g. write `race-splits:migrated = "1"` the first time `migrateLegacy` runs) instead of "index empty + legacy exists"; check and set that flag so migration runs at most once regardless of later deletions.

---

## P2

### F5. `SPORTS.official` is written where the code has `SPORTS[sport].official`
- **Where:** KTD6, propagated into U1 and U4.
- **Source:** coherence.
- **Problem:** KTD6 refers to the constant two ways within two sentences — first the correct per-sport `SPORTS[sport].official`, then a bare `SPORTS.official`; U1/U4 carry the bare form. `SPORTS` is keyed by sport (`index.html:993` confirms `SPORTS[sport].official`), so `SPORTS.official` is undefined. Seeding `dist:{swim,bike,run}` from `SPORTS.official` yields undefined defaults instead of 2000/87000/22000.
- **Why it matters:** New races get undefined leg distances.
- **Proposed solution:** Normalize all references to the per-sport form. Seed a new race as `{swim: SPORTS.swim.official, bike: SPORTS.bike.official, run: SPORTS.run.official}` (equivalently `SPORTS[sport].official` per sport).

### F6. Global edit listeners leak on remount
- **Where:** U2 approach (event-handler lifecycle).
- **Source:** adversarial (feasibility F12 is the same root).
- **Problem:** The edit view's global handlers — `document` keydown (undo), `window` pagehide, `document` visibilitychange — are attached at wire-up. The U2 note offers "or scope them to the mounted container" as an equivalent, but `document`/`window` handlers can't be container-scoped, so that branch guarantees a leak: repeated home↔edit navigation stacks keydown-undo listeners, and one Cmd-Z then pops several undo steps at once. The plan's only leak check is the U5 interval.
- **Why it matters:** Undo becomes destructive after a few navigations.
- **Proposed solution:** Require explicit `removeEventListener` teardown for the document/window handlers on edit unmount, and add a verification gate that repeats home↔edit navigation and confirms a single undo per Cmd-Z with no listener growth.

### F7. "Reset to race" overwrites any race with the Fodaxman seed
- **Where:** R6 / U2.
- **Source:** adversarial.
- **Problem:** The existing `#btn-reset` calls `RACE()`, the hardcoded Fodaxman seed. In a multi-race app, "Reset to race" on any other race replaces its sections with Fodaxman data. R6 says edit behavior is "preserved verbatim" but its preserved-behavior list omits reset, and no unit says whether the control is removed or repurposed — so an implementer preserving it verbatim ships a destructive, race-agnostic reset.
- **Why it matters:** Silent data loss on an unrelated race.
- **Proposed solution:** In U2/U4 explicitly remove the global "Reset to race" control (its single-race semantics no longer apply) or repurpose it to "reset this race to blank", and state which. Do not carry the `RACE()`-based reset into multi-race verbatim.

### F8. Import/backup-restore drops race identity
- **Where:** U2 / R6 / U1.
- **Source:** adversarial.
- **Problem:** The current import ("Replace splits") and legacy backup-restore assign a brand-new object to `state` containing only `start` + `sections`, discarding `id/name/location/date/dist`. Saved through `saveRace()` there's no id to key on, so the active race is orphaned from the index (or written under a bogus key) and its metadata vanishes from the home list. R6 requires import/restore preserved but never says these paths must retain the active race's identity.
- **Why it matters:** Importing splits erases the race's name/metadata and can orphan it.
- **Proposed solution:** Specify that import ("Replace splits") and legacy-format restore replace only `start` + `sections` on the active race, preserving its `id/name/location/date/dist`. Only a full multi-race backup may carry identity, and `saveRace` must assign/require an id rather than key on undefined.

### F9. Follow mode on a splitless race divides by zero
- **Where:** U5 / KTD5 / R3.
- **Source:** design-lens + adversarial (cross-corroborated).
- **Problem:** R3 allows follow on any race, including a freshly created empty one, but the marker math (`elapsedNow / total`) is NaN when `total` is 0. The read-only render also reuses the edit view's empty-bar copy ("No times yet — paste your results or type a split below."), which tells spectators to perform edit actions that don't exist in read-only mode. Risks a console error against the "no uncaught errors" gate.
- **Why it matters:** Broken/mispositioned marker + misleading copy on a reachable path.
- **Proposed solution:** Add a follow-mode empty state to U5: when `total === 0`, suppress the marker and render read-only copy such as "This race has no splits yet" instead of the edit view's paste/type prompt.

### F10. Unnamed race has no identity placeholder
- **Where:** R2 / U4 / R1.
- **Source:** design-lens.
- **Problem:** "New race" drops the user into edit mode with an empty race, and U4 renders the masthead `h1` and location/date from now-blank race data, but the plan never says what an unnamed race shows. Without a placeholder the `h1` renders empty and the home list shows a nameless row — a user with several races can't tell them apart. This is the first thing seen after the primary create action.
- **Why it matters:** Races become indistinguishable in the list and masthead.
- **Proposed solution:** Commit to a single placeholder (e.g. "Untitled race") shown for an empty name in both the editable masthead and the home list row until the user types a name.

### F11. No focus management or announcement on route change
- **Where:** U2 (hash router) / KTD1.
- **Source:** design-lens.
- **Problem:** Client-side routing unmounts and remounts whole views on `hashchange` but doesn't move focus or announce the view change. Keyboard and screen-reader users clicking Open/Follow/back lose their place — focus falls to `body`, they re-tab from the top, with no announcement the view changed. A standard SPA a11y requirement the codebase's existing aria discipline would otherwise meet.
- **Why it matters:** Regresses keyboard/AT navigation the rest of the app respects.
- **Proposed solution:** Specify in U2 that each view mount moves focus to the view's top-level heading and updates the document title (or an aria-live region) so the route change is announced.

---

## P3

### F12. Deferred "duplicate" is scoped to the wrong unit
- **Where:** Scope Boundaries → Deferred to Follow-Up Work.
- **Source:** coherence + scope-guardian (cross-corroborated).
- **Problem:** The note says add race duplicate "if cheap during U4", but duplicate is a home-list action (R4: "from the home view"; the state machine puts it on Home), and the home list is built in U3. U4 is masthead metadata + importer and never touches the list — so the U3 implementer won't add it and the U4 implementer has no list to attach it to.
- **Why it matters:** The "include if cheap" window is misdirected; the option falls through the cracks.
- **Proposed solution:** Change "include if cheap during U4" to "include if cheap during U3".

---

## FYI (advisory — no decision forced)

- **F13. Terminology drift "watch" vs "follow"** (coherence). The read-only view is "follow" everywhere authoritative (route, R8/R9, U5 title), but the Problem Frame calls it "race-day watch mode" and U5's goal calls it "the read-only watch view." Nothing breaks; context disambiguates. *Fix if desired:* replace the minority "watch view/mode" with "follow view/mode".
- **F14. Live marker needs assistive-tech treatment** (design-lens). A 1s-interval marker in an aria-live region would spam screen readers; untreated, AT users get no access to the "where is he now" position that is the view's purpose. *Fix:* make the animated marker `aria-hidden` and expose position via a separate polite aria-live status that updates only on segment change (not every tick). Fold into F6/F9's rework.
- **F15. Edit view has no "watch live" affordance** (design-lens). The state machine names Edit→Follow and U5 gives follow a link back to edit, but nothing routes edit→follow. *Fix:* add a "Watch live" link in the edit masthead → `#/race/:id/follow`, mirroring follow's "open in edit" link.
- **F16. `render()` caches DOM refs at init** (feasibility). `render()` does `$("#start").value = …` directly and writes through the module-level `legsEl = $("#legs")` captured once at IIFE init. Under U2 re-mount a stale `legsEl` points at a detached node (renderLegs paints nothing); in U5 follow mode, if gun time becomes static text with no `input#start`, the `.value` write hits null and throws. KTD4 scopes read-only to the *field builders* and doesn't cover `render()` itself. *Fix:* re-query container refs at the start of each mount, and make `render()` read-only-aware for the gun-time element. Fold into F6/F8's mount-lifecycle rework.

---

## Suggested handling order

1. Fold the four P1 fixes (F1–F4) into the plan — they're correctness bugs that block first-run tests.
2. Apply F5–F11 (P2) — each has a concrete, low-ambiguity fix.
3. F12 (P3) is a one-word edit.
4. FYIs F13–F16: F14 and F16 are worth folding into the F6/F8/F9 mount-lifecycle and follow-view rework rather than tracking separately.
