# site-meta

Meta-repository coordinating four independently-tracked GitHub repos that together form the portfolio site at **[a9l.im](https://a9l.im)**.

This folder is not deployed — it exists only for cross-repo design coordination. Each sub-folder has its own `.git` and is deployed independently to GitHub Pages.

## Projects

| Project | Live | Repo | Description |
|---------|------|------|-------------|
| **Portfolio** | [a9l.im](https://a9l.im) | [a9lim.github.io](https://github.com/a9lim/a9lim.github.io) | Landing page, blog, about — hosts shared design system |
| **Relativistic N-Body** | [a9l.im/physsim](https://a9l.im/physsim) | [physsim](https://github.com/a9lim/physsim) | Barnes-Hut physics simulation with relativistic mechanics |
| **Cellular Metabolism** | [a9l.im/biosim](https://a9l.im/biosim) | [biosim](https://github.com/a9lim/biosim) | Interactive visualization of 8 metabolic pathways |
| **Redistricting Sim** | [a9l.im/gerry](https://a9l.im/gerry) | [gerry](https://github.com/a9lim/gerry) | Gerrymandering simulator on a hex-tile map |

## Shared Design System

All four projects share a unified design language via two files hosted at the root site:

- **`shared-tokens.js`** — Color palette, font stacks, color math helpers, CSS custom property injection
- **`shared-base.css`** — Reset, layout tokens, glass panels, tool buttons, intro screen, tab system, form controls, responsive breakpoints

Zero dependencies across the board — vanilla JS/HTML/CSS, no build step, no bundler.

## Local Development

Serve any project from the `a9lim.github.io/` directory so shared files resolve:

```bash
cd a9lim.github.io
python -m http.server
```

Sub-projects are accessible at `/physsim`, `/biosim`, `/gerry` when their folders are siblings inside the served root.

## License

[AGPL-3.0](LICENSE)
