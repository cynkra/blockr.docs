# blockr.docs — Agent Primer

You are an AI assistant operating in `blockr.docs`, the shared developer documentation repository for the blockr ecosystem. This file orients you.

## What this repo is

Durable, reference-style documentation for the blockr packages: design system, patterns, decision records, testing guidelines, infrastructure notes, ecosystem map.

Not to be confused with:

- **blockr.design** — temporal specs for individual pieces of work (motivation → requirements → design → implementation). Lives its own lifecycle per topic.
- **Per-package `dev/` folders** — package-specific notes tied to current code state (e.g. a particular block's quirks, benchmark snapshots). Stay with the code.

If something would still be true six months from now after the code has moved around, it belongs here. Otherwise it belongs in a package's `dev/` folder or in a blockr.design spec.

## Structure

```
blockr.docs/
  README.md                    # humans land here
  AGENTS.md                    # (this file) agents / LLMs land here
  CLAUDE.md                    # pointer → AGENTS.md (Claude Code auto-loads this)
  decisions/                   # architecture decision records (ADRs), dated, immutable
    README.md                  # index of all decisions
    0001-hand-rolled-vs-libraries.md
    ...
  patterns/                    # durable "how we build X" guides
    README.md
    js-driven-blocks.md
    column-metadata-forwarding.md
    portal-dropdowns.md
  design-system/               # visual primitives + shared components
    README.md
    colors.md
    typography.md
    spacing-and-sizing.md
    components/
      blockr-select.md
      blockr-input.md
  testing/                     # testthat patterns, Playwright workflows, devtools::check
  infra/                       # dev container, git mount caveats, port forwarding
  ecosystem/                   # package map, dep graph, release ordering
  agents/                      # prompt engineering notes, skill descriptions, agent-authored guides
```

Each folder has a `README.md` that indexes its contents. Start there if you're grepping for "everything about X."

## Conventions

### Decision records (ADRs)

- Filename: `NNNN-slug.md` (`0001-...`, `0002-...`, zero-padded, monotonic).
- Frontmatter:
  ```
  ---
  id: 0001
  title: Hand-rolled vs external UI libraries
  status: Accepted
  date: 2026-04-19
  supersedes: (id or none)
  superseded-by: (id or none)
  ---
  ```
- Structure: `Context / Decision / Consequences / Triggers for revisit`.
- Immutable once accepted. If the stance changes, write a *new* ADR and mark the old one `Superseded`.

### Code references

When you reference code, include the file path and line number at the time of writing: `blockr.dplyr/R/filter_block.R:51`. This lets future readers (human or LLM) quickly tell whether the doc has rotted.

### Cross-links

Link to other blockr.docs files using relative paths. Link to package code using `{package}/{path}` style (no leading slash). Link to blockr.design specs with full `blockr.design/open|done|abandoned/<topic>/` path.

### Keep it short

Each doc is a few paragraphs, not an essay. If a doc exceeds ~300 lines, split it.

## How to use this repo when working in other blockr packages

If you're an agent working on `blockr.dplyr` or similar, this repo is your canonical reference for:

- **Visual style** (colors, spacing, typography) → `design-system/`
- **How to structure a new block** → `patterns/js-driven-blocks.md`
- **Testing conventions** → `testing/`
- **Why was X decided** → `decisions/` (search by keyword)
- **Ecosystem layout / who depends on whom** → `ecosystem/package-map.md`

The workspace-level `/workspace/CLAUDE.md` should point here. Follow that pointer for authoritative answers; each package's `dev/` folder overrides only with package-specific exceptions.

## When you modify this repo

- A doc PR is often bundled with a code PR that changes the described pattern. Treat doc updates as part of the code change, not a separate cleanup pass that may never come.
- ADRs get added, not edited. If the answer changed, new ADR.
- Pattern docs get edited freely but include a "last reviewed" line if stale content is a concern.

## Where to look first for common tasks

| Task | File |
|---|---|
| Start a new block type | `patterns/js-driven-blocks.md` |
| Style a new UI element | `design-system/` + `decisions/0001-hand-rolled-vs-libraries.md` |
| Write a testthat test for a block | `testing/testthat-patterns.md` |
| Run end-to-end tests | `testing/playwright-e2e.md` |
| Understand a recent architectural decision | `decisions/README.md` (index) |
| Navigate the devcontainer / git quirks | `infra/` |
