# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-page meeting cost calculator. Deployed at cost.adumb.dev via Vercel (Cloudflare in front, Squarespace domain). Zero dependencies, no build step — one HTML file (`index.html`) with inline CSS and JS. Open it directly or serve the folder as static files.

## Running tests

`test.html` is a browser-based test runner. It loads `index.html` in an iframe, drives it through clicks + synthesized `storage` events, and asserts on observable DOM and `localStorage`. Open it in any browser locally, or visit `cost.adumb.dev/test.html` after deploy. No build step, no Node required.

If you need to run the suite headlessly during development, `jsdom` works: spin up a static server (`python3 -m http.server`) and load `test.html` via `JSDOM.fromURL` with `runScripts: 'dangerously', resources: 'usable'`. The summary div flips to `class="summary pass"` or `summary fail` when the run finishes.

## Architecture

The interesting part is the persistence layer. Read `index.html`'s `<script>` block top-to-bottom; what follows is what's not obvious from a single pass.

**Two `localStorage` keys, one schema-versioned blob:**
- `meeting-cost:session-v1` — `{ v, teamKey, accumulatedMs, isRunning, runStartedAtMs, savedAtMs }`. Survives reloads, tab closes, and browser restarts.
- `meeting-cost:custom-rate-hourly` — bare numeric string, separate so the working custom-rate flow stays untouched.

**Wall-clock timer anchor.** `startTimestamp` is `Date.now()`, not `performance.now()`, specifically so a running session keeps accruing across reloads: on load, `elapsed = accumulatedMs + (Date.now() - runStartedAtMs)`. RAF still drives in-frame updates; the anchor is wall-clock.

**No periodic snapshots.** Writes only on real transitions (Start/Stop/Restart/team change/custom-rate change). The wall-clock anchor reconstructs everything else.

**Stale-session guard.** A "running" session more than 12h old (`STALE_AFTER_MS`) loads as paused with whatever was already accumulated, so a closed laptop doesn't cost $50k.

**`apply*` vs `persist*` split.** This is the core architectural decision and the reason the code reads the way it does:
- `applyTeam(k)`, `applyCustomRate(r)`, `applyClearCustomRate()` — **pure state mutation only**. No `localStorage`. `applyTeam` returns `true` if state changed so callers can skip needless writes.
- `persistSession()`, `clearPersistedSession()`, `persistCustomRate(r)`, `clearPersistedCustomRate()` — **pure I/O only**. No state mutation.
- `applySession(session)` — the single state-installer used for both initial load and cross-tab reconciliation. Idempotent; tolerates partial input (`applySession({})` produces idle state).

User actions compose `apply* + persist*`. The initial load and the `storage`-event handler call only `apply*`, which is what makes cross-tab sync echo-loop-safe: they never write back to storage.

**Cross-tab sync.** A `storage` event listener reacts to changes on `meeting-cost:session-v1` and `meeting-cost:custom-rate-hourly` from other tabs. The event doesn't fire in the originating tab, so there's no echo risk. Same-tab clicks on the already-active tab early-bail in `applyTeam` and skip `persistSession()` via the boolean return — without that, every click would re-write.

**`safeLS`.** All five `localStorage` I/O sites go through one wrapper that swallows private-mode and quota errors. Use it; don't write raw `try { localStorage.* } catch {}`.

**The Konami CEO easter egg is deliberately not persisted.** The opt-out lives at the I/O boundary (`persistSession()` early-returns when `currentTeamKey === 'ceo'`), not at each call site. If you're adding a new persistable team, this is where the rule lives.

## Workflow

- Working branch is set per session (currently `claude/update-cost-counter-D0YA7`); pushes go to that branch and to `main`.
- Don't open PRs unless asked — the user pushes directly to `main`.
- After non-trivial changes, run `test.html` (locally via static server + jsdom, or just open in a browser) before committing. Running `/simplify` after a refactor is established practice.
