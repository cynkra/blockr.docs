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

1. **Identify the package, block type, and what the block does.** Ask only if unclear. Most prompts include the data source and the parameters — that's enough.
2. **Pick the pattern.** Default to **R-driven** for new packages, simple blocks, and first-time block authors. Pick **JS-driven** only if the user explicitly asks for it, OR the target package's existing blocks are uniformly JS-driven. Don't ask the user unless the package signals are mixed.
3. **Pick a constructor name.** Convention: `new_<readable_form>_block()`. Split compound package suffixes on logical word boundaries (`blockr.catfacts` → `new_cat_facts_block`, `blockr.dplyr` → multiple — one per verb). Tell the user the chosen name in your first message so they can flag it before you write code.
4. **Scaffold + register + test + verify** as one unit. Registration is not optional — see "Scaffolding checklist" below.

## Pattern reference

Full chooser: `blockr.docs/patterns/README.md`.

- **R-driven** — pure-R Shiny module returning `expr` + `state`. Faster to write, easier to debug, `testServer()` is sufficient. Right for simple blocks, internal tools, prototypes, the on-ramp into the framework. Used by `blockr.core`'s built-ins.
- **JS-driven** — custom JS class wired through a Shiny input binding. Materially better UX (multi-row builders, autocomplete, drag handles, instant client-side feedback). Used throughout `blockr.dplyr`. Reach for this when polish matters and stock Shiny inputs can't deliver.

## R-driven path

Reference: `blockr.docs/patterns/r-driven-blocks.md`.

### Scaffolding checklist

For an existing package, write/update:
- `R/<name>_block.R` — constructor + server + UI in one file.
- `R/zzz.R` — `.onLoad()` calling `blockr.core::register_blocks(ctor = ..., name = ..., description = ..., category = ..., package = pkgname)`. Always register; otherwise the constructor warns on every call.
- `tests/testthat/test-<name>_block.R` — `testServer()`-based tests.
- `DESCRIPTION` — add the block's runtime deps to `Imports`.
- `NAMESPACE` — `importFrom(blockr.core, bbquote)` plus any roxygen-driven exports.

For a brand-new package, also create: `DESCRIPTION` (with `Imports: blockr.core, shiny` plus block deps), `NAMESPACE` (`export(new_<name>_block)`, `importFrom(blockr.core, bbquote)`), `.Rbuildignore`. The `category` for `register_blocks()` must be one of `blockr.core::suggested_categories()` — `input`, `transform`, `structured`, `plot`, `table`, `model`, `output`, `utility`, `uncategorized`. Data-fetching blocks are `input`, not `data`.

### Construction rules

The constructor returns `new_*_block()`. Pick the variant from what the block does:

| Block does... | Variant | Server signature |
|---|---|---|
| Loads from API / file / database (no upstream) | `new_data_block()` | `function(id)` |
| Reshapes one upstream input | `new_transform_block()` | `function(id, data)` |
| Joins two upstream inputs | `new_join_block()` | `function(id, x, y)` |
| Takes N upstream inputs | `new_variadic_block()` | `function(id, ...args)` |
| Renders a plot | `new_plot_block()` | `function(id, data)` |

The server returns `list(expr = reactive(...), state = list(...))`.

- **State names must match constructor argument names** exactly and in count. Serialization breaks silently otherwise.
- Use `blockr.core::bbquote()` (not base `bquote()`) for expression building, and set `expr_type = "bquoted"` on the parent constructor. Splice the data input via `.(data)`. Never use `paste()` or string interpolation.
- Don't expose data inputs (`data`, `x`, `y`, `...args`) as constructor arguments — those are wired by the framework via the server signature.
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

- Launch the package's preview app (e.g. `dev/preview-all-blocks.R`, or a minimal `shiny::runApp(blockr.core::serve(new_<name>_block(), list(data = ...)), port = 3838)`). Note: `serve()` itself doesn't accept `port` / `launch.browser` — it returns a `shinyApp` which `runApp()` then runs.
- Screenshot the block in its empty state and after a typical interaction.
- Check: UI renders without console errors, the block produces output downstream, the empty → configured transition is clean.

Tests passing isn't the same as the block working in a real Shiny session. Don't skip this step.

### Board demo (multi-block)

Serving the block alone with a static `data = ...` proves it renders, not that it behaves in a pipeline. Wire a small board so data actually flows through the block. Prefer a **dock board** with the DAG extension — that is how blockr is actually used: dockable panels, the block picker, and a live DAG view, so you can add and rewire blocks from the UI instead of only in code.

```r
library(blockr.core)
library(blockr.dock)   # dockable layout + block picker
library(blockr.dag)    # DAG view extension
pkgload::load_all(".")

serve(
  new_dock_board(
    blocks = c(
      data = new_dataset_block("iris"),   # upstream source
      mine = new_<name>_block(),          # the block under test
      out  = new_scatter_block()          # downstream consumer
    ),
    links = c(
      new_link(from = "data", to = "mine", input = "data"),
      new_link(from = "mine", to = "out",  input = "data")
    ),
    extensions = new_dag_extension()
  )
)
```

Adapt the wiring to the block variant:

- **data block** — drop the upstream `data` block; make `mine` the source and link `mine -> out` (e.g. `new_head_block()`).
- **transform / plot block** — source -> `mine` -> consumer, as above.
- **join / variadic block** — two or more upstream `new_dataset_block()`s linked into `x`/`y` (or `...args`).

Then edit the source block and confirm `mine` and `out` update. Reacting to upstream changes (not just its own inputs) is what catches broken link inputs and state-name mismatches that `testServer()` and single-block `serve()` both miss.

If `blockr.dock` / `blockr.dag` aren't available, fall back to a plain `blockr.core::new_board(blocks = ..., links = ...)` with the same wiring — no dock UI or DAG view, but the data still flows end to end.

## Don'ts

- **Don't ask the user to pick the pattern when the answer is obvious.** New package, simple block, first-time author → R-driven. Default and tell them you defaulted; they can override.
- **Don't skip registration.** Without `register_blocks()` every constructor call warns, and the block won't show up in board/AI/MCP discovery.
- **Don't add `shinytest2` to an R-driven block.** `testServer()` covers it in milliseconds. The exception is visual regression on CSS, rarely worth the maintenance.
- **Don't duplicate expression-building logic in the constructor.** Helpers go in `R/expr-builders.R` so they're unit-testable in isolation.
- **Don't skip the Playwright verification.** Tests passing isn't the same as the block actually working.

## When you're done

Tell the user the verification command — `pkgload::load_all("<pkg>")` then `shiny::runApp(blockr.core::serve(<your_constructor>()))` — so they can drop the block into a real session. For anything that consumes or produces upstream data, point them at the board demo above so they see it working in a real pipeline, not just in isolation.
