# Bulwark — Tower Defense

A self-contained HTML5 canvas tower-defense game. **The entire game is one file: `index.html`.**
No build step, no dependencies, no server. To run it: just open `index.html` in any browser
(double-click it). To edit it: open `index.html` in any text editor and change the code — what
you save is exactly what runs. The deployed site at https://bulwark-td.vercel.app serves this
same file verbatim (nothing is compiled or minified).

The only external thing it loads is the Google Fonts stylesheet (`<link>` in the `<head>`); the
game still works offline, it just falls back to system fonts.

## Files in this folder

| File | What it is |
|------|------------|
| `index.html` | **The game.** Edit this. ~1080 lines: CSS + HTML + JS, all inline. |
| `index.html.bak` | Backup of the version *before* the synergy/3-tier update, in case you want to diff or revert. |
| `GOAL_improve-bulwark.md` | The design brief used to drive the big feature update (3-tier trees, combos, synergies). |
| `_integration_spec.md` | The exact, step-by-step spec that was applied to add those features. Useful as a map of where each system lives. |
| `README.md` | This file. |
| `.vercel/` | Vercel project link (so `vercel deploy` knows which project). Safe to ignore. |

## How `index.html` is structured

It's three parts in one file:

1. **`<style>` … `</style>`  (lines ~10–119)** — all CSS. Layout, the HUD, the board frame, the
   start/game-over overlays, the tower shop, the upgrade panel, the synergy chips, and the
   responsive breakpoints at the bottom.
2. **`<body>` (lines ~121–171)** — the static DOM: the header/HUD, the `<canvas id="game">`, the
   two overlays (start screen with difficulty + map pickers, and the game-over screen), the tower
   shop container, the action buttons, and the upgrade `<div id="panel">`. Most of this is
   populated at runtime by JS, so you rarely edit the body directly.
3. **`<script>` … `</script>`  (lines ~172–1079)** — the whole game, wrapped in one IIFE
   `(function(){ … })()`. This is where you'll spend 95% of your editing time.

### Map of the `<script>` (current line numbers)

**Tuning data (edit these to balance the game — they're plain objects):**
- `DIFFS` (line ~180) — the 4 difficulties: starting gold/lives, HP growth per wave, enemy speed
  scaling, spawn count multiplier, kill-reward multiplier, boss frequency.
- `MAPS` (line ~189) — the 4 maps: color palette + the waypoint path the enemies walk.
- `TOWERS` (line ~215) — base stats for all 10 towers (damage, range, fire rate, projectile
  speed, the special `kind`, and the one-line shop description).
- `TORDER` (line ~227) — the order towers appear in the shop (and their `1`–`0` hotkeys).
- `UPGRADES` (line ~230) — **the upgrade trees.** Each tower has two paths (`a`/`b`), each with 3
  tiers. Every tier is `{name, desc, cost, apply:function(t){…}}` where `apply` mutates the live
  tower. Helpers `M.dmg(t,f)`, `M.rate(t,f)`, `M.range(t,n)` are defined just above.
- `PATH_IDENTITY` (line ~323) — the one-line "identity" subtitle shown under each upgrade path.
- `ENEMIES` (line ~346) — all enemy types: HP, speed, reward, radius, and flags like `armor`,
  `flying`, `heal`, `split`, `boss`, `regen`.

**Combat & systems logic:**
- `SHATTER_MIN` / `DETO_*` / `CONDUCT_*` (line ~421) — tunable constants for the three status
  combos.
- `damage()` (~424) — applies damage + the **Shatter** combo. `comboBlast()` (~438) is the
  Detonate explosion.
- `applyStatus()` (~456) — applies slow/poison/burn + triggers the **Detonate** combo.
- `acquire()` (~471) — target selection (also where the Volley-Line range synergy is read).
- `fireProjectile()` / `impact()` (~477) — projectile spawn + on-hit resolution (crits here).
- `teslaFire()` (~508) — Tesla chain logic + the **Conduct** combo.
- `SYNERGIES` table (line ~544) — the 5 named adjacency pairs. `rebuildLinks()` (~553) computes
  which towers are linked (runs on every build/sell); `linkRate()` is the fire-rate aura.
- `update(dt)` (line ~575) — **the simulation tick.** Spawns, enemy movement, status ticks,
  the per-tower firing loop (one branch per tower `kind`: beam / pulse / chain / default),
  projectile movement, cleanup, leak/lives.

**Rendering (HiDPI canvas):**
- `buildBoardLayer()` (~664) + `makePad()` (~682) — **performance caches.** The static board
  (background, grid, path) and each tower's glowing base are pre-rendered once instead of every
  frame. *Don't add per-frame `shadowBlur` to static art* — that's the latency fix from earlier.
- `render(time)` (~692) — draws one frame (board → keep → links → towers → enemies → effects).
- `ART` (line ~744) — the per-tower vector art, one draw function per tower type.
- `drawLinks()`, `drawTowerArt()`, `drawEnemies()`, etc. — the individual draw helpers.

**UI & interaction:**
- `updateHUD()`, `buildShop()` — sync the DOM HUD/shop with game state.
- `renderSelection()` (line ~929) — builds the upgrade panel for the selected tower (the `btn()`
  inside it renders each upgrade card; the synergy "Linked" chips are built here too).
- `placeTower()` (~997), `buyUpgrade()`, `sellSelected()` — the build/upgrade/sell actions.
- event listeners (canvas pointer + keyboard + buttons), then the `audio`/`snd` Web-Audio
  sound effects, then the main `loop()` and the bootstrap line at the very bottom.

## Common edits — where to go

| I want to… | Edit |
|------------|------|
| Make the game easier/harder | `DIFFS` (line ~180) — e.g. lower `hpGrow`, raise starting `gold` |
| Change a tower's base power/cost | `TOWERS` (line ~215) |
| Add/rebalance an upgrade | `UPGRADES` (line ~230) — change a tier's `cost` or `apply` |
| Add a whole new tower | add to `TOWERS`, `TORDER`, `UPGRADES`, `PATH_IDENTITY`, and an `ART` draw fn |
| Tweak an enemy | `ENEMIES` (line ~346) and/or `buildWave()` for when it appears |
| Add a new map | push to `MAPS` (line ~189) with a palette + waypoint path |
| Add/adjust a synergy | `SYNERGIES` (line ~544) |
| Retune the combos | the `SHATTER_MIN` / `DETO_*` / `CONDUCT_*` constants (line ~421) |
| Restyle the UI | the `<style>` block (lines ~10–119) |

After editing, sanity-check the JS still parses:

```sh
# extracts the <script> and runs Node's syntax checker
node -e "const fs=require('fs');const m=fs.readFileSync('index.html','utf8').match(/<script>([\s\S]*?)<\/script>/);fs.writeFileSync('_c.js',m[1])" && node --check _c.js && echo OK && rm _c.js
```

## Deploying

The project is linked to the Vercel project `bulwark-td`. From this folder:

```sh
vercel deploy --prod --yes      # or: npx vercel deploy --prod --yes
```

That pushes the current `index.html` live to https://bulwark-td.vercel.app.
