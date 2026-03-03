# Simulation Improvements Design

**Date:** 2026-03-02
**Scope:** physsim (9), biosim (5), gerry (4) — 18 total improvements
**Priority:** Feature completeness > Educational value > Realism

---

## Physsim Improvements

### P-A: Larmor Radiation

Accelerating charged particles emit photon particles and lose kinetic energy.

**Physics:** Radiated power `P = q²a²/(6πε₀c³)`. In natural units (c=1), `P = q²a²/6π`. Each frame, subtract `P·dt` from particle KE by reducing `|w|`. Spawn visual photon particle flying radially outward from the particle.

**Implementation:**
- New `photon` entity type in `particle.js` with position, velocity (always c), energy, lifetime
- Array `sim.photons[]` managed in `main.js` loop
- In `physics.js` after force calculation: compute `a² = |force/m|²`, compute `P`, reduce `|w|` by `P·dt/γ³` (relativistic correction)
- Emission threshold: only emit visible photon when `P·dt > threshold` (prevents spam from low-acceleration particles)
- Renderer draws photons as small glowing dots with short trails
- New toggle in Settings: "Radiation" (force toggle row)
- New energy stat sub-row: "Radiated" (cumulative energy lost to radiation)

**Dependencies:** None (but P-C builds on this)

### P-B: Tidal Forces / Roche Limit

Differential gravity across a particle's diameter can shred it.

**Physics:** Tidal acceleration `a_tidal = 2GM·r_particle / d³` where d is distance to the massive body. Roche limit: particle breaks when `a_tidal > self-gravity ∝ m/r²`. For a fluid body: `d_Roche = r_primary · (2·M_primary/m_particle)^(1/3)`.

**Implementation:**
- In `physics.js` force loop: when gravity enabled, compute tidal stress on each particle from its strongest gravitational neighbor
- Compare to binding threshold `G·m/(r²)` (self-gravity at surface)
- When exceeded: replace particle with 2-4 smaller fragments conserving total mass, momentum, angular momentum
- Fragment radii: `r_frag = r_orig / n^(1/3)`, charges split proportionally
- New config constant `TIDAL_ENABLED` (default false), toggle in Settings
- Visual: brief flash animation at breakup point

**Dependencies:** None

### P-C: Radiation Pressure

Larmor photons carry momentum and push particles they hit.

**Physics:** Photon momentum `p = E/c`. On absorption: apply impulse to target particle. On reflection: apply `2E/c` impulse.

**Implementation:**
- In main loop after photon position update: check for photon-particle collisions (distance < target radius)
- On collision: add `photon.energy / c` (= photon.energy in natural units) to target's proper velocity in the photon's travel direction
- Remove photon after absorption
- Small effect individually but cumulative for many photons around a radiating charge

**Dependencies:** P-A (Larmor radiation must exist to create photons)

### P-E: EM Field Energy Bookkeeping

Track electromagnetic field energy to improve total energy conservation display.

**Physics:**
- Electrostatic: `U_E = Σ_{i<j} q_i·q_j / r_ij` (already computed as Coulomb PE)
- Magnetostatic dipole: `U_M = Σ_{i<j} μ_i·μ_j / r_ij³` (already computed as magnetic PE)
- Gravitomagnetic: `U_GM = -Σ_{i<j} L_i·L_j / r_ij³` (already computed)
- **New — velocity-dependent field energy**: Approximate the EM field momentum/energy from moving charges. Poynting vector energy density `u = (E² + B²)/2`. For point charges: `U_vel ≈ Σ_{i<j} q_i·q_j·(v_i·v_j) / r_ij` (Darwin Lagrangian O(v²/c²) correction)

**Implementation:**
- In `computeEnergy()`: add Darwin-term computation alongside existing PE terms
- New stat sub-row: "Field Energy" under Energy section
- Adjust "Total" energy to include field energy
- Drift% should improve when magnetic/gravitomagnetic forces are active

**Dependencies:** None

### P-F: Retarded Potentials

Forces propagate at the speed of light instead of instantaneously.

**Physics:** Each particle sees other particles at their retarded position `x_ret = x(t - r/c)`. Solve `|x(t_ret) - x_obs(t)| = c·(t - t_ret)` iteratively.

**Implementation:**
- Each particle gets a circular history buffer: `historyPos[MAX_HISTORY]`, `historyVel[MAX_HISTORY]`, `historyTime[MAX_HISTORY]`
- `MAX_HISTORY = 512`, storing positions at each substep
- In force calculation: for each pair, compute light-travel time `r/c`, look up source particle's retarded position/velocity via linear interpolation in history buffer
- Iterative solver: 2-3 Newton-Raphson iterations to find `t_ret` such that `|x_source(t_ret) - x_target(t)| = c·(t - t_ret)`
- Toggle: only active when Relativity is enabled
- Performance: O(N²) remains the same, but each pair interaction is more expensive (history lookup + interpolation). Cap history buffer writes to every `dt > min_interval` to limit memory
- Visual: optional "light cone" display showing retarded positions as ghost particles

**Dependencies:** Relativity toggle must be on

### P-G: Correct Spin-Orbit Coupling

Re-implement spin-orbit torque correctly via magnetic dipole interaction.

**Physics:** A spinning charged particle has magnetic dipole moment `μ = MAG_MOMENT_K · q · ω · r²`. In an external magnetic field B, it experiences torque `τ = μ × B`. In 2D (spin axis ⊥ plane, B_z only), this reduces to `τ_z = μ · B_perp` where `B_perp` is the in-plane component.

Wait — in the current setup, spin is out-of-plane and B is out-of-plane (B_z from Lorentz). So `μ × B` is in-plane, not a torque on the out-of-plane spin. The correct coupling for out-of-plane dipole in out-of-plane field is zero torque (parallel/antiparallel).

**Revised approach:** The spin-orbit coupling that matters in 2D is the **gradient force** on the magnetic dipole: `F_dipole = ∇(μ·B)`. For a dipole in a non-uniform B_z field: `F_x = μ · ∂B_z/∂x`, `F_y = μ · ∂B_z/∂y`. This is already partially captured by the magnetic dipole force term (1/r⁴). The missing piece is the **back-reaction on spin**: by Newton's 3rd law, the field gradient force on the orbit should extract energy from spin. Correct implementation:

- Compute field gradient `∇B_z` at each particle from all other particles' Lorentz fields
- Energy exchange rate: `dE_spin/dt = -μ · (v · ∇B_z)` (dipole moving through gradient)
- Adjust `spin` accordingly: `dSpin/dt = -dE_spin/(I·ω)` where `I = INERTIA_K·m·r²`
- Conserves total energy (kinetic + spin + field) by construction

**Implementation:**
- In `physics.js`: after Lorentz B_z accumulation, compute B_z gradient via finite differences from neighboring particles
- Apply spin torque in the same substep as the Boris rotation
- Toggle: active when both Magnetic and Relativity are enabled
- Stats: angular momentum should now show correct orbital↔spin transfer

**Dependencies:** None (builds on existing magnetic force infrastructure)

### P-H: Energy Flow Sankey Diagram

Real-time visualization of energy transfers between categories.

**Implementation:**
- Track per-frame energy deltas: `ΔKE`, `ΔPE`, `ΔRadiated`, `ΔFieldEnergy`, `ΔSpin_KE`
- Compute transfer rates: `KE→PE` (gravity capture), `PE→KE` (acceleration), `KE→Radiation` (Larmor), `Spin↔Orbit` (spin-orbit coupling)
- Render as a small overlay (200×150px) in a corner, with animated flow arrows whose width is proportional to transfer rate
- Color-code: KE = warm, PE = cool, Radiation = yellow, Field = cyan, Spin = purple
- Toggle in Display tab: "Show Energy Flow"
- Update rates smoothed over 10 frames to avoid flicker

**Dependencies:** P-A (radiation tracking), P-E (field energy), P-G (spin-orbit) — but can show partial data without all dependencies

### P-I: Phase Space Plot

Small overlay showing selected particle's trajectory in phase space.

**Implementation:**
- When a particle is selected (click-to-select already exists): track `(r, v_r)` or `(x, p_x)` in a circular buffer (500 points)
- `r` = distance to center-of-mass or to most massive body; `v_r` = radial velocity component
- Render as a small canvas overlay (180×180px) in a corner
- Draw trajectory as a fading line (older = more transparent)
- Axis labels: r (horizontal), v_r (vertical)
- Auto-scaling axes based on data range
- Toggle in Display tab: "Show Phase Space"
- Only visible when a particle is selected

**Dependencies:** None

### P-J: Gravitational Potential Heatmap

Background overlay showing the combined potential field.

**Implementation:**
- Compute potential `Φ(x,y) = -Σ m_i / |r - r_i|` (gravity) + `Σ q_i·q_j / |r - r_j|` (Coulomb) on a coarse grid (e.g., 64×64 covering the viewport)
- Use Plummer softening for grid points near particles
- Color map: blue (deep negative = strong gravity well) → white (zero) → red (positive = repulsive). Or use a diverging colormap
- Render as an offscreen canvas, scale up with bilinear interpolation, composite behind particles with low opacity (0.3)
- Update every 5-10 frames (not every frame) for performance
- Toggle in Display tab: "Show Potential Field"
- When Barnes-Hut is on, use the quadtree for O(N log N) potential evaluation on the grid

**Dependencies:** None

---

## Biosim Improvements

### B-B: Oxidative Stress / ROS

Electron leakage creates harmful reactive oxygen species.

**Implementation:**
- In `etc.js` electron chain callbacks: 2% chance per hop at Complex I and Cyt b6f to spawn superoxide (O₂⁻) instead of completing the chain
- New metabolite in `store`: `superoxide` (starts at 0), `h2o2` (starts at 0)
- New enzyme: **SOD** (superoxide dismutase) — drawn on canvas near membrane. 2 O₂⁻ → H₂O₂ + O₂. Auto-fires when superoxide > 0
- New enzyme: **Catalase** — drawn near SOD. 2 H₂O₂ → 2 H₂O + O₂. Auto-fires when h2o2 > 0
- New enzyme: **Glutathione Reductase** — consumes NADPH to regenerate glutathione (simplified: 1 NADPH removes 1 ROS). Gives PPP a survival purpose
- New stat: "Cell Health" bar (0-100%, starts at 100). Decreases by 1% per accumulated superoxide/h2o2 per second. Recovers when ROS are scavenged
- Visual: ROS particles rendered as red sparks near the membrane
- At health < 20%: warning toast. At health 0%: cell death animation (canvas fades to grey, "Cell Death" overlay)
- Info tip explaining oxidative stress

**Dependencies:** None, but benefits from B-G (organism presets may vary ROS sensitivity)

### B-D: ATP Source Tracking

Distinguish substrate-level from oxidative phosphorylation.

**Implementation:**
- In `store`: add `atpSubstrate` and `atpOxidative` counters (cumulative, not pools)
- In glycolysis (PGK step 6, PK step 8): increment `atpSubstrate`
- In Krebs (SCS step 4): increment `atpSubstrate`
- In `advanceATPSynthase()`: increment `atpOxidative`
- In Stats tab: new split bar showing substrate% vs oxidative% with labels
- Per-glucose annotation: "~2 ATP (substrate) + ~30 ATP (oxidative) = ~32 ATP total"
- Reset counters on sim reset

**Dependencies:** None

### B-F: Proton Leak / Uncoupling

Passive proton leak makes ATP synthase realistically inefficient.

**Implementation:**
- In `autoplayTick()` or a separate timer: every 0.5s, leak `floor(gradient * leakRate)` protons (reduce `protonGradient`)
- `leakRate = 0.02` base (2% of gradient per tick)
- New toggle in Controls tab: "Uncoupling Protein" — sets `leakRate = 0.10` (5× increase)
- Info tip: "Uncoupling proteins dissipate the proton gradient as heat instead of making ATP. Found in brown fat for thermogenesis."
- Visual: when uncoupling is on, show faint proton particles leaking downward through the membrane (reverse direction of ATP synthase protons)
- Stats: new "Leak" counter showing total protons leaked

**Dependencies:** None

### B-G: Organism Presets

Preset selector configuring available pathways per organism type.

**Implementation:**
- New UI element: preset dropdown or dialog (similar to physsim preset cards) in the Controls tab header
- 5 presets with configuration objects:

```
Cyanobacterium: { all pathways enabled, light: true, o2: true }
Animal Cell:    { glycolysis, krebs, betaox, resp_etc, fermentation; light: false, o2: true }
Obligate Anaerobe: { glycolysis, fermentation; light: false, o2: false }
Plant Chloroplast: { calvin, photo_etc, cyclic; light: true, o2: true }
Archaeon:       { glycolysis, krebs, betaox, br; light: true, o2: false }
```

- On preset selection: update `simState` pathway enables, lock disabled pathway toggles (grey out with lock icon), set environment defaults, reset metabolite pools to organism-appropriate ratios
- Locked pathways: toggle switches visually disabled, info tip explains why ("Animal cells lack photosynthetic machinery")
- Current manual toggle state preserved as "Custom" preset
- Visual: organism name displayed in topbar or sidebar header

**Dependencies:** None

### B-K: Metabolite Time Series Graphs

Sparkline charts showing cofactor ratios over time.

**Implementation:**
- Circular buffer per tracked value: `atpHistory[300]`, `nadhHistory[300]`, `nadphHistory[300]`, `fadh2History[300]`, `gradientHistory[300]` — 300 points at 5Hz = 60 seconds of history
- Sample every 200ms in the main loop
- Render as small canvas elements (one per metric, ~160×40px) in the Stats tab below the existing bars
- Line color matches the cofactor bar color
- Y-axis: 0-100% for ratios, 0-max for gradient
- No axis labels (space-constrained), but tooltip on hover shows exact value
- Vertical dotted line shows "now" at right edge
- Clear on reset

**Dependencies:** None

---

## Gerry Improvements

### G-A: Seed Persistence

Store PRNG seed with saved plans for reproducible maps.

**Implementation:**
- Replace `Math.random()` in `hex-generator.js` with a seeded PRNG (e.g., mulberry32 or xoshiro128)
- Store seed in `state.seed`, generated randomly on each `randomizeMap()` call
- Plan JSON format: add `seed` field alongside `hexAssignments`
- On `loadPlan()`: if plan has seed, call `randomizeMap(plan.seed)` before applying assignments
- URL hash support: `#seed=12345` generates deterministic map on page load
- Backward compatibility: plans without seed field load as before (assignments on current map)

**Dependencies:** None

### G-B: Auto-Gerrymander & Auto-Fair Algorithms

Automated redistricting demonstrating partisan strategies.

**Implementation:**
- Two new toolbar buttons: "Auto-Gerrymander" (with party selector dropdown) and "Auto-Fair"
- **Pack & Crack algorithm:**
  1. Sort all hexes by target party vote share (descending)
  2. Pack: fill one district with highest-opposition hexes until 110% cap
  3. Crack: distribute remaining opposition hexes across remaining districts, mixing in enough target-party hexes to win each
  4. Greedy optimization: iterate swapping border hexes between districts to maximize target party seats while maintaining contiguity
- **Fair Draw algorithm:**
  1. Start with random seed assignments
  2. Iteratively swap border hexes between adjacent districts
  3. Objective function: minimize `|vote_share - seat_share|` for all parties + compactness bonus + population equality penalty
  4. Simulated annealing: accept worsening swaps with decreasing probability
  5. Run for 1000-5000 iterations
- Both algorithms push undo snapshot before starting, so user can undo the result
- Results display: toast showing "Gerrymandered: Red wins 7/10 seats with 45% of votes" or "Fair: seats match vote shares within 5%"

**Dependencies:** G-A (seed persistence useful for reproducing the map being gerrymandered)

### G-E: Multi-Round Election Simulation

Simulate electoral volatility to test map resilience.

**Implementation:**
- New button in toolbar or sidebar: "Simulate Elections"
- Opens a small overlay/panel showing simulation controls:
  - Swing σ slider (1-10%, default 5%)
  - Number of elections (50/100/500)
  - "Run" button
- Simulation: for each election, add `N(0, σ)` swing to each party's vote share per hex (correlated: same swing direction for all hexes, magnitude varies). Recompute district winners. Record seat outcomes
- Results display:
  - Histogram: X = number of seats for each party, Y = frequency across simulations
  - "Expected seats" (mean) vs "Current seats" comparison
  - "Probability of majority" for each party
  - "Swing needed for majority change" (critical swing value)
- Rendered as a small canvas overlay (300×200px)
- Dismiss overlay to return to normal map editing

**Dependencies:** None

### G-F: Compactness Formula Fix

Replace approximated Polsby-Popper constant with correct hex geometry.

**Implementation:**
- Current: `compactness = (32.648 * hexCount) / perimeter²`
- Correct: `compactness = (4π * area) / perimeter²` where `area = hexCount * hexArea` and `perimeter = edgeCount * edgeLength`
- `hexArea = (3√3/2) * s²` where `s` is hex side length
- `edgeLength = s` (hex side)
- Compute from `CONFIG.HEX_W` and `CONFIG.HEX_H` to get actual pixel-unit values
- Result: 0-1 scale (1 = circle), displayed as percentage 0-100%
- One-line formula change in `metrics.js`

**Dependencies:** None

---

## Implementation Order (Suggested)

### Phase 1: Quick Wins (1-2 days each)
1. G-F — Compactness fix (one formula change)
2. B-D — ATP source tracking (counter + display)
3. B-F — Proton leak (timer + toggle)
4. P-E — EM field energy bookkeeping

### Phase 2: Medium Features (2-4 days each)
5. G-A — Seed persistence (seeded PRNG + plan format)
6. B-K — Time series graphs
7. P-J — Potential heatmap
8. P-I — Phase space plot
9. P-G — Correct spin-orbit coupling
10. B-G — Organism presets

### Phase 3: Major Features (4-7 days each)
11. P-A — Larmor radiation
12. P-B — Tidal forces / Roche limit
13. P-C — Radiation pressure (depends on P-A)
14. G-E — Multi-round election simulation
15. B-B — Oxidative stress / ROS
16. P-H — Energy flow Sankey (depends on P-A, P-E, P-G)

### Phase 4: Complex Features (1-2 weeks each)
17. G-B — Auto-gerrymander/fair algorithms
18. P-F — Retarded potentials
