# Documentation Overhaul Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Rewrite all code comments, READMEs, CLAUDE.md files, and info tooltips across 4 sub-repos + meta-repo for accuracy, conciseness, and usefulness.

**Architecture:** Repo-by-repo, top-down approach. Within each repo: README (approachable) → CLAUDE.md (architect reference) → info tooltips (user-friendly) → code comments (terse, WHY not WHAT). Use parallel sub-agents for independent files within each repo.

**Tech Stack:** Pure documentation changes — no functional code modifications. Vanilla JS/HTML/CSS codebase, zero dependencies.

**Style Rules:**
- README.md: approachable, educational — what it does, how to run, what's interesting
- CLAUDE.md: architect-level reference — file map, data flow, patterns, gotchas
- Info tooltips: user-friendly, concise — 1-3 sentences explaining what a concept means
- Code comments: terse, technical — WHY not WHAT, formulas, invariants, non-obvious decisions
- Section headers (`// ─── ───` / `/* ═══ */`) stay for navigation
- Never use "retarded potential(s)" — use "signal delay" or "finite-speed force propagation"

---

### Task 1: Meta-repo CLAUDE.md

**Files:**
- Modify: `CLAUDE.md` (meta-repo root, 189 lines)

**Step 1: Read and audit current CLAUDE.md for accuracy**
Verify all file paths, token values, module descriptions, and cross-references match the current codebase state.

**Step 2: Rewrite for clarity and accuracy**
- Fix any stale references or inaccurate descriptions
- Tighten prose — remove redundancy, improve information density
- Ensure all shared token values match `shared-tokens.js`
- Verify UI architecture descriptions match current implementations
- Verify responsive breakpoints match current CSS

**Step 3: Commit**
```bash
git add CLAUDE.md
git commit -m "docs: rewrite meta-repo CLAUDE.md for accuracy and clarity"
```

---

### Task 2: Root site (a9lim.github.io) — README + CLAUDE.md

**Files:**
- Modify: `a9lim.github.io/README.md` (46 lines)
- Modify: `a9lim.github.io/CLAUDE.md` (106 lines)

**Step 1: Rewrite README.md**
Approachable tone. Cover: what the site is, live URL, tech choices (zero-dep vanilla JS, WebGL shader bg), how to run locally, shared design system overview, project structure.

**Step 2: Audit and rewrite CLAUDE.md**
Architect reference. Verify all module descriptions, file paths, key patterns, CSS conventions, and gotchas match current code. Fix stale content. Improve information density.

**Step 3: Commit**
```bash
cd a9lim.github.io && git add README.md CLAUDE.md && git commit -m "docs: rewrite README and CLAUDE.md"
```

---

### Task 3: Root site — shared JS modules (comments)

**Files:**
- Modify: `a9lim.github.io/shared-tokens.js` (212 lines)
- Modify: `a9lim.github.io/shared-utils.js` (77 lines)
- Modify: `a9lim.github.io/shared-camera.js` (477 lines)
- Modify: `a9lim.github.io/shared-info.js` (170 lines)
- Modify: `a9lim.github.io/shared-shortcuts.js` (206 lines)
- Modify: `a9lim.github.io/shared-touch.js` (87 lines)

**Step 1: Rewrite comments in all shared-*.js files**
These are consumed by all 4 projects. Comments should document the public API clearly (JSDoc on exports), explain non-obvious implementation choices, and remove restating-the-obvious comments.

**Step 2: Commit**
```bash
cd a9lim.github.io && git add shared-*.js && git commit -m "docs: rewrite shared module comments"
```

---

### Task 4: Root site — project-specific JS, CSS, HTML (comments)

**Files:**
- Modify: `a9lim.github.io/main.js` (50 lines)
- Modify: `a9lim.github.io/colors.js` (exists in root)
- Modify: `a9lim.github.io/index.html` (229 lines)
- Modify: `a9lim.github.io/styles.css` (1403 lines)
- Modify: `a9lim.github.io/src/animations.js` (74 lines)
- Modify: `a9lim.github.io/src/blog.js` (116 lines)
- Modify: `a9lim.github.io/src/card-effects.js` (62 lines)
- Modify: `a9lim.github.io/src/carousel.js` (168 lines)
- Modify: `a9lim.github.io/src/markdown.js` (110 lines)
- Modify: `a9lim.github.io/src/mobile-menu.js` (8 lines)
- Modify: `a9lim.github.io/src/projects-page.js` (20 lines)
- Modify: `a9lim.github.io/src/projects.js` (74 lines)
- Modify: `a9lim.github.io/src/router.js` (57 lines)
- Modify: `a9lim.github.io/src/shader.js` (214 lines)
- Modify: `a9lim.github.io/src/theme.js` (21 lines)
- Modify: `a9lim.github.io/src/world-map.js` (151 lines)

**Step 1: Rewrite comments in all project-specific files**
Focus on: GLSL shader comments (shader.js has complex noise math), carousel pagination logic, markdown parser regex patterns, world-map projection calibration. Remove obvious comments. Add WHY comments where missing.

**Step 2: Rewrite CSS section headers and comments in styles.css**
Keep section header style. Remove redundant comments. Add notes on non-obvious specificity battles or responsive behavior.

**Step 3: Rewrite HTML comments in index.html**
Keep section dividers. Ensure they accurately label what follows.

**Step 4: Commit**
```bash
cd a9lim.github.io && git add main.js colors.js index.html styles.css src/ && git commit -m "docs: rewrite project-specific code comments"
```

---

### Task 5: physsim — README + CLAUDE.md

**Files:**
- Modify: `physsim/README.md` (100 lines)
- Modify: `physsim/CLAUDE.md` (376 lines)

**Step 1: Rewrite README.md**
Approachable tone for portfolio visitors. Cover: what it simulates, physics model highlights (Boris integrator, relativistic, Barnes-Hut), force types, controls overview, how to run, tech choices. Make it showcase the project's sophistication without being impenetrable.

**Step 2: Audit and rewrite CLAUDE.md**
This is already the most detailed CLAUDE.md. Verify all formulas, module descriptions, toggle dependencies, and data flow against current code. Fix any stale content. Improve organization. Ensure "signal delay" terminology is used throughout.

**Step 3: Commit**
```bash
cd physsim && git add README.md CLAUDE.md && git commit -m "docs: rewrite README and CLAUDE.md"
```

---

### Task 6: physsim — info tooltips

**Files:**
- Identify and modify: info tooltip data (likely in `physsim/src/ui.js` inline, similar to gerry pattern)

**Step 1: Find and rewrite all info tooltip text**
User-friendly tone, 1-3 sentences. Each tooltip should explain what the concept is and why a user might care. Cover all physics toggles, engine settings, and stat displays.

**Step 2: Commit**
```bash
cd physsim && git add src/ui.js && git commit -m "docs: rewrite info tooltip text"
```

---

### Task 7: physsim — code comments (physics core)

**Files:**
- Modify: `physsim/src/forces.js` (335 lines)
- Modify: `physsim/src/integrator.js` (763 lines)
- Modify: `physsim/src/energy.js` (127 lines)
- Modify: `physsim/src/signal-delay.js` (315 lines)
- Modify: `physsim/src/topology.js` (129 lines)
- Modify: `physsim/src/relativity.js` (41 lines)
- Modify: `physsim/src/potential.js` (158 lines)
- Modify: `physsim/src/collisions.js` (259 lines)
- Modify: `physsim/src/quadtree.js` (256 lines)

**Step 1: Rewrite comments in physics-critical modules**
These have the most complex algorithms. Ensure formula comments match implementation. Explain sign conventions, coordinate choices, and numerical stability decisions. Use "signal delay" not "retarded potential".

**Step 2: Commit**
```bash
cd physsim && git add src/forces.js src/integrator.js src/energy.js src/signal-delay.js src/topology.js src/relativity.js src/potential.js src/collisions.js src/quadtree.js && git commit -m "docs: rewrite physics core comments"
```

---

### Task 8: physsim — code comments (rendering, UI, utilities)

**Files:**
- Modify: `physsim/main.js` (212 lines)
- Modify: `physsim/colors.js` (27 lines)
- Modify: `physsim/index.html` (415 lines)
- Modify: `physsim/styles.css` (560 lines)
- Modify: `physsim/src/renderer.js` (406 lines)
- Modify: `physsim/src/input.js` (262 lines)
- Modify: `physsim/src/ui.js` (338 lines)
- Modify: `physsim/src/config.js` (54 lines)
- Modify: `physsim/src/particle.js` (79 lines)
- Modify: `physsim/src/photon.js` (19 lines)
- Modify: `physsim/src/vec2.js` (69 lines)
- Modify: `physsim/src/presets.js` (87 lines)
- Modify: `physsim/src/heatmap.js` (83 lines)
- Modify: `physsim/src/phase-plot.js` (120 lines)
- Modify: `physsim/src/sankey.js` (98 lines)
- Modify: `physsim/src/stats-display.js` (92 lines)

**Step 1: Rewrite comments in rendering/UI/utility modules**
Focus on renderer draw pipeline order, input state machine, UI toggle wiring, and visualization algorithm explanations.

**Step 2: Commit**
```bash
cd physsim && git add main.js colors.js index.html styles.css src/renderer.js src/input.js src/ui.js src/config.js src/particle.js src/photon.js src/vec2.js src/presets.js src/heatmap.js src/phase-plot.js src/sankey.js src/stats-display.js && git commit -m "docs: rewrite rendering and UI comments"
```

---

### Task 9: biosim — README + CLAUDE.md

**Files:**
- Modify: `biosim/README.md` (90 lines)
- Modify: `biosim/CLAUDE.md` (210 lines)

**Step 1: Rewrite README.md**
Approachable tone. Cover: what it visualizes (cellular metabolism), key pathways, interactive features (click enzymes, allosteric regulation, organism presets), how to run, tech choices.

**Step 2: Audit and rewrite CLAUDE.md**
Verify regulation table, dispatch maps, color system, draw pipeline, and UI patterns against current code. Fix the `data-theme` on `<body>` vs `<html>` discrepancy (check which is actually true). Tighten prose.

**Step 3: Commit**
```bash
cd biosim && git add README.md CLAUDE.md && git commit -m "docs: rewrite README and CLAUDE.md"
```

---

### Task 10: biosim — info tooltips

**Files:**
- Modify: `biosim/src/info.js` (114 lines)

**Step 1: Rewrite all enzyme and metabolite tooltip text**
User-friendly tone. ~42 enzyme entries and ~26 metabolite entries. Each should explain what the enzyme/metabolite does in the cell and its role in the pathway, in 1-2 sentences. Keep chemical equations accurate. Verify equation Unicode subscripts are correct.

**Step 2: Commit**
```bash
cd biosim && git add src/info.js && git commit -m "docs: rewrite enzyme and metabolite info tooltips"
```

---

### Task 11: biosim — code comments (reactions)

**Files:**
- Modify: `biosim/src/reactions/dispatch.js` (112 lines)
- Modify: `biosim/src/reactions/glycolysis.js` (133 lines)
- Modify: `biosim/src/reactions/krebs.js` (83 lines)
- Modify: `biosim/src/reactions/calvin.js` (39 lines)
- Modify: `biosim/src/reactions/ppp.js` (36 lines)
- Modify: `biosim/src/reactions/etc.js` (103 lines)
- Modify: `biosim/src/reactions/fermentation.js` (52 lines)
- Modify: `biosim/src/reactions/betaoxidation.js` (78 lines)
- Modify: `biosim/src/reactions/ros.js` (59 lines)

**Step 1: Rewrite comments in all reaction modules**
Each reaction step should have a brief comment explaining the biochemistry (substrate → product, cofactor changes). Batch handlers should note what they accomplish. Dispatch map comments should explain the routing logic.

**Step 2: Commit**
```bash
cd biosim && git add src/reactions/ && git commit -m "docs: rewrite reaction module comments"
```

---

### Task 12: biosim — code comments (core modules)

**Files:**
- Modify: `biosim/main.js` (56 lines)
- Modify: `biosim/colors.js` (119 lines)
- Modify: `biosim/index.html` (431 lines)
- Modify: `biosim/styles.css` (659 lines)
- Modify: `biosim/src/renderer.js` (1137 lines — largest file in the project)
- Modify: `biosim/src/enzymes.js` (734 lines)
- Modify: `biosim/src/ui.js` (259 lines)
- Modify: `biosim/src/state.js` (85 lines)
- Modify: `biosim/src/anim.js` (64 lines)
- Modify: `biosim/src/autoplay.js` (122 lines)
- Modify: `biosim/src/dashboard.js` (115 lines)
- Modify: `biosim/src/layout.js` (93 lines)
- Modify: `biosim/src/particles.js` (153 lines)
- Modify: `biosim/src/regulation.js` (168 lines)
- Modify: `biosim/src/sparkline.js` (46 lines)
- Modify: `biosim/src/organisms.js` (53 lines)
- Modify: `biosim/src/theme.js` (23 lines)

**Step 1: Rewrite comments in all core biosim modules**
Focus on: renderer draw pipeline (1137 lines — needs clear pass documentation), enzyme drawing constants, regulation logic, autoplay tick intervals, layout grid calculations, particle trail system.

**Step 2: Commit**
```bash
cd biosim && git add main.js colors.js index.html styles.css src/renderer.js src/enzymes.js src/ui.js src/state.js src/anim.js src/autoplay.js src/dashboard.js src/layout.js src/particles.js src/regulation.js src/sparkline.js src/organisms.js src/theme.js && git commit -m "docs: rewrite core module comments"
```

---

### Task 13: gerry — README + CLAUDE.md

**Files:**
- Modify: `gerry/README.md` (99 lines)
- Modify: `gerry/CLAUDE.md` (171 lines)

**Step 1: Rewrite README.md**
Approachable tone. Cover: what it simulates (redistricting/gerrymandering), key features (brush sizes, metrics, plan save/load, election sim, auto-gerrymander/fair-draw), controls, how to run, tech choices.

**Step 2: Audit and rewrite CLAUDE.md**
Verify all algorithm descriptions, state management, data flow, UI layout, and responsive breakpoints against current code. Tighten prose.

**Step 3: Commit**
```bash
cd gerry && git add README.md CLAUDE.md && git commit -m "docs: rewrite README and CLAUDE.md"
```

---

### Task 14: gerry — info tooltips

**Files:**
- Modify: `gerry/main.js` (492 lines — tooltip data is inline around lines 390-405)

**Step 1: Rewrite all metric tooltip text**
User-friendly tone. 7 metrics: Efficiency Gap, Partisan Symmetry, Competitive Districts, Compactness, Contiguity, Majority-Minority Districts, Population Balance. Each should explain the metric and why it matters for fair redistricting, in 1-3 sentences.

**Step 2: Commit**
```bash
cd gerry && git add main.js && git commit -m "docs: rewrite metric info tooltips"
```

---

### Task 15: gerry — code comments (all modules)

**Files:**
- Modify: `gerry/main.js` (492 lines — also has code comments beyond tooltips)
- Modify: `gerry/colors.js` (78 lines)
- Modify: `gerry/index.html` (472 lines)
- Modify: `gerry/styles.css` (734 lines)
- Modify: `gerry/src/auto-district.js` (369 lines)
- Modify: `gerry/src/config.js` (58 lines)
- Modify: `gerry/src/election-sim.js` (108 lines)
- Modify: `gerry/src/hex-generator.js` (184 lines)
- Modify: `gerry/src/hex-math.js` (21 lines)
- Modify: `gerry/src/input.js` (309 lines)
- Modify: `gerry/src/metrics.js` (201 lines)
- Modify: `gerry/src/noise.js` (28 lines)
- Modify: `gerry/src/palette.js` (52 lines)
- Modify: `gerry/src/plans.js` (145 lines)
- Modify: `gerry/src/prng.js` (15 lines)
- Modify: `gerry/src/renderer.js` (294 lines)
- Modify: `gerry/src/sidebar.js` (195 lines)
- Modify: `gerry/src/state.js` (105 lines)
- Modify: `gerry/src/theme.js` (30 lines)
- Modify: `gerry/src/touch.js` (124 lines)
- Modify: `gerry/src/zoom.js` (91 lines)

**Step 1: Rewrite comments in all gerry modules**
Focus on: auto-districting algorithms (pack & crack, simulated annealing), metric computations (efficiency gap, partisan symmetry), hex grid generation (population model, noise), election simulation (Monte Carlo), SVG rendering pipeline.

**Step 2: Commit**
```bash
cd gerry && git add main.js colors.js index.html styles.css src/ && git commit -m "docs: rewrite all code comments"
```

---

## Parallelization Notes

Within each task, independent files can be processed by parallel sub-agents:
- Task 3: All 6 shared-*.js files are independent
- Task 4: All src/*.js files are independent; CSS and HTML are independent of JS
- Task 7: Physics modules are mostly independent (forces.js, energy.js, topology.js, etc.)
- Task 8: Rendering/UI modules are independent
- Task 11: Reaction modules are independent
- Task 12: Core biosim modules are independent
- Task 15: Gerry modules are independent

Tasks 1-4 (meta + root site) should complete before Tasks 5-8 (physsim) since shared modules inform project-specific docs. Tasks 5-15 can be done in any order since each repo is independent.
