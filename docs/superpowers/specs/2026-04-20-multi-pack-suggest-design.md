# Multi-Pack + Improved Suggest — Design Spec

**Date:** 2026-04-20
**File:** `cointainer_calculator.html`
**Source request:** `improve_item.txt`

## 1. Goals

1. Replace the binary "secondary package on/off" with an extensible **1–3 package** model accessed via a "**+ Add package**" button.
2. Improve **Suggest Size** so it returns realistic dimensions (no more 1cm × 5cm oil barrels) and adds a **partial-auto** sub-mode where the user locks some dimensions and auto-fills the rest.
3. Make all output modes (Max Capacity, Custom Qty, Suggest Size) work with the multi-pack model.
4. Let the user choose how the container is split between packs (axis **L / W / H**) and what fraction each pack gets.
5. Add a **5-calculation soft limit** for demo users to nudge toward licensing.

## 2. Out of Scope (YAGNI)

- True 3D bin-packing (FFD / shelf algorithms).
- Hybrid "vertical fill leftover within a length-split zone."
- Suggesting dims for multiple packs simultaneously.
- Persisting pack configs across sessions.
- More than 3 packs.
- Changes to the login / license / common-license system.
- Changes to the license key algorithm or Firebase schema.

## 3. Data Model

Replace `pkg2Enabled` flag + scattered `p2_*` DOM inputs with a single array:

```
packs = [
  {
    id: 1,                      // 1-indexed, stable
    shape: 'cylinder'|'square'|'rectangle'|'sphere'|'wedge'|'bag',
    dims: { d, h, s, l, w, sd }, // only fields relevant to shape are read
    gap: number,                 // cm
    weight: number,              // kg per unit
    allocPct: number,            // 0–100, must sum to 100 across packs
    color: '#3fb950'|'#4ecca3'|'#f0883e'  // assigned by id
  },
  ...
]
```

Invariants:
- `packs.length` ∈ [1, 3].
- `packs[i].id === i + 1` (array index + 1).
- Sum of `allocPct` across packs = 100 (UI enforces; calc rejects otherwise).
- Pack 1 always exists and cannot be removed.

Global split axis state:

```
splitAxis: 'L' | 'W' | 'H' | 'auto'
```

`'auto'` is only valid in Max Capacity mode; other modes default to `'L'` and expose a radio.

## 4. UI Changes

### 4.1 Packs panel (replaces current "Secondary package" block)

```
┌─ Packages ──────────────────────────────────────┐
│  Split axis: (•) L   ( ) W   ( ) H   [auto*]    │
│                                                 │
│  ┌─ Pack 1 [green]  alloc: [60 %] ──────────┐  │
│  │ Shape: [Cylinder ▾]                      │  │
│  │ Dia: [__] cm   Height: [__] cm            │  │
│  │ Gap: [__]  Weight: [__] kg                │  │
│  │ [Suggest dims for this pack]              │  │
│  └───────────────────────────────────────────┘  │
│                                                 │
│  ┌─ Pack 2 [teal]  alloc: [30 %]  [× Remove] ┐  │
│  │ …                                         │  │
│  └───────────────────────────────────────────┘  │
│                                                 │
│  [+ Add package]   Sum: 100 %  ✓                │
└─────────────────────────────────────────────────┘
```

- "+ Add package" is disabled when `packs.length === 3`.
- Remove button (`×`) appears on Pack 2 and Pack 3 only. Clicking removes that pack; the remaining packs' ids are re-numbered 1..N.
- On add: new pack inherits the previous pack's shape as default; alloc% is auto-rebalanced to equal split (e.g., adding 3rd pack splits 100/3 across all three). User can then adjust sliders.
- Sum badge turns red when sum ≠ 100.
- `*auto` option on split axis is visible only in Max Capacity mode.

### 4.2 Suggest Size panel

```
┌─ Suggest Size ──────────────────────────────────┐
│  Mode: (•) Auto all   ( ) Auto partial          │
│                                                 │
│  [Auto all]                                     │
│    Target count (optional): [__]                │
│    [Advanced ▸]  (min dim overrides)            │
│                                                 │
│  [Auto partial]                                 │
│    Shape: [Cylinder ▾]                          │
│    Dia:    [__] cm  🔒 locked                   │
│    Height: [__] cm  🔓 auto                     │
│    [Advanced ▸]                                 │
│                                                 │
│  [Run Suggest]                                  │
└─────────────────────────────────────────────────┘
```

- The Suggest button inside a **pack card** (Section 4.1) is equivalent to opening this panel with that pack's context. It uses the pack's allocated zone as the container for the search.
- **Advanced** is a collapsible section exposing per-shape min-dim overrides (see §5.3).

## 5. Suggest Size — Algorithm

### 5.1 Auto all

Same as current `runSuggestSize()`, with three changes:

1. **Lower bounds per shape** enforced (see §5.3). The iteration starts at the preset min, not at 1 cm.
2. **Aspect-ratio guard**: reject any candidate where `max(dims) / min(dims) > 10` (protects against 3 cm × 30 cm × 300 cm results that pass volume but are unrealistic).
3. **Search step** for rect/wedge/bag stays at 5 cm (current behavior), but starting bound is now the preset min.

### 5.2 Auto partial

Same iteration loop, but dimensions marked **🔒 locked** are fixed at the user's entered value (not iterated). Only unlocked dims are searched. Ranking (volume utilization, optional target match) is unchanged. If all dims are locked, the suggester returns a single row with the user's values.

### 5.3 Shape-preset minimum dimensions (defaults)

| Shape | Min dim | Rationale |
|---|---|---|
| Cylinder | d ≥ 20, h ≥ 20 | Drums, cans, barrels |
| Square | s ≥ 15, h ≥ 15 | Cardboard boxes |
| Rectangle | l, w, h ≥ 15 | Cardboard boxes |
| Sphere | d ≥ 10 | Balls, fruits |
| Wedge | l, w, h ≥ 15 | Pallet-style wedges |
| Bag | l, w, h ≥ 30 | Sacks, rice bags |

Exposed under **Advanced** as editable number inputs; defaults listed above. Stored in a JS const; user overrides live only in-memory (no persistence).

### 5.4 Multi-pack Suggest

When Suggest is triggered for pack `k` and `packs.length > 1`:

- Build the **reduced container**: the zone allocated to pack `k` given the current split axis and allocPct. Example: L-axis, 40ft container (1219 cm L), pack k = 30% → reduced container is `L=366, W=231, H=239`.
- Run the normal Suggest algorithm against the reduced container.
- Before running, validate: all OTHER packs have non-blank dims. Otherwise show: *"Fill dimensions for the other packs first."*
- If the reduced container has zero volume in any dimension (alloc = 0), result list is empty with message *"No space remaining — increase this pack's allocation."*

## 6. Multi-Pack Calculation

### 6.1 Container zoning

Given `splitAxis` and `packs[].allocPct`, the container (`c.l, c.w, c.h`) is divided along the chosen axis. For axis L with packs [60%, 30%, 10%]:

```
pack1 zone: l = c.l * 0.60, w = c.w, h = c.h, offset = 0
pack2 zone: l = c.l * 0.30, w = c.w, h = c.h, offset = c.l * 0.60
pack3 zone: l = c.l * 0.10, w = c.w, h = c.h, offset = c.l * 0.90
```

Analogous for axis W (split along width) and H (split along height, matches current behavior when splitting 2 packs).

### 6.2 Per-pack calc

For each pack, run the existing `calcGrid()`-style math against its zone dims. Accumulate total count and total weight.

### 6.3 Auto axis (Max Capacity mode only)

When `splitAxis === 'auto'`:
- Try `L`, `W`, `H` in turn.
- For each, compute total packed count `Σ packs[i].count`.
- Pick the axis with the highest total. Tie-break: prefer `L`, then `W`, then `H`.
- Report in status: *"Best split axis: L (tried L=120, W=108, H=96)"*

### 6.4 Max Capacity split behavior

In Max Capacity mode with multiple packs, `allocPct` still governs the split. The "Max" refers to filling each pack's zone to capacity given its dims. There is no solver that redistributes alloc%; user controls that.

### 6.5 Weight check

Sum total weight across all packs; compare to container payload limit. Warn if exceeded. Same UX as today.

## 7. 3D Rendering

- Render each pack in its own zone using its assigned color.
- Pack 1 = `#3fb950` (green, current primary).
- Pack 2 = `#4ecca3` (teal, current secondary).
- Pack 3 = `#f0883e` (amber, reuses existing Y-axis arrow color).
- Legend lists all active packs with count + color swatch + shape name.
- Axis arrows, rotation, and dragging are unchanged.
- Overloaded cells (beyond user's Custom Qty) still drawn red as today.

## 8. Status Readout

Format (one line per pack + total):

```
Mode: 🎯 Max Capacity  |  Split: L (auto: best of 3)
Pack 1 (cylinder): 240 units · Grid 6×4×10 · Zone 731×231×239 cm
Pack 2 (square):    90 units · Grid 5×3×6  · Zone 366×231×239 cm
Pack 3 (bag):       18 units · Grid 3×2×3  · Zone 122×231×239 cm
Total: 348 packages  ·  Vol util: 87%  ·  Weight: 8,430 kg  OK
```

## 9. Demo Mode

Demo restrictions remain as today, adapted to the new model:

- If `isDemo === true`: disable "+ Add package" button and hide Remove buttons. Only Pack 1 is usable. This is the new-model equivalent of today's "Mixed disabled."
- Continue to disable Suggest Size, HC (high-cube) containers, Compare, and Print as today.
- 3D package cap of 200 still applies.
- Watermark unchanged.
- Max Capacity and Custom Qty remain available in demo (single pack only).

## 9a. Demo Calculation Limit

Demo users get **5 successful Calculate runs per browser**. After the 5th, the Calculate button is disabled and a banner prompts for a license.

- **Who:** `isDemo === true` only. Licensed users have no counter.
- **Counter:** `localStorage` key `cpc_demo_calcs` (integer, default 0).
- **Increment:** inside `calculate()`, only after the run completes successfully (no input errors, no alloc-sum failure). Invalid runs do not count.
- **Gate:** at the top of `calculate()`, if `isDemo && cpc_demo_calcs >= 5`, return early and show banner.
- **Banner** (new element `#demoLimitBanner`, hidden by default):
  > "Demo limit reached (5/5 calculations). Enter a license key to continue — free for beta testers: hovanphapbk@gmail.com"
  With a "Enter License" button that opens the existing lock screen.
- **Reset:** when a valid license is activated via `tryUnlock()`, clear `cpc_demo_calcs`. No other reset path — this is intentional (browser-level nudge, not session-level).
- **Counter display:** below the Calculate button in demo mode, show `"Demo: 3 / 5 calculations used"`. Updates live.
- **Bypass awareness:** the user can clear localStorage to reset. This is acceptable for a sales nudge; the intent is friction, not enforcement.

## 10. Migration

No user data persists (packs config is in-memory only), so no migration concerns. On first load:

- `packs[]` initializes with a single Pack 1 (alloc 100%).
- `splitAxis` initializes to `'L'`.
- All current Pack-1 DOM inputs (`p_d`, `p_l`, `p_w`, `p_s`, `p_ch`, `p_gap`, `p_weight`, etc.) are kept as the Pack 1 form inside the new Packs panel. Existing element IDs stay where possible to minimize diff.
- `p2_*` inputs are removed.
- `togglePkg2()`, `setType2()`, `getPkgDims2()`, `pkg2Enabled`, `pkg2Body`, `pkg2Badge` are removed.

## 11. Components / Functions to Add or Change

**New:**
- `addPack()` — append a new pack card, re-balance allocPct, disable "+" at 3.
- `removePack(id)` — remove a pack, re-number, re-balance allocPct.
- `getPackDims(id)` — read DOM → dims object for pack `id`.
- `computePackZones(c, axis, packs)` — return array of `{l, w, h, offset}` zones.
- `tryAllAxes(c, packs)` — for Max Cap + auto-axis, return best axis + totals.
- `suggestForPack(packId)` — run Suggest against pack's allocated zone.
- `applyShapeMins(shape)` — return default min-dim object.
- `passesAspectGuard(dims)` — return true if `max/min ≤ 10`.

**Modified:**
- `calculate()` — loop over `packs[]` instead of pack1 + optional zone2.
- `draw3D()` — loop over zones, draw each in its color with its offset.
- `renderResult()` — emit one status line per pack + total.
- `runSuggestSize()` — accept optional `reducedContainer` and `lockedDims`; apply min-dim and aspect-ratio guard.
- `setMode()` — show/hide the Auto-partial sub-panel; show/hide the `auto` axis option.
- `applyDemoRestrictions()` — disable "+ Add" in demo.

**Removed:**
- `togglePkg2()`, `setType2()`, `getPkgDims2()`, `pkg2Enabled`.

## 12. Testing / Acceptance Criteria

Manual browser tests (no automated test suite in this project):

1. **Single pack** (Pack 1 only, alloc 100%) — all three modes (Max Cap / Custom / Suggest) produce the same result as the current single-pack behavior (regression check).
2. **Two packs, L-split 60/40** — 3D shows green zone front 60% of length, teal zone rear 40%. Status lists both packs + total.
3. **Three packs, W-split 40/40/20** — 3D shows three side-by-side zones across width.
4. **H-split, 2 packs 50/50** — teal stacks on top of green, matches old "secondary on" layout.
5. **Auto axis (Max Cap only)** — status reports which axis won and the totals tried.
6. **Alloc ≠ 100** — sum badge red, calculate button shows warning, no render.
7. **Suggest, single pack, cylinder** — min diameter ≥ 20 cm, no 1 cm results.
8. **Suggest, Auto partial, box with W locked at 50 cm** — all results have W = 50.
9. **Suggest for Pack 2 while Pack 1 is filled** — result fits in pack 2's allocated zone, not the whole container.
10. **Demo mode** — "+ Add package" disabled, Suggest / Max Cap / Print disabled as today.
11. **Remove Pack 2 (of 3)** — Pack 3 renumbers to Pack 2; alloc% rebalances; sum stays 100.
12. **Weight overflow** — warning shown when total weight > container payload.
13. **Demo calc limit** — run Calculate 5 times in demo; 6th click shows banner and does nothing. Counter displays "5 / 5." Activating a license clears counter.
14. **Demo invalid runs** — in demo, clicking Calculate with alloc ≠ 100 does NOT increment counter.

## 13. Deployment

Same as today:
```
cp cointainer_calculator.html index.html
git add index.html cointainer_calculator.html docs/
git commit -m "Multi-pack support + improved Suggest Size"
git push
```

GitHub Pages at `a17255.github.io/container-calculator/` picks it up automatically.

## 14. Risks / Open Questions

- **3D perspective at 3 side-by-side zones** — when one zone is very thin (e.g., 10% of L), individual packages may render as thin slivers. Mitigation: the aspect-ratio guard in Suggest keeps dims reasonable; for Custom-qty, user accepts whatever dims they enter.
- **Mobile layout** — three pack cards stacked vertically plus the allocation controls may push the 3D tab's Setup panel long. Mitigation: collapse inactive pack cards to a header row (shape + count badge) by default on mobile (< 768 px).
- **Backward-compat bookmarks** — none; app has no URL state.
