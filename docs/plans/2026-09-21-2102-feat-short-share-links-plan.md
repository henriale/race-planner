---
title: Short Share Links - Plan
type: feat
date: 2026-09-21
artifact_contract: ce-unified-plan/v1
product_contract_source: ce-brainstorm
execution: code
---

# Short Share Links - Plan

## Goal Capsule

- **Objective:** Pressing Share hands the user a link short enough to paste into a WhatsApp message without it looking like a wall of characters — around 28 characters instead of the present 269 — while the app keeps its identity: one file, no build, no backend, no account.
- **Means:** Shorten the already-working self-contained link through TinyURL's CORS-enabled `api-create.php` at share time, and fall back silently to today's full self-contained link whenever that request cannot be made or cannot be trusted.
- **Authority:** R-IDs win on product behavior. The self-contained link remains the canonical addressing scheme; the short link is a delivery convenience layered on top and never replaces it in storage, routing, or the library.
- **Execution profile:** One file, no build, no dependencies, no test runner. Verified by opening `index.html` in a browser and walking the scenarios, including with the network disabled.
- **Stop conditions:** Stop and ask if TinyURL begins stripping or mangling the URL fragment (verified working 2026-09-21 — the whole payload lives in the fragment, so this is fatal to the approach). Stop and ask if the response cannot be distinguished from an error body reliably. Stop and ask if honouring this requires any change to how a race is stored or routed.
- **Who finishes:** The implementing agent lands the change; the user verifies in-browser, online and offline.

---

## Product Contract

### Summary

The share link today carries the whole race in its fragment: canonical text, deflate-raw compressed, base64url encoded. For the sample race that is a 269-character URL. It is correct, permanent, private and offline-proof — and too long to paste into a message without embarrassment. Measured alternatives that keep the link self-contained (binary re-encoding, a shorter route prefix, brotli) recover at most 16%, because half the payload is the checkpoint names themselves; the floor for a lossless self-contained link is roughly 230 characters.

So the link stops being self-contained at the moment of sharing. Pressing Share sends the full link to TinyURL's public create API and shows the ~28-character result. Everything else about the app is unchanged: races still live in `localStorage`, the router still resolves the full fragment form, and a short link is only ever a redirect to one. When the request fails — offline, service down, a throttle, an unrecognisable response — the full link appears instead, with no error and no extra press.

The cost is real and accepted: the plan text leaves the browser on every share, and a short link outlives the app only as long as TinyURL outlives it. The full link is always still obtainable and always still works.

### Problem Frame

A race plan is shared by messaging it to people who will watch the race — a family group, a support crew. The link is the whole product for those readers. A 269-character string reads as broken or suspicious in a chat window, which costs the share at exactly the moment it matters. Shortening cannot be solved by better compression, because the payload is already compressed and its bulk is irreducible user-authored text.

### Key Decisions

- **TinyURL, not is.gd.** Verified 2026-09-21: `tinyurl.com/api-create.php` returns `access-control-allow-origin` reflecting the caller's Origin, so it is callable from a static page. `is.gd/create.php` sends no CORS header and additionally failed the test request. No other candidate was needed.
- **The fragment survives.** Verified: shortening `https://example.org/#/race/AAAA…/watch` and following the short link returns `301 Location: https://example.org/#/race/AAAA…/watch`. The entire design depends on this.
- **Shortening is one press, behind a flag.** When it is on there is no second press and no toggle: WhatsApp is the target and the common case should be one press. The privacy cost is paid on every share, so the decision to pay it is made once per session by the `shorturl` flag (R43) rather than per share, and is disclosed rather than hidden.
- **The self-contained link stays canonical.** Nothing about routing, storage, `digestText`, or the library changes. A short link is a redirect that lands on the existing route.
- **Failure is silent, not an error.** The fallback is a fully working link, not a degraded one. Telling the user a share "failed" when they are holding a correct link would be false.
- **TinyURL is idempotent.** Verified: the same long URL returns the same slug on repeat calls. Re-sharing an unchanged race creates no new short link and burns no additional quota.

### Requirements

- **R32 — Share yields a short link.** Pressing Share on a race resolves to a TinyURL short link for that race's existing self-contained watch URL, displayed in the share field exactly where the full link is displayed today.
- **R33 — Pending state.** Between the press and the result the share field shows that the link is being prepared. The user is never shown an empty field, a stale link from a previous race, or a frozen control.
- **R34 — Silent fallback to the full link.** If the short link cannot be obtained — offline, network error, timeout, non-2xx, or a response body that is not a well-formed TinyURL URL — the share field shows today's full self-contained link instead, with no error message and no second press required. That link is fully functional.
- **R35 — Bounded wait.** The request is abandoned after a short bounded interval (on the order of three seconds) and treated as a failure under R34. The user never waits indefinitely to share.
- **R36 — Stale results are discarded.** A short link that arrives after the user has navigated away, edited the race, or pressed Share on a different race is discarded and never displayed, following the existing `routeSeq` guard used by Watch Live.
- **R37 — Repeat shares do not re-fetch.** Pressing Share twice for an unchanged race reuses the short link already obtained in this session rather than issuing a second request.
- **R38 — The oversize path is preserved.** The existing behaviour for a race whose full URL exceeds the 2,000-character budget — the `toast.tooBig` warning and the canonical-text clipboard fallback — is unchanged and is decided before any network request is made.
- **R39 — Disclosure.** The app states, in the release notes shown under **What's new**, that sharing sends the race link to TinyURL to shorten it, and that the full link still works if the service is unavailable. The user is not asked to consent per-share.
- **R40 — The README stops claiming a single external request.** The README currently states the Archivo webfont is the only external request. It is corrected to name the share-time TinyURL call and its offline fallback.
- **R41 — Both locales.** Every new user-visible string exists in `en` and `pt`, following the existing `tr()` table.
- **R43 — Behind a query-parameter flag.** Shortening is off unless the page was opened with a `shorturl` query parameter (`index.html?shorturl#/race/<token>`). With the flag off, Share behaves exactly as it did before 0.4.0: the full self-contained link, shown immediately, with no network request. The flag is read once at load, since the router only rewrites the fragment. It is not carried by the shared link — the URL being shortened is rebuilt from origin and pathname — so a reader cannot have the feature switched on for them by opening someone else's link.
- **R42 — Version.** `APP_VERSION`, `RELEASES[0]`, and `CHANGELOG.md` move together to `0.4.0`: the app gains behaviour, and no stored race or existing link stops resolving.

### Success Criteria

- With `?shorturl` set, sharing the sample race (`data/fodaxman-sp5-2025.txt`) produces a link of roughly 28 characters, down from 269. Without it, the full link appears with no request made.
- Opening that short link in a second browser with empty `localStorage` lands on the same watch view the full link lands on.
- With the network disabled, pressing Share still produces a working link, in one press, with no error shown.
- No change in behaviour for stored races, the library, Paste/import, or any existing link ever produced by the app.

### Acceptance Examples

- **Happy path.** A race is open. Press Share → the field briefly shows a pending state → the field shows `https://tinyurl.com/2bpnocmm`, selected and ready to copy. Pasting it into WhatsApp and opening it on a phone shows the race.
- **Offline.** Airplane mode. Press Share → after at most ~3s the field shows `https://race-splits.example/#/race/AaB1…/watch`. No toast, no error. The link works when pasted into another tab.
- **Service error.** TinyURL returns HTTP 200 with the body `Error`. The response is rejected as not a URL and the full link is shown under R34.
- **Stale.** Press Share, then immediately navigate home. The short link arrives and is discarded; the home view is untouched.
- **Oversize race.** A race whose full URL exceeds 2,000 characters. Press Share → the existing `toast.tooBig` path runs, the canonical text goes to the clipboard, and no TinyURL request is made.

### Scope Boundaries

#### Deferred to Follow-Up Work

- Promoting the `shorturl` flag into a real setting — a control in the UI, a remembered per-user choice, or flipping the default to on. The flag is the whole of it for now.
- QR code generation, which the short link now makes practical.
- Binary re-encoding of the token (measured at 236 → 220 characters). It is real but small, and it buys a second serialization format to maintain; the short link makes it moot.
- Trimming the `#/race/…/watch` route prefix. Same reasoning: it only shortens the fallback.

#### Outside this product's identity

- Any backend, account, or database of the app's own.
- Any analytics or tracking attached to shared links.
- Making the short link the canonical address — a race is addressed by its own content, and that does not change.

### Open Questions

- Is the ~28-character result short enough, or does the user also want the fallback link trimmed for the offline case? Deferred above; revisit only if offline sharing turns out to be common.
- TinyURL's unauthenticated rate limit is not documented and was not hit in testing. If a real user ever sees repeated fallbacks online, R34 makes that harmless, but it would be the signal to revisit.
