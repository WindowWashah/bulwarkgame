# Goal: Make Bulwark deeper and more satisfying

You are improving a finished, single-file HTML5 canvas tower-defense game, `index.html`
(deployed at https://bulwark-td.vercel.app). It already works: 10 towers, 4 maps, 4
difficulties, 2 upgrade paths per tower with 2 tiers each, status effects (slow / poison /
burn), bosses, splitters, healers, flyers. Your job is to add **depth and identity** without
breaking what works.

## Primary objectives (in priority order)

1. **Three-tier upgrade trees** — extend every tower's two paths from 2 tiers to **3 tiers**.
   The 3rd tier on each path is a **capstone**: build-defining, expensive, and visibly
   changes how the tower plays (not just "+X% more").
2. **Upgrades read as abilities, not stats** — every tier is a *named ability* with a short
   flavor verb-phrase, and the raw stat shown as a secondary detail. No upgrade should read
   like a spreadsheet cell. (The data already has names — make them evocative and consistent,
   and make the UI lead with the name + effect, stat second.)
3. **Tower synergies / connections** — add two synergy systems so tower *placement* and *mix*
   matter, not just individual upgrades:
   - **Status combos** (cross-tower, automatic): different towers' debuffs combine into bonus
     effects (e.g. a *frozen* enemy hit hard *Shatters* for bonus damage; an enemy that is
     both *burning* and *poisoned* *Detonates* in an AoE).
   - **Adjacency links** (spatial): towers placed next to each other form a visible glowing
     **link** and grant each other a small aura buff, with a few **named pair bonuses** for
     specific type combinations.

## Hard constraints

- Keep it a **single self-contained `index.html`** — vanilla JS, no build step, no new
  dependencies, no external assets (fonts via the existing CDN link are fine).
- **Preserve the existing performance work**: the offscreen `boardLayer` (built in
  `buildBoardLayer`) and the per-color `padCache`/`makePad` tower-base cache. Don't reintroduce
  per-frame `shadowBlur` on static art.
- Stay **data-driven** wherever the existing code is. `buyUpgrade()` and `renderSelection()`
  already handle arbitrary tier counts generically (they read `up.tiers.length`), so adding a
  3rd tier should be **mostly data**, not new control flow.
- Don't break: save/restart flow, difficulty + map selection, the `reset()` lifecycle, mobile
  layout/responsiveness, keyboard shortcuts.
- Keep the visual language (dark fantasy, neon glow, the `PAL` palette per map).

## Architecture you're working with (read before editing)

- `TOWERS` — base stats per tower type (`dmg, range, rate, proj, kind`, plus kind-specific
  fields). `TORDER` is the shop order.
- `UPGRADES[type] = { a:{name, tiers:[ {name, desc, cost, apply(t)} ]}, b:{...} }`.
  `M.dmg/M.rate/M.range` are the stat-mutation helpers used inside `apply`.
- `placeTower(col,row)` builds a live tower object (note `col,row,x,y,aTier,bTier,invested`).
- `buyUpgrade(path)` — generic; guards on `tier >= up.tiers.length`, runs `apply(t)`, bumps
  `aTier`/`bTier`. **Should not need changes** for 3 tiers.
- `renderSelection()` + its inner `btn(path)` render the upgrade panel; it already prints
  `(tier+1)+'/'+u.tiers.length` and a `MAX` state. The stat-summary chips are built from the
  live tower fields (`t.splash`, `t.dot`, `t.crit`, etc.) — extend these as you add effects.
- Enemy status fields on each `e`: `slowTimer/slowMul`, `poisonDps/poisonTimer`,
  `burnDps/burnTimer`. `damage(e,amount,opts)` and `applyStatus(e,p)` are the choke points for
  combo logic.
- `update(dt)` runs the tower firing loop (per-`kind` branches) and the enemy status ticks.
- `drawTowerArt(tw,time)` draws the cached pad + per-type `ART[type]` + level pips
  (`lvl = aTier+bTier`).

---

## Deliverable 1 — Three-tier trees

For **all 10 towers**, give each of the two paths a 3rd `tiers[]` entry. Use this template
(Archer shown fully as the gold standard — match this quality for the other 9):

```js
arrow:{
  a:{name:'Marksman', tiers:[
    {name:'Broadhead',     desc:'+90% damage',                         cost:45,  apply:t=>M.dmg(t,1.9)},
    {name:'Armor-Piercer', desc:'shots punch through armor, +30% dmg', cost:100, apply:t=>{t.pierce=true;M.dmg(t,1.3);}},
    {name:'Deadeye',       desc:'30% chance to crit for x2.5, +range', cost:190, apply:t=>{t.crit=true;t.critChance=0.3;t.critMult=2.5;M.range(t,30);}}]},
  b:{name:'Volley', tiers:[
    {name:'Quickdraw',  desc:'+70% fire rate',          cost:55,  apply:t=>M.rate(t,1.7)},
    {name:'Twin Shot',  desc:'looses at 2 targets',     cost:120, apply:t=>{t.multi=2;M.rate(t,1.2);}},
    {name:'Arrow Storm',desc:'looses at 4 targets, +rate',cost:210,apply:t=>{t.multi=4;M.rate(t,1.4);}}]}},
```

Rules for the capstone (tier 3) on every path:
- It should **change behavior or unlock a mechanic**, not only scale a number. Good capstones:
  add `multi`, add `pierce`, add `crit`, add a new on-hit status, convert a projectile to
  splash, add chains, make a beam fork, etc.
- Cost ≈ 1.6–2.0x the tier-2 cost of that path.
- Capstones on path A vs path B should pull the tower in **different directions** so the two
  paths feel like a real choice (e.g. single-target nuker vs. crowd-clearer).
- Verify each new field your `apply` sets is also (a) initialized in `placeTower`, and (b)
  surfaced in `renderSelection`'s stat chips if it's player-relevant.

After this change `lvl` can reach 6 — **fix the level-pip rendering** in `drawTowerArt` so 6
pips don't overflow the 30px pad (e.g. two rows of 3, or a small "A3·B3" tag).

## Deliverable 2 — Ability-first upgrade copy & UI

- Audit all tier names across all towers for an evocative, consistent fantasy-tech voice
  (e.g. "Deep Freeze", "Fork Lightning", "Napalm", "Arrow Storm"). No generic "Damage 3".
- In `renderSelection`'s `btn(path)`, lead with the **ability name** prominently, the
  flavor/effect line next, and the **stat delta** as a smaller muted detail. Show the path's
  branch name + tier dots (`●●○`) so progression reads at a glance.
- Add a one-line **path identity** subtitle per branch (e.g. Marksman = "precision & pierce",
  Volley = "speed & numbers") so players understand the fork before committing.

## Deliverable 3 — Synergies

### 3a. Status combos (automatic, cross-tower)
Implement in/around `damage()` and `applyStatus()`. Keep to **2–3 crisp, readable combos**,
each with a clear trigger, a clear payoff, and a distinct visual tell (a one-shot ring/spark
in the existing `rings`/`burst` systems, plus a floater label via `addFloater`). Suggested set:

- **Shatter** — when a *slowed/frozen* enemy takes a single hit above a damage threshold
  (heavy hitters: Cannon/Ballista/Arcane crit), deal **+50% bonus** and consume the slow.
- **Detonate** — when an enemy is simultaneously *burning* (Pyre) **and** *poisoned* (Venom),
  trigger a small AoE blast (reuse splash logic) and reset both DoTs. This rewards mixing
  Venom + Pyre.
- **Conduct** — Tesla chains deal **+30%** and arc farther to enemies that are *slowed/wet*
  (frozen). Makes Frost + Tesla a combo.

Each combo must be discoverable: add them to the start-overlay `.notes` "Counters matter" copy.

### 3b. Adjacency links (spatial synergy + the "connecting towers" visual)
- Maintain a derived adjacency graph: two towers are **linked** if orthogonally/diagonally
  adjacent (Chebyshev distance 1 on `col,row`). Recompute only on `placeTower` / `sellSelected`
  (not per frame). Store e.g. `tw.links = [...]`.
- **Visual:** draw a soft glowing line between linked towers (below towers, above the board) —
  this is the literal "connecting" the user asked for. Pulse it subtly.
- **Generic aura:** each linked neighbor grants a small stackable buff, capped (e.g. **+6%
  fire rate per link, max +24%**). Apply as a derived multiplier at fire time so it updates
  live as towers are added/sold (don't bake it into base stats).
- **Named pair bonuses:** a `SYNERGIES` table keyed by sorted type-pair, giving a stronger
  flavored bonus when those two specific types are adjacent. Ship ~4–6, e.g.:
  - `archer+cannon` → **Volley Line**: +20% range to both
  - `frost+venom`  → **Toxic Chill**: poison ticks +25% faster on slowed foes
  - `tesla+prism`  → **Resonance**: beam ramps faster, bolts +1 chain
  - `pyre+cannon`  → **Firestorm Battery**: cannon shells also ignite
  - `mage+ballista`→ **Siege Focus**: +crit chance to both
- Surface active synergies in the selected-tower panel ("Linked: Toxic Chill, +12% rate").

---

## Stretch goals (do if time allows, don't compromise the above)

- **Targeting priority** per tower: First / Last / Strongest / Closest, cycled from the panel.
- **localStorage** persistence for `best` score (currently resets on reload).
- **Next-wave preview**: small icons of what's coming before you hit Start Wave.
- **Endless scaling** past the current curve, with a visible wave-modifier banner.
- A **support/beacon tower** whose whole identity is buffing links (ties the synergy system
  together).

## Balance & feel guidance

- Re-check the difficulty curve after capstones land — tier-3 power may trivialize current
  waves. Tune `DIFFS[*].hpGrow` / `count` or wave composition if so.
- Synergies should be a **bonus for thoughtful play**, not mandatory. A spammed single tower
  should still lose to a synergized mixed defense of equal gold.

## Acceptance criteria (definition of done)

- [ ] Every tower has 2 paths × 3 named tiers; tier 3 is a behavior-changing capstone.
- [ ] Upgrade panel leads with ability name + effect; stat is secondary; tier dots shown.
- [ ] 2–3 status combos fire correctly, each with a visible tell + floater + helptext.
- [ ] Adjacency links render as connecting lines; generic aura + ≥4 named pair bonuses work,
      and update live on build/sell.
- [ ] Level pips (or replacement) render cleanly at max upgrades — no overflow.
- [ ] No regressions: restart, difficulty/map switching, pause, speed, mute, mobile layout.
- [ ] `boardLayer` + `padCache` optimizations intact; no new per-frame `shadowBlur` on statics.
- [ ] JS passes a `node --check` on the extracted `<script>`; manual playthrough to ~wave 12
      on Brutal with no console errors.

## Workflow

1. Read `index.html` end-to-end first; map the integration points above to real line numbers.
2. Land Deliverable 1 (data) → verify → Deliverable 2 (UI) → verify → Deliverable 3a → 3b.
   Commit/redeploy between milestones so each is independently testable.
3. After each milestone: extract the script, `node --check`, then `vercel deploy --prod --yes`.
