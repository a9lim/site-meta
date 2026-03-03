# Simulation Improvements Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Implement 18 improvements across physsim (9), biosim (5), and gerry (4) to deepen feature completeness, educational value, and realism.

**Architecture:** Each improvement is self-contained with its own commit. Changes stay within each project's existing ES6 module structure. No new dependencies — all rendering is vanilla Canvas 2D (physsim/biosim) or SVG (gerry). Tests are manual visual verification (no test framework in these projects).

**Tech Stack:** Vanilla JS (ES6 modules), HTML5 Canvas 2D, SVG, CSS custom properties

---

## Phase 1: Quick Wins

---

### Task 1: G-F — Compactness Formula Fix

**Files:**
- Modify: `gerry/src/metrics.js:55-66`
- Modify: `gerry/src/config.js:26-27` (read HEX_W, HEX_H for geometry)

**Step 1: Understand the current formula**

Current code at `metrics.js:65`:
```js
Math.min(100, Math.round((32.648 * d.hexes.length) / (perimeter * perimeter) * 100))
```
The constant `32.648` is an approximation. The correct Polsby-Popper formula is `4π × area / perimeter²` where area and perimeter are in consistent units.

**Step 2: Compute correct hex geometry constants**

In `config.js`, `HEX_W = SQRT3 * hexSize` (~31.18px) and `HEX_H = 1.5 * hexSize` (27px). For a flat-top hexagon with size `s`:
- Area per hex = `(3√3/2) × s²`
- Edge length = `s` (one hex side)

The perimeter counter in `calculateCompactness` counts exposed edges. Each exposed edge has pixel length `s` = `CONFIG.hexSize` (18px).

**Step 3: Replace the formula in metrics.js**

Open `gerry/src/metrics.js`. At the top, add the hex geometry import if not already present, and at line 65, replace:

```js
// Old:
return Math.min(100, Math.round((32.648 * d.hexes.length) / (perimeter * perimeter) * 100));

// New — correct Polsby-Popper with hex geometry:
const hexArea = (3 * Math.sqrt(3) / 2) * CONFIG.hexSize * CONFIG.hexSize;
const totalArea = d.hexes.length * hexArea;
const totalPerimeter = perimeter * CONFIG.hexSize;
return Math.min(100, Math.round((4 * Math.PI * totalArea) / (totalPerimeter * totalPerimeter) * 100));
```

Ensure `CONFIG` is imported from `./config.js` at the top of `metrics.js`.

**Step 4: Verify**

Serve the project locally, draw a few districts, check that compactness values are reasonable (a roughly circular district should be 70-85%, a long snaking district should be 10-30%). Compare to old values — the new formula should produce similar but more accurate results.

**Step 5: Commit**

```bash
cd gerry
git add src/metrics.js
git commit -m "fix: use correct Polsby-Popper formula for compactness calculation

Replace approximated constant (32.648) with proper 4πA/P² using actual hex
geometry (area = 3√3/2 × s², perimeter in pixel units). Results are now on a
true 0-100% scale comparable to standard political science metrics."
```

---

### Task 2: B-D — ATP Source Tracking

**Files:**
- Modify: `biosim/src/state.js:26-43` (add counters to store)
- Modify: `biosim/src/reactions/glycolysis.js:63-71,80-88` (tag substrate ATP at PGK/PK steps)
- Modify: `biosim/src/reactions/krebs.js:34-43` (tag substrate ATP at SCS step)
- Modify: `biosim/src/reactions/etc.js:67-75` (tag oxidative ATP at ATP synthase)
- Modify: `biosim/src/dashboard.js:53-70` (add split bar display)
- Modify: `biosim/index.html:285-294` (add ATP source stats HTML)

**Step 1: Add tracking counters to store**

In `biosim/src/state.js`, add two new counters inside the `store` object (after the existing metabolite pools around line 31):

```js
atpSubstrate: 0,
atpOxidative: 0,
```

In `resetState()` (line 48), add these to the reset block:
```js
store.atpSubstrate = 0;
store.atpOxidative = 0;
```

**Step 2: Tag ATP production in glycolysis**

In `biosim/src/reactions/glycolysis.js`:
- At step 6 (PGK, around line 66 where `yields: { atp: 2 }`), add after the `store.atp += 2` line:
  ```js
  store.atpSubstrate += 2;
  ```
- At step 9 (PK, around line 84 where `yields: { atp: 2 }`), add after the `store.atp += 2` line:
  ```js
  store.atpSubstrate += 2;
  ```
- In `runGlycolysisLower()` (around line 116 where ATP is produced), add:
  ```js
  store.atpSubstrate += 4;
  ```

**Step 3: Tag ATP production in Krebs**

In `biosim/src/reactions/krebs.js`:
- At step 4 (SCS, around line 38 where `yields: { atp: t }`), after `store.atp += t`:
  ```js
  store.atpSubstrate += t;
  ```
- In `runKrebsCycle()` (around line 80 where `store.atp += 2`):
  ```js
  store.atpSubstrate += 2;
  ```

**Step 4: Tag ATP production at ATP synthase**

In `biosim/src/reactions/etc.js`, in `advanceATPSynthase()` (around line 69), after `store.atp++`:
```js
store.atpOxidative++;
```

**Step 5: Add HTML for the source split display**

In `biosim/index.html`, after the Proton Motive Force stat group (around line 288), add:

```html
<div class="stat-group">
  <div class="group-label">ATP Source</div>
  <div class="stat-row">
    <span>Substrate-Level</span>
    <span class="stat-value" id="atp-substrate">0</span>
  </div>
  <div class="stat-row">
    <span>Oxidative</span>
    <span class="stat-value" id="atp-oxidative">0</span>
  </div>
  <div class="stat-row" style="margin-top:4px">
    <div id="atp-source-bar" style="display:flex;height:6px;border-radius:3px;overflow:hidden;width:100%">
      <div id="atp-source-sub" style="background:var(--palette-orange);height:100%;transition:width .3s"></div>
      <div id="atp-source-ox" style="background:var(--palette-blue);height:100%;transition:width .3s"></div>
    </div>
  </div>
</div>
```

**Step 6: Update dashboard rendering**

In `biosim/src/dashboard.js`, in `updateDashboard()` (around line 53), add after the existing counter updates:

```js
const atpSub = _dom.atpSubstrate;
const atpOx = _dom.atpOxidative;
const atpSubBar = _dom.atpSourceSub;
const atpOxBar = _dom.atpSourceOx;
if (atpSub) atpSub.textContent = store.atpSubstrate;
if (atpOx) atpOx.textContent = store.atpOxidative;
const totalAtp = store.atpSubstrate + store.atpOxidative;
if (totalAtp > 0 && atpSubBar && atpOxBar) {
    atpSubBar.style.width = (store.atpSubstrate / totalAtp * 100) + '%';
    atpOxBar.style.width = (store.atpOxidative / totalAtp * 100) + '%';
}
```

In `biosim/src/ui.js`, in `cacheDOMElements()`, add the new IDs:
```js
dom.atpSubstrate = document.getElementById('atp-substrate');
dom.atpOxidative = document.getElementById('atp-oxidative');
dom.atpSourceSub = document.getElementById('atp-source-sub');
dom.atpSourceOx = document.getElementById('atp-source-ox');
```

**Step 7: Verify**

Run biosim, enable glycolysis + Krebs + autoplay. Watch ATP source counters. Oxidative should dominate (~30 ATP per glucose vs ~4 substrate). The split bar should show mostly blue (oxidative).

**Step 8: Commit**

```bash
cd biosim
git add src/state.js src/reactions/glycolysis.js src/reactions/krebs.js src/reactions/etc.js src/dashboard.js src/ui.js index.html
git commit -m "feat: track ATP source (substrate-level vs oxidative phosphorylation)

Add cumulative counters for substrate-level ATP (glycolysis PGK/PK, Krebs SCS)
and oxidative ATP (ATP synthase). Display split bar in Stats tab showing the
ratio, making it visually clear that oxidative phosphorylation produces ~90%
of cellular ATP."
```

---

### Task 3: B-F — Proton Leak / Uncoupling

**Files:**
- Modify: `biosim/src/state.js:4-24` (add uncoupling state)
- Modify: `biosim/src/autoplay.js:18-76` (add leak logic)
- Modify: `biosim/src/particles.js` (reverse proton animation)
- Modify: `biosim/src/ui.js:62-182` (wire toggle, add info tip)
- Modify: `biosim/src/dashboard.js:53-70` (add leak counter display)
- Modify: `biosim/index.html:199-226` (add uncoupling toggle HTML)

**Step 1: Add state for uncoupling**

In `biosim/src/state.js`, add to `simState` (around line 9):
```js
uncouplingEnabled: false,
```

Add to `store` (around line 31):
```js
protonsLeaked: 0,
```

In `resetState()`, add: `store.protonsLeaked = 0;`

**Step 2: Add leak logic to autoplay**

In `biosim/src/autoplay.js`, add a new timer at the top (around line 9):
```js
let leakTimer = 0;
```

In `resetAutoplayTimers()`, add: `leakTimer = 0;`

In `autoplayTick(dt)`, after the timer increments (around line 23), add a new leak block:

```js
// Proton leak — always active (not just in autoplay)
leakTimer += dt;
if (leakTimer > 0.5) {
    leakTimer = 0;
    const leakRate = simState.uncouplingEnabled ? 0.10 : 0.02;
    const leaked = Math.floor(store.protonGradient * leakRate);
    if (leaked > 0) {
        store.protonGradient -= leaked;
        store.protonsLeaked += leaked;
    }
}
```

Note: The leak should run even when autoplay is off. Move this block to execute before the `if (!simState.autoPlay) return` guard at line 19 — or create a separate exported function `protonLeakTick(dt)` that gets called from the main loop in `main.js` independently of autoplay.

**Step 3: Add uncoupling toggle HTML**

In `biosim/index.html`, in the Environment ctrl-group (around line 219, after the autoplay toggle), add:

```html
<div class="ctrl-row">
    <label for="uncoupling-toggle">Uncoupling <button class="info-trigger" data-info="uncoupling">?</button></label>
    <div class="tog-wrap">
        <input type="checkbox" id="uncoupling-toggle" role="switch">
        <label for="uncoupling-toggle" class="tog tog-uncoupling"><span class="tog-thumb"></span></label>
    </div>
</div>
```

**Step 4: Add leak counter to stats HTML**

In `biosim/index.html`, after the proton gradient stat row (around line 288), add:
```html
<div class="stat-row">
    <span>Protons Leaked</span>
    <span class="stat-value" id="protons-leaked">0</span>
</div>
```

**Step 5: Wire toggle and info tip in UI**

In `biosim/src/ui.js`:
- In `cacheDOMElements()`, add: `dom.uncouplingToggle = document.getElementById('uncoupling-toggle');`
- In `bindEvents()`, after the autoplay toggle wiring (around line 77), add:
  ```js
  dom.uncouplingToggle?.addEventListener('change', e => { simState.uncouplingEnabled = e.target.checked; });
  ```
- In the `infoData` object (around line 172), add:
  ```js
  uncoupling: { title: 'Uncoupling Proteins', body: 'Uncoupling proteins create a passive proton leak across the membrane, dissipating the gradient as heat instead of making ATP. Found in brown fat for thermogenesis. Base leak rate: 2%/tick. With uncoupling protein: 10%/tick.' },
  ```

In `biosim/src/ui.js` `cacheDOMElements()`, add: `dom.protonsLeaked = document.getElementById('protons-leaked');`

**Step 6: Update dashboard**

In `biosim/src/dashboard.js`, in `updateDashboard()`, add:
```js
if (_dom.protonsLeaked) _dom.protonsLeaked.textContent = store.protonsLeaked;
```

**Step 7: Add CSS for toggle color**

In `biosim/styles.css`, add:
```css
.tog-uncoupling { --tog-color: var(--palette-brown); }
```

**Step 8: Verify**

Run biosim with ETC running. Watch proton gradient slowly decay even without ATP synthase. Enable uncoupling protein — gradient should drain 5× faster. Check that ATP production decreases when uncoupling is on.

**Step 9: Commit**

```bash
cd biosim
git add src/state.js src/autoplay.js src/ui.js src/dashboard.js index.html styles.css
git commit -m "feat: add proton leak and uncoupling protein toggle

Passive proton leak (2% per 0.5s tick) makes ATP synthase realistically
less than 100% efficient. Uncoupling protein toggle increases leak rate
to 10%, simulating brown fat thermogenesis. Leak counter in Stats tab."
```

---

### Task 4: P-E — EM Field Energy Bookkeeping

**Files:**
- Modify: `physsim/main.js:100-190` (add Darwin term to computeEnergy)
- Modify: `physsim/index.html:257-264` (add Field Energy stat row)

**Step 1: Add Field Energy stat row to HTML**

In `physsim/index.html`, in the Energy section (around line 264, after the Drift sub-row), add:
```html
<div class="stat-row stat-sub">
    <span>Field</span>
    <span class="stat-value" id="fieldE">0</span>
</div>
```

**Step 2: Add DOM reference**

In `physsim/main.js`, in the constructor where `this.dom` is built (around line 37-53), add:
```js
fieldE: document.getElementById('fieldE'),
```

**Step 3: Compute Darwin Lagrangian field energy**

In `physsim/main.js`, in `computeEnergy()` (around line 100), after the existing PE pair loop but before the totals computation:

The existing pair loop already computes PE terms (gravity, Coulomb, magnetic dipole, gravitomagnetic dipole). Add the Darwin velocity-dependent correction in the same pair loop:

```js
// Darwin Lagrangian O(v²/c²) correction for velocity-dependent field energy
// U_Darwin = -(1/2) Σ_{i<j} (q_i·q_j / r_ij) × [(v_i·v_j) + (v_i·r̂)(v_j·r̂)]
let fieldEnergy = 0;
if (this.physics.coulombEnabled) {
    for (let i = 0; i < n; i++) {
        const pi = particles[i];
        for (let j = i + 1; j < n; j++) {
            const pj = particles[j];
            const dx = pj.pos.x - pi.pos.x;
            const dy = pj.pos.y - pi.pos.y;
            const rSq = dx * dx + dy * dy + SOFTENING_SQ;
            const invR = 1 / Math.sqrt(rSq);
            const rx = dx * invR, ry = dy * invR;
            const viDotVj = pi.vel.x * pj.vel.x + pi.vel.y * pj.vel.y;
            const viDotR = pi.vel.x * rx + pi.vel.y * ry;
            const vjDotR = pj.vel.x * rx + pj.vel.y * ry;
            fieldEnergy -= 0.5 * pi.charge * pj.charge * invR * (viDotVj + viDotR * vjDotR);
        }
    }
}
```

Then update the total energy computation to include field energy:
```js
const total = linearKE + spinKE + pe + fieldEnergy;
```

And update the DOM:
```js
this.dom.fieldE.textContent = fieldEnergy.toFixed(2);
```

**Step 4: Verify**

Load physsim, enable Coulomb + Magnetic forces. Create some charged spinning particles. Check that the Field Energy sub-row appears and shows values. Check that Total Energy drift% improves when magnetic forces are active compared to before this change.

**Step 5: Commit**

```bash
cd physsim
git add main.js index.html
git commit -m "feat: add Darwin Lagrangian field energy to energy bookkeeping

Compute O(v²/c²) velocity-dependent field energy correction using the Darwin
Lagrangian: U = -(1/2)Σ(q_i·q_j/r)[v_i·v_j + (v_i·r̂)(v_j·r̂)]. Display as
'Field' sub-row under Energy in Stats. Improves total energy conservation
accuracy when magnetic/velocity-dependent forces are active."
```

---

## Phase 2: Medium Features

---

### Task 5: G-A — Seed Persistence

**Files:**
- Create: `gerry/src/prng.js` (seeded PRNG)
- Modify: `gerry/src/hex-generator.js:16-182` (replace Math.random with seeded PRNG)
- Modify: `gerry/src/state.js:4-23` (add seed to state)
- Modify: `gerry/src/plans.js:28-42,44-62` (store/load seed)
- Modify: `gerry/main.js:129-146` (pass seed to generateHexes, URL hash support)

**Step 1: Create seeded PRNG module**

Create `gerry/src/prng.js`:

```js
// Mulberry32 — fast, high-quality 32-bit seeded PRNG
export function createPRNG(seed) {
    let s = seed | 0;
    return function() {
        s = (s + 0x6D2B79F5) | 0;
        let t = Math.imul(s ^ (s >>> 15), 1 | s);
        t = (t + Math.imul(t ^ (t >>> 7), 61 | t)) ^ t;
        return ((t ^ (t >>> 14)) >>> 0) / 4294967296;
    };
}

export function randomSeed() {
    return (Math.random() * 4294967296) >>> 0;
}
```

**Step 2: Add seed to state**

In `gerry/src/state.js`, add to `state` object (around line 22):
```js
seed: 0,
```

**Step 3: Replace Math.random in hex-generator**

In `gerry/src/hex-generator.js`:
- Import: `import { createPRNG } from './prng.js';`
- Change `generateHexes()` signature to `generateHexes(seed)`:
  ```js
  export function generateHexes(seed) {
      const rand = createPRNG(seed);
  ```
- Replace every `Math.random()` call (all ~18+ occurrences) with `rand()`. Use find-and-replace across the function body.

**Step 4: Update randomizeMap to use seeds**

In `gerry/main.js`:
- Import: `import { randomSeed } from './src/prng.js';`
- Update `randomizeMap()` (around line 129) to accept an optional seed parameter:
  ```js
  function randomizeMap(seed) {
      if (seed === undefined) seed = randomSeed();
      state.seed = seed;
      state.hexes.clear();
      // ... existing code ...
      generateHexes(seed);
      // ... rest unchanged ...
  }
  ```

**Step 5: Store seed in plans**

In `gerry/src/plans.js`, in `savePlan()` (around line 31-35), add `seed` to the plan object:
```js
const plan = {
    name,
    seed: state.seed,
    hexAssignments,
    timestamp: Date.now()
};
```

In `loadPlan()` (around line 44-62), before restoring assignments, regenerate the map if the plan has a seed:
```js
if (plan.seed !== undefined) {
    // Regenerate the exact same map from the stored seed
    // This requires calling randomizeMap with the seed, but without
    // the final pushUndoSnapshot (loadPlan will do its own)
    state.seed = plan.seed;
    state.hexes.clear();
    hexElements.clear();
    generateHexes(plan.seed);
    // recalculate targetPop, renderMap, etc.
}
```

Note: This requires refactoring `randomizeMap` to expose the map-generation part separately, or accepting that `loadPlan` calls a trimmed version.

**Step 6: Add URL hash support**

In `gerry/main.js`, in `init()` (around line 393), before `generateHexes()`:
```js
const hashSeed = parseInt(location.hash.replace('#seed=', ''), 10);
const initialSeed = Number.isFinite(hashSeed) ? hashSeed : randomSeed();
state.seed = initialSeed;
generateHexes(initialSeed);
```

After `randomizeMap()`, update the URL hash:
```js
history.replaceState(null, '', '#seed=' + state.seed);
```

**Step 7: Verify**

1. Load gerry, note the seed in the URL hash
2. Copy URL, open in new tab — should generate identical map
3. Save a plan, randomize, load plan — should restore exact same map + assignments
4. Plans without seeds (old format) should still load assignments on current map

**Step 8: Commit**

```bash
cd gerry
git add src/prng.js src/hex-generator.js src/state.js src/plans.js main.js
git commit -m "feat: seeded PRNG for reproducible maps and plan persistence

Replace Math.random() with mulberry32 seeded PRNG. Store seed in state,
saved plans, and URL hash (#seed=N). Loading a plan with a seed regenerates
the exact same map before applying district assignments. Old plans without
seeds remain backward-compatible."
```

---

### Task 6: B-K — Metabolite Time Series Graphs

**Files:**
- Create: `biosim/src/sparkline.js` (sparkline rendering module)
- Modify: `biosim/src/state.js:26-43` (add history buffers)
- Modify: `biosim/main.js` (sample values on interval)
- Modify: `biosim/src/dashboard.js` (render sparklines)
- Modify: `biosim/index.html:251-283` (add canvas elements)

**Step 1: Create sparkline module**

Create `biosim/src/sparkline.js`:

```js
const HISTORY_LEN = 300; // 300 samples at 5Hz = 60 seconds

export function createHistory() {
    return { data: new Float32Array(HISTORY_LEN), head: 0, count: 0 };
}

export function pushSample(h, value) {
    h.data[h.head] = value;
    h.head = (h.head + 1) % HISTORY_LEN;
    if (h.count < HISTORY_LEN) h.count++;
}

export function drawSparkline(ctx, h, w, hh, color) {
    if (h.count < 2) return;
    ctx.clearRect(0, 0, w, hh);

    ctx.beginPath();
    ctx.strokeStyle = color;
    ctx.lineWidth = 1.5;

    for (let i = 0; i < h.count; i++) {
        const idx = (h.head - h.count + i + HISTORY_LEN) % HISTORY_LEN;
        const x = (i / (HISTORY_LEN - 1)) * w;
        const y = hh - h.data[idx] * hh; // 0-1 normalized → canvas y
        if (i === 0) ctx.moveTo(x, y);
        else ctx.lineTo(x, y);
    }
    ctx.stroke();

    // "Now" line at right edge of data
    const nowX = (h.count / HISTORY_LEN) * w;
    ctx.setLineDash([2, 2]);
    ctx.strokeStyle = color + '66';
    ctx.beginPath();
    ctx.moveTo(nowX, 0);
    ctx.lineTo(nowX, hh);
    ctx.stroke();
    ctx.setLineDash([]);
}
```

**Step 2: Add history buffers to state**

In `biosim/src/state.js`, import and add histories:

```js
import { createHistory } from './sparkline.js';

// Add after store definition:
export const histories = {
    atp: createHistory(),
    nadh: createHistory(),
    nadph: createHistory(),
    fadh2: createHistory(),
    gradient: createHistory(),
};
```

In `resetState()`, reset histories:
```js
Object.values(histories).forEach(h => { h.head = 0; h.count = 0; h.data.fill(0); });
```

**Step 3: Add canvas elements to HTML**

In `biosim/index.html`, inside each metabolite card (e.g., `mc-atp` around line 254-260), after the existing bar, add a sparkline canvas:

```html
<canvas class="sparkline" id="spark-atp" width="160" height="36"></canvas>
```

Repeat for `spark-nadh`, `spark-nadph`, `spark-fadh2`. Add a gradient sparkline after the proton gradient stat row:
```html
<canvas class="sparkline" id="spark-gradient" width="160" height="36"></canvas>
```

**Step 4: Add CSS for sparklines**

In `biosim/styles.css`:
```css
.sparkline { display: block; width: 100%; height: 36px; margin-top: 4px; border-radius: 4px; background: var(--bg-elevated); }
```

**Step 5: Sample values in main loop**

In `biosim/main.js`, add a sample timer and sampling logic:

```js
import { pushSample } from './src/sparkline.js';
import { histories } from './src/state.js';

let sparkTimer = 0;
// In the main loop, after updateDashboard():
sparkTimer += dt;
if (sparkTimer > 0.2) { // 5Hz sampling
    sparkTimer = 0;
    pushSample(histories.atp, store.atp / store.totalAtpAdp);
    pushSample(histories.nadh, store.nadh / store.totalNad);
    pushSample(histories.nadph, store.nadph / store.totalNadp);
    pushSample(histories.fadh2, store.fadh2 / store.totalFad);
    pushSample(histories.gradient, Math.min(1, store.protonGradient / 40)); // normalize to 0-1
}
```

**Step 6: Render sparklines in dashboard**

In `biosim/src/dashboard.js`, add sparkline rendering to `updateDashboard()`:

```js
import { drawSparkline } from './sparkline.js';
import { histories } from './state.js';

// Cache canvas contexts once:
// (in initDashboard or lazily)
const sparkCtx = {};
function getSparkCtx(id) {
    if (!sparkCtx[id]) {
        const c = document.getElementById(id);
        if (c) sparkCtx[id] = c.getContext('2d');
    }
    return sparkCtx[id];
}

// In updateDashboard(), at the end:
const atpCtx = getSparkCtx('spark-atp');
if (atpCtx) drawSparkline(atpCtx, histories.atp, 160, 36, getComputedStyle(document.documentElement).getPropertyValue('--palette-orange').trim());
// Repeat for nadh (blue), nadph (green), fadh2 (rose), gradient (red)
```

**Step 7: Verify**

Run biosim with autoplay. Watch sparklines populate over 60 seconds. Verify they show oscillatory behavior as pathways cycle. Check that reset clears the graphs.

**Step 8: Commit**

```bash
cd biosim
git add src/sparkline.js src/state.js src/dashboard.js main.js index.html styles.css
git commit -m "feat: add metabolite time series sparkline graphs

Track cofactor ratios (ATP, NADH, NADPH, FADH2) and proton gradient over
60 seconds at 5Hz. Render as sparkline canvas graphs in the Stats tab.
Reveals oscillatory behavior, steady states, and perturbation responses."
```

---

### Task 7: P-J — Gravitational Potential Heatmap

**Files:**
- Create: `physsim/src/heatmap.js` (potential field computation + rendering)
- Modify: `physsim/src/renderer.js:44-93` (draw heatmap before particles)
- Modify: `physsim/src/ui.js:119-131` (add toggle)
- Modify: `physsim/main.js:11-76` (add heatmap instance to sim)
- Modify: `physsim/index.html:228-248` (add checkbox in Display tab)

**Step 1: Create heatmap module**

Create `physsim/src/heatmap.js`:

```js
import { SOFTENING_SQ } from './config.js';

const GRID_SIZE = 48;   // 48×48 grid
const UPDATE_INTERVAL = 6; // update every 6 frames

export default class Heatmap {
    constructor() {
        this.enabled = false;
        this.canvas = document.createElement('canvas');
        this.canvas.width = GRID_SIZE;
        this.canvas.height = GRID_SIZE;
        this.ctx = this.canvas.getContext('2d');
        this.frameCount = 0;
        this.potential = new Float32Array(GRID_SIZE * GRID_SIZE);
    }

    update(particles, camera, width, height) {
        if (!this.enabled) return;
        if (++this.frameCount % UPDATE_INTERVAL !== 0) return;

        const zoom = camera.zoom;
        const cx = camera.x, cy = camera.y;
        const halfW = width / (2 * zoom), halfH = height / (2 * zoom);
        const left = cx - halfW, top = cy - halfH;
        const cellW = (2 * halfW) / GRID_SIZE;
        const cellH = (2 * halfH) / GRID_SIZE;

        let minPhi = 0, maxPhi = 0;

        for (let gy = 0; gy < GRID_SIZE; gy++) {
            for (let gx = 0; gx < GRID_SIZE; gx++) {
                const wx = left + (gx + 0.5) * cellW;
                const wy = top + (gy + 0.5) * cellH;
                let phi = 0;

                for (let i = 0; i < particles.length; i++) {
                    const p = particles[i];
                    const dx = wx - p.pos.x, dy = wy - p.pos.y;
                    const rSq = dx * dx + dy * dy + SOFTENING_SQ;
                    const invR = 1 / Math.sqrt(rSq);
                    phi -= p.mass * invR;        // gravitational
                    phi += p.charge * invR;      // electrostatic (repulsive same-sign)
                }

                this.potential[gy * GRID_SIZE + gx] = phi;
                if (phi < minPhi) minPhi = phi;
                if (phi > maxPhi) maxPhi = phi;
            }
        }

        // Render to offscreen canvas with diverging colormap
        const imgData = this.ctx.createImageData(GRID_SIZE, GRID_SIZE);
        const range = Math.max(Math.abs(minPhi), Math.abs(maxPhi)) || 1;

        for (let i = 0; i < GRID_SIZE * GRID_SIZE; i++) {
            const norm = this.potential[i] / range; // -1 to 1
            const idx = i * 4;
            if (norm < 0) {
                // Negative (gravity well) → blue
                imgData.data[idx] = 40;
                imgData.data[idx + 1] = 80;
                imgData.data[idx + 2] = 200;
                imgData.data[idx + 3] = Math.round(Math.abs(norm) * 80);
            } else {
                // Positive (repulsive) → red
                imgData.data[idx] = 200;
                imgData.data[idx + 1] = 60;
                imgData.data[idx + 2] = 40;
                imgData.data[idx + 3] = Math.round(norm * 80);
            }
        }
        this.ctx.putImageData(imgData, 0, 0);
    }

    draw(ctx, camera, width, height) {
        if (!this.enabled) return;
        // Draw the low-res heatmap scaled to fill viewport, before camera transform
        ctx.save();
        ctx.imageSmoothingEnabled = true;
        ctx.imageSmoothingQuality = 'high';
        ctx.drawImage(this.canvas, 0, 0, width, height);
        ctx.restore();
    }
}
```

**Step 2: Add to Simulation**

In `physsim/main.js`, import and instantiate:
```js
import Heatmap from './src/heatmap.js';
// In constructor:
this.heatmap = new Heatmap();
```

In the main loop, before `renderer.render()`:
```js
this.heatmap.update(this.particles, this.camera, this.width, this.height);
```

**Step 3: Draw heatmap in renderer**

In `physsim/src/renderer.js`, in `render()` (around line 44), after clearing the canvas but before applying the camera transform (before line 48):

```js
// Draw heatmap in screen space (before camera transform)
if (sim.heatmap) sim.heatmap.draw(ctx, sim.camera, this.width, this.height);
```

This requires passing `sim` reference to the renderer, or having the heatmap draw call happen in `main.js` between clear and the camera-transformed render.

**Step 4: Add checkbox to Display tab**

In `physsim/index.html`, in the Display tab (around line 241), add:
```html
<label><input type="checkbox" id="potentialToggle"> Potential Field</label>
```

**Step 5: Wire toggle in UI**

In `physsim/src/ui.js`, in the display toggle section (around line 119-131):
```js
document.getElementById('potentialToggle')?.addEventListener('change', e => {
    sim.heatmap.enabled = e.target.checked;
});
```

**Step 6: Verify**

Load physsim, place a heavy particle. Enable "Potential Field" in Display tab. Should see a blue gradient around the particle showing the gravity well. Add a charged particle — should see red regions where charges repel.

**Step 7: Commit**

```bash
cd physsim
git add src/heatmap.js src/renderer.js src/ui.js main.js index.html
git commit -m "feat: add gravitational/electrostatic potential heatmap overlay

Compute combined gravitational + electrostatic potential on a 48x48 grid.
Render as a diverging blue (wells) / red (repulsive) heatmap behind particles.
Updated every 6 frames for performance. Toggle in Display tab."
```

---

### Task 8: P-I — Phase Space Plot

**Files:**
- Create: `physsim/src/phase-plot.js` (phase space overlay)
- Modify: `physsim/src/renderer.js` (draw phase plot overlay)
- Modify: `physsim/src/ui.js` (add toggle)
- Modify: `physsim/main.js` (update phase plot)
- Modify: `physsim/index.html` (add checkbox)

**Step 1: Create phase plot module**

Create `physsim/src/phase-plot.js`:

```js
const BUFFER_LEN = 500;
const PLOT_SIZE = 180;
const MARGIN = 24;

export default class PhasePlot {
    constructor() {
        this.enabled = false;
        this.canvas = document.createElement('canvas');
        this.canvas.width = PLOT_SIZE;
        this.canvas.height = PLOT_SIZE;
        this.ctx = this.canvas.getContext('2d');
        this.rBuf = new Float32Array(BUFFER_LEN);
        this.vrBuf = new Float32Array(BUFFER_LEN);
        this.head = 0;
        this.count = 0;
        this.trackedId = -1;
    }

    update(particles, selectedParticle) {
        if (!this.enabled || !selectedParticle) return;

        // Track distance to most massive body (or COM)
        const sel = selectedParticle;
        if (sel.id !== this.trackedId) {
            this.trackedId = sel.id;
            this.head = 0;
            this.count = 0;
        }

        let refX = 0, refY = 0, refVx = 0, refVy = 0, maxM = 0;
        for (const p of particles) {
            if (p === sel) continue;
            if (p.mass > maxM) {
                maxM = p.mass;
                refX = p.pos.x; refY = p.pos.y;
                refVx = p.vel.x; refVy = p.vel.y;
            }
        }

        const dx = sel.pos.x - refX, dy = sel.pos.y - refY;
        const r = Math.sqrt(dx * dx + dy * dy) || 1;
        const rx = dx / r, ry = dy / r;
        const dvx = sel.vel.x - refVx, dvy = sel.vel.y - refVy;
        const vr = dvx * rx + dvy * ry; // radial velocity

        this.rBuf[this.head] = r;
        this.vrBuf[this.head] = vr;
        this.head = (this.head + 1) % BUFFER_LEN;
        if (this.count < BUFFER_LEN) this.count++;
    }

    draw(ctx, width, height, isLight) {
        if (!this.enabled || this.count < 2) return;

        // Auto-scale
        let rMin = Infinity, rMax = -Infinity, vrMin = Infinity, vrMax = -Infinity;
        for (let i = 0; i < this.count; i++) {
            const idx = (this.head - this.count + i + BUFFER_LEN) % BUFFER_LEN;
            const r = this.rBuf[idx], vr = this.vrBuf[idx];
            if (r < rMin) rMin = r; if (r > rMax) rMax = r;
            if (vr < vrMin) vrMin = vr; if (vr > vrMax) vrMax = vr;
        }
        const rRange = (rMax - rMin) || 1;
        const vrRange = (vrMax - vrMin) || 1;

        // Draw to offscreen canvas
        const c = this.ctx;
        const ps = PLOT_SIZE;
        c.clearRect(0, 0, ps, ps);

        // Background
        c.fillStyle = isLight ? '#FCF7F244' : '#0C0B0988';
        c.fillRect(0, 0, ps, ps);

        // Axes
        c.strokeStyle = isLight ? '#1A161233' : '#E8DED433';
        c.lineWidth = 0.5;
        c.beginPath();
        c.moveTo(MARGIN, 0); c.lineTo(MARGIN, ps);
        c.moveTo(0, ps - MARGIN); c.lineTo(ps, ps - MARGIN);
        c.stroke();

        // Labels
        c.fillStyle = isLight ? '#1A161288' : '#E8DED488';
        c.font = '9px Noto Sans Mono';
        c.fillText('r', ps - 12, ps - MARGIN + 12);
        c.fillText('vᵣ', MARGIN - 18, 12);

        // Trajectory
        c.beginPath();
        c.lineWidth = 1.2;
        for (let i = 0; i < this.count; i++) {
            const idx = (this.head - this.count + i + BUFFER_LEN) % BUFFER_LEN;
            const x = MARGIN + ((this.rBuf[idx] - rMin) / rRange) * (ps - MARGIN - 4);
            const y = (ps - MARGIN) - ((this.vrBuf[idx] - vrMin) / vrRange) * (ps - MARGIN - 4);
            const alpha = (i / this.count);
            if (i === 0) {
                c.moveTo(x, y);
                c.strokeStyle = `rgba(254,59,1,${alpha * 0.8})`;
            } else {
                c.lineTo(x, y);
            }
        }
        c.strokeStyle = '#FE3B01CC';
        c.stroke();

        // Current point
        const lastIdx = (this.head - 1 + BUFFER_LEN) % BUFFER_LEN;
        const cx = MARGIN + ((this.rBuf[lastIdx] - rMin) / rRange) * (ps - MARGIN - 4);
        const cy = (ps - MARGIN) - ((this.vrBuf[lastIdx] - vrMin) / vrRange) * (ps - MARGIN - 4);
        c.fillStyle = '#FE3B01';
        c.beginPath();
        c.arc(cx, cy, 3, 0, Math.PI * 2);
        c.fill();

        // Composite onto main canvas (bottom-left corner)
        ctx.drawImage(this.canvas, 12, height - ps - 60);
    }
}
```

**Step 2: Add to sim, wire toggle, add checkbox**

Follow the same pattern as the heatmap (Task 7): import in `main.js`, add `this.phasePlot = new PhasePlot()`, call `update()` in loop, call `draw()` after particles, add checkbox `#phaseToggle` in Display tab, wire in `ui.js`.

**Step 3: Verify**

Load physsim, load Solar System preset. Click a planet to select it. Enable "Phase Space" in Display tab. Should see an elliptical trajectory in (r, vr) space for a stable orbit.

**Step 4: Commit**

```bash
cd physsim
git add src/phase-plot.js src/renderer.js src/ui.js main.js index.html
git commit -m "feat: add phase space plot overlay for selected particle

Track (r, v_r) relative to most massive body in a 500-point circular buffer.
Render as a 180x180 overlay with auto-scaling axes. Shows orbital structure,
chaos, and resonances. Toggle in Display tab, visible when a particle is selected."
```

---

### Task 9: P-G — Correct Spin-Orbit Coupling

**Files:**
- Modify: `physsim/src/physics.js:466-534` (compute B_z gradient, apply spin torque)
- Modify: `physsim/src/config.js` (add SPIN_ORBIT_ENABLED flag if needed)
- Modify: `physsim/src/ui.js:102-117` (conditionally gate on magnetic + relativity)

**Step 1: Compute B_z gradient in pair force**

In `physsim/src/physics.js`, in `_pairForce()` (around line 466), the function already computes `p.Bz` from each source particle. To compute the gradient, we need to store `dBz/dx` and `dBz/dy` per particle.

Add to Particle (in `particle.js`):
```js
this.dBzdx = 0;
this.dBzdy = 0;
```

In `_resetForces()` (around line 196), add:
```js
p.dBzdx = 0;
p.dBzdy = 0;
```

In `_pairForce()`, after the Bz accumulation (around line 517), compute the gradient:
```js
// B_z = q_s * cross(v_s, r̂) / r²  →  ∂B_z/∂x_i, ∂B_z/∂y_i
// For a point source B_z ~ cross/(r²), gradient ~ d(cross/r²)/dr
if (magneticEnabled) {
    const Bz_contribution = sCharge * crossSV * invR * invRSq;
    // ∂B_z/∂r ~ -3 * B_z / r  (from d/dr of 1/r³ behavior)
    const dBzdr = -3 * Bz_contribution * invR;
    p.dBzdx += dBzdr * dx * invR; // dx/r = r̂_x
    p.dBzdy += dBzdr * dy * invR;
}
```

**Step 2: Apply spin torque**

In the substep loop (around line 112-118, after the second half-kick), add the spin-orbit energy exchange:

```js
// Spin-orbit coupling: dE_spin/dt = -μ · (v · ∇B_z)
if (magneticEnabled && relativityEnabled) {
    const mu = MAG_MOMENT_K * p.charge * p.angVel * p.radius * p.radius;
    const vDotGradB = p.vel.x * p.dBzdx + p.vel.y * p.dBzdy;
    const dEspin = -mu * vDotGradB * dtSub;
    const I = INERTIA_K * p.mass * p.radius * p.radius;
    if (Math.abs(p.angVel) > 1e-10 && Math.abs(I) > 1e-10) {
        p.spin += dEspin / (I * p.angVel);
        // Re-derive angVel from spin
        const sr = p.spin * p.radius;
        p.angVel = p.spin / Math.sqrt(1 + sr * sr);
    }
}
```

**Step 3: Verify**

Create a spinning charged particle near another charged particle with different spin. Watch angular momentum transfer between spin and orbital motion. Check that total angular momentum (orbital + spin) is better conserved than before.

**Step 4: Commit**

```bash
cd physsim
git add src/physics.js src/particle.js
git commit -m "feat: add correct spin-orbit coupling via B-field gradient

Compute ∂B_z/∂x and ∂B_z/∂y from source particle contributions. Apply
spin torque dSpin/dt = -μ·(v·∇B_z)/(I·ω) to exchange energy between spin
and orbital motion. Conserves total angular momentum by construction.
Active when both Magnetic and Relativity toggles are on."
```

---

### Task 10: B-G — Organism Presets

**Files:**
- Create: `biosim/src/organisms.js` (preset definitions)
- Modify: `biosim/src/state.js` (add activeOrganism state)
- Modify: `biosim/src/ui.js` (add preset selector, lock toggles)
- Modify: `biosim/index.html` (add preset UI in Controls tab)
- Modify: `biosim/styles.css` (preset dialog/selector styling)

**Step 1: Create organism definitions**

Create `biosim/src/organisms.js`:

```js
export const ORGANISMS = {
    cyanobacterium: {
        name: 'Cyanobacterium',
        desc: 'Photosynthetic prokaryote — all pathways available',
        pathways: { glycolysis: true, ppp: true, calvin: true, krebs: true, betaox: true },
        environment: { light: true, oxygen: true },
        initialRatios: { atp: 0.9, nadh: 0.1, nadph: 0.1, fadh2: 0.1 },
    },
    animalCell: {
        name: 'Animal Cell',
        desc: 'Heterotrophic — no photosynthesis or carbon fixation',
        pathways: { glycolysis: true, ppp: true, calvin: false, krebs: true, betaox: true },
        environment: { light: false, oxygen: true },
        initialRatios: { atp: 0.9, nadh: 0.1, nadph: 0.1, fadh2: 0.1 },
        lockedReason: { calvin: 'Animal cells lack RuBisCO and photosystems' },
    },
    obligateAnaerobe: {
        name: 'Obligate Anaerobe',
        desc: 'Fermentation only — no oxidative phosphorylation',
        pathways: { glycolysis: true, ppp: false, calvin: false, krebs: false, betaox: false },
        environment: { light: false, oxygen: false },
        initialRatios: { atp: 0.5, nadh: 0.5, nadph: 0.1, fadh2: 0.1 },
        lockedReason: {
            ppp: 'Simplified anaerobe model',
            calvin: 'No photosynthetic machinery',
            krebs: 'O₂ required for oxidative metabolism',
            betaox: 'O₂ required for β-oxidation',
        },
    },
    plantChloroplast: {
        name: 'Plant Chloroplast',
        desc: 'Light reactions and carbon fixation only',
        pathways: { glycolysis: false, ppp: false, calvin: true, krebs: false, betaox: false },
        environment: { light: true, oxygen: true },
        initialRatios: { atp: 0.3, nadh: 0.1, nadph: 0.5, fadh2: 0.1 },
        lockedReason: {
            glycolysis: 'Chloroplast does not perform glycolysis',
            ppp: 'PPP occurs in cytoplasm, not chloroplast',
            krebs: 'Krebs cycle occurs in mitochondria',
            betaox: 'β-oxidation occurs in mitochondria',
        },
    },
    archaeon: {
        name: 'Archaeon',
        desc: 'Uses bacteriorhodopsin for light-driven proton pumping',
        pathways: { glycolysis: true, ppp: false, calvin: false, krebs: true, betaox: true },
        environment: { light: true, oxygen: false },
        initialRatios: { atp: 0.7, nadh: 0.2, nadph: 0.1, fadh2: 0.1 },
        lockedReason: {
            ppp: 'Simplified archaeal model',
            calvin: 'No photosynthetic carbon fixation',
        },
    },
};
```

**Step 2: Add activeOrganism to state**

In `biosim/src/state.js`:
```js
// In simState:
activeOrganism: 'cyanobacterium',
lockedPathways: {},
```

**Step 3: Add preset selector HTML**

In `biosim/index.html`, at the top of the Controls tab (before the pathway toggles, around line 160), add:

```html
<div class="ctrl-group">
    <div class="group-label">Organism</div>
    <select id="organism-select" class="organism-select">
        <option value="cyanobacterium">Cyanobacterium</option>
        <option value="animalCell">Animal Cell</option>
        <option value="obligateAnaerobe">Obligate Anaerobe</option>
        <option value="plantChloroplast">Plant Chloroplast</option>
        <option value="archaeon">Archaeon</option>
        <option value="custom">Custom</option>
    </select>
    <p class="organism-desc" id="organism-desc"></p>
</div>
```

**Step 4: Wire preset selector in UI**

In `biosim/src/ui.js`:
- Cache DOM: `dom.organismSelect = document.getElementById('organism-select');`
- Import: `import { ORGANISMS } from './organisms.js';`
- Bind change event:

```js
dom.organismSelect?.addEventListener('change', e => {
    const key = e.target.value;
    if (key === 'custom') {
        simState.lockedPathways = {};
        // unlock all toggles
        return;
    }
    const org = ORGANISMS[key];
    if (!org) return;

    simState.activeOrganism = key;
    simState.lockedPathways = org.lockedReason || {};

    // Apply pathway enables
    const pathwayMap = {
        glycolysis: dom.glycToggle,
        ppp: dom.pppToggle,
        calvin: dom.calvinToggle,
        krebs: dom.krebsToggle,
        betaox: dom.betaoxToggle,
    };
    for (const [pw, toggle] of Object.entries(pathwayMap)) {
        const enabled = org.pathways[pw];
        toggle.checked = enabled;
        toggle.dispatchEvent(new Event('change'));
        toggle.disabled = !enabled && org.lockedReason?.[pw];
        toggle.closest('.ctrl-row')?.classList.toggle('locked', !!org.lockedReason?.[pw]);
    }

    // Apply environment
    dom.lightToggle.checked = org.environment.light;
    dom.lightToggle.dispatchEvent(new Event('change'));
    dom.oxygenToggle.checked = org.environment.oxygen;
    dom.oxygenToggle.dispatchEvent(new Event('change'));

    // Reset metabolite pools with organism-specific ratios
    resetState();
    store.atp = Math.round(store.totalAtpAdp * org.initialRatios.atp);
    store.nadh = Math.round(store.totalNad * org.initialRatios.nadh);
    store.nadph = Math.round(store.totalNadp * org.initialRatios.nadph);
    store.fadh2 = Math.round(store.totalFad * org.initialRatios.fadh2);

    // Update description
    dom.organismDesc.textContent = org.desc;
});
```

**Step 5: Add CSS for locked toggles**

In `biosim/styles.css`:
```css
.organism-select { width: 100%; padding: 6px 10px; border-radius: var(--radius-sm); border: 1px solid var(--border); background: var(--bg-panel); color: var(--text); font-family: var(--font-body); font-size: 0.82rem; }
.organism-desc { font-size: 0.75rem; color: var(--text-muted); margin: 4px 0 0; }
.ctrl-row.locked { opacity: 0.4; pointer-events: none; }
```

**Step 6: Verify**

Switch between organism presets. Verify that:
- Animal Cell locks Calvin and hides photo ETC
- Obligate Anaerobe only allows glycolysis + fermentation
- Plant Chloroplast only allows Calvin + photo ETC
- Custom mode unlocks everything
- Metabolite ratios reset appropriately

**Step 7: Commit**

```bash
cd biosim
git add src/organisms.js src/state.js src/ui.js index.html styles.css
git commit -m "feat: add organism preset selector with pathway locking

Five organism presets (Cyanobacterium, Animal Cell, Obligate Anaerobe,
Plant Chloroplast, Archaeon) configure which pathways are available,
environment defaults, and initial metabolite ratios. Locked pathways
are greyed out with explanation. Custom mode preserves manual toggles."
```

---

## Phase 3: Major Features

---

### Task 11: P-A — Larmor Radiation

**Files:**
- Create: `physsim/src/photon.js` (photon entity + pool)
- Modify: `physsim/src/physics.js:76-145` (compute radiation, drain KE)
- Modify: `physsim/src/renderer.js` (draw photons)
- Modify: `physsim/src/config.js` (add radiation constants)
- Modify: `physsim/main.js` (photon array, toggle state, energy tracking)
- Modify: `physsim/src/ui.js` (radiation toggle)
- Modify: `physsim/index.html` (toggle HTML, stat row)

**Step 1: Add config constants**

In `physsim/src/config.js`:
```js
export const LARMOR_K = 1 / (6 * Math.PI); // q²a²/(6π) in natural units (c=G=1)
export const PHOTON_LIFETIME = 300;          // frames before despawn
export const RADIATION_THRESHOLD = 0.01;     // min energy per frame to emit visible photon
export const MAX_PHOTONS = 500;              // photon pool cap
```

**Step 2: Create photon entity**

Create `physsim/src/photon.js`:

```js
import Vec2 from './vec2.js';

export default class Photon {
    constructor(x, y, vx, vy, energy) {
        this.pos = new Vec2(x, y);
        this.vel = new Vec2(vx, vy); // direction, |v| = 1 (c)
        this.energy = energy;
        this.lifetime = 0;
        this.alive = true;
    }

    update(dt) {
        this.pos.x += this.vel.x * dt;
        this.pos.y += this.vel.y * dt;
        this.lifetime++;
    }
}
```

**Step 3: Add radiation computation to physics**

In `physsim/src/physics.js`, after the second half-kick (around line 118), add radiation loss:

```js
if (radiationEnabled && Math.abs(p.charge) > 0) {
    const ax = p.force.x / p.mass;
    const ay = p.force.y / p.mass;
    const aSq = ax * ax + ay * ay;
    const gamma = Math.sqrt(1 + p.w.x * p.w.x + p.w.y * p.w.y);
    const gammaCube = gamma * gamma * gamma;
    const P = LARMOR_K * p.charge * p.charge * aSq; // non-relativistic Larmor
    const dE = P * dtSub / gammaCube; // relativistic correction

    if (dE > 0) {
        // Reduce |w| to drain kinetic energy
        const wMag = Math.sqrt(p.w.x * p.w.x + p.w.y * p.w.y);
        if (wMag > 1e-10) {
            const KE = (gamma - 1) * p.mass;
            const newKE = Math.max(0, KE - dE);
            const newGamma = newKE / p.mass + 1;
            const newWSq = newGamma * newGamma - 1;
            const newWMag = Math.sqrt(Math.max(0, newWSq));
            const scale = newWMag / wMag;
            p.w.x *= scale;
            p.w.y *= scale;
        }
        sim.totalRadiated += dE;

        // Spawn photon if energy exceeds threshold
        if (dE > RADIATION_THRESHOLD && sim.photons.length < MAX_PHOTONS) {
            const angle = Math.atan2(ay, ax) + Math.PI + (Math.random() - 0.5) * 1.0;
            sim.photons.push(new Photon(
                p.pos.x, p.pos.y,
                Math.cos(angle), Math.sin(angle),
                dE
            ));
        }
    }
}
```

**Step 4: Update photons in main loop**

In `physsim/main.js`, in the loop (around line 225):
```js
// Update photons
for (let i = this.photons.length - 1; i >= 0; i--) {
    const ph = this.photons[i];
    ph.update(dt);
    if (ph.lifetime > PHOTON_LIFETIME) {
        this.photons.splice(i, 1);
    }
}
```

**Step 5: Render photons**

In `physsim/src/renderer.js`, add a `drawPhotons(ctx, photons, isLight)` method:

```js
drawPhotons(ctx, photons, isLight) {
    for (const ph of photons) {
        const alpha = 1 - ph.lifetime / PHOTON_LIFETIME;
        const size = 1.5 + ph.energy * 20;
        ctx.fillStyle = isLight ? `rgba(254,59,1,${alpha * 0.6})` : `rgba(255,220,100,${alpha * 0.8})`;
        ctx.beginPath();
        ctx.arc(ph.pos.x, ph.pos.y, size, 0, Math.PI * 2);
        ctx.fill();
        if (!isLight) {
            ctx.shadowBlur = size * 3;
            ctx.shadowColor = `rgba(255,220,100,${alpha * 0.5})`;
            ctx.fill();
            ctx.shadowBlur = 0;
        }
    }
}
```

Call this in `render()` after `drawParticles`.

**Step 6: Add toggle, stat row, energy tracking**

- HTML: add radiation toggle (same pattern as other force toggles), add "Radiated" stat sub-row with id `radiatedE`
- UI: wire toggle → `sim.physics.radiationEnabled`
- main.js: add `this.totalRadiated = 0`, display in computeEnergy, include in total
- Info tip for radiation explaining Larmor formula

**Step 7: Verify**

Enable Coulomb + Radiation. Create two opposite-charge particles that orbit each other. Watch them spiral inward as they radiate energy. Photons should fly outward. Radiated energy counter should increase. Total energy (including radiated) should be better conserved.

**Step 8: Commit**

```bash
cd physsim
git add src/photon.js src/physics.js src/renderer.js src/config.js src/ui.js main.js index.html
git commit -m "feat: add Larmor radiation — accelerating charges emit photons

Compute radiated power P = q²a²/(6π) with relativistic γ³ correction.
Drain kinetic energy from radiating particles by reducing |w|. Spawn
visual photon particles flying outward. Creates orbital decay for
charge-charge systems. New 'Radiation' force toggle and 'Radiated'
energy stat sub-row. Photon pool capped at 500."
```

---

### Task 12: P-B — Tidal Forces / Roche Limit

**Files:**
- Modify: `physsim/src/physics.js` (compute tidal stress, trigger breakup)
- Modify: `physsim/src/config.js` (add tidal constants)
- Modify: `physsim/main.js` (handle fragment spawning)
- Modify: `physsim/src/renderer.js` (breakup flash)
- Modify: `physsim/src/ui.js` (toggle)
- Modify: `physsim/index.html` (toggle HTML)

**Step 1: Add config**

In `physsim/src/config.js`:
```js
export const TIDAL_STRENGTH = 2.0;
export const MIN_FRAGMENT_MASS = 2; // don't fragment below this mass
export const FRAGMENT_COUNT = 3;    // split into 3 pieces
```

**Step 2: Add tidal computation to physics**

In `physsim/src/physics.js`, add a new method `checkTidalBreakup(particles)`:

```js
checkTidalBreakup(particles) {
    if (!this.tidalEnabled) return [];
    const fragments = [];

    for (const p of particles) {
        if (p.mass < MIN_FRAGMENT_MASS * FRAGMENT_COUNT) continue;

        // Find strongest gravitational neighbor
        let maxTidal = 0;
        for (const other of particles) {
            if (other === p) continue;
            const dx = other.pos.x - p.pos.x, dy = other.pos.y - p.pos.y;
            const rSq = dx * dx + dy * dy;
            const r = Math.sqrt(rSq);
            const tidalAccel = TIDAL_STRENGTH * other.mass * p.radius / (r * rSq);
            if (tidalAccel > maxTidal) maxTidal = tidalAccel;
        }

        // Self-gravity at surface: g_self = m / r²
        const selfGravity = p.mass / (p.radius * p.radius);

        if (maxTidal > selfGravity) {
            fragments.push(p);
        }
    }

    return fragments;
}
```

Call this after collisions in the substep loop. For each fragment, replace the particle with `FRAGMENT_COUNT` smaller particles conserving total mass, charge, momentum, and angular momentum.

**Step 3: Implement fragmentation in main.js**

```js
// After physics.update():
const toFragment = this.physics.checkTidalBreakup(this.particles);
for (const p of toFragment) {
    const idx = this.particles.indexOf(p);
    if (idx === -1) continue;
    this.particles.splice(idx, 1);

    const n = FRAGMENT_COUNT;
    const fragMass = p.mass / n;
    const fragCharge = p.charge / n;
    const fragRadius = Math.cbrt(fragMass);

    for (let i = 0; i < n; i++) {
        const angle = (2 * Math.PI * i) / n;
        const offset = p.radius * 1.5;
        const fx = p.pos.x + Math.cos(angle) * offset;
        const fy = p.pos.y + Math.sin(angle) * offset;
        // Each fragment gets parent velocity + tangential kick from spin
        const tangVx = -Math.sin(angle) * p.angVel * offset;
        const tangVy = Math.cos(angle) * p.angVel * offset;
        this.addParticle(fx, fy, p.vel.x + tangVx, p.vel.y + tangVy, {
            mass: fragMass, charge: fragCharge, spin: p.spin
        });
    }
}
```

**Step 4: Add toggle, info tip**

Same pattern as other force toggles. Info tip: "Tidal forces arise from differential gravity across a body's diameter. When tidal stress exceeds self-gravity (Roche limit), the body breaks apart."

**Step 5: Verify**

Create a very massive particle (mass 200) and a small particle (mass 10). Move the small one close. With tidal forces enabled, it should break apart when it gets too close.

**Step 6: Commit**

```bash
cd physsim
git add src/physics.js src/config.js main.js src/renderer.js src/ui.js index.html
git commit -m "feat: add tidal forces and Roche limit breakup

Compute tidal stress from strongest gravitational neighbor. When tidal
acceleration exceeds self-gravity, fragment the particle into 3 pieces
conserving mass, charge, momentum, and angular momentum. New 'Tidal'
toggle in Settings. Minimum fragment mass prevents infinite splitting."
```

---

### Task 13: P-C — Radiation Pressure

**Files:**
- Modify: `physsim/src/physics.js` (photon-particle collision)
- Modify: `physsim/main.js` (collision check in photon update loop)

**Step 1: Add photon-particle collision detection**

In `physsim/main.js`, in the photon update loop (from Task 11), add collision checking:

```js
for (let i = this.photons.length - 1; i >= 0; i--) {
    const ph = this.photons[i];
    ph.update(dt);

    // Radiation pressure: check collision with particles
    if (this.physics.radiationEnabled) {
        for (const p of this.particles) {
            const dx = ph.pos.x - p.pos.x, dy = ph.pos.y - p.pos.y;
            const distSq = dx * dx + dy * dy;
            if (distSq < p.radius * p.radius) {
                // Absorb photon — transfer momentum E/c = E (natural units)
                const impulse = ph.energy / p.mass;
                p.w.x += ph.vel.x * impulse;
                p.w.y += ph.vel.y * impulse;
                ph.alive = false;
                break;
            }
        }
    }

    if (!ph.alive || ph.lifetime > PHOTON_LIFETIME) {
        this.photons.splice(i, 1);
    }
}
```

**Step 2: Verify**

Create a strongly radiating charged particle near a neutral massive particle. Photons hitting the massive particle should push it slightly away from the radiation source.

**Step 3: Commit**

```bash
cd physsim
git add main.js
git commit -m "feat: add radiation pressure from Larmor photons

Photons that collide with particles transfer momentum p = E/c (= E in
natural units) to the target's proper velocity. Completes the radiation
feedback loop: acceleration → photon emission → momentum transfer."
```

---

### Task 14: G-E — Multi-Round Election Simulation

**Files:**
- Create: `gerry/src/election-sim.js` (simulation engine + overlay renderer)
- Modify: `gerry/main.js` (button wiring, overlay management)
- Modify: `gerry/index.html` (button HTML, overlay container)
- Modify: `gerry/styles.css` (overlay styling)

**Step 1: Create election simulation module**

Create `gerry/src/election-sim.js`:

```js
import { CONFIG } from './config.js';
import { state } from './state.js';

// Box-Muller transform for normal distribution
function gaussRandom(mean, stddev) {
    const u1 = Math.random(), u2 = Math.random();
    return mean + stddev * Math.sqrt(-2 * Math.log(u1)) * Math.cos(2 * Math.PI * u2);
}

export function simulateElections(numElections, swingSigma) {
    const parties = ['red', 'blue', 'yellow'];
    const results = { red: [], blue: [], yellow: [] };

    for (let e = 0; e < numElections; e++) {
        // Correlated national swing per party
        const nationalSwing = {};
        for (const p of parties) nationalSwing[p] = gaussRandom(0, swingSigma);

        const seats = { red: 0, blue: 0, yellow: 0 };

        for (let d = 1; d <= CONFIG.numDistricts; d++) {
            const dist = state.districts[d];
            if (!dist || dist.hexes.length === 0) continue;

            // Apply swing to vote shares
            const totalVotes = dist.votes.red + dist.votes.blue + dist.votes.yellow;
            if (totalVotes === 0) continue;

            const swungShares = {};
            let shareSum = 0;
            for (const p of parties) {
                const baseShare = dist.votes[p] / totalVotes;
                const localNoise = gaussRandom(0, swingSigma * 0.3);
                swungShares[p] = Math.max(0.01, baseShare + nationalSwing[p] + localNoise);
                shareSum += swungShares[p];
            }
            // Normalize
            for (const p of parties) swungShares[p] /= shareSum;

            // Determine winner
            let winner = 'red', maxShare = 0;
            for (const p of parties) {
                if (swungShares[p] > maxShare) { maxShare = swungShares[p]; winner = p; }
            }
            seats[winner]++;
        }

        for (const p of parties) results[p].push(seats[p]);
    }

    return results;
}

export function renderHistogram(canvas, results, numDistricts) {
    const ctx = canvas.getContext('2d');
    const w = canvas.width, h = canvas.height;
    ctx.clearRect(0, 0, w, h);

    const parties = ['red', 'blue', 'yellow'];
    const colors = {
        red: getComputedStyle(document.documentElement).getPropertyValue('--party-red').trim(),
        blue: getComputedStyle(document.documentElement).getPropertyValue('--party-blue').trim(),
        yellow: getComputedStyle(document.documentElement).getPropertyValue('--party-yellow').trim(),
    };

    const n = results.red.length;
    const barGroupW = w / (numDistricts + 1);

    for (const p of parties) {
        const counts = new Array(numDistricts + 1).fill(0);
        for (const s of results[p]) counts[s]++;
        const maxCount = Math.max(...counts, 1);

        const barW = barGroupW / 4;
        const offset = parties.indexOf(p) * barW;

        ctx.fillStyle = colors[p] + 'AA';
        for (let s = 0; s <= numDistricts; s++) {
            const barH = (counts[s] / maxCount) * (h - 30);
            const x = s * barGroupW + offset + 2;
            ctx.fillRect(x, h - 20 - barH, barW - 1, barH);
        }
    }

    // X-axis labels
    ctx.fillStyle = getComputedStyle(document.documentElement).getPropertyValue('--text-muted').trim();
    ctx.font = '9px Noto Sans Mono';
    ctx.textAlign = 'center';
    for (let s = 0; s <= numDistricts; s++) {
        ctx.fillText(s, s * barGroupW + barGroupW / 2, h - 4);
    }
    ctx.fillText('Seats →', w / 2, h - 4);

    // Stats text
    const meanSeats = {};
    for (const p of parties) {
        const sum = results[p].reduce((a, b) => a + b, 0);
        meanSeats[p] = (sum / n).toFixed(1);
    }
    ctx.textAlign = 'left';
    ctx.font = '11px Noto Sans';
    let y = 14;
    for (const p of parties) {
        ctx.fillStyle = colors[p];
        ctx.fillText(`${p[0].toUpperCase() + p.slice(1)}: μ=${meanSeats[p]} seats`, 8, y);
        y += 16;
    }
}
```

**Step 2: Add UI elements**

In `gerry/index.html`, add a button to the toolbar (around line 145, near randomize):
```html
<button class="tool-btn" id="simulate-btn" aria-label="Simulate Elections" title="Simulate Elections">
    <svg viewBox="0 0 24 24"><path d="M3 3v18h18M7 14l4-4 4 4 4-4" fill="none" stroke="currentColor" stroke-width="2"/></svg>
</button>
```

Add an overlay container before the closing `</main>`:
```html
<div id="election-overlay" class="election-overlay" hidden>
    <div class="election-panel glass">
        <button class="election-close" id="election-close">&times;</button>
        <h3>Election Simulation</h3>
        <div class="election-controls">
            <label>Swing σ: <input type="range" id="swing-sigma" min="1" max="10" value="5" step="0.5">
                <span id="swing-value">5%</span></label>
            <label>Elections: <select id="election-count">
                <option value="50">50</option>
                <option value="100" selected>100</option>
                <option value="500">500</option>
            </select></label>
            <button class="ghost-btn" id="run-elections">Run Simulation</button>
        </div>
        <canvas id="election-histogram" width="320" height="200"></canvas>
    </div>
</div>
```

**Step 3: Wire in main.js**

```js
import { simulateElections, renderHistogram } from './src/election-sim.js';

// In setupUI():
$.simulateBtn?.addEventListener('click', () => {
    $.electionOverlay.hidden = false;
});
$.electionClose?.addEventListener('click', () => {
    $.electionOverlay.hidden = true;
});
$.runElections?.addEventListener('click', () => {
    const sigma = parseFloat($.swingSigma.value) / 100;
    const count = parseInt($.electionCount.value);
    const results = simulateElections(count, sigma);
    renderHistogram($.electionHistogram, results, CONFIG.numDistricts);
});
$.swingSigma?.addEventListener('input', e => {
    $.swingValue.textContent = e.target.value + '%';
});
```

**Step 4: Add CSS**

In `gerry/styles.css`:
```css
.election-overlay { position: fixed; inset: 0; display: flex; align-items: center; justify-content: center; background: #00000044; z-index: 100; }
.election-panel { padding: 20px; border-radius: var(--radius-lg); max-width: 380px; width: 90%; position: relative; }
.election-close { position: absolute; top: 8px; right: 12px; background: none; border: none; font-size: 1.4rem; cursor: pointer; color: var(--text-muted); }
.election-controls { display: flex; flex-direction: column; gap: 8px; margin: 12px 0; }
.election-controls label { display: flex; align-items: center; gap: 8px; font-size: 0.82rem; }
#election-histogram { width: 100%; border-radius: var(--radius-sm); background: var(--bg-elevated); }
```

**Step 5: Verify**

Draw some districts, click Simulate Elections. Adjust swing σ and number of elections. Run simulation. Histogram should show seat distribution across parties. Higher swing should produce wider distributions.

**Step 6: Commit**

```bash
cd gerry
git add src/election-sim.js main.js index.html styles.css
git commit -m "feat: add multi-round election simulation with swing analysis

Simulate N elections with correlated national swing (normal distribution,
configurable σ). Each election applies per-party + local noise to vote
shares and recomputes district winners. Results displayed as a seat
histogram per party with mean seat counts. Demonstrates map resilience
to electoral volatility."
```

---

### Task 15: B-B — Oxidative Stress / ROS

**Files:**
- Modify: `biosim/src/state.js` (add ROS metabolites, health)
- Create: `biosim/src/reactions/ros.js` (SOD, catalase, glutathione reductase)
- Modify: `biosim/src/reactions/etc.js` (ROS generation chance)
- Modify: `biosim/src/reactions/dispatch.js` (register ROS handlers)
- Modify: `biosim/src/autoplay.js` (auto-fire ROS scavengers)
- Modify: `biosim/src/dashboard.js` (health bar display)
- Modify: `biosim/src/renderer.js` (ROS particle rendering, cell death overlay)
- Modify: `biosim/src/layout.js` (SOD/catalase positions)
- Modify: `biosim/src/info.js` (enzyme info entries)
- Modify: `biosim/index.html` (health bar HTML)

This is a large feature. Key implementation steps:

**Step 1: Add ROS state**

In `biosim/src/state.js` store:
```js
superoxide: 0,
h2o2: 0,
cellHealth: 100,
```

**Step 2: Create ROS reaction handlers**

Create `biosim/src/reactions/ros.js`:
```js
import { store, simState } from '../state.js';
import { showActiveStep, applyYields } from '../dashboard.js';

export function advanceSOD() {
    if (store.superoxide < 2) return false;
    store.superoxide -= 2;
    store.h2o2 += 1;
    store.o2 += 0.5;
    showActiveStep('SOD', '2 O₂⁻ → H₂O₂ + ½O₂', {}, 'ros');
    return true;
}

export function advanceCatalase() {
    if (store.h2o2 < 2) return false;
    store.h2o2 -= 2;
    store.h2o += 2;
    store.o2 += 1;
    showActiveStep('Catalase', '2 H₂O₂ → 2 H₂O + O₂', {}, 'ros');
    return true;
}

export function advanceGlutathioneReductase() {
    if (store.superoxide < 1 || store.nadph < 1) return false;
    store.superoxide -= 1;
    store.nadph -= 1;
    showActiveStep('GR', 'O₂⁻ + NADPH → scavenged', { nadphConsume: 1 }, 'ros');
    return true;
}
```

**Step 3: Add ROS generation to ETC**

In `biosim/src/reactions/etc.js`, in the respiratory ETC callback at Complex I (around line 24) and Cyt b6f (around line 34), add:
```js
// 2% chance of electron leak → superoxide
if (Math.random() < 0.02) {
    store.superoxide++;
    // Don't complete the chain — electron was lost
    return;
}
```

**Step 4: Health decay in autoplay**

In `biosim/src/autoplay.js`, add to the passive drain tick:
```js
// ROS damage
store.cellHealth -= (store.superoxide + store.h2o2) * 0.5;
store.cellHealth = Math.max(0, Math.min(100, store.cellHealth));

// Auto-scavenge
if (store.superoxide > 0) advanceSOD();
if (store.h2o2 > 0) advanceCatalase();
if (store.superoxide > 2 && store.nadph > 0) advanceGlutathioneReductase();

// Recovery when clean
if (store.superoxide === 0 && store.h2o2 === 0) {
    store.cellHealth = Math.min(100, store.cellHealth + 0.2);
}
```

**Step 5: Add health bar HTML and dashboard update**

In `biosim/index.html`, add before the cofactor bars:
```html
<div class="stat-group">
    <div class="group-label">Cell Health</div>
    <div class="mc"><div class="mc-bar-track"><div class="mc-bar-fill" id="health-bar" style="background:var(--palette-green)"></div></div><span class="mc-ratio" id="health-ratio">100%</span></div>
    <div class="stat-row"><span>Superoxide</span><span class="stat-value" id="superoxide-count">0</span></div>
    <div class="stat-row"><span>H₂O₂</span><span class="stat-value" id="h2o2-count">0</span></div>
</div>
```

Update `dashboard.js` to render health and ROS counts.

**Step 6: Verify**

Run biosim with respiratory ETC active. Superoxide should occasionally appear. SOD and catalase should scavenge it. If you disable PPP (cutting NADPH supply), ROS should accumulate faster and health should decrease. At health < 20%, show warning toast.

**Step 7: Commit**

```bash
cd biosim
git add src/reactions/ros.js src/reactions/etc.js src/state.js src/autoplay.js src/dashboard.js src/ui.js index.html
git commit -m "feat: add oxidative stress system with ROS, SOD, catalase

2% electron leak chance at Complex I and Cyt b6f generates superoxide.
SOD converts 2 O₂⁻ → H₂O₂. Catalase converts 2 H₂O₂ → H₂O + O₂.
Glutathione reductase consumes NADPH to scavenge ROS, giving PPP a
survival role. Cell health bar decreases with ROS accumulation.
Warning toast at health < 20%."
```

---

### Task 16: P-H — Energy Flow Sankey Diagram

**Files:**
- Create: `physsim/src/sankey.js` (energy flow visualization)
- Modify: `physsim/main.js` (track energy deltas, draw sankey)
- Modify: `physsim/src/ui.js` (toggle)
- Modify: `physsim/index.html` (checkbox)

**Step 1: Track energy deltas**

In `physsim/main.js`, store previous-frame energy values and compute deltas:

```js
// In constructor:
this.prevEnergy = { linearKE: 0, spinKE: 0, pe: 0, fieldE: 0, radiated: 0 };
this.energyFlows = { keToRad: 0, keToPe: 0, peToKe: 0, spinToOrbit: 0 };
this.flowSmoothing = 0.9; // exponential smoothing

// In computeEnergy(), after computing current values:
const dKE = linearKE - this.prevEnergy.linearKE;
const dPE = pe - this.prevEnergy.pe;
const dRad = this.totalRadiated - this.prevEnergy.radiated;

// Classify flows (smoothed)
const s = this.flowSmoothing;
this.energyFlows.keToRad = s * this.energyFlows.keToRad + (1 - s) * Math.max(0, dRad);
this.energyFlows.keToPe = s * this.energyFlows.keToPe + (1 - s) * Math.max(0, dPE);
this.energyFlows.peToKe = s * this.energyFlows.peToKe + (1 - s) * Math.max(0, -dPE);

this.prevEnergy = { linearKE, spinKE, pe, fieldE: fieldEnergy, radiated: this.totalRadiated };
```

**Step 2: Create Sankey overlay**

Create `physsim/src/sankey.js`:

```js
export default class SankeyOverlay {
    constructor() {
        this.enabled = false;
    }

    draw(ctx, width, height, flows, isLight) {
        if (!this.enabled) return;

        const x = width - 220, y = 12, w = 200, h = 140;

        // Background
        ctx.save();
        ctx.fillStyle = isLight ? '#FCF7F2CC' : '#0C0B09CC';
        ctx.strokeStyle = isLight ? '#1A161222' : '#E8DED422';
        ctx.lineWidth = 1;
        ctx.beginPath();
        ctx.roundRect(x, y, w, h, 8);
        ctx.fill();
        ctx.stroke();

        // Node positions
        const nodes = {
            KE: { x: x + 30, y: y + 35, color: '#CC8E4E' },
            PE: { x: x + 100, y: y + 35, color: '#5C92A8' },
            Rad: { x: x + 170, y: y + 35, color: '#CCA84C' },
            Spin: { x: x + 30, y: y + 95, color: '#9C7EB0' },
            Field: { x: x + 100, y: y + 95, color: '#4AACA0' },
        };

        // Draw nodes
        ctx.font = '9px Noto Sans';
        ctx.textAlign = 'center';
        for (const [name, n] of Object.entries(nodes)) {
            ctx.fillStyle = n.color;
            ctx.beginPath();
            ctx.arc(n.x, n.y, 12, 0, Math.PI * 2);
            ctx.fill();
            ctx.fillStyle = isLight ? '#1A1612' : '#E8DED4';
            ctx.fillText(name, n.x, n.y + 22);
        }

        // Draw flow arrows (width proportional to flow magnitude)
        const maxFlow = Math.max(
            flows.keToRad, flows.keToPe, flows.peToKe, flows.spinToOrbit, 0.001
        );

        this._drawFlow(ctx, nodes.KE, nodes.PE, flows.keToPe / maxFlow, '#CC8E4E88');
        this._drawFlow(ctx, nodes.PE, nodes.KE, flows.peToKe / maxFlow, '#5C92A888');
        this._drawFlow(ctx, nodes.KE, nodes.Rad, flows.keToRad / maxFlow, '#CCA84C88');
        this._drawFlow(ctx, nodes.Spin, nodes.KE, flows.spinToOrbit / maxFlow, '#9C7EB088');

        // Title
        ctx.fillStyle = isLight ? '#1A1612' : '#E8DED4';
        ctx.font = '10px Noto Sans';
        ctx.textAlign = 'left';
        ctx.fillText('Energy Flow', x + 8, y + 14);

        ctx.restore();
    }

    _drawFlow(ctx, from, to, magnitude, color) {
        if (magnitude < 0.01) return;
        const lineWidth = 1 + magnitude * 6;
        ctx.strokeStyle = color;
        ctx.lineWidth = lineWidth;
        ctx.beginPath();
        ctx.moveTo(from.x, from.y);
        ctx.lineTo(to.x, to.y);
        ctx.stroke();

        // Arrowhead
        const dx = to.x - from.x, dy = to.y - from.y;
        const len = Math.sqrt(dx * dx + dy * dy);
        const nx = dx / len, ny = dy / len;
        const ax = to.x - nx * 14, ay = to.y - ny * 14;
        ctx.fillStyle = color;
        ctx.beginPath();
        ctx.moveTo(to.x - nx * 2, to.y - ny * 2);
        ctx.lineTo(ax - ny * 4, ay + nx * 4);
        ctx.lineTo(ax + ny * 4, ay - nx * 4);
        ctx.fill();
    }
}
```

**Step 3: Wire toggle, draw in render**

Same pattern as other overlays. Toggle checkbox "Energy Flow" in Display tab. Draw after all overlays in screen space (not camera-transformed).

**Step 4: Verify**

Enable gravity + radiation. Watch KE↔PE flow arrows animate as particles orbit. KE→Rad arrow should appear when charged particles radiate.

**Step 5: Commit**

```bash
cd physsim
git add src/sankey.js main.js src/ui.js index.html
git commit -m "feat: add real-time energy flow Sankey diagram overlay

Track per-frame energy deltas between KE, PE, Radiation, Spin, and Field
categories. Display as an animated node-and-arrow diagram in the corner.
Arrow width proportional to transfer rate, smoothed over 10 frames.
Toggle in Display tab."
```

---

## Phase 4: Complex Features

---

### Task 17: G-B — Auto-Gerrymander & Auto-Fair Algorithms

**Files:**
- Create: `gerry/src/auto-district.js` (Pack & Crack + Fair Draw algorithms)
- Modify: `gerry/main.js` (buttons, party selector, algorithm invocation)
- Modify: `gerry/index.html` (toolbar buttons, party selector dropdown)
- Modify: `gerry/styles.css` (dropdown styling)

This is the most complex gerry feature. Key implementation:

**Step 1: Create auto-districting module**

Create `gerry/src/auto-district.js` with two main functions:

- `packAndCrack(targetParty)`: Greedy algorithm that maximizes seats for `targetParty` by packing opponents into few districts and cracking the rest across many. Uses BFS flood-fill to ensure contiguity.

- `fairDraw()`: Simulated annealing that minimizes `Σ|vote_share - seat_share|` for all parties + compactness bonus + population equality penalty. Iterates swapping border hexes between adjacent districts.

Both functions should:
1. Call `pushUndoSnapshot()` before modifying anything
2. Clear all existing assignments
3. Build districts from scratch
4. Call `updateMetrics()` after completion
5. Show a result toast

**Step 2: Add toolbar UI**

Two new buttons in toolbar. The gerrymander button has a dropdown to select which party to advantage:

```html
<div class="auto-district-group">
    <button class="tool-btn" id="gerrymander-btn" title="Auto-Gerrymander">
        <svg><!-- gerrymander icon --></svg>
    </button>
    <select id="gerrymander-party" class="party-select">
        <option value="red">Red</option>
        <option value="blue">Blue</option>
        <option value="yellow">Yellow</option>
    </select>
</div>
<button class="tool-btn" id="fair-draw-btn" title="Fair Draw">
    <svg><!-- balance icon --></svg>
</button>
```

**Step 3: Wire algorithms**

In `gerry/main.js`:
```js
$.gerrymanderBtn?.addEventListener('click', () => {
    const party = $.gerrymanderParty.value;
    pushUndoSnapshot();
    packAndCrack(party);
    doUpdateMetrics();
    showToast(`Auto-gerrymandered for ${party}`);
});

$.fairDrawBtn?.addEventListener('click', () => {
    pushUndoSnapshot();
    fairDraw();
    doUpdateMetrics();
    showToast('Fair districts drawn');
});
```

**Step 4: Implement Pack & Crack**

```js
export function packAndCrack(targetParty) {
    // 1. Clear all assignments
    for (const hex of state.hexes.values()) hex.district = 0;

    // 2. Sort hexes by opposition vote share (descending)
    const hexList = [...state.hexes.values()].filter(h => h.population > 0);
    const opposition = hexList.sort((a, b) => {
        const aOpp = 1 - (a.votes[targetParty] / (a.votes.red + a.votes.blue + a.votes.yellow));
        const bOpp = 1 - (b.votes[targetParty] / (b.votes.red + b.votes.blue + b.votes.yellow));
        return bOpp - aOpp;
    });

    // 3. Pack: fill ~2 districts with highest-opposition hexes
    const packCount = 2;
    const targetPopPerDistrict = hexList.reduce((s, h) => s + h.population, 0) / CONFIG.numDistricts;

    for (let d = 1; d <= packCount; d++) {
        let pop = 0;
        // BFS from a seed hex
        const seed = opposition.find(h => h.district === 0);
        if (!seed) break;
        const queue = [seed];
        const visited = new Set();

        while (queue.length > 0 && pop < targetPopPerDistrict * CONFIG.popCapRatio) {
            // Sort queue by opposition vote share (pack highest opposition first)
            queue.sort((a, b) => {
                const aOpp = 1 - (a.votes[targetParty] / (a.votes.red + a.votes.blue + a.votes.yellow || 1));
                const bOpp = 1 - (b.votes[targetParty] / (b.votes.red + b.votes.blue + b.votes.yellow || 1));
                return bOpp - aOpp;
            });

            const hex = queue.shift();
            const key = hex.q + ',' + hex.r;
            if (visited.has(key) || hex.district !== 0) continue;
            if (pop + hex.population > targetPopPerDistrict * CONFIG.popCapRatio) continue;

            visited.add(key);
            hex.district = d;
            pop += hex.population;

            // Add unassigned neighbors to queue
            for (const [dq, dr] of HEX_DIRS) {
                const nk = (hex.q + dq) + ',' + (hex.r + dr);
                const nh = state.hexes.get(nk);
                if (nh && nh.district === 0 && !visited.has(nk)) {
                    queue.push(nh);
                }
            }
        }
    }

    // 4. Crack: distribute remaining hexes across remaining districts
    // ensuring target party wins each
    for (let d = packCount + 1; d <= CONFIG.numDistricts; d++) {
        let pop = 0;
        const remaining = [...state.hexes.values()].filter(h => h.district === 0);
        if (remaining.length === 0) break;

        // Seed from a hex with high target-party share
        remaining.sort((a, b) => {
            const aShare = a.votes[targetParty] / (a.votes.red + a.votes.blue + a.votes.yellow || 1);
            const bShare = b.votes[targetParty] / (b.votes.red + b.votes.blue + b.votes.yellow || 1);
            return bShare - aShare;
        });

        const seed = remaining[0];
        const queue = [seed];
        const visited = new Set();

        while (queue.length > 0 && pop < targetPopPerDistrict * CONFIG.popCapRatio) {
            const hex = queue.shift();
            const key = hex.q + ',' + hex.r;
            if (visited.has(key) || hex.district !== 0) continue;
            if (pop + hex.population > targetPopPerDistrict * CONFIG.popCapRatio) continue;

            visited.add(key);
            hex.district = d;
            pop += hex.population;

            for (const [dq, dr] of HEX_DIRS) {
                const nk = (hex.q + dq) + ',' + (hex.r + dr);
                const nh = state.hexes.get(nk);
                if (nh && nh.district === 0 && !visited.has(nk)) queue.push(nh);
            }
        }
    }

    // Update visuals
    for (const hex of state.hexes.values()) {
        updateHexVisuals(hex.q + ',' + hex.r);
    }
}
```

**Step 5: Implement Fair Draw**

Simulated annealing with objective function:
```js
function objective() {
    const totalVotes = { red: 0, blue: 0, yellow: 0 };
    const seats = { red: 0, blue: 0, yellow: 0 };
    // ... compute from current assignments ...
    const proportionalityError = parties.reduce((sum, p) => {
        return sum + Math.abs(totalVotes[p] / totalVotesAll - seats[p] / totalSeats);
    }, 0);
    const compactnessBonus = avgCompactness / 100;
    const popEquality = 1 - maxDeviation;
    return proportionalityError - 0.3 * compactnessBonus - 0.3 * popEquality;
}
```

Run 3000 iterations, temperature schedule from 1.0 → 0.01.

**Step 6: Verify**

Generate a map. Click "Fair Draw" — districts should be roughly proportional. Click "Auto-Gerrymander for Red" — Red should win disproportionately many seats. Undo should restore previous state.

**Step 7: Commit**

```bash
cd gerry
git add src/auto-district.js main.js index.html styles.css
git commit -m "feat: add auto-gerrymander (pack & crack) and fair draw algorithms

Pack & Crack: maximizes seats for a chosen party by concentrating opponents
into few packed districts and distributing supporters across many cracked
districts. Fair Draw: simulated annealing minimizing |vote_share - seat_share|
with compactness and population equality bonuses. Both maintain contiguity
via BFS flood-fill and support undo."
```

---

### Task 18: P-F — Retarded Potentials

**Files:**
- Modify: `physsim/src/particle.js` (add history buffers)
- Modify: `physsim/src/physics.js` (retarded position lookup, modified force calc)
- Modify: `physsim/src/config.js` (history constants)
- Modify: `physsim/src/renderer.js` (ghost particle rendering)
- Modify: `physsim/src/ui.js` (light cone toggle)
- Modify: `physsim/index.html` (display checkbox)

This is the most complex feature. Implementation outline:

**Step 1: Add history buffer to Particle**

In `physsim/src/particle.js`:
```js
// In constructor:
this.historyX = new Float32Array(512);
this.historyY = new Float32Array(512);
this.historyVx = new Float32Array(512);
this.historyVy = new Float32Array(512);
this.historyTime = new Float64Array(512);
this.histHead = 0;
this.histCount = 0;
```

**Step 2: Record history at each substep**

In `physsim/src/physics.js`, in the substep loop, after position drift (around line 129):
```js
// Record history for retarded potentials
if (this.retardedEnabled) {
    const h = p.histHead;
    p.historyX[h] = p.pos.x;
    p.historyY[h] = p.pos.y;
    p.historyVx[h] = p.vel.x;
    p.historyVy[h] = p.vel.y;
    p.historyTime[h] = this.simTime;
    p.histHead = (h + 1) % 512;
    if (p.histCount < 512) p.histCount++;
}
```

Track `this.simTime += dtSub` in the physics object.

**Step 3: Retarded position lookup**

Add a helper method:
```js
_getRetardedState(source, observer, time) {
    // Find t_ret such that |source(t_ret) - observer(time)| = (time - t_ret)
    // Newton-Raphson: 2-3 iterations
    const ox = observer.pos.x, oy = observer.pos.y;
    let tRet = time - Vec2.dist(source.pos, observer.pos); // initial guess

    for (let iter = 0; iter < 3; iter++) {
        // Interpolate source position at tRet from history buffer
        const sp = this._interpolateHistory(source, tRet);
        if (!sp) return null; // history not long enough

        const dx = sp.x - ox, dy = sp.y - oy;
        const dist = Math.sqrt(dx * dx + dy * dy);
        const residual = dist - (time - tRet);
        if (Math.abs(residual) < 0.01) break;

        // Newton step: d(residual)/d(tRet) ≈ -(v_source · r̂)/r - 1
        tRet += residual / (1 + (sp.vx * dx + sp.vy * dy) / (dist * dist + 1));
    }

    return this._interpolateHistory(source, tRet);
}

_interpolateHistory(p, t) {
    if (p.histCount < 2) return null;
    // Binary search for bracketing time entries
    // Linear interpolation between them
    // ... (standard circular buffer binary search + lerp)
}
```

**Step 4: Use retarded positions in force calculation**

In `_pairForce()`, when `this.retardedEnabled` is true, replace the source particle's current position/velocity with the retarded state from `_getRetardedState()`.

**Step 5: Add ghost particle rendering**

In `physsim/src/renderer.js`, when "Light Cone" display is enabled, draw faint circles at each particle's retarded position as seen by the selected particle. This shows where each particle "was" when the force was emitted.

**Step 6: Wire toggles**

- Retarded potentials toggle: active only when Relativity is on
- Light Cone display checkbox: shows ghost particles
- Info tip explaining retarded potentials and causality

**Step 7: Verify**

Create two particles far apart. Enable relativity + retarded potentials. The force between them should have a visible delay — moving one particle doesn't instantly affect the other. With the Light Cone display on, ghost positions should lag behind actual positions.

**Step 8: Commit**

```bash
cd physsim
git add src/particle.js src/physics.js src/config.js src/renderer.js src/ui.js index.html
git commit -m "feat: add retarded potentials — forces propagate at light speed

Each particle maintains a 512-entry circular history buffer of positions
and velocities. Force calculation uses Newton-Raphson to solve for the
retarded time t_ret such that |x_source(t_ret) - x_observer| = c·(t - t_ret).
Optional 'Light Cone' display shows retarded positions as ghost particles.
Active only when Relativity toggle is on."
```

---

## Post-Implementation

After all 18 tasks are complete:

1. Update each project's `CLAUDE.md` with new features, forces, toggles, and stat rows
2. Update each project's `README.md` with new feature descriptions
3. Update the root `site-meta/CLAUDE.md` with any new shared patterns
4. Visual regression test: check all three sims in both light and dark themes at desktop and mobile breakpoints
