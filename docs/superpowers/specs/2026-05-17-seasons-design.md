# Seasons — Design

Add four seasons (spring, summer, autumn, winter) to the procedural world background in `index.html`. Purely cosmetic: no gameplay effects, no extra panels, no save state. Visual companion to the existing day-night and weather systems.

## Goals

- A clear sense of seasonal change in the background — palette shifts, autumn foliage, winter snow blanket.
- Auto-cycles in sync with the existing time-speed multiplier.
- Manually advanceable via a new HUD chip, matching the existing UX for time and weather.

## Non-goals

- No effect on crops, animals, fires, or any logic system.
- No persistence — refresh starts in summer (parallel to how the day cycle starts at "day").
- No new mechanic tied to season (no "harvest only in autumn", etc.).

## Architecture

A `season` system parallel to the existing day-night cycle:

- `world.dataset.season` ∈ `spring | summer | autumn | winter`.
- Auto-cycle: 1 season lasts 2 day-night cycles (≈ 4 min at 1×). One full year ≈ 16 min at 1×; scales with `SPEED_OPTIONS` exactly like the day cycle.
- HUD chip `#hud-season` to the left of `#hud-time`. Click advances to the next season (manual override, same pattern as time chip).
- Season change writes a one-line entry to the chronicle (`logEvent`).

## CSS variable plumbing

All season-affected colors become `:root` CSS variables with summer defaults. `.world[data-season="..."]` rules swap the variables and a global `transition` fades them over ~3s.

Variables introduced:

- `--grass`, `--grass-dark` — grass strip gradient.
- `--leaf-light`, `--leaf-mid`, `--leaf-dark`, `--leaf-shadow` — tree canopy palette.
- `--hill-far-fill`, `--hill-near-fill`, `--mountain-fill` — silhouette colors. Hard-coded fills in the SVG paths are removed and replaced with `fill="var(...)"`.
- `--snow-opacity` — drives every snow overlay simultaneously. `0` outside winter, `1` in winter, transitioned.

## Trees

Existing tree SVGs in `makeTreeSVG()` are rewritten once to use the `--leaf-*` and `--bark-*` variables instead of inline hex.

Each `.tree` gets `data-species` (`oak | pine | fruit`) so seasonal CSS rules can target species:

- **Spring/summer/autumn**: all canopies visible, colored from the leaf vars.
- **Winter**:
  - Oak + fruit canopies hidden (`opacity: 0`); bare trunk + skeletal branches visible. A small bare-branch SVG group is added behind the canopy and shown only in winter.
  - Pine canopies stay visible; a white snow-cap rect overlay layered on top fades in via `--snow-opacity`.

Fruit pellets on the fruit tree are hidden in autumn + winter (no out-of-season fruit).

## Snow particles

The existing rain canvas (`tickRain`) gains a snow mode. When `season === "winter"`, drops are drawn as small white circles with a slow vertical fall and sine-wave horizontal wobble. Intensity is capped (~110 flakes on desktop, ~70 on mobile) and runs in addition to current weather (snow on a stormy winter day is fine).

Rain and snow do not coexist: if it's winter, the canvas renders snow regardless of `weather`. The weather veil still applies for cloudy/storm overlay.

## Snow caps

All snow overlays read `var(--snow-opacity)` for their `opacity` and `transition: opacity 3s ease`, so they fade in/out smoothly when the season changes:

- **Grass top**: a `::before` white blanket pseudo-element on `.grass`, soft top edge.
- **Hills**: a second white SVG path layered on top of each existing hill silhouette, tracing the ridge — opacity driven by `--snow-opacity`.
- **Mountains**: same approach — a white peaks path on top of the existing mountain shape.
- **House roofs**: a `::after` overlay on `.house` clipped to a thin band at the top of the silhouette. Single rule, fades in via `--snow-opacity`. Acceptable that the cap is a generic shape rather than per-roof traced — keeps the change minimal.

## Season palettes

Approximate hex values; final tuning happens during implementation:

| Var | Spring | Summer (current) | Autumn | Winter |
|---|---|---|---|---|
| `--grass` | `#4d7a30` | `#3a5a28` | `#7a6228` | `#bfc7cf` |
| `--grass-dark` | `#356018` | `#25401a` | `#5a4818` | `#8a92a0` |
| `--leaf-light` | `#7ab050` | `#4a6a36` | `#e08838` | `#3a5a26` (pine) |
| `--leaf-mid` | `#5a8a30` | `#3a5a26` | `#c86028` | — |
| `--leaf-dark` | `#3a6a20` | `#2a4a1a` | `#8a3818` | — |
| `--hill-far-fill` | `#1c2208` | `#1a1208` | `#2a2010` | `#3a4248` |
| `--hill-near-fill` | `#2c3018` | `#2a2018` | `#3a2818` | `#586068` |
| `--mountain-fill` | `#5e7080` | `#5a6878` | `#6a6a68` | `#8a98a8` |

## HUD chip behavior

`#hud-season` appended to the existing HUD. Displays `SPRING` / `SUMMER` / `AUTUMN` / `WINTER`. Click advances to the next season; the season cycle resumes from the new position.

## Spec self-review

- No TBDs.
- Internal consistency: snow opacity drives all overlays uniformly; tree vs. snow vs. palette behavior consistent across seasons.
- Scope: one implementation plan can cover this — palette + chip + snow particles + overlays.
- Ambiguity: "fruit hidden in autumn" — chosen explicitly, fruit appears only in summer + spring.
