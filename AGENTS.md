# blockr.docs — Agent Primer

Durable, reference-style documentation for the blockr ecosystem: block-building
patterns, starter packages, decision records. The design system lives in
blockr.ui (`vignettes/articles/design-system.Rmd`).

If something would still be true six months from now after the code has moved,
it belongs here. Otherwise:

- **Specs for in-progress work** → the workspace's specs repo, not here.
- **Notes tied to a package's current code state** → that package's `dev/`
  folder.

## Layout

```
patterns/       "How we build X". README.md is the chooser.
scaffolds/      Copy-and-rename starter block packages (rblock, jsblock), tests green.
decisions/      ADRs: numbered, dated, immutable. New ADR for any change of stance.
agents/         Claude Code skills and prompt notes.
```

Each folder has a `README.md` that indexes its contents. Start there.

## Where to look first

| Task | File |
|---|---|
| Start a new block | `patterns/README.md` (chooser) → `r-driven-blocks.md` or `js-driven-blocks.md`; new package → copy from `scaffolds/` |
| Style a UI element, or decide how a control behaves | blockr.ui, `vignettes/articles/design-system.Rmd` |
| Write tests for a block | the testing section of the relevant `patterns/` guide, plus the scaffold's `tests/` |
| Understand a past decision | `decisions/README.md` |
| Install a Claude Code skill | `agents/skills/README.md` |

## Conventions

- **Code refs** with file path and line number (`blockr.dplyr/R/filter_block.R:51`)
  so rot is obvious.
- **Relative links** within blockr.docs; `{package}/{path}` for code.
- **This repo is public.** No internal branch names, no links into private
  repos, no pointers to a package's `dev/` folder as if a reader could open it.
  If a fact only exists in an internal note, restate the fact here rather than
  linking to it.
- **Keep docs tight** — paragraphs, not essays. Split anything past ~300 lines.
- **Doc PRs travel with code PRs** that change the described pattern. Don't
  defer.
- **ADRs are append-only** — see `decisions/README.md` for the format.
