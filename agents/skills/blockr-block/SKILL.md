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
- `app.R` — a runnable demo board at the package root (see "Board demo"). Add `^app\.R$` to `.Rbuildignore` so R CMD check stays clean.
- `DESCRIPTION` — add the block's runtime deps to `Imports`.
- `NAMESPACE` — `importFrom(blockr.core, bbquote)` plus any roxygen-driven exports.

For a brand-new package, also create: `DESCRIPTION` (with `Imports: blockr.core, shiny` plus block deps), `NAMESPACE` (`export(new_<name>_block)`, `importFrom(blockr.core, bbquote)`), `app.R` (the demo board), and `.Rbuildignore` (including `^app\.R$`). The `category` for `register_blocks()` must be one of `blockr.core::suggested_categories()` — `input`, `transform`, `structured`, `plot`, `table`, `model`, `output`, `utility`, `uncategorized`. Data-fetching blocks are `input`, not `data`.

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

Once tests pass, verify the block runs end-to-end in a browser. **The verification artifact is the `app.R` board demo below** — a dock board with the DAG extension where data actually flows through the block. This is a required deliverable, not optional.

Do **not** verify by serving the block standalone (`serve(new_<name>_block())` or `serve(new_<name>_block(), list(data = ...))`). That only proves the UI renders in isolation; it exercises no links, no upstream data, and no state round-trip, so it hides exactly the bugs verification is meant to catch (broken link `input`s, state-name mismatches). Use it at most as a throwaway smoke test, never as the thing you hand back.

Once `app.R` is written and launched, hand off to the **`blockr-playwright`** skill to drive the running app:

- Screenshot the block in its empty state and after a typical interaction.
- Check: UI renders without console errors, the block produces output downstream, the empty → configured transition is clean.

> **`blockr-playwright` is not in this repo.** It ships from `cynkra/blockr.dev` at [`.claude/skills/blockr-playwright`](https://github.com/cynkra/blockr.dev/tree/align-workflow/.claude/skills/blockr-playwright) (end-to-end debugging of blockr Shiny apps via the Playwright MCP). Install it per `blockr.docs/agents/skills/README.md`. If it isn't installed, drive the running `app.R` manually with the `shiny-chromote-inspect` skill or `{chromote}` instead — but still verify against the board demo, not a standalone serve.

Tests passing isn't the same as the block working in a real Shiny session. Don't skip this step.

### Board demo (multi-block) — the verification app

Write a runnable **`app.R` at the package root** so data actually flows through the block. Use a **dock board** with the DAG extension — that is how blockr is actually used: dockable panels, the block picker, and a live DAG view, so you can add and rewire blocks from the UI instead of only in code. This `app.R` is the app you launch and hand to `blockr-playwright`; there is no separate standalone step.

**Launch it so the working directory is the package root.** Run `shiny::runApp("<path-to-package>")` (or RStudio's **Run App** on `app.R`). `runApp()` switches the working directory to the app's folder for the session, so `pkgload::load_all(".")` inside `app.R` loads *this* package no matter where you launched from. Don't `pkgload::load_all(".")` from a parent directory or a console whose working directory isn't the package root — it'll load the wrong directory and fail. The launch path is relative to the package, not your shell.

**Don't install or reinstall anything for the demo.** Assume `blockr.core`, `blockr.dock`, `blockr.dag` and the package's deps are already installed; `library()` loads them and `load_all()` loads the package under development. Never run `install.packages()`, `pak::pak()`, `remotes::install_*()`, or `devtools::install()` here — that can clobber working versions with wrong ones. If a dependency is genuinely missing, tell the user and stop; don't silently install.

**The board layout depends on the block's variant.** A data block is an entry point — it is the source, so don't add another data block. Transform/plot blocks need an upstream source. Pick the matching shape:

`app.R` header (all variants):

```r
# Launch with shiny::runApp("<this package dir>") or RStudio Run App.
# runApp sets the working directory to this folder, so load_all(".") loads
# this package. Deps are assumed already installed — do not (re)install them.
library(blockr.core)
library(blockr.dock)   # dockable layout + block picker
library(blockr.dag)    # DAG view extension
pkgload::load_all(".")
```

**Data block (entry point — API / file / DB).** The block under test is the source; link it straight to a consumer so you can see what it returns. No upstream data block.

```r
serve(
  new_dock_board(
    blocks = c(
      mine = new_<name>_block(),     # the block under test IS the source
      out  = new_head_block()        # consumer, to view the data
    ),
    links = c(new_link(from = "mine", to = "out", input = "data")),
    extensions = new_dag_extension()
  )
)
```

**Transform / plot block.** Needs an upstream source feeding it:

```r
serve(
  new_dock_board(
    blocks = c(
      data = new_dataset_block("iris"),   # upstream source
      mine = new_<name>_block(),          # the block under test
      out  = new_scatter_block()          # downstream consumer (transform only)
    ),
    links = c(
      new_link(from = "data", to = "mine", input = "data"),
      new_link(from = "mine", to = "out",  input = "data")
    ),
    extensions = new_dag_extension()
  )
)
```

**Join / variadic block.** Two or more upstream `new_dataset_block()`s linked into `x`/`y` (or `...args`).

Then exercise the flow: for a data block, change its own inputs and confirm `out` updates; for the rest, edit the upstream source and confirm `mine` and its consumer update. Reacting to changes (not just rendering once) is what catches broken link inputs and state-name mismatches that `testServer()` and single-block `serve()` both miss.

If `blockr.dock` / `blockr.dag` aren't available, fall back to a plain `blockr.core::new_board(blocks = ..., links = ...)` with the same per-variant wiring — no dock UI or DAG view, but the data still flows end to end.

**Keep R CMD check clean.** A top-level `app.R` is a non-standard package file, so add `^app\.R$` to `.Rbuildignore`. The demo then stays out of the built tarball (no "non-standard top-level file" NOTE) while remaining runnable from the source tree. If you'd rather ship the demo with the package, put it at `inst/examples/app.R` instead and run it via `shiny::runApp(system.file("examples", package = "<pkg>"))` — `inst/` content doesn't need ignoring.

## Don'ts

- **Don't ask the user to pick the pattern when the answer is obvious.** New package, simple block, first-time author → R-driven. Default and tell them you defaulted; they can override.
- **Don't skip registration.** Without `register_blocks()` every constructor call warns, and the block won't show up in board/AI/MCP discovery.
- **Don't add `shinytest2` to an R-driven block.** `testServer()` covers it in milliseconds. The exception is visual regression on CSS, rarely worth the maintenance.
- **Don't duplicate expression-building logic in the constructor.** Helpers go in `R/expr-builders.R` so they're unit-testable in isolation.
- **Don't skip the Playwright verification.** Tests passing isn't the same as the block actually working.
- **Don't install or reinstall packages for the demo.** Deps are assumed preinstalled — `install.packages()` / `pak` / `remotes` can replace working versions with wrong ones. `library()` the deps, `load_all()` the package, and if something's missing, say so instead of installing.
- **Don't `load_all(".")` from a parent directory.** Launch the demo with `shiny::runApp("<pkg>")` so the working directory is the package root; the path is relative to the package, not your shell.

## When you're done

Tell the user how to run the demo — `shiny::runApp("<path-to-package>")` (or RStudio's **Run App** on `app.R`) — so they see the block working in a real pipeline, not just in isolation. Pass the package path so `runApp()` sets the working directory there; don't assume the shell is already inside the package. The `app.R` board demo (dock board + DAG extension) is the deliverable you hand back — not a standalone `serve(<your_constructor>())`. Confirm `^app\.R$` is in `.Rbuildignore` so `R CMD check` stays clean, and that you didn't install or reinstall any packages to get here.
