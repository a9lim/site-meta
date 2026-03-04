# Physsim Physics Batch 3: Post-Newtonian Effects & Bug Fixes

**Date**: 2026-03-03
**Scope**: physsim only
**Goal**: Add missing post-Newtonian gravitational effects, spin-curvature coupling, radiation pressure, and fix two existing bugs.

## Context

The simulation currently models:
- Newtonian gravity + Coulomb (inverse-square)
- Magnetic dipole + gravitomagnetic dipole (inverse-cube)
- Lorentz + linear gravitomagnetic via Boris rotation
- Landau-Lifshitz radiation reaction (EM only)
- Spin-orbit energy transfer + frame-dragging spin alignment
- Signal delay (finite-speed force propagation)

Missing from a complete post-Newtonian treatment:
- 1PN velocity-dependent corrections to gravity (perihelion precession)
- Spin-sourced gravitomagnetic/magnetic fields (Lense-Thirring orbital precession)
- Spin-curvature coupling force (Mathisson-Papapetrou / Stern-Gerlach)
- Radiation pressure from photon absorption

## 1. 1PN Gravitational Correction (Perihelion Precession)

### Physics

The 1PN Einstein-Infeld-Hoffmann correction adds O(v^2/c^2) terms to gravity. For particle 1 accelerated by particle 2 (natural units, G=c=1):

```
a_1PN = (m2/r^2) * {
  n * [-v1^2 - 2*v2^2 + 4*(v1.v2) + (3/2)*(n.v2)^2 + 5*m1/r + 4*m2/r]
  + (v1 - v2) * [4*(n.v1) - 3*(n.v2)]
}
```

where `n = (r2 - r1)/|r2 - r1|` is the unit separation vector.

This produces perihelion precession of ~6*pi*M / (a*(1-e^2)) radians per orbit.

### Toggle

New "1PN" toggle, child of Gravity (sibling to Gravitomagnetic). Requires Relativity enabled.

### Integration: Velocity Verlet

The 1PN force is velocity-dependent, so applying it purely via kicks loses accuracy. Use a velocity-Verlet correction for second-order accuracy:

1. Compute F_1PN at current (v, x) -> F_1PN_old
2. Half-kick: w += (F_E + F_1PN_old) * dt/2 / m
3. Boris rotation (B-like forces, unchanged)
4. Half-kick: w += (F_E + F_1PN_old) * dt/2 / m
5. Drift: x += v * dt
6. Recompute F_1PN at new (v, x) -> F_1PN_new
7. Correction kick: w += (F_1PN_new - F_1PN_old) * dt/2 / m

### Implementation

**forces.js**: New function `compute1PNForce(p, sx, sy, svx, svy, sMass, out)` that computes the EIH 1PN correction. Accumulates into `out` and `p.force1PN` (new per-particle display vector). Uses coordinate velocities (p.vel, not p.w) since EIH is written in coordinate velocity.

**integrator.js**: Store per-particle `_f1pn` Vec2 before substep. After drift, recompute 1PN forces and apply the correction kick `(F_new - F_old) * dt/2 / m`.

**potential.js**: Add 1PN PE correction term:
```
U_1PN = -(m1*m2/r) * [3*(v1^2 + v2^2)/2 - 7*(v1.v2)/2 - (v1.n)*(v2.n)/2 + m1/r + m2/r]
```

**config.js**: No new constants needed.

### Barnes-Hut Compatibility

O(N^2) in pairwise mode, O(N log N) with BH using aggregated node velocities (average velocity = total_momentum / total_mass).

## 2. Spin-Curvature Coupling (Mathisson-Papapetrou / Stern-Gerlach Force)

### Physics

A spinning body doesn't follow a geodesic in curved spacetime. It experiences the Mathisson-Papapetrou force from coupling between its spin and the field gradient. The EM analog is the Stern-Gerlach force.

**EM (Stern-Gerlach force on magnetic dipole in B field gradient):**
```
F_x = +mu * dBz/dx
F_y = +mu * dBz/dy
```
where mu = MAG_MOMENT_K * q * omega * r^2

**GM (Mathisson-Papapetrou force on spin in Bg field gradient):**
```
F_x = -L * dBgz/dx
F_y = -L * dBgz/dy
```
where L = INERTIA_K * m * omega * r^2. Negative sign for GEM (gravity is attractive).

### Key Insight

The field gradients dBz/dx, dBz/dy, dBgz/dx, dBgz/dy are ALREADY computed and stored on each particle (p.dBzdx, etc.) for the existing spin-orbit energy transfer. This force uses them for a center-of-mass kick.

### Toggle

Part of the existing Spin-Orbit toggle (requires Relativity + Magnetic or Gravitomagnetic).

### Implementation

**integrator.js**: After the spin-orbit energy transfer block, add center-of-mass kicks:

```js
// EM Stern-Gerlach force
if (hasMagnetic && relOn && this.spinOrbitEnabled) {
    for (const p of particles) {
        const pRSq = p.radius * p.radius;
        const mu = MAG_MOMENT_K * p.charge * p.angVel * pRSq;
        p.w.x += mu * p.dBzdx * dtSub / p.mass;
        p.w.y += mu * p.dBzdy * dtSub / p.mass;
    }
}

// GM Mathisson-Papapetrou force
if (hasGM && relOn && this.spinOrbitEnabled) {
    for (const p of particles) {
        const pRSq = p.radius * p.radius;
        const L = INERTIA_K * p.mass * p.angVel * pRSq;
        p.w.x -= L * p.dBgzdx * dtSub / p.mass;
        p.w.y -= L * p.dBgzdy * dtSub / p.mass;
    }
}
```

### Effect

Spinning particles orbit differently from non-spinning ones. A spinning particle near a massive body experiences a force that depends on its spin orientation relative to the field gradient.

## 3. Lense-Thirring Enhancement (Spin-Sourced Fields)

### Physics

Currently, Bgz and Bz only include contributions from particles' LINEAR velocity. A spinning body also generates fields from its angular momentum:

**GM spin-sourced field:**
```
Bgz_spin = -2 * L_source / r^3
```
where L = INERTIA_K * m * omega * r^2 (source angular momentum).

**EM spin-sourced field (dipole magnetic field):**
```
Bz_spin = +mu_source / r^3
```
where mu = MAG_MOMENT_K * q * omega * r^2 (source magnetic moment).

Without these, a non-moving spinning body produces zero Bgz/Bz, so it can't frame-drag nearby orbits or deflect passing charged particles via Lorentz force.

### No New Toggle

Completes the existing Gravitomagnetic and Magnetic toggles.

### Implementation

**forces.js** in `pairForce()`, added after existing field accumulations:

```js
if (toggles.gravitomagEnabled) {
    // Spin-sourced Bgz: -2 * L_source / r^3
    p.Bgz -= 2 * sAngMomentum * invR * invRSq;

    // Gradient: d(-2*L/r^3)/dpx = -6*L*rx/r^5
    p.dBgzdx -= 6 * sAngMomentum * rx * invRSq * invRSq * invR;
    p.dBgzdy -= 6 * sAngMomentum * ry * invRSq * invRSq * invR;
}

if (toggles.magneticEnabled) {
    // Dipole-sourced Bz: +mu_source / r^3
    p.Bz += sMagMoment * invR * invRSq;

    // Gradient: d(mu/r^3)/dpx = +3*mu*rx/r^5
    p.dBzdx += 3 * sMagMoment * rx * invRSq * invRSq * invR;
    p.dBzdy += 3 * sMagMoment * ry * invRSq * invRSq * invR;
}
```

### Gradient Derivation

With `rx = sx - px` (source minus observer) and `rSq = rx^2 + ry^2 + SOFTENING_SQ`:

```
d(rSq)/dpx = -2*rx
d(r^-3)/dpx = d(rSq^(-3/2))/dpx = (-3/2) * rSq^(-5/2) * (-2*rx) = +3*rx/r^5

For Bgz_spin = -2*L/r^3:
  d/dpx = -2*L * 3*rx/r^5 = -6*L*rx/r^5

For Bz_dipole = +mu/r^3:
  d/dpx = +mu * 3*rx/r^5 = +3*mu*rx/r^5
```

### Barnes-Hut Compatibility

The quadtree already aggregates `totalMagneticMoment` and `totalAngularMomentum`, so spin-sourced fields work with BH out of the box.

## 4. Radiation Pressure (Photon Absorption)

### Physics

Currently, photons are spawned by the Landau-Lifshitz radiation reaction and displayed visually but don't interact with other particles. The LL force already handles radiation RECOIL on the emitter. Missing: radiation PRESSURE on absorbing particles.

### Implementation

**Photon-particle collision detection** (each substep, inside the fixed-timestep loop):
- For each photon, query the existing quadtree for nearby particles: `pool.query(root, photon.x, photon.y, maxRadius*2, maxRadius*2)`
- If `dist < particle.radius`: absorb the photon
- Complexity: O(P * log N) per substep via quadtree

**Momentum transfer on absorption:**
```js
const impulse = photon.energy;  // p = E/c = E (c=1)
particle.w.x += impulse * photon.dx / particle.mass;
particle.w.y += impulse * photon.dy / particle.mass;
```

**Energy bookkeeping:**
- `sim.totalRadiated -= photon.energy` (fixes existing photon absorption bookkeeping gap)
- `sim.totalRadiatedPx -= photon.energy * photon.dx`
- `sim.totalRadiatedPy -= photon.energy * photon.dy`
- Mark photon for removal; compact array after absorption pass

**Self-absorption guard**: Track `photon.emitterId` and skip absorption by the emitter within a grace period of 2 substeps (track `photon.birthStep`).

### Toggle

Part of the existing Radiation toggle. No separate toggle.

## 5. Bug Fixes

### 5a. Signal Delay Newton-Raphson Denominator

**File**: `signal-delay.js:32`

**Current** (wrong power in denominator):
```js
const denom = 1 + (sp.vx * dx + sp.vy * dy) / (dist * dist + 1);
```

The derivative of the light-cone residual is `(v_s . r) / |r| + 1`, requiring division by `dist`, not `dist^2`. Using the softening constant for regularization:

**Fixed**:
```js
const denom = 1 + (sp.vx * dx + sp.vy * dy) / (dist + SOFTENING);
```

This restores correct Newton-Raphson convergence while using the same softening scale as force calculations.

### 5b. Darwin Field Momentum Sign for GM

**File**: `energy.js:96-98`

**Current** (GM field momentum uses same sign as Coulomb):
```js
const coeff = mmInvR * 0.5;
fieldPx += coeff * (svx + rx * svDotR);
fieldPy += coeff * (svy + ry * svDotR);
```

The GM Darwin Lagrangian has opposite sign from EM. The GM field momentum should be subtracted:

**Fixed**:
```js
fieldPx -= coeff * (svx + rx * svDotR);
fieldPy -= coeff * (svy + ry * svDotR);
```

## Summary

| # | Feature | Type | Toggle | Integration | BH? |
|---|---------|------|--------|-------------|-----|
| 1 | 1PN perihelion precession | New force | New "1PN" under Gravity (req. Relativity) | Velocity Verlet | Yes |
| 2 | Spin-curvature (MP/SG) | New force | Existing Spin-Orbit | E-like kicks | Yes |
| 3 | Spin-sourced Bgz/Bz | Field completion | Existing Magnetic/Gravitomagnetic | pairForce() accumulation | Yes |
| 4 | Radiation pressure | Photon absorption | Existing Radiation | O(P*logN) quadtree | Yes |
| 5a | Signal delay NR fix | Bug fix | -- | denominator: dist+SOFTENING | -- |
| 5b | Darwin momentum sign | Bug fix | -- | negate GM field momentum | -- |

### Files Modified

- `forces.js`: 1PN force function, spin-sourced field contributions
- `integrator.js`: 1PN velocity Verlet, MP force kicks, photon absorption loop
- `potential.js`: 1PN PE correction
- `energy.js`: Darwin momentum sign fix, 1PN field energy if needed
- `signal-delay.js`: NR denominator fix
- `config.js`: SOFTENING import in signal-delay.js
- `ui.js`: New 1PN toggle wiring, info tip text
- `particle.js`: New `force1PN` Vec2, `_f1pn` Vec2 for Verlet
- `photon.js`: Add `emitterId` and `birthStep` fields

### Toggle Dependency Update

```
Gravity → Gravitomagnetic (existing)
        → 1PN (new, requires Relativity)
Coulomb → Magnetic (existing)
Relativity → Radiation (existing, now includes photon absorption)
           → Spin-Orbit (existing, now includes MP/SG force)
Relativity + BH off → Signal Delay (existing)
Tidal (independent)
```
