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

Two files at the root site are loaded by all four projects via absolute paths (`/shared-tokens.js`, `/shared-base.css`):

- **`shared-tokens.js`**: Color math helpers (`_r`, `_parseHex`, `_rgb2hsl`, `_hsl2hex`, `_darken`), `_FONT`, and `_PALETTE` (shared surface/text/accent/shadow tokens plus `extended` sub-object with 10 cross-project colors). Loaded as a plain `<script>` in `<head>` before project-specific `colors.js`.
- **`shared-base.css`**: Reset, `:root` layout tokens (`--radius-*`, `--toolbar-h: 52px`, `--panel-w: 350px`, `--ease-*`), body base, `.glass`, `.tool-btn` (34×34 sim-style with SVG defaults), shared keyframes, intro screen, sidebar stat patterns (`.group-label`, `.stats-header`, `.stat-row`, `.stat-value`, `.stat-group`), tab system (`.tabs-wrap`, `.tab-bar`, `.tab-btn`, `.tab-panels`, `.tab-panel`), control group/row (`.ctrl-group`, `.ctrl-row`), `.slider-value`, preset dialog (`.preset-dialog`, `.preset-content`, `.preset-grid`, `.preset-card`), shared responsive breakpoints (900px/600px/440px toolbar/brand/sep rules), `.hide-sm`, form controls (range sliders, `.mode-toggles`, checkboxes, `.ghost-btn`), and `prefers-reduced-motion`.

### HTML `<head>` Loading Order

All four projects follow this order in `<head>`:
1. Google Fonts `<link>` tags
2. `<link rel="stylesheet" href="/shared-base.css">` (relative `shared-base.css` on root site)
3. `<link rel="stylesheet" href="styles.css">` (project-specific overrides)
4. `<script src="/shared-tokens.js"></script>` (relative on root site)
5. `<script src="colors.js"></script>` (project-specific color extensions)

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
- **Tab system** (shared-base.css): `.tabs-wrap`, `.tab-bar`, `.tab-btn`, `.tab-panels`, `.tab-panel` — used by biosim sidebar
- **Control group/row** (shared-base.css): `.ctrl-group`, `.ctrl-row` — used by biosim and physsim
- **Slider value** (shared-base.css): `.slider-value` — monospace numeric display
- **Preset dialog** (shared-base.css): `.preset-dialog`, `.preset-content`, `.preset-grid`, `.preset-card` — used by physsim
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
| `blue` | `#5C92A8` | biosim (Krebs), gerry (Blue party) |
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

- **a9lim.github.io** (root site): ES6 modules, traditional page layout (not floating panels over canvas). Hosts `shared-tokens.js` and `shared-base.css`. Overrides `--toolbar-h: 56px` and `.tool-btn` to 36×36. WebGL shader background. `main.js` + `src/` subdirectory (router, theme, animations, shader, carousel, blog, world-map, markdown).
- **physsim**: ES6 modules (`import`/`export`), Canvas 2D rendering, `window.sim` global instance, `src/` subdirectory (physics, quadtree, vec2, renderer, input, particle, presets, ui). Preset dialog uses shared `.preset-dialog` class. Responsive merges to single 900px breakpoint (bottom sheet + toolbar).
- **biosim**: ES6 modules (`import`/`export`), Canvas 2D, `_BASE`/`_ROLE`/`_THEME` color pipeline. `main.js` + `src/` subdirectory with `src/reactions/` for pathway logic. Theme toggle on `<body>` (not `<html>`). Three-state theme: Simulation/Light/Dark. Uses shared tab system for sidebar. Responsive at 900px/600px.
- **gerry**: ES6 modules (`import`/`export`), SVG rendering (not Canvas), `$` DOM cache object in `main.js`. `main.js` + `src/` subdirectory (config, hex-math, noise, hex-generator, state, metrics, renderer, input, touch, zoom, sidebar, palette, theme). Overrides `--palette-h: 56px`. Responsive at 1100px/900px/600px/440px (inherits shared toolbar/brand/sep rules).
