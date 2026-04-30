# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-page meeting cost calculator. Deployed at lesstalk.ing via Vercel (Namecheap-registered domain). Zero dependencies, no build step — one HTML file (`index.html`) with inline CSS and JS. Open it directly or serve the folder as static files.

The FFP-branded variant lives in a sibling repo: [CalamityAdam/ffp-meeting-cost](https://github.com/CalamityAdam/ffp-meeting-cost) (deployed at cost.adumb.dev). The two repos share an architectural lineage but have diverged.

## Running tests

`test.html` is a browser-based test runner. It loads `index.html` in an iframe, drives it through clicks + synthesized `storage` events, and asserts on observable DOM and `localStorage`. Open it in any browser locally, or visit `lesstalk.ing/test.html` after deploy. No build step, no Node required.

If you need to run the suite headlessly during development, `jsdom` works: spin up a static server (`python3 -m http.server`) and load `test.html` via `JSDOM.fromURL` with `runScripts: 'dangerously', resources: 'usable'`. The summary div flips to `class="summary pass"` or `summary fail` when the run finishes.

## Architecture

The interesting part is the persistence + URL-share layer. Read `index.html`'s `<script>` block top-to-bottom; what follows is what's not obvious from a single pass.

**Three rate sources, one state machine.** `currentSource` is one of `'rate' | 'attendees' | 'elon'`. The `'rate'` source uses `currentRatePerHour`; `'attendees'` uses `currentAttendees` (an array of `{ role, count, rate }` rows); `'elon'` is the Konami easter egg. UI rendering and cost computation branch on `currentSource`.

**One `localStorage` key, schema-versioned blob:**
- `meeting-cost:generic-session-v1` — `{ v, source, ratePerHour, attendees, currency, label, accumulatedMs, isRunning, runStartedAtMs, savedAtMs }`. Survives reloads, tab closes, and browser restarts.

**Wall-clock timer anchor.** `startTimestamp` is `Date.now()`, not `performance.now()`, specifically so a running session keeps accruing across reloads: on load, `elapsed = accumulatedMs + (Date.now() - runStartedAtMs)`. RAF still drives in-frame updates; the anchor is wall-clock.

**No periodic snapshots.** Writes only on real transitions (Start/Stop/Restart/source change/currency change/label change). The wall-clock anchor reconstructs everything else.

**Stale-session guard.** A "running" session more than 12h old (`STALE_AFTER_MS`) loads as paused with whatever was already accumulated, so a closed laptop doesn't cost $50k.

**`apply*` vs `persist*` split.** This is the core architectural decision and the reason the code reads the way it does:
- `applyRate(r)`, `applyAttendees(rows)`, `applyCurrency(code)`, `applyLabel(label)`, `applySession(s)` — **pure state mutation only**. No `localStorage`.
- `persistSession()`, `clearPersistedSession()`, `readSession()` — **pure I/O only**. No state mutation.
- `applySession(session)` is idempotent and tolerates partial input (`applySession({})` parks the UI in a clean idle state, used by the cross-tab Restart handler).

User actions compose `apply* + persist*`. The initial load and the `storage`-event handler call only `apply*`, which is what makes cross-tab sync echo-loop-safe: they never write back to storage.

**Cross-tab sync.** A `storage` event listener reacts to changes on `meeting-cost:generic-session-v1` from other tabs. The event doesn't fire in the originating tab, so there's no echo risk.

**`safeLS`.** All `localStorage` I/O sites go through one wrapper that swallows private-mode and quota errors. Use it; don't write raw `try { localStorage.* } catch {}`.

**Konami easter egg deliberately not persisted.** The opt-out lives at the I/O boundary (`persistSession()` early-returns when `currentSource === 'elon'`), not at each call site. Konami should always feel surprising — never restore it from storage.

**URL-shareable state.** `buildShareUrl()` encodes the current rate setup into query params; `applyFromQuery()` reads them on load. URL wins over `localStorage` so a shared link always reflects the sender's setup. Canonical share URL is always `<origin>/?...` — index.html is the default file at root, so `/index.html` is never preserved in the path.

## Workflow

- Default branch is `master`. Don't open PRs unless asked — push directly to `master`.
- After non-trivial changes, run `test.html` (locally via static server + jsdom, or just open in a browser) before committing. Running `/simplify` after a refactor is established practice.
