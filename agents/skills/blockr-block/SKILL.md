---
name: blockr-block
description: |
  Use when adding a new block to a blockr package (transform, plot, data,
  join, variadic). Walks through the R-driven vs JS-driven choice, scaffolds
  the files, writes the matching tests, and verifies the block in a browser
  via Playwright. Trigger on phrases like "create a block", "add a new
  block", "write a transform block", "build a select/filter/mutate block in
  blockr.foo".
argument-hint: "[block-name] [package]"
---

# blockr-block

Add a new block to a blockr package the right way: pick a pattern, scaffold the files, write the matching tests, verify it works in a real Shiny session.

## On invocation

1. **Identify the package and block type.** If `$ARGUMENTS` doesn't make both clear, ask.
2. **Pick the pattern.** Two options with different cost/UX trade-offs and different testing strategies. **Don't guess** — ask the user.

## Picking the pattern

Full chooser: `blockr.docs/patterns/README.md`. Short version:

- **R-driven** — pure-R Shiny module returning `expr` + `state`. Faster to write, easier to debug, `testServer()` is sufficient. Right for simple blocks, internal tools, prototypes, the on-ramp into the framework. Used by `blockr.core`'s built-ins.
- **JS-driven** — custom JS class wired through a Shiny input binding. Materially better UX (multi-row builders, autocomplete, drag handles, instant client-side feedback). Used throughout `blockr.dplyr`. Reach for this when polish matters and stock Shiny inputs can't deliver.

If the target package's existing blocks are uniformly one pattern, follow suit unless the user explicitly wants the other.

## R-driven path

Reference: `blockr.docs/patterns/r-driven-blocks.md`.

Scaffold:

- `R/<name>_block.R` — constructor + server + UI in one file
- `tests/testthat/test-<name>_block.R` — `testServer()`-based tests

The constructor returns `new_*_block()` (`new_transform_block`, `new_data_block`, etc.). The server returns `list(expr = reactive(...), state = list(...))`. **State names must match constructor argument names exactly** — serialization breaks silently otherwise.

Rules that bite:
- Use `bquote()` for expression building, never `paste()` or string interpolation.
- Don't expose data inputs (`data`, `x`, `y`, `...args`) as constructor arguments.
- Forward `...` to the parent constructor.

### Testing (R-driven)

Two tiers, **no `shinytest2`**:

1. **Unit tests** for any pure helpers (expression builders, parsers).
2. **`testServer()`** for everything Shiny. Pattern A: `expr_server` for expression + state. Pattern B: `block_server` for materialized result. Both are documented in r-driven-blocks.md "Testing".

`session$setInputs()` simulates UI interaction. No browser needed.

## JS-driven path

Reference: `blockr.docs/patterns/js-driven-blocks.md`.

Scaffold (four files):

- `inst/js/<name>-block.js` — JS class + Shiny input binding
- `inst/css/<name>-block.css` — block-specific CSS
- `R/<name>_block.R` — R constructor with `expr_type = "bquoted"`, `external_ctrl = TRUE`, `allow_empty_state = "state"`
- `R/expr-builders.R` — append the pure-R expression builder for the new block

Reference implementations: `blockr.dplyr/R/filter_block.R` + `blockr.dplyr/inst/js/filter-block.js`.

Rules that bite:
- Single `state` reactiveVal whose name matches the constructor parameter.
- Bidirectional sync requires the `self_write` env guard to prevent R→JS→R loops.
- Use `Blockr.Select` / `Blockr.Input` shared components rather than rolling your own dropdowns or inputs.

### Testing (JS-driven)

Three tiers — `shinytest2` is necessary because `session$setInputs()` cannot drive a custom JS input binding:

1. **Unit tests** for the expression builder in `R/expr-builders.R`. Use the `eval_bquoted` helper from the pattern doc.
2. **`testServer()`** to verify constructor state → expression. Inject state via the constructor or push to `r_state` directly inside the test.
3. **`shinytest2`** for the JS round-trip. Drive the block via `app$run_js()` calling `el._block.setState(...)` and `el._block._submit()`. Reference test app: `blockr.dplyr/tests/testthat/apps/dplyr-e2e/app.R`. Keep one happy-path test per block here — push everything else into Tiers 1 and 2.

## Verification

Once tests pass, verify the block runs end-to-end in a browser. Hand off to the **`blockr-playwright`** skill:

- Launch the package's preview app (e.g. `dev/preview-all-blocks.R`, or a minimal `serve(new_<name>_block(), list(data = ...))` on port 3838).
- Screenshot the block in its empty state and after a typical interaction.
- Check: UI renders without console errors, the block produces output downstream, the empty → configured transition is clean.

Tests passing isn't the same as the block working in a real Shiny session. Don't skip this step.

## Don'ts

- **Don't skip the pattern choice.** Silently defaulting wastes work later.
- **Don't add `shinytest2` to an R-driven block.** `testServer()` covers it in milliseconds. The exception is visual regression on CSS, rarely worth the maintenance.
- **Don't duplicate expression-building logic in the constructor.** Helpers go in `R/expr-builders.R` so they're unit-testable in isolation.
- **Don't skip the Playwright verification.** Tests passing isn't the same as the block actually working.

## When you're done

If the block is being registered for AI/MCP discovery, add a `register_blocks()` call — see js-driven-blocks.md "Registry" section. Update the package's pkgdown reference if needed.
