# Simulation Improvements Batch 2 — Design Document

**Date**: 2026-03-03
**Scope**: 4 physsim tasks (P-Ga, P-Ba, P-Aa, P-Ab), 3 biosim tasks (B-Fa, B-Ga, B-Ba)

---

## Task Summary

| ID | Project | Title | Complexity |
|----|---------|-------|------------|
| B-Ba | biosim | Remove NADPH passive drain in autoplay | Trivial |
| B-Ga | biosim | Enable PPP for Anaerobe and Archaeon | Trivial |
| B-Fa | biosim | Add UCP visual complex to membrane | Medium |
| P-Aa | physsim | Photon spawn direction toggle (mouse direction) | Medium |
| P-Ab | physsim | Abraham-Lorentz radiation via Landau-Lifshitz | High |
| P-Ba | physsim | Combined surface force disintegration | Medium |
| P-Ga | physsim | Gravitomagnetic spin coupling (spin-orbit + frame-drag) | High |

---

## B-Ba: Remove NADPH Passive Drain in Autoplay

**File**: `biosim/src/autoplay.js` (lines 28-33)

**Change**: Remove `store.nadph -= 2` from the passive drain tick. Keep the `-3 ATP` drain intact. Other NADPH consumers (Calvin -12/cycle, FA synthesis -14/cycle, glycolysis-reverse -2, ROS scavenging -1/tick) remain unchanged.

**Rationale**: The passive NADPH drain was a simplification to simulate biosynthetic demand, but it causes NADPH to deplete unrealistically fast in organisms that depend on PPP for NADPH production.

---

## B-Ga: Enable PPP for Anaerobe and Archaeon

**File**: `biosim/src/organisms.js`

**Changes**:
- Obligate Anaerobe: set `ppp: true` in pathways object, remove `ppp` from `lockedReason`
- Archaeon: set `ppp: true` in pathways object, remove `ppp` from `lockedReason`

Both organisms will use the standard PPP implementation (G6PDH, Lactonase, 6PGDH) with existing NADPH regulation (G6PDH activated when NADPH < 20%, inhibited when > 80%).

---

## B-Fa: Add Uncoupling Protein to Membrane

### Visual Design
- **Position**: 14th membrane complex at `colW(14)` in layout.js
- **Shape**: Transmembrane channel pore (narrower than ATP Synthase, no F1 head — simple barrel/funnel)
- **Color role**: `uncoupling` → `_BASE.red` (heat/dissipation connotation)
- **Glow**: Only when `simState.uncouplingEnabled && store.protonGradient > 0`
- **Dimmed**: Faded when uncoupling is disabled (alpha ~0.3)
- **Proton arrow**: Downward "H+" label when active (protons leak down gradient)
- **Proton particles**: Spawn downward from UCP position during leak ticks (when uncoupling enabled)

### Reaction Integration
No new reaction step. Existing `protonLeakTick()` in autoplay.js handles the 10% gradient leak when uncoupling is enabled. The only addition is visual proton particle spawning from the UCP's canvas position.

### Files to Modify
1. `biosim/src/enzymes.js` — `drawUCP()` shape function + `uncoupling` role in `_ROLE`
2. `biosim/src/layout.js` — increment `numComplexes` to 14, add UCP at `colW(14)`
3. `biosim/src/renderer.js` — draw UCP in `drawETCChain()`, add hitbox, add `_etcInfoKey` entry
4. `biosim/src/autoplay.js` — spawn downward proton particles from UCP position during leak tick
5. `biosim/src/info.js` — add `UCP` entry (Uncoupling Protein 1, dissipates gradient as heat)

---

## P-Aa: Photon Spawn Direction Toggle

### Mouse Direction Tracking
- Add `prevMouseWorld` Vec2 to `InputHandler` (previous mouse world position)
- On each `mousemove`, shift: `prevMouseWorld = currentPos` (before update), `currentPos = new position`
- Direction = `normalize(currentPos - prevMouseWorld)`
- Fallback: if positions are identical (stationary mouse), use random direction

### Spawn Mode
- Add 4th mode `'photon'` alongside existing `place`/`shoot`/`orbit`
- New mode button in the mode toggle group (with photon/light SVG icon)
- On click (mouseup in photon mode):
  - Spawn a `Photon` at click world position
  - Velocity = normalized mouse direction (magnitude 1 = speed of light)
  - Energy = 1.0 (default; could add slider later)
  - Adds to `sim.photons` array (same pool as Larmor radiation photons)
  - Respects `MAX_PHOTONS` cap

### Files to Modify
1. `physsim/src/input.js` — add `prevMouseWorld`, direction tracking in `onMouseMove`, photon spawn in `onMouseUp`
2. `physsim/src/ui.js` — add "Photon" to mode toggle setup
3. `physsim/index.html` — add photon mode button to `.mode-toggles`

---

## P-Ab: Abraham-Lorentz Radiation via Landau-Lifshitz Approximation

### Problem
Current implementation (physics.js lines 146-199) computes Larmor power and directly rescales `|w|` to drain KE. This causes NaN when `dE > KE` and doesn't apply radiation reaction as a proper force.

### Solution: Landau-Lifshitz Force
Replace KE drain with the Landau-Lifshitz radiation reaction force applied as a kick in the Boris integrator.

**Particle changes**: Add `prevForce` Vec2 (stores force from previous substep).

**LL force computation** (after Boris rotation, before drift):
```
τ = 2 * LARMOR_K * q² / m                    // radiation reaction timescale

// Jerk term: rate of change of external force
jerkX = (force.x - prevForce.x) / dt
jerkY = (force.y - prevForce.y) / dt

// Schott damping term: |F|² · v / m (dominant energy dissipation)
Fsq = force.x² + force.y²
schottX = Fsq * vel.x / mass
schottY = Fsq * vel.y / mass

// Total LL force
F_radX = τ * (jerkX + schottX)
F_radY = τ * (jerkY + schottY)

// Relativistic correction: divide by γ³
if (relativity) { F_rad /= γ³ }
```

**Integration**: Applied as additional kick to `p.w`:
```
p.w.x += F_radX * dt / p.mass
p.w.y += F_radY * dt / p.mass
```

**Safety guards**:
- Clamp `|F_rad * dt / mass|` to `LL_FORCE_CLAMP * |w|` (default 0.5) to prevent sign flips
- Skip if `|w| < 1e-10`
- NaN check after application

**Photon emission**: Computed from Larmor power `P = LARMOR_K * q² * a²` for visualization. Photon energy capped to actual energy change.

**Energy tracking**: `sim.totalRadiated += |KE_before - KE_after|` (measured, not computed from P).

### Files to Modify
1. `physsim/src/particle.js` — add `prevForce` Vec2
2. `physsim/src/physics.js` — replace lines 146-199 with LL force computation + kick + photon emission
3. `physsim/src/config.js` — add `LL_FORCE_CLAMP = 0.5`

---

## P-Ba: Combined Surface Force Disintegration

### Current State
Tidal breakup checks `tidal_accel > self_gravity` per nearby body. Fragmentation spawns `FRAGMENT_COUNT` (3) pieces.

### Design: Generalized Breakup Condition
Compute total outward force at particle surface vs self-gravitational binding:

```
// Binding force (inward)
F_bind = mass / radius²                        // surface self-gravity (G=1)

// Outward forces
F_tidal = TIDAL_STRENGTH * M_other * radius / r³   // tidal stretching (existing)
F_centrifugal = angVel² * radius                    // spin centrifugal
F_coulomb_self = charge² / (4 * radius²)            // Coulomb self-repulsion

// Breakup condition
if (F_tidal + F_centrifugal + F_coulomb_self > F_bind) → disintegrate
```

**Gating**: Existing `tidalEnabled` toggle gates the entire surface force check. The check runs in the existing tidal breakup loop (per-pair for tidal, but centrifugal and Coulomb self-repulsion are per-particle and only need checking once per particle per frame — if either alone exceeds binding, the particle disintegrates without needing a nearby body).

**Per-particle self-disruption**: Add a separate check outside the pair loop:
- If `F_centrifugal + F_coulomb_self > F_bind`, particle disintegrates on its own (no tidal partner needed)
- This handles fast-spinning or highly-charged bodies that tear themselves apart

**Fragment properties**: Mass/3, charge/3, inherit parent spin, tangential velocity kick from spin (existing logic).

### Files to Modify
1. `physsim/src/physics.js` — extend `checkTidalBreakup()` with centrifugal + Coulomb terms, add per-particle self-disruption check
2. `physsim/main.js` — ensure self-disruption fragments are spawned in the same code path

---

## P-Ga: Gravitomagnetic Spin Coupling

### A. GM Spin-Orbit Coupling (Energy Exchange)

Mirrors the existing EM spin-orbit coupling (physics.js lines 127-144) but uses gravitomagnetic field `Bgz` instead of `Bz`.

```
// Angular momentum (gravitational "magnetic moment" analog)
L = INERTIA_K * mass * angVel * radius²

// GM field gradient (same form as EM)
dBgz_dr = -3 * Bgz / r
dBgz_dx = dBgz_dr * dx / r
dBgz_dy = dBgz_dr * dy / r

// Energy exchange (GM analog of -μ·(v·∇Bz))
dE_spin = -L * (vel.x * dBgz_dx + vel.y * dBgz_dy) * dt

// Update spin: spin += dE_spin / (I * angVel)
// Guard: only if I*angVel > threshold
```

**Gating**: `gravitomagEnabled && relativityEnabled` (mirrors EM spin-orbit gate pattern).

### B. Frame-Dragging Spin-Spin Torque (Lense-Thirring-like)

In 2D (scalar spin), frame-dragging drives nearby spins toward co-rotation:

```
// Alignment torque from mass_other's angular momentum
torque = FRAME_DRAG_K * m_other * (angVel_other - angVel_self) / r³

// Update spin
dSpin = torque * dt / I_self
p.spin += dSpin
```

This naturally:
- Drives anti-aligned spins toward alignment
- Has zero effect when already co-rotating
- Scales as 1/r³ (frame-dragging falloff)

**New constant**: `FRAME_DRAG_K` in config.js (default 0.1, tunable).

**Gating**: `gravitomagEnabled` (no relativity requirement — frame-dragging is inherently a GR effect but the toggle gates all GM).

**Barnes-Hut**: Aggregate nodes already store `totalAngularMomentum`. Frame-dragging torque uses `node.totalAngularMomentum / node.totalMass` as approximate `angVel_other` for distant clusters.

### Files to Modify
1. `physsim/src/physics.js` — add GM spin-orbit block after EM spin-orbit, add frame-drag torque in force computation loop
2. `physsim/src/config.js` — add `FRAME_DRAG_K = 0.1`
3. `physsim/src/quadtree.js` — verify `totalAngularMomentum` is usable for frame-drag (likely already sufficient)

---

## Dependency Order

No hard dependencies between tasks. Recommended implementation order (simplest first):

1. **B-Ba** — 1-line change
2. **B-Ga** — 2-line change
3. **B-Fa** — medium, self-contained in biosim
4. **P-Aa** — medium, self-contained input change
5. **P-Ba** — medium, extends existing tidal code
6. **P-Ab** — high, replaces core radiation logic
7. **P-Ga** — high, new physics terms
