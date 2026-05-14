# Village Economy & Defense Overhaul — Design Spec

**Date:** 2026-05-14
**Scope:** Background medieval village simulation in `index.html` (single-file portfolio site)
**Approach:** Big-bang single pass — all systems land together

## Goals

Make the background simulation feel like a coherent economy that builds itself up before any external threat appears:

1. Castles are visually dominant ~4× landmarks, not small icons.
2. Houses come in multiple silhouettes and accumulate decorations as they upgrade.
3. The road grows from raw grass through five increasingly refined stone surfaces; better roads attract more and "better" people.
4. Shops appear, merchants trade with passers-by, occasional arguments break out.
5. Guard posts arise that actually fight dragons with arrow volleys.
6. Dragons are hard-gated: they do not appear at all until the village has reached real economic and defensive maturity, and they are rare even then.

## Architecture Notes

Everything ships inside the existing `index.html` (no new files, no build step). New systems plug into the current pattern of independent spawner `loop(fn, min, max)` calls and the `houses[]` registry. Existing structures and animations are not removed; the dragon spawn loops are wrapped with milestone checks rather than torn out.

Key existing anchors:
- `structureSVG(kind, palette)` — switch-statement that returns SVG markup per structure kind.
- `STRUCTURE_KINDS` array, weighted spawn list in `spawnBuilder()`.
- `houses[]` — central registry of all placed structures with `{el, x, kind, stage, upgrade, palette}`.
- `setRoadLevel(n)` / `tryUpgradeRoad()` — road state machine driven off `world.dataset.road`.
- `loop(fn, min, max)` — generic interval loop used by every spawner.
- `spawnDragon()` / `spawnHugeDragon()` — current dragon entries; will be gated, not replaced.

---

## Section 1 — Castle Redesign

**Dimensions:** new SVG viewBox `200×150` (was `96×72`, ~4× the area). CSS: `.house[data-kind="castle"] { width: 200px; height: 150px; }`. L4 keeps its `1.5×` scale → on-screen ~`300×225`. L4's golden filter + animated glow halo + invincibility-to-normal-dragons preserved.

**Spawn density:** cap at 2 castles on screen at once. If the weighted spawn picker picks `castle` while two already exist, re-roll for a different kind.

**Visual content** (drawn into the new viewBox):

*Stage 1 — `.hs-walls` (revealed after `midDelay`):*
- Wide stone-block foundation with grout lines and a few moss tufts.
- Full-width curtain wall with masonry block hints (two-tone), arrow slits at regular intervals.
- **Four** corner towers (was 2), taller than the wall, each with arrow slits and a small balcony.
- Central gatehouse with portcullis bars, flanked by two short guard blocks.
- Wooden drawbridge angling down across a small moat strip in front of the gate.
- Inner bailey wall visible behind the main one (slight overlap creates depth).

*Stage 2 — `.hs-roof` (revealed after `doneDelay`):*
- Crenellations along curtain wall and every corner tower (~12-16 merlons total).
- Tall central keep with steep tile roof and prominent banner pole.
- Conical roofs on the four corner towers (replacing flat tops) with weathervanes on the two front towers.
- Royal banner (large) on the keep; smaller pennants (4) on corner towers.
- Decorative quoins (corner stone trim) on the keep.

*Lit windows — `.hs-windows-lit`:* ~12 windows total (was 6) — gate windows, two-level windows on each corner tower, keep windows, and a warm glow seeping under the gate.

**Build timing:** keep current `midDelay: 7000`, `doneDelay: 14000`.

---

## Section 2 — House Variety

Add **3 new house silhouettes** alongside the current cottage. All share the existing palette system and the four `.up-1/2/3/4` upgrade decoration groups.

| Variant | Size (px) | Silhouette | Spawn weight |
|---|---|---|---|
| `cottage` (current) | 48×42 | 1-story, gable roof | 50% |
| `two-story` | 44×58 | tall narrow, two window rows, prominent chimney | 25% |
| `a-frame` | 48×52 | steep triangle roof reaching ~⅔ down, attic window | 15% |
| `longhouse` | 64×38 | wide low rectangle, multi-window, hay-loft door | 10% |

**Implementation:**
- A weighted `HOUSE_VARIANTS` list, rolled at spawn time in `spawnBuilder()` when `kind === "house"`.
- The variant is stored on the `houses[]` record (`rec.variant`) and on `dataset.variant` of the `.house` element so CSS can size it per variant.
- `case "house"` in `structureSVG` switches on `palette.variant` (variant passed in via the palette object for compatibility) or accepts a separate `variant` argument.
- Each variant must implement all four `.up-N` decoration groups so upgrades work uniformly.

**Per-upgrade decorations** (all variants):
- **up-1:** chimney with animated wisp of smoke (CSS keyframe drift + fade). Smoke fades to 0 under `[data-weather="rain"]` and `[data-weather="storm"]`.
- **up-2:** flower box under one window (3 small colored squares); short fence segment in front (palette-tinted wood).
- **up-3:** garden plot beside the house (3-4 tiny crops); either a hanging laundry line or an awning over the door (variant-appropriate).
- **up-4:** painted shutters (palette accent color); house number sign; climbing vine up one wall.

Spawn frequency for houses overall does not change in this section; pacing changes live in Section 3.

---

## Section 3 — Road Progression + Spawn-Gating

### 3.1 Six road tiers (add grass as tier 0)

Insert a new tier at the bottom. World starts on raw grass with no road strip visible.

| Tier | Surface | CSS class | Visible road-strip? |
|---|---|---|---|
| 0 | grass (new) | — | No (`.world[data-road="0"] .road-strip { display: none; }`) |
| 1 | dirt | `.t-dirt` | Yes |
| 2 | packed dirt | `.t-packed` | Yes |
| 3 | dressed stone | `.t-stone` | Yes |
| 4 | cobblestone | `.t-cobble` | Yes |
| 5 | flagstone | `.t-flagstone` | Yes |

**CSS migration:** every existing `.world[data-road="N"] .road-tier.t-X` selector shifts up by 1. The starting `data-road="0"` in the HTML already matches (grass), and `setRoadLevel` clamps to `[0, 5]` instead of `[0, 4]`.

**Upgrade thresholds** (replaces `builtCount >= cur*2 + 2`):

| Going to tier | Requires completed structures (`stage >= 2`) |
|---|---|
| 1 (dirt) | 2 |
| 2 (packed) | 5 |
| 3 (stone) | 9 |
| 4 (cobble) | 14 |
| 5 (flagstone) | 20 |

Road-upgrade loop interval stays 35-80s; it simply no-ops until the threshold is met.

**Logs:** add a new entry to `ROAD_LOG` at index 0 for the grass → dirt transition (current log array becomes indices 1-5). The grass tier itself has no transition-in message (it is the starting state).

### 3.2 Spawn-gating by road tier

Wrap the existing `loop(fn, min, max)` helper to accept an **optional** options object:

```
loop(fn, min, max)                                  // unchanged behavior — backwards-compatible
loop(fn, min, max, { minRoad, rateScale })          // new gating behavior
```

Behavior:
- Calls without `opts` behave exactly as today (no gating, scale = 1.0). Every existing call site that we do not explicitly migrate continues to work.
- With `opts`: on every tick, read `world.dataset.road` (current tier `t`).
  - If `t < (opts.minRoad ?? 0)`, skip this firing and reschedule using the original `(min, max)`.
  - Else fire `fn()` and use `(min/scale, max/scale)` for the next interval, where `scale = opts.rateScale?.[t] ?? 1.0`.
- All spawners in the table below are migrated to the new form (those omitted retain the unwrapped call).

Per-spawner configuration:

| Spawner | minRoad | Notes |
|---|---|---|
| `spawnLoneBuilder` | 0 | always |
| `spawnRandomTraveler` (peasants/groups) | 0 | always |
| trees, sheep+shepherd, dogs (rare path) | 0 | always |
| `spawnLumberjack` | 0 | always |
| `spawnDog` (common) | 2 | uncommon below |
| `spawnHorse` (horseman) | 2 | |
| `spawnDuel` | 1 | |
| `spawnBandit` | 1 | |
| `spawnAdventurerCamp` | 1 | |
| `plantCropPatch` | 2 | |
| `spawnWedding` | 2 | |
| `spawnCaravan` | 3 | |
| `spawnCamelCaravan` | 3 | |
| `spawnRoyalCastle` | 4 | |
| `spawnWizardTower` | 4 | |
| `spawnSkyWizard` | 4 | |
| `spawnFlyingCastle` | 4 | |
| `spawnArmy` | 5 | |
| `maybeSolarHalo` | 0 | unaffected |
| `tryUpgradeRoad` | 0 | self-driving |
| `spawnUpgrader`, `checkDevelopment` | 0 | unaffected |

All spawners use the same `rateScale: [1.0, 1.1, 1.3, 1.5, 1.7, 2.0]` table unless individually overridden.

**Result:** at grass tier the realm has only lone peasants, builders, and lumberjacks; "better people" (horsemen, carts, royalty) emerge as the road improves, and the overall spawn rate roughly doubles by tier 5.

---

## Section 4 — Market Stalls + Merchants + Customers

### 4.1 New structure: `marketstall`

- Size 24×34 px. Wooden frame, striped awning (palette-tinted), basket of goods on the counter, hanging sign.
- 2-stage build like other structures: stage 1 = bare frame, stage 2 = awning + goods + sign.
- Added to `STRUCTURE_KINDS`.
- Added to `STRUCTURE_BUILD_LOG` with `mid` = `"A merchant raises a market stall."`, `done` = `"The stall opens for business!"`.

### 4.2 Spawn rules

- Dedicated loop `spawnStallBuilder`, interval 25-50s, `minRoad: 2`.
- Placement: scan existing `houses[]` for the nearest tavern / well / guildhall / marketstall. If any anchor exists, place the new stall within `±150px` of it (random offset). If none, fall back to a random screen-x position.
- Cap: max 8 stalls present at once. Loop no-ops while at cap.
- Uses standard `spawnBuilder` flow with `kind = "marketstall"`.

### 4.3 Merchant sprite

- On stall completion (stage 2), spawn a stationary `.person.merchant` at the stall's x.
- Stored on the stall record: `rec.merchant = personEl`.
- Apron color = stall's awning palette accent.
- Idle: occasional "stirring goods" arm-bob animation (~every 4-6s).
- Lives for the lifetime of the stall.

### 4.4 Customer behaviors

Every spawned walker gets a one-time `maybeStopAtStall()` chance attached. As the walker moves:
- When within ~40px of any stall, roll once.
- Result probabilities:
  - **75% — ignore.** Keep walking.
  - **20% — purchase.** Pause 1.5-2.5s, face merchant, emit 2-3 `.coin` sparkles (reuses existing class), then continue. Log: `"A villager buys goods at the market."` (variants: bread / cloth / cheese / wares).
  - **4% — argue.** Both the customer and merchant shake side-to-side (~0.25s, 4-5 oscillations) via a new `.arguing` class. A red `!` particle appears above each head. Customer storms off (1.4× walk speed for 2s). Log: `"Voices rise at the market — a dispute!"`.
  - **1% — long browse.** 4-5s pause, 4-6 coins, may attract a second customer walking by. Log: `"A long bargain at the stall draws onlookers."`.
- Each walker rolls at most once per stall to avoid loops.

### 4.5 Higher-tier polish

- At road tier ≥ 4, stalls gain a pole banner + a second basket via an additive `.stall-up-1` SVG group.
- At road tier 5, a richer "noble buyer" can stop (purple-trim person, more coins, longer browse) — 10% of stall stops at tier 5 use this variant.
- At night, an `.stall-lantern` glows on completed stalls (lit windows treatment, same `hs-windows-lit` toggle).

---

## Section 5 — Guard Posts + Arrow Volleys + Dragon Arc

### 5.1 New structure: `guardpost`

- Size 24×44 px. Square stone base, crenellated top, helmeted watchman with a bow.
- 2-stage build: stage 1 = stone base, stage 2 = crenellations + watchman + bow.
- At night, a small brazier glows on top (toggle in `hs-windows-lit` group).
- Added to `STRUCTURE_KINDS` and `STRUCTURE_BUILD_LOG` (`mid` = `"A guard post rises beside the road."`, `done` = `"Watchmen take their posts."`).

### 5.2 Spawn rules

- Dedicated loop `spawnGuardpostBuilder`, interval 40-90s, `minRoad: 3`.
- Placement: spread evenly along the road; reject candidate x if within 200px of an existing guard post.
- Cap: max 8 guard posts at once (gives breathing room above the 6-post milestone for huge-dragon gating).
- Uses `spawnBuilder` with `kind = "guardpost"`.

### 5.3 Arrow volleys

New entity class `.arrow`:
- 8×2 px SVG, dark-wood shaft + iron tip, points along travel direction.
- Fired from `guardpost` records during any dragon flyby.

Firing behavior:
- When a dragon is on-screen (`small` or `huge`), each `guardpost` with `stage >= 2` fires one arrow every 0.6-1.0s (random offset per post; staggered so they do not visually clump).
- Arrow flight: positioned at post-top `(x, y_top)`. The arrow **locks onto the dragon's position at the moment of firing** (read once, no in-flight tracking) and animates via CSS transition over 0.4s to that fixed target `(x, y)`. Slight parabolic arc using a midpoint keyframe. Because the dragon keeps moving, this naturally explains the 30% miss rate without extra logic.
- Hit roll: 70% hit, 30% miss. A miss flies past without effect.
- Hit effect: `.dragon` gains a transient `.flinch` class (CSS shake 0.15s); a small "hit" sparkle appears at the impact point; the dragon's `hp` is decremented.

Dragon hp:
- Small dragon: `hp = 4`.
- Huge dragon: `hp = 14`.

Outcomes:
- If `hp > 0` when the dragon's normal flyby duration ends → existing behavior (it may attack a building as before).
- If `hp <= 0` → the dragon retreats immediately: redirect its `transition` toward off-screen with a wobble. Log: `"Arrows drive the dragon back!"` (small) or `"Volleys force the great dragon to retreat!"` (huge).

L4 castle remains an absolute shield (existing behavior preserved). Guard posts make the village survivable below L4 and add visible drama during huge-dragon raids.

### 5.4 Dragon hard-gating (milestones)

Replace the always-firing dragon spawns with milestone gates. The loop wrapper (`minRoad` etc.) is not enough here — both spawners check a milestone function inside the loop body and skip-and-reschedule when not eligible.

| Threat | Required state (all must be true) |
|---|---|
| **Small dragon** | road tier ≥ 3 **AND** ≥ 1 castle (any upgrade) **AND** ≥ 4 marketstalls **AND** ≥ 2 guard posts |
| **Huge dragon** | road tier = 5 **AND** ≥ 1 castle at upgrade ≥ 4 **AND** ≥ 6 marketstalls **AND** ≥ 6 guard posts |

Rebalanced timers (applied once gating opens):
- Small dragon loop: `35-80s` → `120-240s`.
- Huge dragon loop: `240-420s` → `600-1200s`.

**First-unlock emphasis:** the first time each milestone passes, prepend a thematic log line before the dragon spawn:
- Small: `"A dark shadow falls over the realm — dragons return."`
- Huge: `"The horizon trembles. Something vast stirs in the mountains."`

These first-unlock lines fire exactly once per page load (tracked by booleans `firstSmallDragonAnnounced`, `firstHugeDragonAnnounced`).

### 5.5 Resulting arc

The village now follows a clean progression visible to an idle viewer:

`grass → first houses & lumberjacks → road becomes dirt → tavern + well appear → packed dirt → first castle starts → markets cluster around the tavern → stone road → guard posts rise along it → first small dragon raid (arrows fly!) → cobble road, royal procession, wizard tower → flagstone road → L4 castle stands invincible → great dragon raid → defended by guard volleys`

This is the "see the entire economy build before dragons attack" arc.

---

## Testing & Verification

- Visual smoke test in the live browser (the project is the file itself). After all changes land:
  - Reload the page; verify it starts on grass with no road strip.
  - Skip-forward by triggering manual `seedStructure` calls in the console to reach each road tier and confirm visuals + spawn changes.
  - Force a small dragon (call `spawnDragon()` in the console) without milestones met — confirm it is blocked. Then satisfy milestones via seeds and confirm it fires.
  - Force a huge dragon similarly.
  - Confirm arrow volleys hit dragons and drive them back when hp drops to 0.
- Existing animations (mill blades, dragon flyby, wedding, weather) must continue to work unchanged.
- Reduced-motion path (`reduced` branch) remains a no-op for all new spawners (early return in each loop body).

## Out of Scope

- No new build tooling, no module split, no asset files — everything stays inline in `index.html`.
- No persistence across reloads.
- No interactivity (clicks, hover) — this remains a background simulation.
- No additional weather effects, no new time-of-day phases.
- No localization / accessibility changes beyond preserving `aria-hidden="true"` on the world.
