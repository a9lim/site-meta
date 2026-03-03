# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Purpose

This is a meta-repository coordinating four independently-tracked GitHub repos that together form the portfolio site at **a9l.im** (hosted via `a9lim.github.io`). The goal is design, style, and theme consistency across all sub-sites.

## Repository Map

| Folder | GitHub Repo | Domain Path | Description |
|--------|------------|-------------|-------------|
| `a9lim.github.io/` | `a9lim/a9lim.github.io` | `a9l.im` (root) | Portfolio landing page |
| `physsim/` | `a9lim/physsim` | `a9l.im/physsim` | Relativistic N-body simulation |
| `biosim/` | `a9lim/biosim` | `a9l.im/biosim` | Cellular metabolism visualization |
| `gerry/` | `a9lim/gerry` | `a9l.im/gerry` | Gerrymandering/redistricting simulator |

Each sub-folder has its own `.git`, `CLAUDE.md`, and is deployed independently to GitHub Pages. This meta-folder is **not** itself a git repo — it exists only for cross-repo coordination.

## Shared Tech Stack

All four projects are **zero-dependency vanilla JS/HTML/CSS** static sites. No build step, no bundler, no npm. Serve any project locally with:
```bash
python -m http.server          # or
npx serve .
```

## Shared Design System

### Shared Files (in `a9lim.github.io/`)

Six files at the root site are loaded by all four projects via absolute paths:

- **`shared-tokens.js`**: Color math helpers (`_r`, `_parseHex`, `_rgb2hsl`, `_hsl2hex`, `_darken`), `_FONT`, and `_PALETTE` (shared surface/text/accent/shadow tokens plus `extended` sub-object with 10 cross-project colors). Loaded as a plain `<script>` in `<head>` before project-specific `colors.js`.
- **`shared-utils.js`**: Utility functions used across all projects: `escapeHtml()`, `debounce()`, `throttle()`, `clamp()`, `lerp()`, `cubicBezier()` (Newton-Raphson CSS bezier solver), `showToast()`. Loaded after `shared-tokens.js`.
- **`shared-camera.js`**: Reusable viewport/camera module via `createCamera(opts)`. Supports zoom/pan with mouse wheel, touch pinch, middle-click pan. Coordinate transforms (`screenToWorld`/`worldToScreen`), Canvas 2D (`applyToCanvas`) and SVG (`getViewBox`/`setFromViewBox`) integration, animated transitions, configurable bounds/easing. `bindZoomButtons(opts)` wires zoom-in/out/reset buttons and a zoom percentage display to the camera with animated zoom (default 1.25× factor, 200ms easeOutCubic). Loaded by the 3 sim projects (not root site).
- **`shared-info.js`**: Info tip popover system via `createInfoTip(triggerEl, { title, body })`. Creates `[role="tooltip"]` popovers: hover on desktop, tap-to-toggle on mobile (coarse pointer). Auto-positions above/below based on viewport. CSS: `.info-trigger` (the `?` button), `.info-popover`, `.info-popover-title`, `.info-popover-body`, `.info-popover-close`. Loaded by the 3 sim projects.
- **`shared-shortcuts.js`**: Keyboard shortcut registry via `initShortcuts(shortcuts, { helpTitle })`. Shortcuts: `[{ key, label, group, action }]`. Ignores input when focus is in `<input>`/`<textarea>`/`<select>`. `?` key opens help overlay (`.shortcut-overlay` modal). `Esc` closes overlays. CSS: `.shortcut-overlay`, `.shortcut-group`, `.shortcut-row`, `.shortcut-key`, `.shortcut-label`. Loaded by the 3 sim projects.
- **`shared-base.css`**: Reset, `:root` layout tokens (`--radius-*`, `--toolbar-h: 52px`, `--panel-w: 350px`, `--ease-*`), body base, `.glass`, `.tool-btn` (34×34 sim-style with SVG defaults), shared keyframes, intro screen, sidebar stat patterns (`.group-label`, `.stats-header`, `.stat-row`, `.stat-value`, `.stat-group`), sim layout components (`.sim-toolbar`, `.sim-panel`, `.sim-bar`, `.sim-controls`), small utilities (`.zoom-level`, `.ctrl-sep`, `.scrollbar-thin`, `.panel-hint`), toast notifications, toggle switch CSS (`.tog-wrap`, `.tog`, `.tog-thumb`, `.tog-icon`), info tip popover styles, keyboard shortcut overlay styles, accessibility utilities (`.skip-link`, `.sr-only`, focus ring improvements), shared responsive breakpoints (900px/600px/440px toolbar/brand/sep rules), `.hide-sm`, and `prefers-reduced-motion`.

### HTML `<head>` Loading Order

All four projects follow this order in `<head>`:
1. Google Fonts `<link>` tags
2. `<link rel="stylesheet" href="/shared-base.css">` (relative `shared-base.css` on root site)
3. `<link rel="stylesheet" href="styles.css">` (project-specific overrides)
4. `<script src="/shared-tokens.js"></script>` (relative on root site)
5. `<script src="/shared-utils.js"></script>` (relative on root site)
6. `<script src="/shared-camera.js"></script>` (sim projects only; relative on root site)
7. `<script src="colors.js"></script>` (project-specific color extensions)
8. `<script src="/shared-info.js"></script>` (sim projects only; info tip popovers)
9. `<script src="/shared-shortcuts.js"></script>` (sim projects only; keyboard shortcut registry)

### colors.js Contract

Every project has a `colors.js` that **extends** the shared tokens from `shared-tokens.js` with project-specific colors:

1. `shared-tokens.js` already loaded — provides mutable `_PALETTE`, `_FONT`, `_r(hex, alpha)`, color math helpers
2. `colors.js` adds project-specific keys to `_PALETTE` and `_FONT` (often referencing `_PALETTE.extended.*`)
3. Freezes `_PALETTE.extended`, `_PALETTE.light`, `_PALETTE.dark`, `_FONT`, then `_PALETTE`
4. An IIFE injects `<style id="project-vars">` with project-specific CSS custom properties
5. JS modules and canvas code read `_PALETTE` directly; CSS reads `var(--*)` custom properties

When changing shared design tokens, change them in `shared-tokens.js` — they propagate to all projects. Project-specific tokens go in each project's `colors.js`.

### Shared Token Values

These values **must match** across all four projects (enforced by `shared-tokens.js`):

| Token | Value |
|-------|-------|
| Accent | `#FE3B01` |
| Accent light | `#FF7642` |
| Light canvas | `#F0EBE4` |
| Dark canvas | `#0C0B09` |
| Light panel | `#FCF7F2` |
| Dark panel | `#1A1510` |
| Light text | `#1A1612` |
| Dark text | `#E8DED4` |
| Font display | Noto Serif |
| Font body | Noto Sans |
| Font mono | Noto Sans Mono |
| Shadow sm/md/lg | 3-tier system (same hex alpha values) |
| Glass effect | `backdrop-filter: blur(24px) saturate(1.3)` |
| Border radius | `8px / 14px / 20px / 9999px` |

### UI Architecture

The three simulation projects share these layout conventions (root site uses a traditional page layout instead):
- **Floating glass panels** over full-viewport canvas/SVG
- **Fixed topbar** (52px height, set by shared-base.css) with glass effect
- **Right-side slide-in panel** (350px desktop, bottom sheet on mobile)
- **Intro screen** with themed splash, instruction cards, and CTA — shared CSS in `shared-base.css`, uses `--intro-warm`/`--intro-cool`/`--intro-warm-hover` CSS vars (set by each project's `colors.js`)
- **Light/dark theme** toggled via `data-theme` attribute on `<html>` (physsim/gerry/root) or `<body>` (biosim). `<html data-theme="light">` in markup prevents FOUC.
- **Entrance animations** gated by `.app-ready` class on `<body>`
- **`.glass` utility class** (shared-base.css): semi-transparent bg + blur + border + shadow
- **`.tool-btn`** pattern (shared-base.css): 34×34 icon buttons with SVG defaults. Root site overrides to 36×36.
- **Sidebar stat patterns** (shared-base.css): `.group-label`, `.stats-header`, `.stat-row`, `.stat-value` — used by biosim and gerry
- **Toast notifications** (shared-base.css): `#toast-container`, `.toast` — used by all sims via `showToast()` from `shared-utils.js`
- **Tab system** (biosim `styles.css`): `.tabs-wrap`, `.tab-bar`, `.tab-btn`, `.tab-panels`, `.tab-panel` — biosim sidebar only
- **Control group/row** (biosim `styles.css`): `.ctrl-group`, `.ctrl-row` — biosim only
- **Preset dialog** (physsim `styles.css`): `.preset-dialog`, `.preset-content`, `.preset-grid`, `.preset-card` — physsim only
- **Toggle switches** (shared-base.css): `.tog-wrap`, `.tog`, `.tog-thumb`, `.tog-icon` — used by biosim and physsim. Per-toggle color via `--tog-color` CSS var.
- **Info tips** (shared-info.js + shared-base.css): `createInfoTip()` popover system. `?` buttons with `.info-trigger` class, data objects per project. Hover on desktop, tap on mobile.
- **Keyboard shortcuts** (shared-shortcuts.js + shared-base.css): `initShortcuts()` registry. `?` key opens help overlay. Each project registers project-specific shortcuts.
- **Accessibility** (shared-base.css): `.skip-link` (skip to content), `.sr-only` (screen-reader-only), `role` attributes (`toolbar`, `complementary`, `switch`), `aria-expanded`/`aria-label`/`aria-live` on interactive elements.
- **Form controls** (physsim `styles.css`): `.slider-value`, range sliders, `.mode-toggles`, checkboxes, `.ghost-btn` — physsim and gerry
- **Shared responsive blocks** (shared-base.css): toolbar, brand, separator rules at 900px/600px/440px — all sims inherit

### Typography

- **Instrument Serif**: display/headings (large titles, preset cards)
- **Sora**: section headers (uppercase, 0.68rem, 600 weight, 0.12em tracking) — biosim/gerry sidebars
- **Geist**: body text, controls, UI labels
- **Geist Mono**: numeric/data values (`font-variant-numeric: tabular-nums`)
- Fonts loaded via `<link>` tags in HTML — never `@import` in CSS

### CSS Variable Names (Harmonized)

All four projects use these exact CSS variable names for shared tokens:

**Surfaces**: `--bg-canvas`, `--bg-panel`, `--bg-panel-solid`, `--bg-elevated`, `--bg-hover`
**Text**: `--text`, `--text-secondary`, `--text-muted`
**Borders**: `--border`, `--border-strong`
**Accent**: `--accent`, `--accent-light`, `--accent-subtle`, `--accent-glow`
**Shadows**: `--shadow-sm`, `--shadow-md`, `--shadow-lg`
**Fonts**: `--font-display`, `--font-body`, `--font-mono`
**Layout** (in shared-base.css, overridden per-project in styles.css): `--toolbar-h`, `--panel-w`, `--radius-sm/md/lg/pill`, `--ease-out`, `--ease-in-out`, `--ease-spring`

### CSS Conventions

- All color values come from CSS custom properties injected by `shared-tokens.js` + `colors.js` — no hardcoded colors in CSS
- Shared layout tokens and base patterns live in `shared-base.css`; project-specific overrides in each `styles.css`
- Alpha via 8-digit hex (`#RRGGBBAA`), not `rgba()`
- Responsive breakpoints (harmonized across all projects):

| Tier | Breakpoint | Purpose |
|------|-----------|---------|
| XL | 1100px | Panel narrowing (gerry only) |
| LG | **900px** | Primary mobile — toolbar shrinks to 48px, bottom sheet |
| MD | **600px** | Small phone — brand shrinks, tighter spacing |
| SM | 440px | Minimal UI — `.hide-sm` hides non-essential elements |

### _FONT / _PALETTE Keys (Harmonized)

**`_FONT`** keys: `display`, `body`, `mono` (all four projects, defined in `shared-tokens.js`). Biosim adds `emoji`.

**`_PALETTE`** shared keys (defined in `shared-tokens.js`):
- Top-level: `accent`, `accentLight`
- `light` / `dark` sub-objects: `canvas`, `panelSolid`, `elevated`, `text`, `textSecondary`, `textMuted`
- `extended` sub-object (10 cross-project colors):

| Key | Hex | Used by |
|-----|-----|---------|
| `blue` | `#5C92A8` | biosim (Krebs, NADH cofactor bar), gerry (Blue party) |
| `green` | `#509878` | biosim (Calvin), gerry (Green/minority) |
| `slate` | `#8A7E72` | biosim (neutral), gerry (None), physsim (neutral particle) |
| `orange` | `#CC8E4E` | biosim (Glycolysis), gerry (Yellow party) |
| `rose` | `#C46272` | biosim (PPP), gerry (Red party) |
| `purple` | `#9C7EB0` | biosim (Cyclic) |
| `brown` | `#9C6840` | biosim (Fermentation) |
| `red` | `#C05048` | biosim (Protons) |
| `cyan` | `#4AACA0` | biosim (Electrons) |
| `yellow` | `#CCA84C` | biosim (Photons) |

## Cross-Repo Changes

When modifying the **shared** design system:
1. Change `shared-tokens.js` (for shared color/font tokens) or `shared-base.css` (for shared CSS patterns) in `a9lim.github.io/`
2. All four projects pick up the change automatically (they load these files via absolute path `/shared-tokens.js` and `/shared-base.css`)
3. Test both light and dark themes in each project

When modifying **project-specific** tokens:
1. Change `colors.js` in the specific project
2. If the change affects a shared token value, update `shared-tokens.js` instead
3. The four `colors.js` files are not identical — each has project-specific colors (pathway colors in biosim, particle hues in physsim, party colors in gerry) — but shared tokens are centralized in `shared-tokens.js`

## Project-Specific Details

Each sub-folder's `CLAUDE.md` has full architecture docs. Key differences:

- **a9lim.github.io** (root site): ES6 modules, traditional page layout (not floating panels over canvas). Hosts `shared-tokens.js`, `shared-utils.js`, `shared-camera.js`, and `shared-base.css`. Overrides `--toolbar-h: 56px` and `.tool-btn` to 36×36. WebGL shader background (on-demand rendering). `main.js` + `src/` subdirectory (projects, projects-page, card-effects, router, theme, animations, shader, carousel, blog, world-map, markdown). Carousel and projects page cards are rendered dynamically from a shared `PROJECTS` data array in `src/projects.js`. Blog has loading skeletons. Footer has social links and nav.
- **physsim**: ES6 modules (`import`/`export`), Canvas 2D rendering, `window.sim` global instance, `src/` subdirectory (physics, quadtree, vec2, renderer, input, particle, presets, ui, config, relativity). Uses `shared-camera.js` for viewport management. **Boris integrator** (half-kick/rotate/half-kick/drift) for exact velocity-dependent force handling. Force toggles (gravity, Coulomb, magnetic, gravitomagnetic, relativity). **Barnes-Hut toggle** (O(N log N) approximate vs O(N²) exact pairwise). Energy conservation display (total energy with linear KE, spin KE, potential, drift% as sub-rows). Conserved quantities display (momentum with drift%, angular momentum with orbital/spin split and drift%). Force/velocity/per-force-component vector overlays. Particle inspection (hover tooltip + click-to-select sidebar detail). Merge collision conserves angular momentum. Configurable bounce friction slider. Keyboard shortcuts and info tips via shared modules. Preset dialog and form controls CSS in project `styles.css`. Responsive merges to single 900px breakpoint (bottom sheet + toolbar).
- **biosim**: ES6 modules (`import`/`export`), Canvas 2D, `_BASE`/`_ROLE`/`_THEME` color pipeline. `main.js` + `src/` subdirectory (renderer, layout, particles, enzymes, state, anim, theme, dashboard, autoplay, ui, info, regulation) with `src/reactions/` for pathway logic. **Allosteric regulation** gates enzyme reactions by cofactor ratios (PFK by ATP, IDH by NADH, G6PDH by NADPH, RuBisCO by ATP). Per-step and batch (`run_*`) regulation separated; batch keys compute minimum factor across all regulated steps. Enzyme/metabolite info data in `src/info.js`. Unavailable/inhibited enzymes visually dimmed on canvas. Active step display colored by pathway; shared enzymes show alternate pathway color in reverse direction via `_reverseColorPathway` map. Realistic initial cofactor ratios (90% ATP, 10% NADH/NADPH/FADH₂). Keyboard shortcuts and info tips via shared modules. Theme toggle on `<body>` (not `<html>`). Three-state theme: Simulation/Light/Dark. Tab system and ctrl-group/row CSS in project `styles.css`. Responsive at 900px/600px.
- **gerry**: ES6 modules (`import`/`export`), SVG rendering (not Canvas), `$` DOM cache object in `main.js`. `main.js` + `src/` subdirectory (config, hex-math, noise, hex-generator, state, metrics, renderer, input, touch, zoom, sidebar, palette, theme, plans). **All-party efficiency gap** (3-party wasted votes), **partisan symmetry**, **competitive districts** metrics. Dynamic majority-minority district requirement. **Plan save/load** (localStorage + JSON export/import). **Brush sizes** (1/3/7 hex radius). **Auto-fill** (greedy nearest-neighbor). Keyboard shortcuts and info tips via shared modules. Overrides `--palette-h: 56px`. Responsive at 1100px/900px/600px/440px (inherits shared toolbar/brand/sep rules).
