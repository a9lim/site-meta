# Simulation Improvements Batch 2 — Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Implement 7 features across biosim (3 tasks) and physsim (4 tasks), from trivial config changes to new physics terms and membrane visualizations.

**Architecture:** Biosim changes are all in `biosim/src/` — autoplay timing, organism config, enzyme rendering, and layout. Physsim changes are in `physsim/src/` — particle data, physics integrator, input handling, and UI. No shared files modified.

**Tech Stack:** Vanilla JS ES6 modules, Canvas 2D rendering, no build system or dependencies.

---

## Task 1: B-Ba — Remove NADPH Passive Drain

**Files:**
- Modify: `biosim/src/autoplay.js:31`

**Step 1: Remove the NADPH drain line**

In `autoplay.js`, remove line 31 (`if (store.nadph > 1) store.nadph -= 2;`) from the passive drain block. Keep the ATP drain on line 30.

Before:
```javascript
    // Passive ATP/NADPH drain to mimic cellular maintenance
    if (passiveDrainTimer > 1.6) {
        passiveDrainTimer = 0;
        if (store.atp > 2) store.atp -= 3;
        if (store.nadph > 1) store.nadph -= 2;
        updateDashboard();
    }
```

After:
```javascript
    // Passive ATP drain to mimic cellular maintenance
    if (passiveDrainTimer > 1.6) {
        passiveDrainTimer = 0;
        if (store.atp > 2) store.atp -= 3;
        updateDashboard();
    }
```

**Step 2: Verify locally**

Run: `cd biosim && python -m http.server`
- Enable autoplay, observe NADPH bar stabilizes instead of draining to zero
- PPP should still produce NADPH; Calvin/FA synthesis should still consume it

**Step 3: Commit**

```bash
cd biosim
git add src/autoplay.js
git commit -m "fix: remove passive NADPH drain from autoplay maintenance tick"
```

---

## Task 2: B-Ga — Enable PPP for Anaerobe and Archaeon

**Files:**
- Modify: `biosim/src/organisms.js:21,25,47,51`

**Step 1: Enable PPP and remove lock reasons**

In `organisms.js`, change `ppp: false` to `ppp: true` for both organisms and remove the `ppp` key from their `lockedReason` objects.

Obligate Anaerobe (line 21):
```javascript
        pathways: { glycolysis: true, ppp: true, calvin: false, krebs: false, betaox: false },
```
And remove line 25 (`ppp: 'Simplified anaerobe model',`) from its `lockedReason`.

Archaeon (line 47):
```javascript
        pathways: { glycolysis: true, ppp: true, calvin: false, krebs: true, betaox: true },
```
And remove line 51 (`ppp: 'Simplified archaeal model',`) from its `lockedReason`.

**Step 2: Verify locally**

- Select "Obligate Anaerobe" from organism dropdown — PPP toggle should be enabled, not locked
- Select "Archaeon" — PPP toggle should be enabled, not locked
- Click G6PDH on canvas — PPP should advance

**Step 3: Commit**

```bash
cd biosim
git add src/organisms.js
git commit -m "feat: enable PPP pathway for obligate anaerobe and archaeon organisms"
```

---

## Task 3: B-Fa — Add Uncoupling Protein to Membrane

**Files:**
- Modify: `biosim/src/enzymes.js` (add `drawUCP` method + `uncoupling` role)
- Modify: `biosim/src/layout.js:19,23-37` (bump numComplexes, add UCP position)
- Modify: `biosim/src/renderer.js:31-34,458-627` (add `_etcInfoKey`, draw UCP, add hitbox)
- Modify: `biosim/src/autoplay.js:80-89` (spawn proton particles during leak)
- Modify: `biosim/src/info.js` (add UCP entry)

### Step 1: Add `uncoupling` role to enzymes.js

In `_ROLE` object (after line 66 `nnt: _BASE.brown,`), add:
```javascript
  uncoupling: _BASE.red,
```

### Step 2: Add `drawUCP` method to EnzymeStyles

After the `drawNNT` method (after line 432), add the UCP drawing function. UCP is a simple transmembrane channel (narrower barrel, no F1 head):

```javascript
  drawUCP(ctx, cx, cy, w, h, glow, lightMode) {
    const p = this.getPalette('uncoupling', lightMode, glow);
    const chanW = w * 0.5, chanH = h * 0.85;
    const r = 4;
    // Simple barrel channel shape
    this.roundedRect(ctx, cx - chanW / 2, cy - chanH / 2, chanW, chanH, r);
    this.applyStyle(ctx, p, glow);
    // Internal channel lines (pore opening)
    ctx.save();
    ctx.globalAlpha *= 0.2;
    ctx.strokeStyle = p.stroke; ctx.lineWidth = 1;
    ctx.beginPath();
    ctx.moveTo(cx - chanW * 0.15, cy - chanH * 0.35);
    ctx.lineTo(cx - chanW * 0.15, cy + chanH * 0.35);
    ctx.moveTo(cx + chanW * 0.15, cy - chanH * 0.35);
    ctx.lineTo(cx + chanW * 0.15, cy + chanH * 0.35);
    ctx.stroke();
    ctx.restore();
    this.drawLabel(ctx, 'UCP', cx, cy, p.stroke, 10);
  },
```

### Step 3: Update layout.js — add UCP position

Change `numComplexes` from 13 to 14 (line 19):
```javascript
    const numComplexes = 14;
```

Add UCP entry to `etcComplexes` (after line 36 `nnt:`):
```javascript
        ucp:    { cx: colW(14), cy: memMid },
```

### Step 4: Add `_etcInfoKey` entry in renderer.js

In the `_etcInfoKey` object (line 31-34), add:
```javascript
    'ucp:0': 'UCP',
```

### Step 5: Draw UCP in `drawETCChain` in renderer.js

After the NNT drawing block (after line 602, before the ETC hitboxes section), add:

```javascript
        // UCP — uncoupling protein, visible when uncoupling enabled, glows when gradient + uncoupling
        {
            const ucpAlpha = state.uncouplingEnabled ? 1 : 0.3;
            const ucpGlow = (state.uncouplingEnabled && state.protonGradient > 0) ? pulse : 0;
            ctx.save();
            ctx.globalAlpha = ucpAlpha;
            EnzymeStyles.drawUCP(ctx, c.ucp.cx, c.ucp.cy, cxW - 4, cxH * 0.6, ucpGlow, lm);
            if (state.uncouplingEnabled && state.protonGradient > 0) {
                const ucpBotY = c.ucp.cy + cxH * 0.6 / 2;
                this.drawSmallProtonArrow(ctx, c.ucp.cx, ucpBotY + 3, 'H⁺', 'down');
                ctx.font = _F.mono500_10;
                ctx.fillStyle = EnzymeStyles.roleColors.uncoupling.stroke;
                ctx.textAlign = 'center';
                ctx.fillText('heat', c.ucp.cx, ucpBotY + 47);
            }
            ctx.restore();
        }
```

Add UCP hitbox in the hitboxes section (after line 612, the BR hitbox):
```javascript
        this.enzymeHitboxes.push({ cx: c.ucp.cx, cy: c.ucp.cy, w: cxW + 10, h: cxH + 10, pathway: 'ucp', stepIndex: 0 });
```

### Step 6: Spawn proton particles during leak in autoplay.js

In `protonLeakTick`, after the leaked protons are subtracted (after line 88), add visual proton spawning when uncoupling is active. Import Particles at the top of autoplay.js and access layout data:

At the top of autoplay.js, add import:
```javascript
import Particles from './particles.js';
```

After line 88 (`store.protonsLeaked += leaked;`), add:
```javascript
            // Visual: spawn downward protons through UCP when uncoupling active
            if (simState.uncouplingEnabled && simState._renderer) {
                const r = simState._renderer;
                const count = Math.min(leaked, 3); // cap visual particles
                for (let i = 0; i < count; i++) {
                    setTimeout(() => {
                        Particles.spawnProton(r.membraneY, r.membraneH, r.etcComplexes.ucp.cx, 'down');
                    }, i * 120);
                }
            }
```

Note: `simState._renderer` needs to be set during initialization (in main.js where the Renderer is created). Check if this reference already exists — if not, add `simState._renderer = renderer;` after renderer creation in main.js.

### Step 7: Add UCP info entry in info.js

After the BR entry (line 62), add:
```javascript
    UCP:    { name: 'Uncoupling Protein', pathway: 'Membrane', eq: 'H⁺ gradient → heat', desc: 'Dissipates the proton gradient as heat instead of driving ATP synthesis. Active when uncoupling is enabled.' },
```

### Step 8: Verify locally

- Toggle uncoupling ON → UCP should glow, show downward "H⁺" arrow and "heat" label
- Toggle uncoupling OFF → UCP should be dimmed (alpha 0.3)
- Click on UCP → info popup should appear
- Proton particles should flow downward through UCP during leak ticks

### Step 9: Commit

```bash
cd biosim
git add src/enzymes.js src/layout.js src/renderer.js src/autoplay.js src/info.js
git commit -m "feat: add uncoupling protein (UCP) visual complex to membrane"
```

---

## Task 4: P-Aa — Photon Spawn Direction Toggle

**Files:**
- Modify: `physsim/src/input.js:10,167-168,220-260`
- Modify: `physsim/index.html:175-178`

### Step 1: Add mouse direction tracking to InputHandler

In `input.js` constructor (after line 10 `this.currentPos`), add:
```javascript
        this.prevMouseWorld = new Vec2(0, 0);  // previous mouse world position for direction
```

### Step 2: Update onMouseMove to track direction

In `onMouseMove` (line 167-168), save previous position before updating:
```javascript
    onMouseMove(e) {
        this.prevMouseWorld.x = this.currentPos.x;
        this.prevMouseWorld.y = this.currentPos.y;
        this.currentPos = this.getPos(e.clientX, e.clientY);
```

### Step 3: Add photon import

At the top of `input.js`, add:
```javascript
import Photon from './photon.js';
import { MAX_PHOTONS } from './config.js';
```

### Step 4: Add photon spawn logic to `spawnParticle`

In `spawnParticle` (line 220), before the existing mode checks, add the photon mode:

```javascript
    spawnParticle(endPos) {
        if (this.mode === 'photon') {
            if (this.sim.photons.length >= MAX_PHOTONS) return;
            let dx = this.currentPos.x - this.prevMouseWorld.x;
            let dy = this.currentPos.y - this.prevMouseWorld.y;
            const mag = Math.sqrt(dx * dx + dy * dy);
            if (mag < 1e-6) {
                // Mouse stationary — random direction
                const angle = Math.random() * 2 * Math.PI;
                dx = Math.cos(angle);
                dy = Math.sin(angle);
            } else {
                dx /= mag;
                dy /= mag;
            }
            this.sim.photons.push(new Photon(
                this.dragStart.x, this.dragStart.y,
                dx, dy, 1.0  // energy = 1.0
            ));
            return;
        }

        const dragVector = Vec2.sub(this.dragStart, endPos);
        // ... rest of existing spawnParticle
```

### Step 5: Add Photon mode button to HTML

In `index.html` (after line 178, the Orbit button), add:
```html
                                <button class="mode-btn" data-mode="photon">Photon</button>
```

### Step 6: Verify locally

- Select "Photon" mode from interaction toggles
- Move mouse in a direction and click → photon spawns moving in that direction
- Photon should move at c (|vel| = 1), interact with particles via radiation pressure
- Check MAX_PHOTONS cap is respected

### Step 7: Commit

```bash
cd physsim
git add src/input.js index.html
git commit -m "feat: add photon spawn mode — click to emit photon in mouse direction"
```

---

## Task 5: P-Ab — Abraham-Lorentz Radiation via Landau-Lifshitz

**Files:**
- Modify: `physsim/src/particle.js:14` (add `prevForce`)
- Modify: `physsim/src/config.js` (add `LL_FORCE_CLAMP`)
- Modify: `physsim/src/physics.js:146-199` (replace KE drain with LL force)

### Step 1: Add `prevForce` to Particle

In `particle.js`, after line 14 (`this.force = new Vec2(0, 0);`), add:
```javascript
        this.prevForce = new Vec2(0, 0); // previous step force (for Landau-Lifshitz jerk)
```

### Step 2: Add LL_FORCE_CLAMP to config.js

After line 39 (MAX_PHOTONS), add:
```javascript
export const LL_FORCE_CLAMP = 0.5;        // max |F_rad·dt/m| as fraction of |w|
```

### Step 3: Replace Larmor KE-drain with Landau-Lifshitz force

In `physics.js`, replace the entire Larmor radiation block (lines 146-199) with:

```javascript
            // Abraham-Lorentz radiation reaction via Landau-Lifshitz approximation
            // Replaces direct KE drain with a proper force: jerk + Schott damping terms
            if (this.radiationEnabled && this.sim) {
                for (let i = 0; i < n; i++) {
                    const p = particles[i];
                    if (Math.abs(p.charge) < 1e-10) continue;

                    const wMagSq = p.w.x * p.w.x + p.w.y * p.w.y;
                    if (wMagSq < 1e-20) {
                        // Store force for next step's jerk computation
                        p.prevForce.x = p.force.x;
                        p.prevForce.y = p.force.y;
                        continue;
                    }

                    const gamma = relOn ? Math.sqrt(1 + wMagSq) : 1;
                    const qSq = p.charge * p.charge;
                    const tau = 2 * LARMOR_K * qSq / p.mass;

                    // Jerk term: τ · dF/dt ≈ τ · (F - F_prev) / dt
                    const invDt = 1 / dtSub;
                    const jerkX = (p.force.x - p.prevForce.x) * invDt;
                    const jerkY = (p.force.y - p.prevForce.y) * invDt;

                    // Schott damping term: τ · |F|² · v / m
                    const Fsq = p.force.x * p.force.x + p.force.y * p.force.y;
                    const schottScale = Fsq / p.mass;
                    const schottX = schottScale * p.vel.x;
                    const schottY = schottScale * p.vel.y;

                    // Total LL radiation reaction force
                    let fRadX = tau * (jerkX + schottX);
                    let fRadY = tau * (jerkY + schottY);

                    // Relativistic correction: divide by γ³
                    if (relOn && gamma > 1) {
                        const invG3 = 1 / (gamma * gamma * gamma);
                        fRadX *= invG3;
                        fRadY *= invG3;
                    }

                    // Clamp to prevent instability: |F_rad · dt / m| ≤ LL_FORCE_CLAMP · |w|
                    const impulseX = fRadX * dtSub / p.mass;
                    const impulseY = fRadY * dtSub / p.mass;
                    const impulseMag = Math.sqrt(impulseX * impulseX + impulseY * impulseY);
                    const wMag = Math.sqrt(wMagSq);
                    const maxImpulse = LL_FORCE_CLAMP * wMag;

                    if (impulseMag > maxImpulse && impulseMag > 1e-20) {
                        const scale = maxImpulse / impulseMag;
                        fRadX *= scale;
                        fRadY *= scale;
                    }

                    // Measure KE before applying
                    const keBefore = relOn ? (gamma - 1) * p.mass : 0.5 * p.mass * (p.vel.x * p.vel.x + p.vel.y * p.vel.y);

                    // Apply as kick to proper velocity
                    p.w.x += fRadX * dtSub / p.mass;
                    p.w.y += fRadY * dtSub / p.mass;

                    // NaN guard
                    if (isNaN(p.w.x) || isNaN(p.w.y)) {
                        p.w.x = 0; p.w.y = 0;
                    }

                    // Measure KE after for energy tracking
                    const wMagSqAfter = p.w.x * p.w.x + p.w.y * p.w.y;
                    const gammaAfter = relOn ? Math.sqrt(1 + wMagSqAfter) : 1;
                    const keAfter = relOn ? (gammaAfter - 1) * p.mass : 0.5 * p.mass * wMagSqAfter / (gammaAfter * gammaAfter);
                    const dE = Math.max(0, keBefore - keAfter);
                    this.sim.totalRadiated += dE;

                    // Spawn photon if energy exceeds threshold
                    if (dE > RADIATION_THRESHOLD && this.sim.photons.length < MAX_PHOTONS) {
                        const ax = p.force.x / p.mass, ay = p.force.y / p.mass;
                        const angle = Math.atan2(ay, ax) + Math.PI + (Math.random() - 0.5) * 1.0;
                        this.sim.photons.push(new Photon(
                            p.pos.x, p.pos.y,
                            Math.cos(angle), Math.sin(angle),
                            dE
                        ));
                    }

                    // Store force for next step's jerk computation
                    p.prevForce.x = p.force.x;
                    p.prevForce.y = p.force.y;
                }
            }
```

### Step 4: Add LL_FORCE_CLAMP import

In `physics.js`, add `LL_FORCE_CLAMP` to the config import (line ~1-5 area):
```javascript
import { ..., LL_FORCE_CLAMP } from './config.js';
```

### Step 5: Verify locally

- Enable radiation toggle
- Spawn two charged particles in close orbit
- Verify particles don't freeze or produce NaN
- Verify energy conservation display shows radiated energy increasing
- Photons should still emit from accelerating charges
- Compare orbital decay behavior: should see smooth spiral-in rather than sudden freeze

### Step 6: Commit

```bash
cd physsim
git add src/particle.js src/config.js src/physics.js
git commit -m "feat: implement Abraham-Lorentz radiation reaction via Landau-Lifshitz force"
```

---

## Task 6: P-Ba — Combined Surface Force Disintegration

**Files:**
- Modify: `physsim/src/physics.js:720-747` (extend `checkTidalBreakup`)

### Step 1: Extend checkTidalBreakup with centrifugal and Coulomb terms

Replace the entire `checkTidalBreakup` method with:

```javascript
    checkTidalBreakup(particles) {
        if (!this.tidalEnabled) return [];
        const fragments = [];

        for (const p of particles) {
            if (p.mass < MIN_FRAGMENT_MASS * FRAGMENT_COUNT) continue;

            const rSq = p.radius * p.radius;

            // Self-gravity binding force at surface: F_bind = m / r²
            const selfGravity = p.mass / rSq;

            // Per-particle self-disruption forces (no neighbor needed)
            // Centrifugal: F = ω² · r
            const centrifugal = p.angVel * p.angVel * p.radius;
            // Coulomb self-repulsion: uniform charge sphere surface field
            // F = q² / (4·r²) in natural units (k=1)
            const coulombSelf = (p.charge * p.charge) / (4 * rSq);

            if (centrifugal + coulombSelf > selfGravity) {
                fragments.push(p);
                continue; // already marked for breakup
            }

            // Tidal stretching from nearby bodies
            let maxTidal = 0;
            for (const other of particles) {
                if (other === p) continue;
                const dx = other.pos.x - p.pos.x, dy = other.pos.y - p.pos.y;
                const distSq = dx * dx + dy * dy;
                const r = Math.sqrt(distSq);
                const tidalAccel = TIDAL_STRENGTH * other.mass * p.radius / (r * distSq);
                if (tidalAccel > maxTidal) maxTidal = tidalAccel;
            }

            // Combined: all outward forces vs binding
            if (maxTidal + centrifugal + coulombSelf > selfGravity) {
                fragments.push(p);
            }
        }

        return fragments;
    }
```

### Step 2: Verify locally

- Enable tidal toggle
- Spawn a particle with very high spin (e.g. spin=5) → should disintegrate from centrifugal force alone
- Spawn a particle with very high charge (e.g. q=50, mass=5) → should disintegrate from Coulomb self-repulsion
- Spawn two particles close together → original tidal breakup should still work
- Verify fragments inherit mass/3, charge/3, parent spin

### Step 3: Commit

```bash
cd physsim
git add src/physics.js
git commit -m "feat: generalize tidal breakup to include centrifugal and Coulomb self-disruption"
```

---

## Task 7: P-Ga — Gravitomagnetic Spin Coupling

**Files:**
- Modify: `physsim/src/config.js` (add `FRAME_DRAG_K`)
- Modify: `physsim/src/physics.js:127-144` (add GM spin-orbit after EM spin-orbit)
- Modify: `physsim/src/physics.js:640-653` (add Bgz gradients + frame-drag torque)
- Modify: `physsim/src/particle.js` (add `dBgzdx`, `dBgzdy`)

### Step 1: Add FRAME_DRAG_K to config.js

After the TIDAL_STRENGTH line (line 45), add:
```javascript
export const FRAME_DRAG_K = 0.1;           // frame-dragging spin alignment strength
```

### Step 2: Add Bgz gradient fields to Particle

In `particle.js`, after line 29 (`this.dBzdy = 0;`), add:
```javascript
        this.dBgzdx = 0;       // Gravitomagnetic field gradient x-component
        this.dBgzdy = 0;       // Gravitomagnetic field gradient y-component
```

### Step 3: Clear Bgz gradients in force reset

In `physics.js`, in the force reset loop (around line 298-300 where `p.Bgz = 0` etc.), add:
```javascript
            p.dBgzdx = 0;
            p.dBgzdy = 0;
```

### Step 4: Compute Bgz gradients in force calculation

In `physics.js`, inside the `gravitomagEnabled` block (after line 652 where `p.Bgz` is accumulated), add:

```javascript
            // Bgz gradient for GM spin-orbit coupling (same pattern as EM dBz)
            const Bgz_contribution = sMass * crossSV * invR * invRSq;
            const dBgzdr = -3 * Bgz_contribution * invR;
            p.dBgzdx += dBgzdr * rx * invR;
            p.dBgzdy += dBgzdr * ry * invR;
```

### Step 5: Add frame-dragging torque in force calculation

Still inside the `gravitomagEnabled` block, after the Bgz gradient lines, add:

```javascript
            // Frame-dragging torque: drives spins toward co-rotation
            const Ip = INERTIA_K * p.mass * p.radius * p.radius;
            const torque = FRAME_DRAG_K * sMass * (sAngVel - p.angVel) * invR * invRSq;
            p._frameDragTorque = (p._frameDragTorque || 0) + torque;
```

Note: We accumulate the torque during force computation, then apply it during the integration step. Add `p._frameDragTorque = 0;` to the force reset loop.

### Step 6: Add GM spin-orbit coupling in integration step

In `physics.js`, after the existing EM spin-orbit block (after line 143), add:

```javascript
            // GM Spin-orbit coupling: dE_spin/dt = -L · (v · ∇Bgz)
            if (hasGM && relOn) {
                for (let i = 0; i < n; i++) {
                    const p = particles[i];
                    if (Math.abs(p.angVel) < 1e-10) continue;
                    const pRSq = p.radius * p.radius;
                    const L = INERTIA_K * p.mass * p.angVel * pRSq;
                    const vDotGradBg = p.vel.x * p.dBgzdx + p.vel.y * p.dBgzdy;
                    const dEspin = -L * vDotGradBg * dtSub;
                    const I = INERTIA_K * p.mass * pRSq;
                    if (Math.abs(I * p.angVel) > 1e-10) {
                        p.spin += dEspin / (I * p.angVel);
                        const sr = p.spin * p.radius;
                        p.angVel = p.spin / Math.sqrt(1 + sr * sr);
                    }
                }
            }
```

### Step 7: Apply frame-dragging torque in integration step

After the GM spin-orbit block, add:

```javascript
            // Frame-dragging spin alignment torque
            if (hasGM) {
                for (let i = 0; i < n; i++) {
                    const p = particles[i];
                    if (!p._frameDragTorque) continue;
                    const I = INERTIA_K * p.mass * p.radius * p.radius;
                    p.spin += p._frameDragTorque * dtSub / I;
                    const sr = p.spin * p.radius;
                    p.angVel = relOn ? p.spin / Math.sqrt(1 + sr * sr) : p.spin;
                }
            }
```

### Step 8: Add imports

Add `FRAME_DRAG_K` to the config import in physics.js.

### Step 9: Verify locally

- Spawn two particles with opposite spins (spin=2, spin=-2), enable gravitomagnetism
- Frame-dragging should slowly align their spins toward co-rotation
- Spawn a spinning particle orbiting a massive one — GM spin-orbit should exchange energy between spin and orbit
- Check angular momentum conservation display (expect small drift from GM spin-orbit, similar to EM spin-orbit)

### Step 10: Commit

```bash
cd physsim
git add src/config.js src/particle.js src/physics.js
git commit -m "feat: add gravitomagnetic spin coupling — GM spin-orbit and frame-dragging torque"
```

---

## Dependency Order

All tasks are independent. Recommended sequence (simplest first, builds confidence):

1. **Task 1** (B-Ba) — 1-line removal
2. **Task 2** (B-Ga) — 2-line changes
3. **Task 3** (B-Fa) — medium, self-contained biosim
4. **Task 4** (P-Aa) — medium, self-contained physsim
5. **Task 5** (P-Ab) — high, core physics change
6. **Task 6** (P-Ba) — medium, extends existing
7. **Task 7** (P-Ga) — high, new physics terms

Tasks 1-2 and 4-7 can be parallelized across biosim/physsim repos.
