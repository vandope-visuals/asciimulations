```
                _ _                 _      _   _
  __ _ ___  ___(_|_)_ __ ___  _   _| | __ _| |_(_) ___  _ __  ___
 / _` / __|/ __| | | '_ ` _ \| | | | |/ _` | __| |/ _ \| '_ \/ __|
| (_| \__ \ (__| | | | | | | | |_| | | (_| | |_| | (_) | | | \__ \
 \__,_|___/\___|_|_|_| |_| |_|\__,_|_|\__,_|\__|_|\___/|_| |_|___/
```

there's a quiet kind of wonder in watching math draw itself: a fractal diving forever, a flock turning as one, a strange attractor sketching a shape no hand designed. ASCIImulations collects 100 of these living systems, every one a tunable character grid, all running off a single shared engine in one HTML file. cellular automata, fractals, flocking, 3D solids, physics, noise fields, demoscene effects. pick a tile and watch it run.

built by [vandope](https://instagram.com/vandope__), visual engineer and creative technologist, Manila, Philippines. same hands as [m-or.ph](https://m-or.ph), same 1-bit aesthetic.

---

## table of contents

- [quick start](#quick-start)
- [what it does](#what-it-does)
- [the stack](#the-stack)
- [architecture](#architecture)
  - [the character grid](#the-character-grid)
  - [the engine](#the-engine)
  - [the sketch contract](#the-sketch-contract)
  - [charsets & shading](#charsets--shading)
  - [color & glow](#color--glow)
- [the simulations](#the-simulations)
- [controls](#controls)
- [keyboard shortcuts](#keyboard-shortcuts)
- [themes & display](#themes--display)
- [performance](#performance)
- [browser support](#browser-support)
- [project status](#project-status)

---

## quick start

open it live at **[asciimulations.pages.dev](http://asciimulations.pages.dev/)**, or run it yourself:

```
1. open index.html in Chrome (or any modern browser)
2. that's it
```

no build step. no dependencies to install. no server required (`file://` works fine). the only network request is Google Fonts for the three pixel typefaces (Silkscreen, VT323, Departure Mono).

the landing page features 100 live thumbnails. each tile is a real running miniature of its simulation, not a screenshot. click one to open it full-size with its controls.

## what it does

- **100 simulations** across 11 categories — cellular automata, fractals, flocking, 3D-in-ASCII, physics, noise & fields, geometry, signal, demoscene, flow & particles, animation
- **a live thumbnail gallery** — every tile animates; hover for the name, click to enter
- **per-sim tuning** — each simulation exposes its own labelled sliders (seed, speed, spin, detail, cohesion, etc.) plus a universal **zoom** that means something different per sim (cell coarseness, field scale, dive rate, solid size…)
- **three charsets** — `ascii` (12-level ramp), `dots` (round glyphs), `blocks` (solid shades); swap live without restarting the sim
- **four themes** — green phosphor, white, amber, and flat paper (black-on-white, no glow)
- **fullscreen mode** — pure simulation, all chrome stripped away
- **a live HUD** per sim — gen counts, particle counts, fps, and the keys that do something
- **prev / next / random** navigation to surf the whole set
- **pause and reseed** any running sim from the keyboard
- **feeds straight into [m-or.ph](https://m-or.ph)** — open a sim in fullscreen, then add it as a **screen capture** source in m-or.ph; now any simulation here becomes a live VJ input you can crossfade, glitch, and run through the whole effects chain

## the stack

| layer | choice | notes |
|---|---|---|
| language | vanilla JavaScript (ES2020+) | zero frameworks, zero build tooling, zero npm |
| rendering | **DOM text**, not canvas | each frame is a `<pre>` of CSS-grid rows; glyphs are real characters you could select and copy |
| architecture | **single HTML file** | CSS + markup + all 100 sims + engine in one artifact |
| styling | hand-rolled CSS, 1-bit Mac System 7 chrome | SVG data-URI dither/stripe patterns, two theme colors (`--ink` / `--paper`), `shape-rendering: crispEdges`, pixel fonts |
| fonts | 3 pixel/terminal faces via Google Fonts | Silkscreen (chrome), VT323 + Departure Mono (the grid) |
| state | one `Engine` instance per active sim + a plain `state` object the sketch owns | no store library; the render loop is the source of truth |
| persistence | none | no localStorage, no cookies, no analytics, no accounts |

the single-file constraint is deliberate, same as m-or.ph: the whole thing is one artifact you can email, host anywhere, or open from disk.

## architecture

the system splits cleanly in two: a **renderer** that knows nothing about any particular simulation, and **100 sketches** that know nothing about how they're drawn to the screen. they meet at a small contract.

### the character grid

everything renders onto a fixed **100×44 grid of cells**. the `Grid` class holds three flat arrays (the character at each cell, its color, and its alpha) and exposes a single `set(x, y, char, color)`. sketches never touch the DOM; they only `set` cells.

a frame is emitted as one `<pre>` containing 44 `<span class="gr">` rows, each a CSS grid of fixed-width columns. this keeps every glyph in a perfect monospace cell regardless of the actual font metrics: a round dot and a `@` occupy exactly the same box, so nothing ever shifts.

### the engine

one `Engine` wraps one sketch and runs a fixed-timestep loop on `requestAnimationFrame`:

```
accumulate elapsed time
while acc >= step:
    sketch.update(engine)     // advance simulation state
    acc -= step
sketch.draw(engine)           // paint the current state onto the grid
emit <pre> innerHTML
```

key properties:

- **update and draw are separate.** simulation logic advances in fixed steps (so a sim at 14fps stays correct regardless of display refresh); drawing happens once per animation frame.
- **fps is per-sim and live.** a sketch can rewrite `e.fps` / `e.step` from a slider mid-run (the speed knob does exactly this).
- **`prefers-reduced-motion` is respected** — the loop caps to a single update step per frame when the OS requests reduced motion.
- the engine carries the grid, current `cols`/`rows`, the `frame` counter, the active `charMode`, and the sketch's `params` (the slider values, plus the universal `zoom`).

### the sketch contract

every one of the 100 simulations is a plain object with the same shape. nothing more is required:

| field | role |
|---|---|
| `name`, `cat` | display name and category |
| `fps` | default step rate |
| `sliders` | array of `{id, label, min, max, step, def}` — auto-rendered into the control panel |
| `setup(e)` | (re)initialize `e.state`; called on open, on reseed, and when zoom changes the grid |
| `update(e)` | advance the simulation one tick |
| `draw(e)` | paint `e.state` onto `e.grid` via `set` |
| `hud(e, fps)` | the status line under the sim |
| `icon` | a tiny multi-line ASCII string used as the panel icon |

because the contract is this small, adding the 101st simulation is just writing one object. the gallery, controls, navigation, theming, and fullscreen all come for free.

### charsets & shading

three ramps, switchable live:

| mode | glyphs | levels |
|---|---|---|
| `ascii` | `` .,-~:;=!*#$@`` | 12, already smooth |
| `dots` | space → `·∘•●⬤` | 7 |
| `blocks` | space → `░▒▓█` | 4 |

a sketch asks for a brightness `t` in 0–1 and `shadeChar(ramp, t)` returns the right glyph. because `dots` and `blocks` have few levels, the renderer **sub-shades** them: it fades each glyph's opacity toward the next step using the fractional part of `t`, so a 4-glyph block ramp still reads as a smooth gradient. `ascii` is dense enough to skip this and runs at full opacity.

### color & glow

within a frame, cells are tinted on a three-stop scale: `COL3(t)` returns `--dim` / `--ink` / `--hud` for low / mid / high brightness. every theme defines those three plus a `--glow`, and the grid carries a `text-shadow` glow whose strength scales with brightness, so bright cells bloom like real phosphor. the `paper` theme zeroes the glow out entirely for a flat 1-bit print look.

## the simulations

100 sims, grouped by what makes them tick:

| category | count | examples |
|---|---|---|
| **noise & fields** | 23 | Plasma, Perlin Flow, Worley Noise, Curl Noise, Metaballs, Aurora, Lava Lamp |
| **3d in ascii** | 19 | Donut, Wire Cube, Globe, Torus Knot, Tunnel, Hypercube, Icosahedron, Sphere Lattice |
| **flow & particles** | 13 | Flow Field, Matrix Rain, Fireworks, Fountain, Snowfall, Smoke, Fireflies |
| **physics** | 12 | Double Pendulum, Bouncing Balls, Wave Field, Gravity Wells, Plinko, Cloth |
| **fractals** | 11 | Mandelbrot, Julia Set, Burning Ship, Newton, Sierpinski, Koch, Apollonian |
| **cellular automata** | 11 | Game of Life, Rule 30, Brian's Brain, Wireworld, Langton's Ant, Falling Sand, HighLife |
| **geometry** | 9 | Spirograph, Phyllotaxis, Maurer Rose, Superformula, Truchet, Mandala |
| **signal** | 5 | Spectrum, Oscilloscope, Waveform, Radial EQ, Sonar |
| **demoscene** | 4 | Rotozoomer, Twister, Copper Bars, Hyperspace |
| **flocking** | 3 | Boids, Particle Swarm, Slime Mould |
| **animation** | 1 | Pipes |

the strange-attractor and chaos sims (Lorenz, Clifford, De Jong, Hopalong, Double Pendulum) genuinely never repeat. the CA sims (Life, Rule 30, Wireworld, Langton's Ant) are exact implementations, not approximations.

## controls

each sim's panel auto-builds from its `sliders` array plus two universal controls:

- **characters** — `ascii` / `dots` / `blocks`, live
- **two tuning knobs** — whatever that sim declared (seed/speed, spin/girth, detail/dive, count/cohere…)
- **zoom** — universal, but context-dependent per sim: cell coarseness for CA, field scale for flow, dive rate for fractals, overall size for 3D solids. changing it re-runs `setup` where the grid dimensions matter.

a live HUD line sits under the stage with per-sim telemetry (generation, particle count, fps) and the active keys.

## keyboard shortcuts

| key | action |
|---|---|
| `space` | pause / resume the current sim |
| `r` | reseed / restart (re-runs `setup`) |
| `Esc` | exit fullscreen |

(the grid must have focus, so click into the stage first.)

## themes & display

- **four themes** — `green` (default phosphor), `white`, `amber`, `paper`. the chrome (dither, stripes, borders, fonts) is identical across all four; only `--ink` / `--paper` / `--glow` swap. `paper` is the only one with no glow, for a flat black-on-white look.
- **1-bit Mac System 7 chrome** — striped title bar, dithered surfaces via SVG data-URI masks tinted to the theme ink, hard 1px borders, blocky invert-on-hover buttons.
- **fullscreen** — strips all chrome and runs the bare simulation edge to edge; `Esc` exits.
- the layout reflows on small screens — control strip collapses to a 2×2 grid, theme toggles pin to a footer.

## performance

- **DOM text rendering** keeps the whole thing simple and dependency-free, and it's why glyphs are real selectable characters. the trade is that the grid is fixed at 100×44 — dense enough to read as an image, light enough that 4,400 cells repaint cleanly each frame.
- **fixed timestep** decouples simulation correctness from display rate. heavy sims declare a low `fps` and stay accurate.
- **nothing persists between frames** unless a sketch explicitly keeps buffers in its own `e.state` (the CA grids, particle arrays, and history rings do).
- **reduced-motion aware** — the loop throttles itself when the OS asks.

## browser support

- **Chrome / Chromium-based browsers recommended**, but any modern browser runs it — there are no exotic APIs here, just DOM, CSS grid, and `requestAnimationFrame`.
- no permissions, no popups, no network beyond the font CDN.
- `prefers-reduced-motion` is honored automatically.

## project status

current build: **100 simulations**, single file, one shared engine. the sketch contract is small enough that the set grows by writing one object at a time, so expect the count to climb.

---

*ASCIImulations built by vandope. also made [m-or.ph](https://m-or.ph). follow him on [instagram.com/vandope__](https://instagram.com/vandope__)*
