# CLAUDE.md

Architect-level reference for Claude Code working across the a9l.im meta-repository.

## Style Rule

Never use the phrase "retarded potential(s)" in code, comments, or user-facing text. Use "signal delay" or "finite-speed force propagation" instead.

## Purpose

Meta-repository coordinating four independently-tracked GitHub repos that form the portfolio site at **a9l.im** (hosted via `a9lim.github.io`). Enforces design, style, and theme consistency across all sub-sites.

## Repository Map

| Folder | GitHub Repo | Domain Path | Description |
|--------|------------|-------------|-------------|
| `a9lim.github.io/` | `a9lim/a9lim.github.io` | `a9l.im` (root) | Portfolio landing page |
| `physsim/` | `a9lim/physsim` | `a9l.im/physsim` | No-Hair -- N-body simulation |
| `biosim/` | `a9lim/biosim` | `a9l.im/biosim` | Cellular metabolism visualization |
| `gerry/` | `a9lim/gerry` | `a9l.im/gerry` | Gerrymandering/redistricting simulator |

Each sub-folder has its own `.git` and `CLAUDE.md`, deployed independently to GitHub Pages. This meta-folder is itself a git repo for cross-repo coordination (design docs, plans, this file).

## Shared Tech Stack

All four projects are **zero-dependency vanilla JS/HTML/CSS** static sites. No build step, no bundler, no npm. Serve any project locally with:
```bash
python -m http.server          # or
npx serve .
```

## Shared Design System

### Shared Files (in `a9lim.github.io/`)

Nine files at the root site are loaded by sub-projects via absolute paths (`/shared-*.js`, `/shared-*.css`). The root site loads them via relative paths.

| File | Loaded by | Description |
|------|-----------|-------------|
| `shared-tokens.js` | all 4 | `_PALETTE`, `_FONT`, color math (`_r`, `_parseHex`, `_rgb2hsl`, `_hsl2hex`, `_darken`). Injects `<style id="palette-vars">` with all shared CSS custom properties. |
| `shared-utils.js` | all 4 | `escapeHtml()`, `debounce()`, `throttle()`, `clamp()`, `lerp()`, `cubicBezier()` (Newton-Raphson), `showToast()`. |
| `shared-haptics.js` | all 4 | `_haptics.trigger(type)` -- haptic feedback via Web Vibration API. Named presets: `light`/`medium`/`heavy` (impact), `success`/`warning`/`error` (notification), `selection` (discrete stepping), `buzz`/`nudge`/`soft`/`rigid` (extra). Silently no-ops on unsupported platforms. |
| `shared-tabs.js` | 3 sims | Tab switching IIFE for `.tab-btn`/`.tab-panel` sidebar tabs. Loaded as plain `<script>` (not module) so tabs work even if main module fails. |
| `shared-camera.js` | 3 sims | `createCamera(opts)` -- zoom/pan with mouse wheel, touch pinch, middle-click pan. Coordinate transforms (`screenToWorld`/`worldToScreen`), Canvas 2D (`applyToCanvas`) and SVG (`getViewBox`/`setFromViewBox`) integration. `bindZoomButtons(opts)` wires zoom-in/out/reset buttons with animated zoom (1.25x factor, 200ms easeOutCubic). |
| `shared-touch.js` | 3 sims | `initSwipeDismiss(panel, { onDismiss, handleSelector })` -- swipe-to-dismiss for bottom-sheet panels at <=900px. Drag handle, velocity threshold, 30% dismiss ratio. |
| `shared-info.js` | 3 sims | `createInfoTip(triggerEl, { title, body, maxWidth })` -- `[role="tooltip"]` popovers. Hover on desktop, tap-to-toggle on mobile. Auto-positions above/below. KaTeX math rendering if loaded (guarded `typeof`). |
| `shared-shortcuts.js` | 3 sims | `initShortcuts(shortcuts, { helpTitle })` -- `[{ key, label, group, action, when }]`. Ignores input in `<input>`/`<textarea>`/`<select>`/`contentEditable`. `?` opens help overlay, `Esc` closes. |
| `shared-base.css` | all 4 | Reset, layout tokens, `.glass`, `.tool-btn`, tab system, toggle switches, control rows/groups, form controls (range sliders, segmented controls, checkboxes), select dropdowns (`.sim-select`), overlay/dialog (`.sim-overlay`), ghost buttons, theme toggle icons, sim layout components, intro screen, sidebar stat patterns, toast, info tip styles, shortcut overlay styles, accessibility utilities, responsive breakpoints, `prefers-reduced-motion`. |

### HTML `<head>` Loading Order

Loading order varies slightly per project. The general pattern:

**Root site (`a9lim.github.io`):**
1. Google Fonts `<link>` tags
2. `<script src="shared-tokens.js"></script>` (relative path)
3. `<script src="shared-utils.js"></script>` (relative path)
4. `<script src="shared-haptics.js"></script>` (relative path)
5. `<link rel="stylesheet" href="shared-base.css">`
6. `<link rel="stylesheet" href="styles.css">`
7. No `colors.js` -- root site uses shared tokens only

**Sim projects (physsim, biosim, gerry):**
1. Google Fonts `<link>` tags (+ KaTeX CSS for physsim and biosim)
2. `<link rel="stylesheet" href="/shared-base.css">`
3. `<link rel="stylesheet" href="styles.css">`
4. `<script src="/shared-tokens.js"></script>`
5. `<script src="/shared-touch.js"></script>`
6. `<script src="/shared-utils.js"></script>`
7. `<script src="/shared-haptics.js"></script>`
8. `<script src="/shared-camera.js"></script>`
9. `<script src="/shared-info.js"></script>` (physsim and biosim load KaTeX scripts before this)
10. `<script src="/shared-shortcuts.js"></script>`
11. `<script src="colors.js"></script>`

All three sim projects also load `<script src="/shared-tabs.js"></script>` at the end of `<body>` (after sidebar markup, before `main.js`).

Note: gerry loads `colors.js` before `shared-touch.js`/`shared-info.js`/`shared-shortcuts.js` (different order). The key invariant is: `shared-tokens.js` before `colors.js`, `shared-tabs.js` after sidebar DOM, and all shared scripts before `main.js`.

### colors.js Contract

Each sim project has a `colors.js` that **extends** the shared tokens:

1. `shared-tokens.js` already loaded -- provides mutable `_PALETTE`, `_FONT`, `_r()`, color math helpers
2. `colors.js` adds project-specific keys to `_PALETTE` (and `_FONT` in biosim's case: `emoji`)
3. Freezes `_PALETTE.extended`, `_PALETTE.light`, `_PALETTE.dark`, `_FONT`, then `_PALETTE`
4. May inject `<style id="project-vars">` with project-specific CSS custom properties
5. JS modules read `_PALETTE` directly; CSS reads `var(--*)` custom properties

The root site has no `colors.js` -- shared tokens are sufficient.

### Shared Token Values

Enforced by `shared-tokens.js`:

| Token | Value |
|-------|-------|
| Accent | `#FE3B01` |
| Accent light | `#FF7642` |
| Light canvas | `#F0EBE4` |
| Light panel | `#FCF7F2` |
| Light elevated | `#FDF8F3` |
| Light text | `#1A1612` |
| Light text secondary | `#78706A` |
| Light text muted | `#A8A098` |
| Dark canvas | `#0C0B09` |
| Dark panel | `#1A1510` |
| Dark elevated | `#201B16` |
| Dark text | `#E8DED4` |
| Dark text secondary | `#8A8278` |
| Dark text muted | `#5A544C` |
| Font display | `'Noto Serif', Georgia, 'Times New Roman', serif` |
| Font body | `'Noto Sans', system-ui, -apple-system, sans-serif` |
| Font mono | `'Noto Sans Mono', 'SF Mono', 'Menlo', monospace` |
| Glass | `backdrop-filter: blur(24px) saturate(1.3)` |
| Border radius | `8px / 14px / 20px / 9999px` |

### CSS Custom Properties (from shared-tokens.js)

`shared-tokens.js` injects a `<style id="palette-vars">` with these CSS variables (`:root` for light, `[data-theme="dark"]` for dark overrides):

**Themed (light/dark variants):**
`--bg-canvas`, `--bg-panel`, `--bg-panel-solid`, `--bg-elevated`, `--bg-hover`, `--text`, `--text-secondary`, `--text-muted`, `--border`, `--border-strong`, `--slider-track`

**Accent (same both themes):**
`--accent`, `--accent-light`, `--accent-subtle`, `--accent-glow`

**Shadows (different values per theme):**
`--shadow-sm`, `--shadow-md`, `--shadow-lg`

**Fonts:**
`--font-display`, `--font-body`, `--font-mono`

**Toggle switches:**
`--tog-bg`, `--tog-thumb-on`, `--tog-border`, `--tog-shadow`, `--tog-thumb-shadow`, `--tog-checked-inset`

**Intro screen:**
`--intro-warm`, `--intro-warm-hover`, `--intro-cool`

**Overlay/card (root site uses these):**
`--backdrop`, `--overlay-base`, `--overlay-60`, `--overlay-87`, `--overlay-full`, `--overlay-hover-12`, `--overlay-hover-25`, `--overlay-hover-19`, `--card-bg-end`, `--overlay-text`, `--overlay-text-dim`, `--overlay-tint`, `--overlay-tint-dim`, `--shimmer`, `--shimmer-subtle`

**Extended palette (same both themes):**
`--ext-blue`, `--ext-green`, `--ext-slate`, `--ext-orange`, `--ext-rose`, `--ext-purple`, `--ext-brown`, `--ext-red`, `--ext-cyan`, `--ext-yellow`, `--ext-magenta`, `--ext-lime`, `--ext-indigo`

### Layout Tokens (from shared-base.css)

```css
--radius-sm: 8px;  --radius-md: 14px;  --radius-lg: 20px;  --radius-pill: 9999px;
--toolbar-h: 52px;  --panel-w: 350px;
--ease-out: cubic-bezier(0.23, 1, 0.32, 1);
--ease-spring: cubic-bezier(0.34, 1.56, 0.64, 1);
--ease-in-out: cubic-bezier(0.4, 0, 0.2, 1);
```

### UI Architecture

The three sim projects share these layout conventions (root site uses traditional page layout):

- **Floating glass panels** over full-viewport canvas/SVG
- **Fixed topbar** (`.sim-toolbar`, 52px, glass) with brand, tool buttons, separator
- **Right-side slide-in panel** (`.sim-panel`, 350px desktop, bottom sheet on mobile)
- **Bottom pill bar** (`.sim-bar`) and **left control stack** (`.sim-controls`)
- **Intro screen** (`#intro-screen`) with themed splash, instruction cards, CTA -- uses `--intro-warm`/`--intro-cool`/`--intro-warm-hover`
- **Light/dark theme** via `data-theme` on `<html>` (all four projects). `<html data-theme="light">` in markup prevents FOUC.
- **Entrance animations** gated by `.app-ready` class on `<body>`
- **`.glass`**: semi-transparent bg + blur(24px) saturate(1.3) + border + shadow
- **`.tool-btn`**: 34x34 icon buttons with SVG defaults (root site overrides to 36x36)
- **Tab system** (`.tabs-wrap`, `.tab-bar`, `.tab-btn`, `.tab-panels`, `.tab-panel`) -- CSS in shared-base.css, JS in shared-tabs.js
- **Toggle switches** (`.tog-wrap`, `.tog`, `.tog-thumb`, `.tog-icon`) -- per-toggle color via `--tog-color`
- **Control rows** (`.ctrl-row`, `.ctrl-sub`, `.ctrl-disabled`, `.ctrl-group`, `.panel-section`, `.ctrl-row.locked`) -- in shared-base.css
- **Form controls** -- range sliders (`input[type=range]`), segmented controls (`.mode-toggles`/`.mode-btn`), checkbox toggles (`.checkbox-label`), slider value display (`.slider-value`) -- in shared-base.css
- **Select dropdowns** (`.sim-select`) -- styled select element for presets/organisms -- in shared-base.css
- **Overlay/dialog** (`.sim-overlay`, `.sim-overlay-panel`, `.sim-overlay-body`) -- centered modal with backdrop blur, responsive fullscreen at 600px -- in shared-base.css
- **Ghost buttons** (`.ghost-btn`) -- outline-style action buttons -- in shared-base.css
- **Theme toggle icons** (`.icon-sun`, `.icon-moon`) -- hidden by default, shown via `[data-theme]` -- in shared-base.css. Biosim overrides for 3-state theme (adds `.icon-auto`)
- **Sidebar stat patterns** (`.group-label`, `.stats-header`, `.stat-row`, `.stat-value`, `.stat-group`)
- **Toast notifications** (`#toast-container`, `.toast`) via `showToast()` from shared-utils.js
- **Info tips** via `createInfoTip()` -- `?` buttons with `.info-trigger` class
- **Keyboard shortcuts** via `initShortcuts()` -- `?` key opens help overlay
- **Accessibility**: `.skip-link`, `.sr-only`, `role` attributes, `aria-*`, focus ring improvements
- **Sheet handle** (`.sheet-handle`) for mobile bottom-sheet drag, visible at <=900px

### Typography

All four projects use the **Noto** font family, loaded via Google Fonts `<link>` tags (never `@import` in CSS):

| Role | Font | CSS var |
|------|------|---------|
| Display/headings | Noto Serif | `--font-display` |
| Body/UI | Noto Sans | `--font-body` |
| Mono/data | Noto Sans Mono | `--font-mono` |

Section headers use the body font (Noto Sans) with uppercase, 0.68rem, weight 600, 0.12em tracking. Data values use `font-variant-numeric: tabular-nums`.

### CSS Conventions

- All color values from CSS custom properties -- no hardcoded colors
- Alpha via 8-digit hex (`#RRGGBBAA`), not `rgba()`
- Shared layout tokens in `shared-base.css`; project overrides in each `styles.css`
- Responsive breakpoints:

| Tier | Breakpoint | Purpose |
|------|-----------|---------|
| XL | 1100px | Panel narrowing (gerry only) |
| LG | **900px** | Primary mobile -- toolbar 48px, bottom sheet, swipe dismiss |
| MD | **600px** | Small phone -- brand shrinks, tighter spacing |
| SM | 440px | `.hide-sm` hides non-essential elements |

### _PALETTE / _FONT Keys

**`_FONT`** keys (defined in `shared-tokens.js`): `display`, `body`, `mono`. Biosim adds `emoji`.

**`_PALETTE`** keys (defined in `shared-tokens.js`):
- Top-level: `accent`, `accentLight`
- `light` sub-object: `canvas`, `panelSolid`, `elevated`, `text`, `textSecondary`, `textMuted`
- `dark` sub-object: same keys as `light`
- `extended` sub-object (13 cross-project colors):

| Key | Hex | Used by |
|-----|-----|---------|
| `blue` | `#5C92A8` | biosim (Krebs, NADH), gerry (Blue party) |
| `green` | `#509878` | biosim (Calvin, NADPH), gerry (Green/minority) |
| `slate` | `#8A7E72` | biosim (neutral), gerry (None), physsim (neutral particle) |
| `orange` | `#CC8E4E` | biosim (Glycolysis), gerry (Yellow party) |
| `rose` | `#C46272` | biosim (PPP, FADH2), gerry (Red party) |
| `purple` | `#9C7EB0` | biosim (Cyclic) |
| `brown` | `#9C6840` | biosim (Fermentation) |
| `red` | `#C05048` | biosim (Protons) |
| `cyan` | `#4AACA0` | biosim (Electrons) |
| `yellow` | `#CCA84C` | biosim (Photons, ATP) |
| `magenta` | `#B4689C` | physsim (unused — legacy) |
| `lime` | `#82A857` | physsim (Higgs field/force) |
| `indigo` | `#6C79AC` | physsim (Axion field/force) |

## Cross-Repo Changes

**Shared design system changes:**
1. Edit `shared-tokens.js` (tokens/colors) or `shared-base.css` (CSS patterns) in `a9lim.github.io/`
2. All four projects pick up changes automatically via absolute-path loading
3. Test both light and dark themes in each project

**Project-specific token changes:**
1. Edit `colors.js` in the target project
2. If the value should be shared, put it in `shared-tokens.js` instead

## Project-Specific Details

Each sub-folder's `CLAUDE.md` has full architecture docs. Key differences:

- **a9lim.github.io** (root site): ES6 modules, traditional page layout (not floating panels). Hosts all 7 shared files. Overrides `--toolbar-h: 56px` and `.tool-btn` to 36x36. WebGL shader background. No `colors.js`. `main.js` + `src/` (projects, projects-page, card-effects, router, theme, mobile-menu, animations, shader, carousel, blog, world-map, markdown).
- **physsim**: ES6 modules, Canvas 2D, `window.sim` global. Boris integrator with adaptive substepping. Force toggles (gravity, Coulomb, magnetic, gravitomagnetic, 1PN relativity). Barnes-Hut O(N log N) toggle. Signal delay (light-cone solver). Radiation (Landau-Lifshitz). 4-tab sidebar (Settings/Engine/Stats/Particle). KaTeX for math tooltips.
- **biosim**: ES6 modules, Canvas 2D, `_BASE`/`_ROLE`/`_THEME` color pipeline. Allosteric regulation gates enzyme reactions. Three-state theme (Simulation/Light/Dark). Tab system for sidebar (Controls/Stats/Reference). KaTeX for math in reference pages.
- **gerry**: ES6 modules, SVG rendering, `$` DOM cache. Hex-tile map with axial coordinates. 3-party efficiency gap, partisan symmetry, competitive districts metrics. Plan save/load (localStorage + JSON). Brush sizes (1/3/7 hex radius). Auto-fill. Overrides `--palette-h: 56px`.
