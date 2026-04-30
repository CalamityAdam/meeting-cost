# lesstalk.ing — your meeting is burning money

A single-page meeting cost calculator. Set a rate, build the bill by attendee, or just watch the dollars climb. The page reacts as the cost piles up.

Live at **[lesstalk.ing](https://lesstalk.ing)**.

Open `index.html` in a browser, or deploy the folder as a static site (Vercel, Netlify, GitHub Pages — no build step).

## Features

- Set your hourly rate, or build it by attendee (count × rate per role).
- Currency switch (USD/EUR/GBP/JPY/etc.) — symbol-only, no FX conversion.
- Shareable link encodes your rate setup so anyone can join the same meter.
- Timer survives refreshes, tab closes, and browser restarts. Multiple tabs stay in sync via the `storage` event.
- 12h stale-session guard so a forgotten meeting doesn't rack up a fake $50k.

## Stack

Zero dependencies. One HTML file with inline CSS and JS. `requestAnimationFrame` + `Date.now()` for a wall-clock anchor that survives reloads.

## Tests

`test.html` loads `index.html` in an iframe and asserts on observable DOM + `localStorage` behavior — open it in a browser, no build step.
