# blockr.docs

Shared developer documentation for the blockr ecosystem: design system, patterns, architectural decisions, testing, infra.

**Agents / LLMs: see [AGENTS.md](./AGENTS.md) first.**

## What belongs here

Durable reference material that stays true across refactors. If a doc would still make sense six months from now after the code has moved, it belongs here.

What does *not* belong:

- **Temporal specs** for planned or in-progress work → `blockr.design/open|done|abandoned/<topic>/` (motivation → requirements → design → implementation).
- **Package-specific notes** tied to current code state → stay in the package's `dev/` folder.

## Layout

```
decisions/       — Architecture decision records (ADRs). Numbered, dated, immutable.
patterns/        — "How we build X" guides: JS-driven blocks, column metadata forwarding, etc.
design-system/   — Colors, typography, spacing, shared component specs (select, input, …).
testing/         — testthat patterns, Playwright e2e, devtools::check workflows.
infra/           — Dev container, git mount caveats, port forwarding notes.
ecosystem/       — Package dependency map, release ordering, who owns what.
agents/          — Prompt engineering notes, skill docs, agent-authored guides.
```

Each folder has a `README.md` indexing its contents.

## Quick links

- [Decisions index](./decisions/README.md) — all ADRs in chronological order.
- [Agent primer](./AGENTS.md) — how AI tooling should operate in this repo.
