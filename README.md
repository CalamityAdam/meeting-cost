# Meeting Cost-o-Meter

A single-page meeting cost calculator. $40/minute, ticking in real time, with an escalating mood that goes from ☕ to 🔥 as the dollars pile up.

Open `index.html` in a browser, or deploy the folder as a static site (Vercel, Netlify, GitHub Pages — no build step).

## Features

- Pick a team rate or set your own custom hourly rate.
- Timer survives refreshes, tab closes, and browser restarts. Multiple tabs stay in sync via the `storage` event.
- 12h stale-session guard so a forgotten meeting doesn't rack up a fake $50k.

## Stack

Zero dependencies. One HTML file with inline CSS and JS. `requestAnimationFrame` + `Date.now()` for a wall-clock anchor that survives reloads.

## Tests

`test.html` loads `index.html` in an iframe and asserts on observable DOM + `localStorage` behavior — open it in a browser, no build step.
