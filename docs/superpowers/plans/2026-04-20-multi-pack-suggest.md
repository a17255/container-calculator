# Multi-Pack + Improved Suggest Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the on/off secondary package with a 1–3 pack model (via "+"), add split-axis zoning (L/W/H/auto), overhaul Suggest Size (shape-preset mins, aspect guard, auto-partial with locked dims), and add a 5-calculation soft limit for demo users.

**Architecture:** Single-file HTML app. Introduce a `packs[]` array as the single source of truth for package config (replacing `pkg2Enabled` + scattered `p2_*` inputs). All modes (`calculate`, `draw3D`, `renderResult`, `runSuggestSize`) read from this array. Zone splitting done on one chosen axis per calculation.

**Tech Stack:** Vanilla JS + HTML + Canvas 2D (no build tools, no test framework). Manual browser testing only.

**Reference spec:** `docs/superpowers/specs/2026-04-20-multi-pack-suggest-design.md`

**Testing convention:** Because there is no test harness, each task ends with a **manual browser check** step — open `cointainer_calculator.html` in a browser, perform the listed action, confirm the listed observable result.

---

## File Structure

Only one code file is modified: `cointainer_calculator.html` (the single-file app).

Line ranges referenced below use the pre-change numbering (file has 1892 lines at start).

| Region | Pre-change lines | What lives there |
|---|---|---|
| CSS | ~10–380 | styles; we add `.pack-card`, `.pack-hdr`, `.alloc-slider`, `.demo-banner`, `.lock-toggle` |
| HTML — packs panel | 425–470 | current `pkg2-hdr` block + p2_* inputs → replaced with `#packsPanel` |
| HTML — suggest panel | 495–510 | mode_suggest button + sug_target → add mode radio + Advanced block + lock toggles |
| JS — state | ~1100–1160 | add `packs[]`, `splitAxis`, demo counter init |
| JS — mode/UI | 1150–1270 | `setMode`, `togglePkg2` (delete), new `addPack`/`removePack`/`setSplitAxis` |
| JS — calculate | 1218–1340 | loop over `packs[]`; zone math; demo-limit gate |
| JS — draw3D | 1574–1650 | multi-zone render with offsets + colors |
| JS — renderResult | 1290–1340 | per-pack status lines + total |
| JS — Suggest | 1781–1900 | shape-preset mins + aspect guard + auto-partial + per-pack context |
| HTML — index copy | n/a | copied from `cointainer_calculator.html` at deploy |

---

## Task Decomposition

Order is chosen so the app keeps running after every task. Early tasks refactor behind a feature-compat shim so existing behavior works; later tasks add the new UI.

| # | Task | Ships working app? |
|---|---|---|
| 1 | Introduce `packs[]` state, keep legacy DOM | Yes |
| 2 | Extract `getPackDims(id)` and reroute calculate to array | Yes |
| 3 | Zone computation helper `computePackZones` | Yes |
| 4 | Refactor `calculate()` to iterate zones | Yes |
| 5 | Refactor `draw3D()` to iterate zones | Yes |
| 6 | Refactor `renderResult()` to per-pack lines | Yes |
| 7 | Build new Packs panel UI (HTML + CSS) | Yes |
| 8 | `addPack` / `removePack` / alloc rebalancing | Yes |
| 9 | Split-axis control + auto-axis solver | Yes |
| 10 | Remove legacy pkg2 code | Yes |
| 11 | Suggest: shape-preset mins + aspect guard | Yes |
| 12 | Suggest: Auto-partial mode + lock toggles | Yes |
| 13 | Suggest: multi-pack per-pack button | Yes |
| 14 | Demo 5-calc limit + banner | Yes |
| 15 | Demo multi-pack restriction | Yes |
| 16 | Self-review pass + deploy (copy to index.html, commit, push) | Yes |

---

## Task 1: Introduce `packs[]` state, parallel to legacy DOM

**Goal:** Create the new array-based state model while keeping all current p_* and p2_* inputs working. The array is built from current DOM on each calculate.

**Files:**
- Modify: `cointainer_calculator.html` (JS state region, ~line 1100)

- [ ] **Step 1: Add `packs` and `splitAxis` module-level vars**

Locate the main `<script>` tag area where `pkg2Enabled`, `outputMode`, and similar state live. Add:

```javascript
let packs = [
  { id: 1, shape: 'cylinder', dims: {}, gap: 0, weight: 0, allocPct: 100, color: '#3fb950' }
];
let splitAxis = 'L'; // 'L' | 'W' | 'H' | 'auto'
const PACK_COLORS = ['#3fb950', '#4ecca3', '#f0883e'];
```

- [ ] **Step 2: Write a "sync DOM → packs[]" helper**

Add function `syncPacksFromDOM()` that reads current p_* inputs into `packs[0]`, and if `pkg2Enabled === true`, pushes a `packs[1]` built from p2_* inputs. For now alloc is 50/50 if 2 packs, 100 if 1:

```javascript
function syncPacksFromDOM(){
  const shape1 = document.querySelector('.chip[data-type].active')?.dataset.type || 'cylinder';
  const gap1 = +(document.getElementById('p_gap')?.value || 0);
  const w1 = +(document.getElementById('p_weight')?.value || 0);
  const d1 = readDimsForShape(shape1, '');  // '' prefix for primary inputs
  packs = [{ id:1, shape:shape1, dims:d1, gap:gap1, weight:w1, allocPct:100, color:PACK_COLORS[0] }];
  if(pkg2Enabled){
    const shape2 = currentType2 || 'cylinder';
    const gap2 = +(document.getElementById('p2_gap')?.value || 0);
    const w2 = +(document.getElementById('p2_weight')?.value || 0);
    const d2 = readDimsForShape(shape2, '2');
    packs[0].allocPct = 50;
    packs.push({ id:2, shape:shape2, dims:d2, gap:gap2, weight:w2, allocPct:50, color:PACK_COLORS[1] });
  }
}

function readDimsForShape(shape, suffix){
  const g = (id) => +(document.getElementById('p'+suffix+'_'+id)?.value || 0);
  switch(shape){
    case 'cylinder': return { d:g('d'), h:g('ch') };
    case 'square':   return { s:g('s'), h:g('sh') };
    case 'rectangle':return { l:g('l'), w:g('w'), h:g('rh') };
    case 'sphere':   return { d:g('sd') };
    case 'wedge':    return { l:g('l'), w:g('w'), h:g('rh') };
    case 'bag':      return { l:g('l'), w:g('w'), h:g('rh') };
  }
  return {};
}
```

- [ ] **Step 3: Call `syncPacksFromDOM()` at the top of `calculate()`**

Add `syncPacksFromDOM();` as the first line inside `calculate()` (around line 1218).

- [ ] **Step 4: Manual browser check**

Open the file in a browser. Open DevTools console. Set a cylinder package (d=50, h=80). Click Calculate. In console, type `packs`. Expect:
```
[{id:1, shape:'cylinder', dims:{d:50, h:80}, gap:..., weight:..., allocPct:100, color:'#3fb950'}]
```
Turn on Secondary Package (pkg2Enabled=true), set any pack2 dims, click Calculate. Expect `packs.length === 2` and both have `allocPct:50`.

- [ ] **Step 5: Commit**

```bash
git add cointainer_calculator.html
git commit -m "Introduce packs[] state + DOM sync helper (parallel to legacy)"
```

---

## Task 2: Extract `getPackDims(id)` and use it in calculate

**Goal:** Give the existing calculate a uniform way to read pack dimensions regardless of which pack index.

**Files:**
- Modify: `cointainer_calculator.html`

- [ ] **Step 1: Add `getPackDims(packIndex)` returning the same shape as current `getPkgDims()`**

Place near existing `getPkgDims`:

```javascript
function getPackDims(idx){
  const p = packs[idx];
  if(!p) return null;
  // Return object in the same shape as current getPkgDims() output
  return packDimsToPkgObj(p);
}

function packDimsToPkgObj(p){
  // Mirror the object shape the current calculate() expects from getPkgDims
  // Fields: type, footL, footW, stackH, vol, gap, weight
  const gap = p.gap || 0;
  let footL, footW, stackH, vol;
  switch(p.shape){
    case 'cylinder': footL = footW = p.dims.d; stackH = p.dims.h;
                     vol = Math.PI*Math.pow(p.dims.d/2,2)*p.dims.h; break;
    case 'square':   footL = footW = p.dims.s; stackH = p.dims.h;
                     vol = p.dims.s*p.dims.s*p.dims.h; break;
    case 'rectangle':footL = p.dims.l; footW = p.dims.w; stackH = p.dims.h;
                     vol = p.dims.l*p.dims.w*p.dims.h; break;
    case 'sphere':   footL = footW = p.dims.d; stackH = p.dims.d;
                     vol = (4/3)*Math.PI*Math.pow(p.dims.d/2,3); break;
    case 'wedge':    footL = p.dims.l; footW = p.dims.w; stackH = p.dims.h;
                     vol = 0.5*p.dims.l*p.dims.w*p.dims.h; break;
    case 'bag':      footL = p.dims.l; footW = p.dims.w; stackH = p.dims.h;
                     vol = p.dims.l*p.dims.w*p.dims.h; break; // bag vol uses raw l*w*h now
  }
  return { type:p.shape, footL, footW, stackH, vol, gap, weight:p.weight, color:p.color };
}
```

- [ ] **Step 2: Manual browser check**

Console: after a calculate, type `getPackDims(0)`. Expect object `{type:'cylinder', footL:50, footW:50, stackH:80, vol:~157080, gap, weight, color:'#3fb950'}`.

- [ ] **Step 3: Commit**

```bash
git add cointainer_calculator.html
git commit -m "Add getPackDims(idx) helper for packs[] reads"
```

---

## Task 3: Zone computation helper

**Goal:** Compute per-pack container zones given axis + allocPct.

**Files:**
- Modify: `cointainer_calculator.html`

- [ ] **Step 1: Add `computePackZones(c, axis, packs)`**

```javascript
function computePackZones(c, axis, packsArr){
  // Returns array aligned with packsArr: [{l,w,h, offL, offW, offH}, ...]
  const zones = [];
  let cursor = 0;
  for(const p of packsArr){
    const frac = (p.allocPct || 0) / 100;
    let z = { l:c.l, w:c.w, h:c.h, offL:0, offW:0, offH:0 };
    if(axis === 'L'){ z.l = c.l * frac; z.offL = cursor; cursor += z.l; }
    else if(axis === 'W'){ z.w = c.w * frac; z.offW = cursor; cursor += z.w; }
    else if(axis === 'H'){ z.h = c.h * frac; z.offH = cursor; cursor += z.h; }
    zones.push(z);
  }
  return zones;
}
```

- [ ] **Step 2: Manual browser check**

Console:
```
computePackZones({l:1219,w:231,h:239}, 'L', [{allocPct:60},{allocPct:40}])
```
Expect `[{l:731.4, w:231, h:239, offL:0}, {l:487.6, w:231, h:239, offL:731.4}]`.

- [ ] **Step 3: Commit**

```bash
git add cointainer_calculator.html
git commit -m "Add computePackZones helper"
```

---

## Task 4: Refactor `calculate()` to iterate zones

**Goal:** Replace the `pkg2Enabled` branch with a generic loop that works for 1..3 packs.

**Files:**
- Modify: `cointainer_calculator.html` — `calculate()` body (~lines 1218–1340)

- [ ] **Step 1: Replace zone-2 branch with a zones loop**

Inside `calculate()`, after `syncPacksFromDOM()` and the container `c` is built, replace the block around the current lines 1232–1242 with:

```javascript
// Validate alloc sum
const allocSum = packs.reduce((a,p)=>a + (p.allocPct||0), 0);
if(Math.abs(allocSum - 100) > 0.5){
  showStatus('Pack allocation must sum to 100% (current: ' + allocSum.toFixed(0) + '%).', 'error');
  return;
}

const zones = computePackZones(c, splitAxis === 'auto' ? 'L' : splitAxis, packs);
const zoneResults = [];
let totalCount = 0, totalVol = 0, totalWeight = 0;
for(let i=0; i<packs.length; i++){
  const p = getPackDims(i);
  const z = zones[i];
  const grid = calcGrid(z, p); // reuse existing calcGrid(container,pkg) → {perL,perW,perH,count}
  zoneResults.push({ pack: packs[i], pkgObj: p, zone: z, grid });
  totalCount += grid.count;
  totalVol += grid.count * p.vol;
  totalWeight += grid.count * (p.weight || 0);
}
```

- [ ] **Step 2: Delete the old zone2 handling**

Remove the block that reads `zone2` and the separate pack2 calc. All downstream references must use `zoneResults` now.

- [ ] **Step 3: Pipe `zoneResults` to `renderResult` and `draw3D`**

Change the calls at the end of `calculate()`:

```javascript
renderResult({ mode: outputMode, container: c, zoneResults, totalCount, totalVol, totalWeight });
draw3D(c, zoneResults);
```

- [ ] **Step 4: Manual browser check**

Single pack mode: set cylinder (d=50,h=80) in 20ft container. Click Calculate. 3D view and status should look the same as before (regression).

Enable secondary pack (existing toggle, we'll remove later): set pack2 box (15×15×15). Click Calculate. Expect both packs rendered in 3D with 50/50 L-axis split (new behavior — different from current H-stack).

- [ ] **Step 5: Commit**

```bash
git add cointainer_calculator.html
git commit -m "Refactor calculate() to iterate over zones from packs[]"
```

---

## Task 5: Refactor `draw3D()` to iterate zones

**Goal:** Render each zone's packs at their offset with the pack's color.

**Files:**
- Modify: `cointainer_calculator.html` — `draw3D()` body (~line 1574)

- [ ] **Step 1: Change the function signature and loop**

```javascript
function draw3D(c, zoneResults){
  clearCanvas();
  drawContainerWireframe(c);
  for(const zr of zoneResults){
    const { grid, pkgObj, zone } = zr;
    const color = pkgObj.color;
    for(let iz=0; iz<grid.perH; iz++){
      for(let iy=0; iy<grid.perW; iy++){
        for(let ix=0; ix<grid.perL; ix++){
          const px = zone.offL + ix * (pkgObj.footL + pkgObj.gap);
          const py = zone.offW + iy * (pkgObj.footW + pkgObj.gap);
          const pz = zone.offH + iz * (pkgObj.stackH + pkgObj.gap);
          drawPkg3d(pkgObj.type, px, py, pz, pkgObj, color);
        }
      }
    }
  }
  drawAxisArrows(c);
  drawLegend(zoneResults);
}
```

Note: `drawPkg3d` is the generic wrapper around existing `drawCyl3d`, `drawSq3d`, etc. If it doesn't exist as a single switch, add it:

```javascript
function drawPkg3d(type, x, y, z, p, color){
  switch(type){
    case 'cylinder': return drawCyl3d(x,y,z, p.footL, p.stackH, color);
    case 'square':   return drawSq3d(x,y,z, p.footL, p.stackH, color);
    case 'rectangle':return drawRect3d(x,y,z, p.footL, p.footW, p.stackH, color);
    case 'sphere':   return drawSphere3d(x,y,z, p.footL, color);
    case 'wedge':    return drawWedge3d(x,y,z, p.footL, p.footW, p.stackH, color);
    case 'bag':      return drawBag3d(x,y,z, p.footL, p.footW, p.stackH, color);
  }
}
```

- [ ] **Step 2: Update `drawLegend`**

```javascript
function drawLegend(zoneResults){
  const el = document.getElementById('legend');
  el.innerHTML = zoneResults.map((zr,i) => {
    const p = zr.pack;
    return `<span class="legend-chip" style="background:${p.color}"></span>`
      + ` Pack ${p.id} (${p.shape}): ${zr.grid.count}`;
  }).join('&nbsp;&nbsp;');
}
```

- [ ] **Step 3: Manual browser check**

Two-pack L-split 60/40: pack1 front 60% green, pack2 rear 40% teal. Rotate view with drag — both zones rotate coherently. Legend shows both colors with correct counts.

- [ ] **Step 4: Commit**

```bash
git add cointainer_calculator.html
git commit -m "Refactor draw3D to render per-zone with per-pack colors"
```

---

## Task 6: Refactor `renderResult()` to per-pack lines

**Goal:** Emit one status line per pack plus a total line.

**Files:**
- Modify: `cointainer_calculator.html` — `renderResult()` (~line 1290)

- [ ] **Step 1: Rewrite the body**

```javascript
function renderResult(r){
  const rb = document.getElementById('rbar');
  if(!rb) return;
  const { mode, container, zoneResults, totalCount, totalVol, totalWeight } = r;
  const containerVol = container.l * container.w * container.h;
  const utilPct = containerVol > 0 ? (totalVol / containerVol * 100) : 0;
  const payloadKg = container.payloadKg || 0;
  const weightOK = payloadKg <= 0 || totalWeight <= payloadKg;

  let lines = [];
  lines.push(`Mode: ${modeLabel(mode)}  |  Split: ${splitAxis === 'auto' ? 'auto' : splitAxis}`);
  for(const zr of zoneResults){
    const p = zr.pack, g = zr.grid, z = zr.zone;
    lines.push(
      `Pack ${p.id} (${p.shape}): ${g.count} units · `
      + `Grid ${g.perL}×${g.perW}×${g.perH} · `
      + `Zone ${Math.round(z.l)}×${Math.round(z.w)}×${Math.round(z.h)} cm`
    );
  }
  lines.push(
    `Total: ${totalCount} packages  ·  Vol util: ${utilPct.toFixed(1)}%  ·  `
    + `Weight: ${totalWeight.toLocaleString()} kg ${weightOK ? 'OK' : '⚠ over payload'}`
  );
  rb.innerHTML = lines.map(l => `<div>${l}</div>`).join('');
}

function modeLabel(m){
  return m === 'auto' ? '🎯 Max Capacity'
       : m === 'custom' ? '📦 Custom Qty'
       : m === 'suggest' ? '🔍 Suggest Size' : m;
}
```

- [ ] **Step 2: Manual browser check**

Set 40ft container + two packs 60/40 L-split. Status bar shows:
- `Mode: 🎯 Max Capacity  |  Split: L`
- `Pack 1 (cylinder): N units · Grid a×b×c · Zone 731×231×239 cm`
- `Pack 2 (square): M units · Grid d×e×f · Zone 488×231×239 cm`
- `Total: (N+M) packages · Vol util: P% · Weight: W kg OK`

- [ ] **Step 3: Commit**

```bash
git add cointainer_calculator.html
git commit -m "Refactor renderResult to per-pack lines + total"
```

---

## Task 7: Build new Packs panel UI (HTML + CSS)

**Goal:** Replace the `pkg2-hdr` block with a proper Packs panel that renders pack cards dynamically from `packs[]`.

**Files:**
- Modify: `cointainer_calculator.html` (HTML + CSS regions)

- [ ] **Step 1: Add CSS for pack cards**

In the `<style>` block, append:

```css
.packs-panel { border:1px solid #30363d; border-radius:6px; padding:12px; margin-top:12px; }
.packs-hdr { display:flex; justify-content:space-between; align-items:center; margin-bottom:8px; font-weight:600; }
.split-axis { display:flex; gap:8px; align-items:center; margin-bottom:10px; font-size:13px; }
.split-axis label { display:flex; align-items:center; gap:4px; cursor:pointer; }
.pack-card { border-left:4px solid var(--pack-color); border:1px solid #30363d; border-radius:4px; padding:10px; margin-bottom:10px; }
.pack-card-hdr { display:flex; justify-content:space-between; align-items:center; font-weight:600; margin-bottom:6px; }
.pack-color-swatch { display:inline-block; width:12px; height:12px; border-radius:2px; margin-right:6px; vertical-align:middle; }
.alloc-row { display:flex; align-items:center; gap:8px; margin:6px 0; }
.alloc-row input[type=range] { flex:1; }
.btn-remove { background:#da3633; color:#fff; border:none; border-radius:4px; padding:2px 8px; cursor:pointer; font-size:12px; }
.btn-add-pack { background:#238636; color:#fff; border:none; border-radius:4px; padding:6px 12px; cursor:pointer; width:100%; margin-top:8px; }
.btn-add-pack:disabled { background:#484f58; cursor:not-allowed; }
.alloc-sum { font-size:13px; font-weight:600; }
.alloc-sum.bad { color:#f85149; }
.alloc-sum.ok { color:#3fb950; }
```

- [ ] **Step 2: Replace the HTML pkg2 block (lines 425–470)**

Delete the old `pkg2-hdr` + `pkg2Body` block. Insert:

```html
<div class="packs-panel">
  <div class="packs-hdr">
    <span>Packages</span>
    <span class="alloc-sum ok" id="allocSum">Sum: 100% ✓</span>
  </div>
  <div class="split-axis" id="splitAxisRow">
    <span>Split axis:</span>
    <label><input type="radio" name="splitAxis" value="L" checked> L</label>
    <label><input type="radio" name="splitAxis" value="W"> W</label>
    <label><input type="radio" name="splitAxis" value="H"> H</label>
    <label id="autoAxisLabel"><input type="radio" name="splitAxis" value="auto"> auto*</label>
  </div>
  <div id="packsList"></div>
  <button class="btn-add-pack" id="btnAddPack" onclick="addPack()">+ Add package</button>
</div>
```

- [ ] **Step 3: Add `renderPacksPanel()` to build cards from `packs[]`**

```javascript
function renderPacksPanel(){
  const list = document.getElementById('packsList');
  list.innerHTML = packs.map((p,idx) => packCardHTML(p, idx)).join('');
  updateAllocSum();
  // wire up inputs
  packs.forEach((p,idx) => wirePackCard(idx));
  document.getElementById('btnAddPack').disabled = packs.length >= 3;
  document.getElementById('autoAxisLabel').style.display =
    (outputMode === 'auto') ? '' : 'none';
}

function packCardHTML(p, idx){
  const removeBtn = idx > 0
    ? `<button class="btn-remove" onclick="removePack(${idx})">× Remove</button>` : '';
  return `
  <div class="pack-card" style="--pack-color:${p.color}" data-pack-idx="${idx}">
    <div class="pack-card-hdr">
      <span><span class="pack-color-swatch" style="background:${p.color}"></span>Pack ${p.id}</span>
      ${removeBtn}
    </div>
    <label>Shape:
      <select class="pack-shape">
        ${['cylinder','square','rectangle','sphere','wedge','bag']
          .map(s => `<option value="${s}"${s===p.shape?' selected':''}>${s}</option>`).join('')}
      </select>
    </label>
    <div class="pack-dims">${packDimsInputsHTML(p)}</div>
    <label>Gap (cm): <input type="number" class="pack-gap" value="${p.gap||0}" min="0" step="1"></label>
    <label>Weight (kg): <input type="number" class="pack-weight" value="${p.weight||0}" min="0" step="0.1"></label>
    <div class="alloc-row">
      <label>Alloc:</label>
      <input type="range" class="pack-alloc" min="0" max="100" step="1" value="${p.allocPct}">
      <span class="pack-alloc-val">${p.allocPct}%</span>
    </div>
    <button class="btn-suggest-pack" onclick="suggestForPack(${idx})">🔍 Suggest dims for this pack</button>
  </div>`;
}

function packDimsInputsHTML(p){
  switch(p.shape){
    case 'cylinder': return `<label>Dia: <input class="pack-dim" data-key="d" type="number" value="${p.dims.d||''}" min="0"></label>
                             <label>Height: <input class="pack-dim" data-key="h" type="number" value="${p.dims.h||''}" min="0"></label>`;
    case 'square':   return `<label>Side: <input class="pack-dim" data-key="s" type="number" value="${p.dims.s||''}" min="0"></label>
                             <label>Height: <input class="pack-dim" data-key="h" type="number" value="${p.dims.h||''}" min="0"></label>`;
    case 'rectangle':
    case 'wedge':
    case 'bag':      return `<label>L: <input class="pack-dim" data-key="l" type="number" value="${p.dims.l||''}" min="0"></label>
                             <label>W: <input class="pack-dim" data-key="w" type="number" value="${p.dims.w||''}" min="0"></label>
                             <label>H: <input class="pack-dim" data-key="h" type="number" value="${p.dims.h||''}" min="0"></label>`;
    case 'sphere':   return `<label>Dia: <input class="pack-dim" data-key="d" type="number" value="${p.dims.d||''}" min="0"></label>`;
  }
  return '';
}

function wirePackCard(idx){
  const card = document.querySelector(`.pack-card[data-pack-idx="${idx}"]`);
  if(!card) return;
  card.querySelector('.pack-shape').addEventListener('change', e => {
    packs[idx].shape = e.target.value; packs[idx].dims = {};
    renderPacksPanel();
  });
  card.querySelectorAll('.pack-dim').forEach(inp => {
    inp.addEventListener('input', e => { packs[idx].dims[e.target.dataset.key] = +e.target.value; });
  });
  card.querySelector('.pack-gap').addEventListener('input', e => { packs[idx].gap = +e.target.value; });
  card.querySelector('.pack-weight').addEventListener('input', e => { packs[idx].weight = +e.target.value; });
  const alloc = card.querySelector('.pack-alloc');
  const allocVal = card.querySelector('.pack-alloc-val');
  alloc.addEventListener('input', e => {
    packs[idx].allocPct = +e.target.value;
    allocVal.textContent = packs[idx].allocPct + '%';
    updateAllocSum();
  });
}

function updateAllocSum(){
  const sum = packs.reduce((a,p)=>a+(p.allocPct||0),0);
  const el = document.getElementById('allocSum');
  if(!el) return;
  el.textContent = `Sum: ${sum.toFixed(0)}% ${sum===100?'✓':'✗'}`;
  el.className = `alloc-sum ${sum===100?'ok':'bad'}`;
}
```

- [ ] **Step 4: Update `syncPacksFromDOM()` to read from the new panel instead of legacy inputs**

Replace its body:

```javascript
function syncPacksFromDOM(){
  // packs[] is already the source of truth (bound by wirePackCard listeners).
  // No-op stub kept for call-site compat.
}
```

- [ ] **Step 5: Initialize the panel on page load**

In whatever `init()` / DOMContentLoaded handler exists, append:

```javascript
renderPacksPanel();
document.querySelectorAll('input[name=splitAxis]').forEach(r => {
  r.addEventListener('change', e => { splitAxis = e.target.value; });
});
```

- [ ] **Step 6: Manual browser check**

Page loads with ONE pack card labeled "Pack 1" colored green, alloc 100%, sum badge "Sum: 100% ✓". Shape selector works — switching to rectangle changes dim inputs to L/W/H. Entering dims and clicking Calculate packs correctly.

- [ ] **Step 7: Commit**

```bash
git add cointainer_calculator.html
git commit -m "Add Packs panel UI rendering from packs[] state"
```

---

## Task 8: Add / Remove pack + alloc rebalancing

**Goal:** Implement the + button and × Remove buttons with automatic alloc rebalancing.

**Files:**
- Modify: `cointainer_calculator.html`

- [ ] **Step 1: Add `addPack()` and `removePack(idx)`**

```javascript
function addPack(){
  if(packs.length >= 3) return;
  const last = packs[packs.length - 1];
  const newId = packs.length + 1;
  packs.push({
    id: newId,
    shape: last.shape,
    dims: {},
    gap: last.gap || 0,
    weight: 0,
    allocPct: 0, // reset below
    color: PACK_COLORS[packs.length]
  });
  rebalanceAlloc();
  renderPacksPanel();
}

function removePack(idx){
  if(idx <= 0 || idx >= packs.length) return;
  packs.splice(idx, 1);
  // Re-number and re-color
  packs.forEach((p,i) => { p.id = i + 1; p.color = PACK_COLORS[i]; });
  rebalanceAlloc();
  renderPacksPanel();
}

function rebalanceAlloc(){
  const n = packs.length;
  const even = Math.floor(100 / n);
  packs.forEach((p,i) => { p.allocPct = even; });
  // Put remainder on pack 1 so sum is exactly 100
  const sum = packs.reduce((a,p)=>a+p.allocPct, 0);
  packs[0].allocPct += (100 - sum);
}
```

- [ ] **Step 2: Manual browser check**

Click "+ Add package". Second card (teal) appears; sum still 100 (50/50). Click again → third card (amber) appears (33/33/34). Click again → button disabled. Click × on Pack 2 → card removes; Pack 3 renumbers to Pack 2; sum = 100 (50/50).

- [ ] **Step 3: Commit**

```bash
git add cointainer_calculator.html
git commit -m "Add addPack/removePack with alloc rebalancing"
```

---

## Task 9: Split-axis control + auto-axis solver

**Goal:** Wire up axis radios and implement auto-axis for Max Cap.

**Files:**
- Modify: `cointainer_calculator.html`

- [ ] **Step 1: Add `tryAllAxes(c, packs)` and hook it into calculate**

```javascript
function tryAllAxes(c, packsArr){
  const axes = ['L','W','H'];
  let best = null;
  for(const ax of axes){
    const zones = computePackZones(c, ax, packsArr);
    let total = 0;
    const perPack = [];
    for(let i=0;i<packsArr.length;i++){
      const p = getPackDims(i);
      const g = calcGrid(zones[i], p);
      perPack.push(g);
      total += g.count;
    }
    if(!best || total > best.total) best = { axis: ax, total, zones, perPack };
  }
  return best;
}
```

In `calculate()`, after the alloc-sum check, replace the `zones` construction with:

```javascript
let chosenAxis = splitAxis;
let autoReport = '';
if(splitAxis === 'auto' && outputMode === 'auto'){
  const tries = ['L','W','H'].map(ax => {
    const zz = computePackZones(c, ax, packs);
    let t = 0;
    for(let i=0;i<packs.length;i++) t += calcGrid(zz[i], getPackDims(i)).count;
    return { ax, t };
  });
  const best = tries.reduce((b,x)=>x.t>b.t?x:b, tries[0]);
  chosenAxis = best.ax;
  autoReport = ` (tried L=${tries[0].t}, W=${tries[1].t}, H=${tries[2].t})`;
}
const zones = computePackZones(c, chosenAxis === 'auto' ? 'L' : chosenAxis, packs);
```

Pass `chosenAxis` and `autoReport` into `renderResult` so the split line reads e.g. `Split: L (auto: best of 3) — tried L=120, W=108, H=96`.

- [ ] **Step 2: Toggle auto-axis radio visibility on mode change**

In `setMode()`, after setting `outputMode`, call `renderPacksPanel()` or toggle `#autoAxisLabel` visibility:

```javascript
const autoLbl = document.getElementById('autoAxisLabel');
if(autoLbl) autoLbl.style.display = (outputMode === 'auto') ? '' : 'none';
if(outputMode !== 'auto' && splitAxis === 'auto'){
  splitAxis = 'L';
  const r = document.querySelector('input[name=splitAxis][value=L]');
  if(r) r.checked = true;
}
```

- [ ] **Step 3: Manual browser check**

- Switch to Max Capacity mode → auto* radio visible.
- Set 3 packs 40/30/30. Pick "auto". Click Calculate.
- Status shows `Split: L (auto) — tried L=..., W=..., H=...` with the best axis reported.
- Switch to Custom Qty mode → auto* hidden; if previously selected, falls back to L.

- [ ] **Step 4: Commit**

```bash
git add cointainer_calculator.html
git commit -m "Add auto-axis solver + split-axis visibility rules"
```

---

## Task 10: Remove legacy pkg2 code

**Goal:** Delete all the `p2_*` inputs, `pkg2Enabled`, `togglePkg2`, `setType2`, `getPkgDims2`, and the old `pkg2-hdr` HTML. The new panel has replaced it.

**Files:**
- Modify: `cointainer_calculator.html`

- [ ] **Step 1: Delete HTML**

Ensure the old `pkg2-hdr`, `pkg2Body`, `inputs2_*` blocks (lines 425–470 pre-change) are gone. They should be gone after Task 7 — this step is verification. Grep for `pkg2` and `p2_` to confirm no residual references.

- [ ] **Step 2: Delete JS**

Remove:
- `let pkg2Enabled = ...;`
- `function togglePkg2(){...}`
- `function setType2(...){...}`
- `function getPkgDims2(){...}`
- Any `currentType2` variable.

Search for each symbol with grep — remove all hits or rewrite callers to use `packs[]`.

- [ ] **Step 3: Manual browser check**

Console: `typeof pkg2Enabled` → `"undefined"`. Page still works; no console errors. Full regression: single pack runs, two packs run, three packs run.

- [ ] **Step 4: Commit**

```bash
git add cointainer_calculator.html
git commit -m "Remove legacy pkg2 code (now handled by packs[])"
```

---

## Task 11: Suggest — shape-preset mins + aspect guard

**Goal:** Add the shape-preset minimum dimensions and the aspect-ratio guard to `runSuggestSize`.

**Files:**
- Modify: `cointainer_calculator.html` — `runSuggestSize` (~line 1781)

- [ ] **Step 1: Add the preset table and helpers**

```javascript
const SHAPE_MIN_DEFAULTS = {
  cylinder:  { d:20, h:20 },
  square:    { s:15, h:15 },
  rectangle: { l:15, w:15, h:15 },
  sphere:    { d:10 },
  wedge:     { l:15, w:15, h:15 },
  bag:       { l:30, w:30, h:30 }
};
let shapeMinOverrides = {}; // populated from Advanced UI in Task 12

function getShapeMin(shape, key){
  return shapeMinOverrides[shape]?.[key] ?? SHAPE_MIN_DEFAULTS[shape][key];
}

function passesAspectGuard(dimsObj){
  const vals = Object.values(dimsObj).filter(v => v > 0);
  if(vals.length < 2) return true;
  const mx = Math.max(...vals), mn = Math.min(...vals);
  return (mx / mn) <= 10;
}
```

- [ ] **Step 2: Update `runSuggestSize(c)` iteration bounds**

Inside each shape branch, replace the starting bound of each loop from `1` to `getShapeMin(shape, key)`. Examples:

```javascript
// cylinder
for(let d = getShapeMin('cylinder','d'); d <= Math.min(c.l, c.w); d++){
  for(let h = getShapeMin('cylinder','h'); h <= c.h; h++){
    // ... existing body ...
    if(!passesAspectGuard({d, h})) continue;
    // push candidate
  }
}

// rectangle (step stays 5)
for(let l = getShapeMin('rectangle','l'); l <= c.l; l += 5){
  for(let w = getShapeMin('rectangle','w'); w <= c.w; w += 5){
    for(let h = getShapeMin('rectangle','h'); h <= c.h; h += 5){
      if(!passesAspectGuard({l, w, h})) continue;
      // push
    }
  }
}
```

Apply to `square`, `sphere`, `wedge`, `bag` analogously.

- [ ] **Step 3: Manual browser check**

Suggest mode, cylinder, 20ft container, no target count. Results should all have `d ≥ 20` and `h ≥ 20`. No more 1 cm × 5 cm rows. Ratio max/min ≤ 10 across results.

- [ ] **Step 4: Commit**

```bash
git add cointainer_calculator.html
git commit -m "Suggest: enforce shape-preset min dims + aspect-ratio guard"
```

---

## Task 12: Suggest — Auto-partial mode + lock toggles + Advanced

**Goal:** Add the "Auto all / Auto partial" radio and per-dimension lock toggles; add Advanced collapsible for min-dim overrides.

**Files:**
- Modify: `cointainer_calculator.html` — Suggest panel HTML (~line 495) + `runSuggestSize`

- [ ] **Step 1: Extend Suggest panel HTML**

Replace the suggest panel block with:

```html
<div class="suggest-panel" id="suggestPanel">
  <div class="suggest-mode-row">
    Mode:
    <label><input type="radio" name="sgMode" value="all" checked> Auto all</label>
    <label><input type="radio" name="sgMode" value="partial"> Auto partial</label>
  </div>
  <div id="sgAllBlock">
    <label>Target count (optional): <input type="number" id="sug_target" min="0"></label>
  </div>
  <div id="sgPartialBlock" style="display:none">
    <label>Shape: <select id="sgShape">
      <option value="cylinder">cylinder</option>
      <option value="square">square</option>
      <option value="rectangle">rectangle</option>
      <option value="sphere">sphere</option>
      <option value="wedge">wedge</option>
      <option value="bag">bag</option>
    </select></label>
    <div id="sgPartialDims"></div>
  </div>
  <details class="advanced">
    <summary>Advanced: min-dim overrides</summary>
    <div id="sgAdvancedBody"></div>
  </details>
  <button onclick="runSuggestFromPanel()">Run Suggest</button>
</div>
```

- [ ] **Step 2: Render partial dims with lock toggles**

```javascript
function renderSgPartialDims(){
  const shape = document.getElementById('sgShape').value;
  const keys = {
    cylinder:['d','h'], square:['s','h'],
    rectangle:['l','w','h'], sphere:['d'],
    wedge:['l','w','h'], bag:['l','w','h']
  }[shape];
  document.getElementById('sgPartialDims').innerHTML = keys.map(k => `
    <label>${k.toUpperCase()}: <input type="number" class="sg-dim" data-key="${k}" min="0"></label>
    <button class="sg-lock" data-key="${k}" data-locked="0">🔓 auto</button><br>
  `).join('');
  document.querySelectorAll('.sg-lock').forEach(btn => {
    btn.addEventListener('click', e => {
      const locked = e.target.dataset.locked === '1';
      e.target.dataset.locked = locked ? '0' : '1';
      e.target.textContent = locked ? '🔓 auto' : '🔒 locked';
    });
  });
}
```

Wire up mode radio and shape change:

```javascript
document.querySelectorAll('input[name=sgMode]').forEach(r => {
  r.addEventListener('change', e => {
    document.getElementById('sgAllBlock').style.display = e.target.value==='all' ? '' : 'none';
    document.getElementById('sgPartialBlock').style.display = e.target.value==='partial' ? '' : 'none';
    if(e.target.value==='partial') renderSgPartialDims();
  });
});
document.getElementById('sgShape').addEventListener('change', renderSgPartialDims);
```

- [ ] **Step 3: Render Advanced min-dim overrides**

```javascript
function renderSgAdvanced(){
  const body = document.getElementById('sgAdvancedBody');
  body.innerHTML = Object.entries(SHAPE_MIN_DEFAULTS).map(([shape, mins]) => `
    <div><b>${shape}:</b> ${Object.entries(mins).map(([k,v]) => `
      ${k.toUpperCase()} ≥ <input class="sg-min" data-shape="${shape}" data-key="${k}" type="number" value="${v}" min="1" style="width:50px">
    `).join(' ')}</div>
  `).join('');
  body.querySelectorAll('.sg-min').forEach(inp => {
    inp.addEventListener('input', e => {
      const s = e.target.dataset.shape, k = e.target.dataset.key;
      shapeMinOverrides[s] ??= {};
      shapeMinOverrides[s][k] = +e.target.value;
    });
  });
}
```

Call `renderSgAdvanced()` once in init.

- [ ] **Step 4: Update `runSuggestSize` to honor locked dims**

```javascript
function runSuggestFromPanel(){
  const sgMode = document.querySelector('input[name=sgMode]:checked').value;
  const c = getContainerDims();
  let locked = {};
  let shape;
  if(sgMode === 'partial'){
    shape = document.getElementById('sgShape').value;
    document.querySelectorAll('.sg-lock').forEach(btn => {
      if(btn.dataset.locked === '1'){
        const key = btn.dataset.key;
        const val = +document.querySelector(`.sg-dim[data-key="${key}"]`).value;
        if(val > 0) locked[key] = val;
      }
    });
  } else {
    shape = document.querySelector('.chip[data-type].active')?.dataset.type || 'cylinder';
  }
  const target = +document.getElementById('sug_target').value || 0;
  const rows = runSuggestSize(c, shape, locked, target);
  renderSuggest(rows, target, c);
}
```

Modify `runSuggestSize(c, shape, lockedDims={}, target=0)` so each iteration loop uses `lockedDims[key]` if present (single value) instead of iterating:

```javascript
function iterRange(shape, key, c, lockedDims){
  if(lockedDims[key]) return [lockedDims[key]];
  const min = getShapeMin(shape, key);
  const maxCap = (key==='l') ? c.l : (key==='w') ? c.w : (key==='h') ? c.h :
                 (key==='s') ? Math.min(c.l,c.w) : (key==='d') ? Math.min(c.l,c.w) : c.h;
  const step = ['l','w','h'].includes(key) && ['rectangle','wedge','bag'].includes(shape) ? 5 : 1;
  const out = [];
  for(let v = min; v <= maxCap; v += step) out.push(v);
  return out;
}
```

Rewrite the shape branches to use `iterRange`.

- [ ] **Step 5: Manual browser check**

- Pick "Auto partial" → form switches. Select rectangle shape. Fill W=50, lock W. Leave L, H unlocked. Click Run Suggest. All result rows have W=50.
- Open Advanced, change cylinder d min from 20 to 30. Switch to Auto all + cylinder. Results all have d ≥ 30.

- [ ] **Step 6: Commit**

```bash
git add cointainer_calculator.html
git commit -m "Suggest: add auto-partial mode with lock toggles + Advanced min-dim overrides"
```

---

## Task 13: Suggest — multi-pack per-pack button

**Goal:** When user clicks "Suggest dims for this pack" inside a pack card, run Suggest against that pack's allocated zone.

**Files:**
- Modify: `cointainer_calculator.html`

- [ ] **Step 1: Implement `suggestForPack(idx)`**

```javascript
function suggestForPack(idx){
  // Validate that all OTHER packs have filled dims
  for(let i=0; i<packs.length; i++){
    if(i === idx) continue;
    const d = packs[i].dims;
    const required = {
      cylinder:['d','h'], square:['s','h'], rectangle:['l','w','h'],
      sphere:['d'], wedge:['l','w','h'], bag:['l','w','h']
    }[packs[i].shape] || [];
    for(const k of required){
      if(!(d[k] > 0)){
        alert(`Fill dimensions for Pack ${packs[i].id} (${packs[i].shape}) first.`);
        return;
      }
    }
  }
  const c = getContainerDims();
  const zones = computePackZones(c, splitAxis === 'auto' ? 'L' : splitAxis, packs);
  const z = zones[idx];
  if(z.l <= 0 || z.w <= 0 || z.h <= 0){
    alert('No space remaining — increase this pack\\'s allocation.');
    return;
  }
  const shape = packs[idx].shape;
  const reducedContainer = { l: z.l, w: z.w, h: z.h };
  const rows = runSuggestSize(reducedContainer, shape, {}, 0);
  renderSuggest(rows, 0, reducedContainer);
}
```

- [ ] **Step 2: Manual browser check**

Add 2 packs. Fill Pack 1 (cylinder d=50 h=80) fully. On Pack 2 card (empty), click "Suggest dims for this pack". Results appear, all feasible within Pack 2's allocated zone (not the full container).

With all packs empty, clicking Suggest on Pack 2 shows alert "Fill dimensions for Pack 1 first."

- [ ] **Step 3: Commit**

```bash
git add cointainer_calculator.html
git commit -m "Suggest: per-pack button operates against pack's allocated zone"
```

---

## Task 14: Demo 5-calc limit + banner

**Goal:** Cap demo users at 5 successful Calculate runs, show banner, reset on license activation.

**Files:**
- Modify: `cointainer_calculator.html`

- [ ] **Step 1: Add banner HTML (hidden by default)**

Place near the top of the main content area:

```html
<div id="demoLimitBanner" class="demo-banner" style="display:none">
  <b>Demo limit reached (5/5 calculations).</b>
  Enter a license key to continue — free for beta testers: hovanphapbk@gmail.com
  <button onclick="showLockScreen()">Enter License</button>
</div>
```

Add CSS:

```css
.demo-banner { background:#4a2a00; color:#ffd33d; padding:10px; border-radius:6px; margin:10px 0; }
.demo-counter { font-size:12px; color:#8b949e; text-align:right; margin-top:4px; }
```

And counter display under Calculate button:

```html
<div id="demoCounter" class="demo-counter" style="display:none"></div>
```

- [ ] **Step 2: Add counter helpers**

```javascript
const DEMO_LIMIT = 5;
function demoCount(){ return +(localStorage.getItem('cpc_demo_calcs') || 0); }
function demoBump(){
  const n = demoCount() + 1;
  localStorage.setItem('cpc_demo_calcs', String(n));
  updateDemoCounterUI();
}
function demoReset(){
  localStorage.removeItem('cpc_demo_calcs');
  updateDemoCounterUI();
}
function updateDemoCounterUI(){
  const counterEl = document.getElementById('demoCounter');
  const bannerEl = document.getElementById('demoLimitBanner');
  if(!counterEl || !bannerEl) return;
  if(!isDemo){
    counterEl.style.display = 'none';
    bannerEl.style.display = 'none';
    return;
  }
  const n = demoCount();
  counterEl.style.display = '';
  counterEl.textContent = `Demo: ${Math.min(n, DEMO_LIMIT)} / ${DEMO_LIMIT} calculations used`;
  bannerEl.style.display = (n >= DEMO_LIMIT) ? '' : 'none';
}
```

- [ ] **Step 3: Gate `calculate()`**

At the very top of `calculate()`, after input validation but before the alloc-sum check:

```javascript
if(isDemo && demoCount() >= DEMO_LIMIT){
  updateDemoCounterUI();
  return;
}
```

At the **end** of a successful calculate (after `renderResult` and `draw3D`), add:

```javascript
if(isDemo) demoBump();
```

- [ ] **Step 4: Reset on license activation**

In `tryUnlock()`, after the license is confirmed valid and stored, call `demoReset()`. Also call it inside the success path of `checkStoredLicense()` if it transitions from demo to licensed.

- [ ] **Step 5: Init + demo-mode toggles**

After page load (and after `applyDemoRestrictions()` runs), call `updateDemoCounterUI()`.

- [ ] **Step 6: Manual browser check**

1. Open app, click "Try Demo" (or set `isDemo=true` in console).
2. Click Calculate 5 times with a valid setup. Counter shows `1/5 … 5/5`. No banner yet.
3. Click Calculate again → banner appears, nothing calculates.
4. Reload page → banner still shows (localStorage persists), counter still 5/5, Calculate still gated.
5. Enter valid license via lock screen → banner hides, counter hides, Calculate works.
6. As licensed user: no counter, no banner, unlimited calcs.

- [ ] **Step 7: Commit**

```bash
git add cointainer_calculator.html
git commit -m "Demo: 5-calculation soft limit with banner + counter"
```

---

## Task 15: Demo multi-pack restriction

**Goal:** In demo, lock to Pack 1 only (disable "+ Add package", hide Remove buttons).

**Files:**
- Modify: `cointainer_calculator.html` — `applyDemoRestrictions()`

- [ ] **Step 1: Extend `applyDemoRestrictions()`**

```javascript
function applyDemoRestrictions(){
  // ... existing disables ...
  if(isDemo){
    document.getElementById('btnAddPack').disabled = true;
    document.querySelectorAll('.btn-remove').forEach(b => b.style.display = 'none');
    // If demo somehow starts with >1 pack, truncate
    if(packs.length > 1){ packs = packs.slice(0,1); packs[0].allocPct = 100; renderPacksPanel(); }
  }
}
```

Call `applyDemoRestrictions()` at the end of `renderPacksPanel()` so added-then-demo flows also respect it.

- [ ] **Step 2: Manual browser check**

Enter demo mode. "+ Add package" button is visibly disabled. If demo starts after packs were added (e.g., logout), list truncates to Pack 1. Other modes (Suggest, Compare, HC, Print) still disabled as today.

- [ ] **Step 3: Commit**

```bash
git add cointainer_calculator.html
git commit -m "Demo: lock to single pack (disable + and Remove)"
```

---

## Task 16: Full regression pass + deploy

**Goal:** Run the full acceptance list from the spec §12, fix any found issues, then copy to `index.html` and push.

**Files:**
- Modify: `index.html` (full overwrite from `cointainer_calculator.html`)

- [ ] **Step 1: Run all §12 acceptance tests manually**

Open `cointainer_calculator.html` in a fresh browser (or hard-reload, clear localStorage first). Run each test in turn; check each off:

1. Single pack (100%) — Max Cap / Custom / Suggest all match pre-change behavior.
2. Two packs L-split 60/40 — 3D zones correct, status lists both.
3. Three packs W-split 40/40/20 — 3 zones across width.
4. Two packs H-split 50/50 — teal stacked above green.
5. Auto axis (Max Cap) — status reports winning axis + counts tried.
6. Alloc ≠ 100 — sum badge red, Calculate shows error, no render.
7. Suggest single-pack cylinder — all dims ≥ 20 cm.
8. Suggest auto-partial, box W=50 locked — all rows W=50.
9. Suggest Pack 2 while Pack 1 filled — result fits Pack 2 zone.
10. Demo mode — "+" disabled; Suggest/HC/Compare/Print disabled.
11. Remove Pack 2 of 3 — renumber works, alloc rebalances.
12. Weight overflow — warning on status.
13. Demo 5-calc — 6th click gated, banner shown; license clears counter.
14. Demo invalid runs don't count — alloc≠100 in demo: counter unchanged.

Fix any failures inline (extra commit per fix).

- [ ] **Step 2: Copy to index.html and verify**

```bash
cp cointainer_calculator.html index.html
```

Check `diff cointainer_calculator.html index.html` is empty.

- [ ] **Step 3: Commit + push**

```bash
git add cointainer_calculator.html index.html
git commit -m "Multi-pack + improved Suggest + demo 5-calc limit (deploy)"
git push
```

- [ ] **Step 4: Smoke test live site**

Open `https://a17255.github.io/container-calculator/` after Pages rebuild (~1 min). Repeat acceptance #1, #2, #13 on the live URL.

---

## Notes for the implementer

- **Do not** touch `common-license\CLAUDE.md` or the license key algorithm constants (`_KA`, `_KB`, `_KC`). Those are cross-project shared state.
- The license block in this app is the reference implementation for BEM and RBH. Any accidental change there needs to be mirrored — so don't accidentally change it.
- Existing 3D camera/rotation math works in world coordinates; zones only add an offset. No changes to view projection are needed.
- If `calcGrid(container, pkg)` doesn't exist as a named function, it's the per-axis integer division logic near line 1240 — extract it into a named helper as part of Task 4 (it's reused by Task 9's `tryAllAxes`).
- Cylinder lying orientations (`drawCylXaxis` / `drawCylYaxis`) are handled inside `drawCyl3d` — don't duplicate that logic.
