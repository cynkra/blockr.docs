# R-driven blocks

The simpler of the two block patterns: a pure-R Shiny module that returns an expression and a state list. No custom JavaScript, no input bindings.

Right for simple blocks, internal tools, prototypes, and the on-ramp into the framework. `blockr.core`'s built-ins are built this way. For polished, user-facing block libraries (think `blockr.dplyr`) the JS-driven pattern is usually the better fit — see [js-driven-blocks.md](./js-driven-blocks.md) and [README.md](./README.md) for the trade-off.

For the full tutorial with worked examples, see `vignette("create-block", package = "blockr.core")`. This guide is the condensed "how we build these" reference.

## Anatomy

A block is a Shiny module that returns two things:

- **`expr`** — a reactive yielding a `bbquote()`d expression. Defines the block's computation as code that can be evaluated outside any reactive context. This is what makes the pipeline reproducible and exportable.
- **`state`** — a list of reactives, one per constructor parameter. Drives serialization (save/restore) and external control of the block.

A block is built from three pieces in a single constructor:

1. **UI function** — `function(id) { ... }` returning a `tagList`. Use `NS(id, "name")` for input IDs.
2. **Server function** — `function(id, data, ...)` wrapping a `moduleServer()`. Returns `list(expr = ..., state = ...)`.
3. **Constructor** — `new_my_block(...)` exposing every UI-controllable parameter as an argument and calling `new_block()` (or `new_data_block()`, `new_transform_block()`, `new_plot_block()` for the typed variants).

Server signature varies by block type:

| Block type | Server signature |
|---|---|
| Data | `function(id)` |
| Transform | `function(id, data)` |
| Join | `function(id, x, y)` |
| Variadic | `function(id, ...args)` |

## Reference implementation

`blockr.core/R/transform-head.R:12` (`new_head_block()`) is the canonical minimal example.

```r
new_head_block <- function(n = 6L, ...) {
  new_transform_block(
    function(id, data) {
      moduleServer(id, function(input, output, session) {
        n_rows <- reactiveVal(n)

        observeEvent(input$n, n_rows(input$n))

        observeEvent(nrow(data()), {
          updateNumericInput(
            inputId = "n", value = n_rows(),
            min = 1L, max = nrow(data())
          )
        })

        list(
          expr = reactive(
            bbquote(utils::head(.(data), n = .(n)), list(n = n_rows()))
          ),
          state = list(n = n_rows)
        )
      })
    },
    function(id) {
      tagList(
        numericInput(
          inputId = NS(id, "n"),
          label = "Number of rows",
          value = n,
          min = 1L
        )
      )
    },
    class = "head_block",
    expr_type = "bquoted",
    ...
  )
}
```

## Rules that bite

- **State names must match constructor argument names**, exactly and in count. Serialization breaks silently otherwise.
- **Don't expose data inputs as constructor arguments**. `data` / `x` / `y` / `...args` are wired by the framework via the server signature.
- **Forward `...`** to the parent constructor so framework-level options (`allow_empty_state`, `dat_val`, `class`, etc.) can be set by callers.
- **Use `blockr.core::bbquote()`** (not base `bquote()`) to splice reactive values *and* the data input into the expression — never `paste()` or string interpolation. Pair it with `expr_type = "bquoted"` on the parent constructor. The expression must be valid R code, not a string.
- **The expression must evaluate outside a reactive context**. If your `expr` only works because a reactive is in scope, the export pipeline will fail.

## Testing

Two tiers cover everything. `shinytest2` is not needed.

### Tier 1 — Unit tests for pure R helpers

If your block uses helper functions to build expressions or parse user input, test them as plain R functions, no Shiny in scope.

```r
test_that("parse_mutate handles single expression", {
  result <- parse_mutate("new_col = old_col * 2")
  expect_equal(result[[1]]$name, "new_col")
  expect_equal(result[[1]]$expr, "old_col * 2")
})
```

### Tier 2 — `testServer()` for everything Shiny

`testServer()` can simulate every UI interaction your block supports via `session$setInputs()`. It reaches reactives, state, and the returned expression directly — no browser, no chromote, ~0.2s per test.

There are two complementary patterns:

**Pattern A — `expr_server`: test the expression and reactive state.**

Use this to verify that user inputs produce the right code, that state reacts correctly, and that `state` exposes the expected reactives.

```r
test_that("head block builds correct expression", {
  blk <- new_head_block(n = 6L)

  testServer(
    blk$expr_server,
    args = list(data = reactive(mtcars)),
    {
      session$setInputs(n = 10L)
      session$flushReact()

      expr_result <- session$returned$expr()
      expect_equal(eval(expr_result, list(data = mtcars)), head(mtcars, 10))
      expect_equal(session$returned$state$n(), 10L)
    }
  )
})
```

**Pattern B — `block_server`: test the materialized result.**

Use this when you want to assert on the actual data the framework would render. It runs the same path the framework runs in production, so it catches integration issues `expr_server` can miss (e.g. `block_output` S3 dispatch).

```r
test_that("head block returns a data frame", {
  blk <- new_head_block(n = 5L)

  testServer(
    blockr.core:::get_s3_method("block_server", blk),
    args = list(x = blk, data = list(data = function() mtcars)),
    {
      session$flushReact()
      expect_equal(nrow(session$returned$result()), 5L)
    }
  )
})
```

Pick A by default; switch to B when you specifically want to test the materialized output or suspect a framework-integration bug.

### What about `shinytest2`?

Don't reach for it for R-driven blocks. Everything `shinytest2` would do — set inputs, click buttons, read outputs — `testServer()` already does in milliseconds. `blockr.dplyr` deleted ~1000 lines of `shinytest2` tests in late 2025 once it was confirmed `testServer()` covered every case. The only legitimate reason to add `shinytest2` to an R-driven block is visual regression on CSS, which is rarely worth the maintenance cost.

The exception is JS-driven blocks, where `session$setInputs()` can't drive custom input bindings — see [js-driven-blocks.md](./js-driven-blocks.md).

## Registering the block

Without a registration call, every constructor invocation emits `No block metadata available for block <name>_block.`. Register on package load:

```r
# R/zzz.R
.onLoad <- function(libname, pkgname) {
  blockr.core::register_blocks(
    ctor = "new_cat_facts_block",
    name = "cat facts block",
    description = "Fetch cat facts from catfact.ninja",
    category = "input",
    package = pkgname
  )
}
```

Notes:
- `register_blocks()` (plural) is the vectorised version — one call can register many blocks; `ctor`, `name`, `description`, `category` accept parallel character vectors.
- `category` must be one of `blockr.core::suggested_categories()`: `input`, `transform`, `structured`, `plot`, `table`, `model`, `output`, `utility`, `uncategorized`. Anything else warns. Data-fetching blocks are `input`, not `data`.
- Pass `package = pkgname` from `.onLoad` so blocks are namespaced to your package.

Full reference: `vignette("blocks-registry", package = "blockr.core")`.
