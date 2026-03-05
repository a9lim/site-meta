# Documentation Overhaul Design

## Goal

Rewrite all code comments, READMEs, CLAUDE.md files, and info tooltips across all 4 sub-repositories and the meta-repo to be accurate, concise, informative, and useful.

## Style Guide

| Document | Audience | Tone | Focus |
|----------|----------|------|-------|
| README.md | Visitors, employers, collaborators | Approachable, educational | What it does, how to run it, what's interesting |
| CLAUDE.md | AI dev assistant (Claude Code) | Architect-level reference | File map, data flow, patterns, gotchas, invariants |
| Info tooltips | End users of the sims | User-friendly, concise | What the concept means, 1-3 sentences |
| Code comments | Developers reading source | Terse, technical | WHY not WHAT, formulas, invariants, non-obvious decisions |

## Code Comment Principles

- Remove comments that restate what the code does
- Add comments where the why is non-obvious (physics formulas, algorithmic choices, perf hacks)
- Keep section headers (`// --- Section ---`) for navigation
- JSDoc on exported functions with non-obvious signatures
- Fix any stale or incorrect comments

## Execution Order

1. Meta-repo CLAUDE.md — verify accuracy, rewrite for clarity
2. a9lim.github.io (root): README, CLAUDE.md, shared-*.js, src/*.js, CSS, HTML
3. physsim: README, CLAUDE.md, info tooltips, all src/*.js, CSS, HTML
4. biosim: README, CLAUDE.md, info tooltips, all src/*.js + src/reactions/*.js, CSS, HTML
5. gerry: README, CLAUDE.md, info tooltips (inline in main.js), all src/*.js, CSS, HTML

## Constraints

- Keep file structure and module boundaries unchanged
- Keep section header comment style
- Keep CLAUDE.md depth similar to current
- Enforce "signal delay" over "retarded potential" terminology
- No functional code changes — documentation only
