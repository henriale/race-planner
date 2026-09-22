# Changelog

Race Planner ships as one file, so it carries one version number:
`APP_VERSION` in `index.html`. The entries below are the same ones the app
shows under **What's new** in the footer (`RELEASES` in `index.html`) — change
one and change the other, and keep `RELEASES[0].v` equal to `APP_VERSION`.

Versions follow [semantic versioning](https://semver.org): the minor number
moves when the app gains behaviour, the patch number when it only gets fixes,
and the major number when a stored race or an existing link would stop
resolving.

## 0.4.0 — 2026-09-21

- **A short share link, behind a flag.** Open the app with `?shorturl` in the
  address (`index.html?shorturl#/race/…`) and pressing Share gives you a
  ~28-character link instead of the ~270-character one, short enough to paste
  into a message without it reading as broken. Without the flag, Share behaves
  exactly as it did before. The long, self-contained link is still what the
  app addresses a race by; the short one is a redirect to it.
- **Shortening sends the race link to TinyURL.** It happens only with the flag
  on and only when you press Share. The flag is per-session and is not carried
  by the link you share. If TinyURL is unavailable, or you are offline, Share
  falls back to the full self-contained link — no error, no second press, and
  that link works exactly as it always has.

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
