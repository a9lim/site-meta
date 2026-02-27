# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Purpose

This is a meta-repository coordinating three independently-tracked GitHub repos that together form the portfolio site at **a9l.im** (hosted via `a9lim.github.io`). The goal is design, style, and theme consistency across all sub-sites.

## Repository Map

| Folder | GitHub Repo | Domain Path | Description |
|--------|------------|-------------|-------------|
| `a9lim.github.io/` | `a9lim/a9lim.github.io` | `a9l.im` (root) | Portfolio landing page |
| `physsim/` | `a9lim/physsim` | `a9l.im/physsim` | Relativistic N-body simulation |
| `biosim/` | `a9lim/biosim` | `a9l.im/biosim` | Cellular metabolism visualization |
| `gerry/` | `a9lim/gerry` | `a9l.im/gerry` | Gerrymandering/redistricting simulator |

Each sub-folder has its own `.git`, `CLAUDE.md`, and is deployed independently to GitHub Pages. This meta-folder is **not** itself a git repo — it exists only for cross-repo coordination.

## Shared Tech Stack

All three projects are **zero-dependency vanilla JS/HTML/CSS** static sites. No build step, no bundler, no npm. Serve any project locally with:
```bash
python -m http.server          # or
npx serve .
```

## Shared Design System

### colors.js Contract

Every project has a `colors.js` that is the **single source of truth** for colors and fonts. It follows this exact pattern:

1. Loaded as a plain `<script>` in `<head>` before any other scripts or stylesheets that depend on it
2. Exposes frozen globals: `_PALETTE`, `_FONT`, `_r(hex, alpha)`
3. An IIFE injects `<style id="palette-vars">` with CSS custom properties (`:root` for light, `[data-theme="dark"]` for dark)
4. JS modules and canvas code read `_PALETTE` directly; CSS reads `var(--*)` custom properties

When changing design tokens, change them in `colors.js` — they propagate to both CSS and JS automatically.

### Shared Token Values

These values **must match** across all three projects:

| Token | Value |
|-------|-------|
| Accent | `#FE3B01` |
| Accent light | `#FF6B3D` |
| Light canvas | `#F0EDE4` |
| Dark canvas | `#0C0B09` |
| Light panel | `#FCFAF4` |
| Dark panel | `#181612` |
| Light text | `#1A1612` |
| Dark text | `#E8E2D4` |
| Font display | Instrument Serif |
| Font body | Geist |
| Font mono | Geist Mono |
| Shadow sm/md/lg | 3-tier system (same hex alpha values) |
| Glass effect | `backdrop-filter: blur(24px) saturate(1.3)` |
| Border radius | `8px / 14px / 20px / 9999px` |

### UI Architecture

All projects share these layout conventions:
- **Floating glass panels** over full-viewport canvas/SVG (not traditional page layout)
- **Fixed topbar** (52px height) with glass effect
- **Right-side slide-in panel** (350px desktop, bottom sheet on mobile)
- **Light/dark theme** toggled via `data-theme` attribute on `<html>` (physsim/gerry) or `<body>` (biosim). `<html data-theme="light">` in markup prevents FOUC.
- **Entrance animations** gated by `.app-ready` class on `<body>`
- **`.glass` utility class**: semi-transparent bg + blur + border + shadow
- **`.tool-btn`** pattern: 34x34 icon buttons with SVG defaults set via CSS

### Typography

- **Instrument Serif**: display/headings (large titles, preset cards)
- **Sora**: section headers (uppercase, 0.68rem, 600 weight, 0.12em tracking) — biosim/gerry sidebars
- **Geist**: body text, controls, UI labels
- **Geist Mono**: numeric/data values (`font-variant-numeric: tabular-nums`)
- Fonts loaded via `<link>` tags in HTML — never `@import` in CSS

### CSS Variable Names (Harmonized)

All three projects now use these exact CSS variable names for shared tokens:

**Surfaces**: `--bg-canvas`, `--bg-panel`, `--bg-panel-solid`, `--bg-elevated`, `--bg-hover`
**Text**: `--text`, `--text-secondary`, `--text-muted`
**Borders**: `--border`, `--border-strong`
**Accent**: `--accent`, `--accent-light`, `--accent-subtle`, `--accent-glow`
**Shadows**: `--shadow-sm`, `--shadow-md`, `--shadow-lg`
**Fonts**: `--font-display`, `--font-body`, `--font-mono`
**Layout** (in styles.css): `--toolbar-h`, `--panel-w`, `--radius-sm/md/lg/pill`, `--ease-out`, `--ease-in-out`, `--ease-spring`

### CSS Conventions

- All color values come from CSS custom properties injected by `colors.js` — no hardcoded colors in `styles.css`
- Layout-only tokens (`--radius-*`, `--toolbar-h`, `--panel-w`, `--ease-*`) remain in `styles.css`
- Alpha via 8-digit hex (`#RRGGBBAA`), not `rgba()`
- Responsive breakpoints: ~900px (mobile/bottom-sheet), ~600px (small phone), ~440px (minimal UI)

### _FONT / _PALETTE Keys (Harmonized)

**`_FONT`** keys: `display`, `body`, `mono` (all three projects).

**`_PALETTE`** shared keys: `accent`, `accentLight`, then `light`/`dark` sub-objects each with: `canvas`, `panelSolid`, `elevated`, `text`, `textSecondary`, `textMuted`.

## Cross-Repo Changes

When modifying the design system:
1. Change `colors.js` in the originating project
2. Propagate matching changes to the other two `colors.js` files
3. Verify the `_PALETTE` light/dark surface values, accent colors, shadow definitions, and font stacks remain identical
4. Test both light and dark themes in each project

The three `colors.js` files are not identical — each has project-specific colors (pathway colors in biosim, particle hues in physsim, party colors in gerry) — but the **shared tokens** listed above must stay synchronized.

## Project-Specific Details

Each sub-folder's `CLAUDE.md` has full architecture docs. Key differences:

- **physsim**: ES6 modules (`import`/`export`), Canvas 2D rendering, `window.sim` global instance, `src/` subdirectory
- **biosim**: IIFE pattern (non-module `<script>` tags, load-order dependent), Canvas 2D, `_BASE`/`_ROLE`/`_THEME` color pipeline
- **gerry**: IIFE pattern, SVG rendering (not Canvas), `$` DOM cache object, `_darken()` color helper
