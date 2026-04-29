# blockr.docs — Agent Primer

Durable, reference-style documentation for the blockr ecosystem: design system, patterns, decision records, testing guidelines, infra notes, ecosystem map.

If something would still be true six months from now after the code has moved, it belongs here. Otherwise:

- **Temporal specs** for in-progress work → `blockr.design/`.
- **Package-specific notes** tied to current code state → that package's `dev/` folder.

## Layout

```
decisions/      ADRs: numbered, dated, immutable. New ADR for any change of stance.
patterns/       "How we build X". README.md is the chooser.
design-system/  Colors, typography, spacing, shared components.
testing/        testthat patterns, Playwright workflows, devtools::check.
infra/          Dev container, git mount caveats, port forwarding.
ecosystem/      Package map, dep graph, release ordering.
agents/         Prompt/skill notes.
```

Each folder has a `README.md` that indexes its contents. Start there.

## Where to look first

| Task | File |
|---|---|
| Start a new block | `patterns/README.md` (chooser) → `r-driven-blocks.md` or `js-driven-blocks.md` |
| Style a UI element | `design-system/` + `decisions/0001-hand-rolled-vs-libraries.md` |
| Write tests for a block | `testing/` (and the testing section of the relevant pattern doc) |
| Understand a past decision | `decisions/README.md` |
| Devcontainer / git quirks | `infra/` |

## Conventions

- **Code refs** with file path and line number (`blockr.dplyr/R/filter_block.R:51`) so rot is obvious.
- **Relative links** within blockr.docs; `{package}/{path}` for code; full path for blockr.design specs.
- **Keep docs tight** — paragraphs, not essays. Split anything past ~300 lines.
- **Doc PRs travel with code PRs** that change the described pattern. Don't defer.
- **ADRs are append-only** — see `decisions/README.md` for the format.
