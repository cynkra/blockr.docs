# blockr.docs

How blockr is put together, and how to build with it. Shared developer
documentation for the whole ecosystem: design system, block-building patterns,
starter packages, architectural decisions.

For what blockr *is* and how to use it, start at
[blockr.site](https://blockr.site). This repo is for
people writing blockr code.

**Agents / LLMs: read [AGENTS.md](./AGENTS.md) first.**

## Start here

| If you want to… | Go to |
|---|---|
| Write your first block | [patterns/README.md](./patterns/README.md) — pick R-driven or JS-driven, then follow that guide |
| Start a whole new block package | [scaffolds/](./scaffolds/) — copy `rblock/` or `jsblock/`, rename, morph. Both open with tests green |
| Style a control so it looks native | [design-system/](./design-system/) — tokens, components, and the behavioural rules |
| Know how a control should *behave* | [design-system/ux-principles.md](./design-system/ux-principles.md) — defaults, required-empty cues, what each label and hint is allowed to say |
| Generate UI with an LLM | [design-system/design-system-prompt.md](./design-system/design-system-prompt.md) — the whole system as one paste-ready brief |
| Understand why something is built the way it is | [decisions/](./decisions/) |
| Install the Claude Code skills | [agents/skills/README.md](./agents/skills/README.md) |

## The shape of the ecosystem

blockr is a Shiny dashboard toolkit. A **block** is a small self-contained
unit of UI plus logic (filter, mutate, plot); blocks compose into a **board**,
a DAG where each block's output feeds the next. Packages layer:

```
blockr.core     data model — board / block / link / stack, reactivity,
                serialization, the block registry. No CSS.
blockr.ui       presentation — design tokens, block card, app shell, sidebar
blockr.dock     docking — the DockView workspace, views, layouts
blockr.theme    data colour — chart palettes, ramps, scale maps
blockr.{dplyr, viz, ggplot, io, dm, …}   the blocks themselves
```

Two things follow from that layering, and most of this repo is downstream of
them:

- **A block owns its own UI.** There is no framework rendering your controls
  for you, which is why the [design system](./design-system/) is written down
  rather than enforced by a component library.
- **A block returns an expression, not a result.** Blocks build code; the board
  evaluates it. That is what makes a board serializable and exportable to a
  script, and it shapes how blocks are written and tested — see
  [patterns/](./patterns/).

## What belongs in this repo

Durable reference material that stays true across refactors. If a doc would
still make sense six months from now after the code has moved, it belongs here.

What does *not*:

- **Specs for planned or in-progress work** — those are temporal by nature and
  live in a separate specs repo, one folder per topic, following motivation →
  requirements → design → implementation.
- **Notes tied to a package's current code state** — those stay in that
  package's `dev/` folder.

The test is whether the doc describes a *decision* or a *situation*. Decisions
last; situations get fixed and the doc rots.

## Layout

```
patterns/        "How we build X": R-driven vs JS-driven blocks, scale maps.
scaffolds/       Copy-and-rename starter block packages, tests green.
design-system/   Tokens, components, UX principles, the LLM brief.
decisions/       Architecture decision records. Numbered, dated, immutable.
agents/          Claude Code skills and prompt notes.
```

Each folder has a `README.md` indexing its contents.
