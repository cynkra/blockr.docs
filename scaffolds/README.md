# Block scaffolds

Two copy-and-rename starter packages for writing a custom block. Both ship
the **same block** — `myplot`: pick a grouping column and a numeric column,
get a boxplot with jittered points — built once per pattern, so the diff
between the two packages is exactly the R-driven vs JS-driven decision
(`../patterns/README.md`).

| | [`rblock/`](./rblock) | [`jsblock/`](./jsblock) |
|---|---|---|
| Package | `blockr.rblock` | `blockr.jsblock` |
| Pattern | R-driven (`patterns/r-driven-blocks.md`) | JS-driven (`patterns/js-driven-blocks.md`) |
| Controls | stock Shiny `selectInput` | JS class + shared `Blockr.Select` |
| Tests | tiers 1–2 (`testServer()`, no browser) | tiers 1–3 (incl. one `shinytest2` round-trip) |
| Extra setup | none | typed JS (`tsconfig.json`, `inst/js/types.d.ts`) |

Both open **green**: tests pass, the block registers, and `app.R` serves a
dock board (dataset → myplot, with the DAG view) where the block works. The
point is that an agent's job is never "invent a valid block from scratch" —
it is "keep this green while turning myplot into the block you actually
want."

## Install

Both scaffolds need blockr.core, blockr.dock and blockr.dag; jsblock also
needs blockr.dplyr. The blockr package installs all of them:

```r
pak::pak("BristolMyersSquibb/blockr@dev")
```

## What to edit, per scaffold

- **The plot** lives in `R/expr-builders.R` (`make_myplot_expr()`) in both
  packages — a pure function from state to quoted ggplot2 code, unit-tested
  in isolation. This is where a new visualization starts.
- **The controls**: `rblock` wires stock inputs in `R/myplot_block.R`;
  `jsblock` owns them in `inst/js/myplot-block.js` (state contract in
  `inst/js/types.d.ts` — update it first, it is the R↔JS protocol).
- **Do not edit** `jsblock/R/js-block.R` (the vendored sync/lifecycle
  factory) — it is the part that breeds bugs when reinvented.

## Prompts

The procedure (copy the scaffold, rename per the checklist below, morph,
keep tests green, verify in the board demo) lives in
`../agents/skills/blockr-block/SKILL.md` — user prompts only need to supply
what the skill cannot infer: the package name, what the block does, and the
pattern (the skill defaults to R-driven unless asked). Final user-facing
wording lives with the blockr.site learn page.

> Read blockr.docs/agents/skills/blockr-block/SKILL.md and follow it to
> create a new package blockr.<name> with a <visualization you want, e.g.
> "an event timeline block: one horizontal lane per subject, a point per
> event positioned by a time column, colored by event type">.
> Use the JS-driven pattern.

Drop the last sentence (or say "Use the R-driven pattern") for the R track.

## Renaming checklist

Renaming the **package**:

1. Folder + `Package:` in DESCRIPTION. jsblock: renaming the `inst/js` /
   `inst/css` files counts as an asset edit — bump `Version:` (htmlDependency
   caches by package version).
2. `tests/testthat.R` and the `library()` call in it.
3. jsblock only: the `pkg <-` line in `R/js-block.R`.

Renaming the **block** (myplot → yours), both scaffolds:

4. Constructor name + `class =` + file names in `R/` and `tests/testthat/`,
   the `ctor` string in `R/zzz.R`, and the export line in NAMESPACE.
   NAMESPACE is maintained **by hand** in these scaffolds despite its
   roxygen header — edit the export directly (or wire up roxygen2 yourself,
   the roxygen comments are kept consistent).
5. jsblock only: the `name = "myplot"` convention end to end — JS
   `registerBlock()` + message names, asset file names, container class,
   `types.d.ts`, the tier-2 `session$setInputs(myplot_input = ...)` name,
   and the input id + plot selector in `tests/testthat/test-shinytest2.R`.

Done-signal: `grep -rn "myplot" .` returns nothing. A missed rename in a
test id fails silently — the tier-3 test drives nothing.
