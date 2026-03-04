# Physsim Batch 3: Post-Newtonian Effects & Bug Fixes — Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Add 1PN perihelion precession, spin-curvature coupling (Mathisson-Papapetrou), spin-sourced Lense-Thirring fields, photon absorption (radiation pressure), and fix two bugs (signal delay NR denominator, Darwin field momentum sign).

**Architecture:** Six independent changes to the physsim physics engine, all following the existing modular toggle pattern. 1PN uses a velocity-Verlet correction for accuracy on velocity-dependent terms. Spin-sourced fields and MP force reuse already-computed field gradients. Photon absorption uses the existing quadtree for O(P log N) collision detection.

**Tech Stack:** Vanilla ES6 modules, Canvas 2D, zero dependencies. Natural units (G=c=1, epsilon_0=1/4pi). No test framework — verify by running the simulation and checking energy/momentum conservation in the Stats tab.

---

### Task 1: Bug Fix — Signal Delay Newton-Raphson Denominator

**Files:**
- Modify: `physsim/src/signal-delay.js:6,32`

**Step 1: Add SOFTENING import**

In `signal-delay.js:6`, change:
```js
import { HISTORY_SIZE } from './config.js';
```
to:
```js
import { HISTORY_SIZE, SOFTENING } from './config.js';
```

**Step 2: Fix the NR denominator**

In `signal-delay.js:32`, change:
```js
        const denom = 1 + (sp.vx * dx + sp.vy * dy) / (dist * dist + 1);
```
to:
```js
        const denom = 1 + (sp.vx * dx + sp.vy * dy) / (dist + SOFTENING);
```

The derivative of the light-cone residual is `(v_s . r)/|r| + 1`. The old code used `dist^2 + 1` (wrong power), badly distorting convergence. Using `dist + SOFTENING` gives the correct power with consistent regularization.

**Step 3: Verify**

Serve from `a9lim.github.io/`, open `/physsim`. Enable Relativity + Signal Delay (disable Barnes-Hut). Place 2 particles. Verify delayed ghost positions appear and track reasonably. Check console for NaN errors.

**Step 4: Commit**
```
fix: correct signal delay NR denominator from dist^2 to dist
```

---

### Task 2: Bug Fix — Darwin Field Momentum Sign for GM

**Files:**
- Modify: `physsim/src/energy.js:96-98`

**Step 1: Negate the GM field momentum**

In `energy.js:96-98`, change:
```js
                    const coeff = mmInvR * 0.5;
                    fieldPx += coeff * (svx + rx * svDotR);
                    fieldPy += coeff * (svy + ry * svDotR);
```
to:
```js
                    const coeff = mmInvR * 0.5;
                    fieldPx -= coeff * (svx + rx * svDotR);
                    fieldPy -= coeff * (svy + ry * svDotR);
```

The GM Darwin Lagrangian has opposite sign from EM: field energy is `+0.5 * mm/r * velTerm` (line 95 is correct), so field momentum must also flip relative to EM.

**Step 2: Verify**

Enable Gravity + Gravitomagnetic, disable Coulomb/Magnetic. Place two massive particles orbiting each other. Check Stats tab: momentum drift should be smaller than before (the GM field momentum was being added in the wrong direction).

**Step 3: Commit**
```
fix: negate GM Darwin field momentum to match Lagrangian sign
```

---

### Task 3: Spin-Sourced Fields (Lense-Thirring / Dipole Bz)

**Files:**
- Modify: `physsim/src/forces.js:139-157,159-179`

**Step 1: Add spin-sourced Bz to the magnetic block**

In `forces.js`, after line 156 (the last gradient line in the `magneticEnabled` block), before the closing `}` on line 157, add:

```js
        // Dipole-sourced Bz: static magnetic field from spinning charged body
        // Bz_dipole = +mu_source / r^3 (equatorial field of z-aligned dipole)
        p.Bz += sMagMoment * invR * invRSq;
        // Gradient: d(mu/r^3)/dpx = +3*mu*rx/r^5
        p.dBzdx += 3 * sMagMoment * rx * invRSq * invRSq * invR;
        p.dBzdy += 3 * sMagMoment * ry * invRSq * invRSq * invR;
```

**Step 2: Add spin-sourced Bgz to the gravitomagnetic block**

In `forces.js`, after line 174 (the last gradient line in the `gravitomagEnabled` block), before the frame-drag torque on line 176, add:

```js
        // Spin-sourced Bgz: gravitomagnetic field from spinning massive body
        // Bgz_spin = -2 * L_source / r^3 (GEM analog of dipole field)
        p.Bgz -= 2 * sAngMomentum * invR * invRSq;
        // Gradient: d(-2*L/r^3)/dpx = -6*L*rx/r^5
        p.dBgzdx -= 6 * sAngMomentum * rx * invRSq * invRSq * invR;
        p.dBgzdy -= 6 * sAngMomentum * ry * invRSq * invRSq * invR;
```

**Step 3: Verify**

1. Magnetic test: Place a spinning charged particle at rest (charge=20, spin=0.8). Place a second charged particle nearby with some velocity. With Magnetic enabled, the Boris rotation should now deflect the moving particle even though the source is stationary (it generates Bz from its spin).
2. GM test: Place a spinning massive body at rest (mass=80, spin=0.8). Place a lighter particle in orbit. With Gravitomagnetic enabled, the orbit should precess (Lense-Thirring effect from spin-sourced Bgz).
3. Check no NaN in console.

**Step 4: Commit**
```
feat: add spin-sourced Bz and Bgz fields for Lense-Thirring effect
```

---

### Task 4: Spin-Curvature Coupling (Mathisson-Papapetrou / Stern-Gerlach Force)

**Files:**
- Modify: `physsim/src/integrator.js:227` (after GM spin-orbit block)

**Step 1: Add MP/SG force kicks after spin-orbit coupling**

In `integrator.js`, after line 227 (end of GM spin-orbit block), before the frame-drag torque block at line 229, add:

```js
            // Stern-Gerlach / Mathisson-Papapetrou force: center-of-mass kick from
            // spin-field gradient coupling. Uses the same gradients as spin-orbit
            // but applies a translational force instead of a spin torque.
            if (hasMagnetic && relOn && this.spinOrbitEnabled) {
                for (let i = 0; i < n; i++) {
                    const p = particles[i];
                    if (Math.abs(p.angVel) < 1e-10 || Math.abs(p.charge) < 1e-10) continue;
                    const pRSq = p.radius * p.radius;
                    const mu = MAG_MOMENT_K * p.charge * p.angVel * pRSq;
                    // F_SG = +mu * grad(Bz)
                    p.w.x += mu * p.dBzdx * dtSub / p.mass;
                    p.w.y += mu * p.dBzdy * dtSub / p.mass;
                }
            }
            if (hasGM && relOn && this.spinOrbitEnabled) {
                for (let i = 0; i < n; i++) {
                    const p = particles[i];
                    if (Math.abs(p.angVel) < 1e-10) continue;
                    const pRSq = p.radius * p.radius;
                    const L = INERTIA_K * p.mass * p.angVel * pRSq;
                    // F_MP = -L * grad(Bgz)  (GEM sign flip)
                    p.w.x -= L * p.dBgzdx * dtSub / p.mass;
                    p.w.y -= L * p.dBgzdy * dtSub / p.mass;
                }
            }
```

**Step 2: Verify**

Enable Gravity + Gravitomagnetic + Relativity + Spin-Orbit. Place a spinning particle (spin=0.9, mass=5) orbiting a massive body (mass=80). Compare with spin=0: the spinning particle's orbit should differ visibly (MP force deflects it based on spin orientation). Flip sign of spin and verify opposite deflection.

**Step 3: Commit**
```
feat: add Mathisson-Papapetrou and Stern-Gerlach spin-curvature forces
```

---

### Task 5: 1PN Perihelion Precession — Particle & Config Setup

**Files:**
- Modify: `physsim/src/particle.js:19`
- Modify: `physsim/src/forces.js:12,27`
- Modify: `physsim/src/integrator.js:7,30`

**Step 1: Add force1PN Vec2 to Particle**

In `particle.js`, after line 19 (`this.forceGravitomag = new Vec2(0, 0);`), add:
```js
        this.force1PN = new Vec2(0, 0);
```

**Step 2: Reset force1PN in resetForces**

In `forces.js`, in the `resetForces()` function (line 12-27), after line 18 (`p.forceGravitomag.set(0, 0);`), add:
```js
        p.force1PN.set(0, 0);
```

**Step 3: Add onePNEnabled toggle to Physics**

In `integrator.js`, after line 30 (`this.spinOrbitEnabled = false;`), add:
```js
        this.onePNEnabled = false;
```

And in the `_toggles` object (after line 47), add:
```js
            onePNEnabled: false,
```

And in `_syncToggles()` (after line 56), add:
```js
        this._toggles.onePNEnabled = this.onePNEnabled;
```

**Step 4: Commit**
```
feat: add 1PN particle force vector and physics toggle scaffolding
```

---

### Task 6: 1PN Force Computation

**Files:**
- Modify: `physsim/src/forces.js:104-180`

**Step 1: Add 1PN force computation to pairForce**

In `forces.js`, after the Coulomb block (line 137, before `if (toggles.magneticEnabled)`), add:

```js
    if (toggles.onePNEnabled) {
        // 1PN Einstein-Infeld-Hoffmann correction (natural units, G=c=1)
        // Uses coordinate velocities (not proper velocity) per EIH formulation
        const pvx = p.vel.x, pvy = p.vel.y;
        const v1Sq = pvx * pvx + pvy * pvy;
        const v2Sq = svx * svx + svy * svy;
        const v1DotV2 = pvx * svx + pvy * svy;
        const nx = rx * invR, ny = ry * invR;
        const nDotV1 = nx * pvx + ny * pvy;
        const nDotV2 = nx * svx + ny * svy;

        // Radial term coefficient
        const radial = -v1Sq - 2 * v2Sq + 4 * v1DotV2
            + 1.5 * nDotV2 * nDotV2
            + 5 * p.mass * invR + 4 * sMass * invR;

        // Tangential term coefficient (along v1 - v2)
        const tangential = 4 * nDotV1 - 3 * nDotV2;
        const dvx = pvx - svx, dvy = pvy - svy;

        // a_1PN = (m2/r^2) * { n * radial + (v1-v2) * tangential }
        const base = sMass * invRSq * invR;
        const fx = base * (rx * radial + dvx * tangential * r);
        const fy = base * (ry * radial + dvy * tangential * r);

        // Accumulate into total force (for kicks) and per-type display vector
        out.x += fx;
        out.y += fy;
        p.force1PN.x += fx;
        p.force1PN.y += fy;
    }
```

Note: `rx * radial` gives `n * r * radial`, and `base = m2 / r^3`, so `base * rx * radial = m2 * nx * radial / r^2`. For the tangential term, `base * dvx * tangential * r = m2 * (v1-v2) * tangential / r^2`. This matches the EIH formula.

**Step 2: Pass toggles to 1PN in BH mode**

The existing BH aggregated-node path (`calculateForce` in forces.js:209-212) calls `pairForce()` with aggregated mass and velocity. 1PN will automatically work through this path since it uses the same `pairForce()` function.

**Step 3: Verify**

Enable Gravity + 1PN + Relativity. Load the Solar System preset. A planet in elliptical orbit should show perihelion precession (the orbit doesn't close — the ellipse slowly rotates). Without 1PN, orbits should close.

**Step 4: Commit**
```
feat: implement 1PN EIH gravitational correction in pairForce
```

---

### Task 7: 1PN Velocity Verlet Correction in Integrator

**Files:**
- Modify: `physsim/src/integrator.js`

**Step 1: Store pre-step 1PN force**

Before step 1 (line 149), add storage of the current 1PN force for each particle:

```js
            // Store 1PN forces for velocity-Verlet correction (recomputed after drift)
            const has1PN = toggles.onePNEnabled;
            if (has1PN) {
                for (let i = 0; i < n; i++) {
                    const p = particles[i];
                    if (!p._f1pnOld) p._f1pnOld = { x: 0, y: 0 };
                    p._f1pnOld.x = p.force1PN.x;
                    p._f1pnOld.y = p.force1PN.y;
                }
            }
```

**Step 2: Add correction kick after drift**

After the drift+history block (after `this.simTime += dtSub;`, around line 360), before the tree rebuild, add the velocity-Verlet correction:

```js
            // 1PN velocity-Verlet correction: recompute 1PN at new positions/velocities
            // and apply half the difference as a correction kick
            if (has1PN) {
                // Derive velocities from updated proper velocities (after drift)
                for (let i = 0; i < n; i++) {
                    const p = particles[i];
                    const invG = relOn ? 1 / Math.sqrt(1 + p.w.magSq()) : 1;
                    p.vel.x = p.w.x * invG;
                    p.vel.y = p.w.y * invG;
                }
                // Reset only 1PN force vectors
                for (let i = 0; i < n; i++) particles[i].force1PN.set(0, 0);
                // Recompute 1PN forces at new state
                if (this.barnesHutEnabled) {
                    // BH: need a quick tree for the correction
                    const corrRoot = this.pool.build(this.boundary.x, this.boundary.y, this.boundary.w, this.boundary.h, particles);
                    for (const p of particles) {
                        // Temporarily zero non-1PN toggles to only get 1PN
                        const savedToggles = { ...toggles };
                        toggles.gravityEnabled = false;
                        toggles.coulombEnabled = false;
                        toggles.magneticEnabled = false;
                        toggles.gravitomagEnabled = false;
                        const tempForce = { x: 0, y: 0, set(a,b){ this.x=a; this.y=b; } };
                        // Actually, simpler: just recompute 1PN pairwise for correction
                        // since it's a small correction, exact pairwise is fine even in BH mode
                        Object.assign(toggles, savedToggles);
                    }
                    // Simpler approach: 1PN correction is O(v^2/c^2), small correction
                    // Use exact pairwise even in BH mode for the correction term
                    for (let i = 0; i < n; i++) {
                        const p = particles[i];
                        for (let j = 0; j < n; j++) {
                            if (i === j) continue;
                            const o = particles[j];
                            // Inline 1PN-only force computation
                            const rx = o.pos.x - p.pos.x;
                            const ry = o.pos.y - p.pos.y;
                            const rSq = rx * rx + ry * ry + SOFTENING_SQ;
                            const r = Math.sqrt(rSq);
                            const invR = 1 / r;
                            const invRSq = 1 / rSq;
                            const nx = rx * invR, ny = ry * invR;
                            const pvx = p.vel.x, pvy = p.vel.y;
                            const svx = o.vel.x, svy = o.vel.y;
                            const v1Sq = pvx * pvx + pvy * pvy;
                            const v2Sq = svx * svx + svy * svy;
                            const v1DotV2 = pvx * svx + pvy * svy;
                            const nDotV1 = nx * pvx + ny * pvy;
                            const nDotV2 = nx * svx + ny * svy;
                            const radial = -v1Sq - 2 * v2Sq + 4 * v1DotV2
                                + 1.5 * nDotV2 * nDotV2
                                + 5 * p.mass * invR + 4 * o.mass * invR;
                            const tangential = 4 * nDotV1 - 3 * nDotV2;
                            const dvx = pvx - svx, dvy = pvy - svy;
                            const base = o.mass * invRSq * invR;
                            p.force1PN.x += base * (rx * radial + dvx * tangential * r);
                            p.force1PN.y += base * (ry * radial + dvy * tangential * r);
                        }
                    }
                } else {
                    // Exact pairwise 1PN recomputation
                    for (let i = 0; i < n; i++) {
                        const p = particles[i];
                        for (let j = 0; j < n; j++) {
                            if (i === j) continue;
                            const o = particles[j];
                            const rx = o.pos.x - p.pos.x;
                            const ry = o.pos.y - p.pos.y;
                            const rSq = rx * rx + ry * ry + SOFTENING_SQ;
                            const r = Math.sqrt(rSq);
                            const invR = 1 / r;
                            const invRSq = 1 / rSq;
                            const nx = rx * invR, ny = ry * invR;
                            const pvx = p.vel.x, pvy = p.vel.y;
                            const svx = o.vel.x, svy = o.vel.y;
                            const v1Sq = pvx * pvx + pvy * pvy;
                            const v2Sq = svx * svx + svy * svy;
                            const v1DotV2 = pvx * svx + pvy * svy;
                            const nDotV1 = nx * pvx + ny * pvy;
                            const nDotV2 = nx * svx + ny * svy;
                            const radial = -v1Sq - 2 * v2Sq + 4 * v1DotV2
                                + 1.5 * nDotV2 * nDotV2
                                + 5 * p.mass * invR + 4 * o.mass * invR;
                            const tangential = 4 * nDotV1 - 3 * nDotV2;
                            const dvx = pvx - svx, dvy = pvy - svy;
                            const base = o.mass * invRSq * invR;
                            p.force1PN.x += base * (rx * radial + dvx * tangential * r);
                            p.force1PN.y += base * (ry * radial + dvy * tangential * r);
                        }
                    }
                }
                // Apply correction kick: w += (F_1PN_new - F_1PN_old) * dt/2 / m
                for (let i = 0; i < n; i++) {
                    const p = particles[i];
                    const halfDtOverM = dtSub * 0.5 / p.mass;
                    p.w.x += (p.force1PN.x - p._f1pnOld.x) * halfDtOverM;
                    p.w.y += (p.force1PN.y - p._f1pnOld.y) * halfDtOverM;
                }
            }
```

Note: The duplicated 1PN computation should be extracted into a helper function for DRY. Refactor after verifying correctness.

**Step 3: Extract 1PN helper to avoid duplication**

Create a standalone function `compute1PNPairwise(particles, SOFTENING_SQ)` in `forces.js` that only computes the 1PN force on all particles (resets and fills `force1PN`). Use it both in the Verlet correction and as the main 1PN computation path.

**Step 4: Verify**

Solar System preset with 1PN + Relativity. A tight elliptical orbit should visibly precess. Mercury-like orbit: mass=80 center, mass=1 particle at r=120 with v=0.7*v_circular. The orbit should not close.

**Step 5: Commit**
```
feat: add velocity-Verlet correction for 1PN force accuracy
```

---

### Task 8: 1PN Potential Energy

**Files:**
- Modify: `physsim/src/potential.js:102-119`

**Step 1: Add 1PN PE term to pairPE**

In `potential.js`, the `pairPE()` function needs access to velocities. Currently it doesn't take velocity parameters. Since the 1PN PE depends on velocities, we need to add velocity parameters.

Change `pairPE` signature (line 102) to accept velocities:
```js
export function pairPE(p, sx, sy, svx, svy, sMass, sCharge, sAngVel, sMagMoment, sAngMomentum, toggles) {
```

After line 117 (`pe -= (pAngMomentum * sAngMomentum) * invR * invRSq;`), add:
```js
    if (toggles.onePNEnabled) {
        // 1PN PE: -(m1*m2/r) * [3(v1^2+v2^2)/2 - 7(v1.v2)/2 - (v1.n)(v2.n)/2 + m1/r + m2/r]
        const pvx = p.vel.x, pvy = p.vel.y;
        const v1Sq = pvx * pvx + pvy * pvy;
        const v2Sq = svx * svx + svy * svy;
        const v1DotV2 = pvx * svx + pvy * svy;
        const nx = rx * invR, ny = ry * invR;
        const v1DotN = pvx * nx + pvy * ny;
        const v2DotN = svx * nx + svy * ny;
        pe -= p.mass * sMass * invR * (
            1.5 * (v1Sq + v2Sq) - 3.5 * v1DotV2 - 0.5 * v1DotN * v2DotN
            + p.mass * invR + sMass * invR
        );
    }
```

Update all call sites of `pairPE` to pass velocity arguments (in `computePE` and `treePE`). In the pairwise loop (line 32), add `o.vel.x, o.vel.y`. In the BH aggregated node path (line 76), use `avgVx, avgVy` (compute from total momentum / total mass).

**Step 2: Verify**

Enable 1PN. Check Stats tab: total energy (KE + PE + Field) should be better conserved with the 1PN PE correction than without.

**Step 3: Commit**
```
feat: add 1PN correction to potential energy computation
```

---

### Task 9: 1PN UI Toggle & Info Tip

**Files:**
- Modify: `physsim/index.html:190` (after gravitomagnetic toggle)
- Modify: `physsim/src/ui.js:110,116,154,296`
- Modify: `physsim/src/renderer.js:9-14,274,285-289`
- Modify: `physsim/styles.css` (add `.tog-onepn` color)

**Step 1: Add HTML toggle**

In `index.html`, after line 190 (gravitomagnetic toggle closing `</div></div>`), add:
```html
                        <div class="ctrl-row ctrl-sub"><label for="onepn-toggle">1PN <button class="info-trigger" data-info="onepn" aria-label="1PN info">?</button></label>
                            <div class="tog-wrap"><input type="checkbox" id="onepn-toggle" role="switch" aria-checked="false">
                            <label for="onepn-toggle" class="tog tog-onepn"><span class="tog-thumb"></span></label></div></div>
```

Note: unchecked by default (1PN off by default).

**Step 2: Add toggle binding in ui.js**

In `ui.js`, in the `forceToggles` array (after line 110), add:
```js
        { id: 'onepn-toggle', prop: 'onePNEnabled' },
```

**Step 3: Add dependency logic**

1PN requires both Gravity and Relativity. In `ui.js`, after the BH dependency block (line 154), add:

```js
    // ─── 1PN dependency: requires Gravity + Relativity ───
    const gravEl = document.getElementById('gravity-toggle');
    const pnEl = document.getElementById('onepn-toggle');
    const updatePnDeps = () => {
        const ok = gravEl.checked && relativityEl.checked;
        pnEl.disabled = !ok;
        pnEl.closest('.ctrl-row').classList.toggle('ctrl-disabled', !ok);
    };
    gravEl.addEventListener('change', updatePnDeps);
    relativityEl.addEventListener('change', updatePnDeps);
    updatePnDeps();
```

**Step 4: Add info tip**

In `ui.js`, in the `infoData` object (before the closing `};` on line 298), add:
```js
        onepn: { title: '1PN Correction', body: 'First post-Newtonian correction to gravity (Einstein\u2013Infeld\u2013Hoffmann equations). Adds O(v\u00B2/c\u00B2) velocity- and mass-dependent terms to the gravitational force. Produces perihelion precession (~6\u03C0M/a(1\u2212e\u00B2) rad/orbit). Integrated with velocity-Verlet for second-order accuracy on velocity-dependent terms. Requires Gravity and Relativity.' },
```

**Step 5: Add force component color for renderer**

In `renderer.js`, add to `_forceCompColors` (after line 13):
```js
    onepn:       { light: _r(_PAL.extended.yellow, 0.7), dark: _r(_PAL.extended.yellow, 0.8) },
```

In the total force sum (line 274), add `+ p.force1PN.x` and `+ p.force1PN.y`.

In the `forces` array (line 285-289), add:
```js
            { key: 'force1PN', color: _forceCompColors.onepn[theme] },
```

**Step 6: Add toggle color in styles.css**

Add the toggle background color (matching yellow extended palette):
```css
.tog-onepn { --tog-color: var(--ext-yellow, #CCA84C); }
```

**Step 7: Verify**

The 1PN toggle should appear as a sub-toggle under Gravity (indented, like Gravitomagnetic). It should be disabled when Gravity or Relativity is off. When enabled, force component vectors should show a yellow 1PN component. The info tip should appear on hover/tap.

**Step 8: Commit**
```
feat: add 1PN toggle UI with dependency logic and force visualization
```

---

### Task 10: Radiation Pressure (Photon Absorption)

**Files:**
- Modify: `physsim/src/photon.js:4-10`
- Modify: `physsim/src/integrator.js` (after collision handling, around line 369)

**Step 1: Add emitter tracking to Photon**

In `photon.js`, update the constructor (lines 4-10):
```js
    constructor(x, y, vx, vy, energy, emitterId = -1) {
        this.pos = new Vec2(x, y);
        this.vel = new Vec2(vx, vy);
        this.energy = energy;
        this.lifetime = 0;
        this.alive = true;
        this.emitterId = emitterId;
        this.age = 0;  // substep counter for self-absorption guard
    }
```

**Step 2: Pass emitterId when spawning photons**

In `integrator.js`, at the photon spawn (around line 323), change:
```js
                            this.sim.photons.push(new Photon(
                                p.pos.x, p.pos.y,
                                Math.cos(spawnAngle), Math.sin(spawnAngle),
                                dE
                            ));
```
to:
```js
                            this.sim.photons.push(new Photon(
                                p.pos.x, p.pos.y,
                                Math.cos(spawnAngle), Math.sin(spawnAngle),
                                dE, p.id
                            ));
```

**Step 3: Add photon absorption loop**

In `integrator.js`, after the collision handling block (around line 369, after `handleCollisions(...)`), add:

```js
            // Photon absorption: transfer momentum from photons to particles
            if (this.radiationEnabled && this.sim && this.sim.photons.length > 0) {
                const photons = this.sim.photons;
                for (let pi = photons.length - 1; pi >= 0; pi--) {
                    const ph = photons[pi];
                    if (!ph.alive) continue;
                    ph.age++;
                    // Query tree for nearby particles
                    const candidates = this.pool.query(lastRoot >= 0 ? lastRoot : root,
                        ph.pos.x, ph.pos.y, SOFTENING, SOFTENING);
                    for (const p of candidates) {
                        // Self-absorption guard: skip emitter for first 2 substeps
                        if (p.id === ph.emitterId && ph.age < 3) continue;
                        const dx = ph.pos.x - p.pos.x;
                        const dy = ph.pos.y - p.pos.y;
                        if (dx * dx + dy * dy < p.radius * p.radius) {
                            // Absorb: transfer photon momentum to particle
                            const impulse = ph.energy; // p = E/c = E (c=1)
                            p.w.x += impulse * ph.vel.x / p.mass;
                            p.w.y += impulse * ph.vel.y / p.mass;
                            // Fix energy bookkeeping
                            this.sim.totalRadiated -= ph.energy;
                            this.sim.totalRadiatedPx -= ph.energy * ph.vel.x;
                            this.sim.totalRadiatedPy -= ph.energy * ph.vel.y;
                            ph.alive = false;
                            break; // photon absorbed, stop checking particles
                        }
                    }
                }
            }
```

Note: The `lastRoot` variable tracks the most recent tree from the substep loop. Use it for the quadtree query.

**Step 4: Verify**

Enable Radiation + Relativity. Place a heavy charged body (mass=80, charge=30) and a lighter charged body orbiting it. Accelerating charges emit photons. Place a third particle far away — when photons reach it, it should receive a small momentum kick. Check that `totalRadiated` in Stats decreases when photons are absorbed.

**Step 5: Commit**
```
feat: add photon absorption with momentum transfer (radiation pressure)
```

---

### Task 11: Update CLAUDE.md

**Files:**
- Modify: `physsim/CLAUDE.md`

**Step 1: Update force types documentation**

Add 1PN to the radial forces list:
```
- **1PN** (`onePNEnabled`): EIH O(v^2/c^2) correction, velocity-Verlet integration
```

Add spin-sourced fields note to Magnetic and Gravitomagnetic descriptions:
```
Includes spin-sourced Bz from magnetic moments (mu/r^3) and spin-sourced Bgz from angular momentum (-2L/r^3).
```

Add MP/SG force to spin-orbit description:
```
Also applies Stern-Gerlach (F = mu*grad(Bz)) and Mathisson-Papapetrou (F = -L*grad(Bgz)) center-of-mass forces.
```

Add radiation pressure note:
```
Photon absorption: photons transfer momentum E*dir to absorbing particles, fix totalRadiated bookkeeping.
```

**Step 2: Update toggle dependencies**

```
Gravity → Gravitomagnetic (existing)
        → 1PN (new, requires Gravity + Relativity)
```

**Step 3: Update sign conventions**

Document the gradient signs for spin-sourced fields.

**Step 4: Commit**
```
docs: update CLAUDE.md with batch 3 physics additions
```
