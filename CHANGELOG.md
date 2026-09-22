# Changelog

Race Planner ships as one file, so it carries one version number:
`APP_VERSION` in `index.html`. The entries below are the same ones the app
shows under **What's new** in the footer (`RELEASES` in `index.html`) — change
one and change the other, and keep `RELEASES[0].v` equal to `APP_VERSION`.

Versions follow [semantic versioning](https://semver.org): the minor number
moves when the app gains behaviour, the patch number when it only gets fixes,
and the major number when a stored race or an existing link would stop
resolving.

## 0.4.6 — 2026-09-21

- **Map pins are shareable.** A checkpoint's map link previously lived only
  in the browser that set it — share links and watch sessions dropped it.
  It now rides in the same canonical race text as everything else, so it
  survives a share link, a watch session, and copy/paste of a plan.
- The canonical text format gains an optional `map` field between `split`
  and `name`, recognized only when it looks like a URL, so old links and
  pasted plans without one still parse exactly as before.

## 0.4.5 — 2026-09-21

- **Km column covers swim too.** Previously run and bike only; now every
  distance-bearing leg (swim, bike, run) shows its running total.
- **Km's unit is inline now.** Same style as Pace and Split Dist.: value
  plus a muted unit beside it, instead of a plain concatenated string.

## 0.4.4 — 2026-09-21

- **Split columns reordered and relabeled.** New order: Split Dist., Split
  Time, Pace, Elapsed, Clock, Km — the cumulative distance column moves to
  the end and is now labeled "Km" (was "Dist. so far"); the per-checkpoint
  distance column is now "Split Dist." (was the bare unit, e.g. "km"), and
  "Split" is now "Split Time".
- **Inline unit on split distance.** The split distance field now shows its
  unit next to the value (e.g. "1.20 km"), styled the same way Pace shows
  "km/h".

## 0.4.3 — 2026-09-21

- **Cumulative distance column.** Run and bike sections show a read-only
  running total of distance covered so far, next to each checkpoint's own
  distance.

## 0.4.2 — 2026-09-21

- **Link previews.** Sharing a Race Planner link in WhatsApp, iMessage, Slack,
  etc. now shows a title, description, and a static branded image instead of
  a bare URL, via `og:title`/`og:description`/`og:image` and Twitter Card
  meta tags in `index.html`.
- The preview image (`assets/og-image.png`) is fixed — it does not reflect
  the sender's actual plan or timeline, since that would need a server to
  render per-link. Same static image for every share.

## 0.4.1 — 2026-09-21

- **Short links removed.** Share goes back to the self-contained link that
  carries the race in its own URL. It is long, but it needs no service to
  resolve — it works today, offline, and in ten years, which the short link
  could not promise.
- **Why it went.** TinyURL's `api-create.php` is a deprecated endpoint, and
  links created through it show an 8-second interstitial ad in a real browser
  (an HTTP-level check misses this — curl sees a clean 301). Beyond that one
  vendor, any pointer link trades permanence for characters: the payload moves
  onto someone else's server, and the plan is unrecoverable from the link once
  that server is gone.
- **Nothing leaves the browser when you share** again. The Archivo webfont is
  once more the only external request the app makes.

## 0.4.0 — 2026-09-21

- **Short share links via TinyURL,** behind an off-by-default `?shorturl`
  flag. Removed in 0.4.1; see above.

## 0.3.0 — 2026-09-21

- **A revised timeline.** In watch mode the bar is drawn against the revised
  plan rather than the frozen one: every checkpoint already passed is as wide
  as it really took, and the rest of the day keeps its planned splits stacked
  on top of the last time logged, so the bar's right edge and the revised
  finish are the same instant.
- **Measured segments.** One that stands for a recorded time carries that
  split as its label, an underline, and the plan it replaced in its tooltip.
- **Pins.** The actual pin sits at the athlete's own elapsed time on that
  measured axis; the plan pin is mapped onto it, so it still points at the
  checkpoint the frozen plan says is due right now.

## 0.2.0 — 2026-09-21

- **Watch view.** One-tap arrivals, a running elapsed counter, a pinned course
  strip, and a clock-first finish.
- **Library.** Cards lead with Watch Live, keep Edit beside it, and tuck
  rename / duplicate / delete behind the cog.
- **Content-addressed links.** A link carries the plan itself and files it in
  the reader's library on arrival; opening the same link twice matches by
  digest and files nothing new.
- **Templates and paste.** Start from a known course, or paste results in the
  canonical race-text format.
- **Português (Brasil).** Alongside English, switchable from the flags at the
  top right. Stored and shared text stays canonical, so a link made in one
  language opens identically in the other.
- **Editing.** Pace is an input wherever the arithmetic is defined; the sport
  hues are reserved and anything live wears amber.
- **This footer.** The running version, and these notes, on every view.

## 0.1.0 — 2026-09-18

- Several races in one library, with create, rename, duplicate and delete.
- Edit view: change a split and watch the rest of the day move, with the
  finish held or shifted by the column you type in.
- Read-only follow view with a live position marker.
- Races stored in this browser, with a copyable text backup.
- Non-destructive migration of the original single-race tool
  (FODAXMAN XTRI Solo Point Five 2025).
