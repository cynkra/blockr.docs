# JS-driven blocks

The advanced approach: a block whose UI is a custom JavaScript class, registered as a Shiny input binding, that sends a JSON state to R. R turns that state into a `dplyr` (or other) expression via `bbquote()`.

Use this only when [r-driven-blocks.md](./r-driven-blocks.md) doesn't fit — see the decision guidance in [README.md](./README.md). The canonical reference implementation is `blockr.dplyr`.

## Anatomy

Each JS-driven block has four files:

```
inst/js/<name>-block.js     # JS class + Shiny input binding
inst/css/<name>-block.css   # Block-specific CSS
R/<name>_block.R            # R constructor + server + UI + htmlDependency
R/expr-builders.R           # Expression builder (shared file, one function per block)
```

Use `blockr.dplyr/R/filter_block.R` and `blockr.dplyr/inst/js/filter-block.js` as the reference.

## R constructor pattern

```r
new_my_block <- function(state = list(param1 = default1), ...) {
  new_transform_block(
    function(id, data) {
      moduleServer(id, function(input, output, session) {
        ns <- session$ns
        r_state <- reactiveVal(state)

        # Bidirectional sync guard: prevents R→JS→R loops
        self_write <- new.env(parent = emptyenv())
        self_write$active <- FALSE

        # Push column info to JS when data changes
        observeEvent(data(), {
          session$sendCustomMessage("my-columns",
            list(id = ns("my_input"), columns = colnames(data())))
        })

        # JS → R
        observeEvent(input$my_input, {
          self_write$active <- TRUE
          r_state(input$my_input)
        })

        # R → JS (external control: programmatic state changes)
        observeEvent(r_state(), {
          if (self_write$active) {
            self_write$active <- FALSE
          } else {
            session$sendCustomMessage("block-update",
              list(id = ns("my_input"), state = r_state()))
          }
        }, ignoreInit = TRUE)

        list(
          expr = reactive({
            s <- r_state()
            make_my_expr(s$param1 %||% default1)
          }),
          state = list(state = r_state)
        )
      })
    },
    function(id) {
      tagList(
        blockr_core_js_dep(), blockr_blocks_css_dep(),
        blockr_select_dep(),  # if using Blockr.Select
        blockr_input_dep(),   # if using Blockr.Input
        my_block_dep(),
        div(class = "block-container",
          div(id = NS(id, "my_input"), class = "my-block-container"))
      )
    },
    class = "my_block",
    expr_type = "bquoted",
    external_ctrl = TRUE,
    allow_empty_state = "state",
    ...
  )
}
```

Key points:

- Single `state` reactiveVal whose name matches the constructor parameter.
- `expr_type = "bquoted"` — expressions use the `.(data)` placeholder, resolved by `blockr.core` at evaluation time.
- `allow_empty_state = "state"` — the framework passes data through unchanged when state is empty.
- No `req()` — return an identity expression instead.

## JS class pattern

```javascript
(() => {
  'use strict';

  class MyBlock {
    constructor(el) {
      this.el = el;
      this._callback = null;
      this._submitted = false;
      this._buildDOM();
    }

    _autoSubmit() {
      clearTimeout(this._debounceTimer);
      this._debounceTimer = setTimeout(() => this._submit(), 300);
    }

    _buildDOM() { /* build UI, attach event listeners */ }
    _compose() { return { /* state JSON */ }; }
    _submit() { this._submitted = true; this._callback?.(true); }
    getValue() { return this._submitted ? this._compose() : null; }
    setState(state) { /* rebuild UI from state, do NOT fire _callback */ }
    updateColumns(cols) { /* refresh dropdowns with new column names */ }
  }

  const binding = new Shiny.InputBinding();
  Object.assign(binding, {
    find: (scope) => $(scope).find('.my-block-container'),
    getId: (el) => el.id || null,
    getValue: (el) => el._block?.getValue() ?? null,
    setValue: (el, value) => el._block?.setState(value),
    subscribe: (el, callback) => { if (el._block) el._block._callback = () => callback(true); },
    unsubscribe: (el) => { if (el._block) el._block._callback = null; },
    initialize: (el) => {
      el._block = new MyBlock(el);
      if (el._pendingColumns) { el._block.updateColumns(el._pendingColumns); delete el._pendingColumns; }
      if (el._pendingState)   { el._block.setState(el._pendingState);     delete el._pendingState; }
    }
  });
  Shiny.inputBindings.register(binding, 'blockr.my');

  // Global message handlers (dispatch by msg.id)
  Shiny.addCustomMessageHandler('my-columns', (msg) => {
    const el = document.getElementById(msg.id);
    if (el?._block) el._block.updateColumns(msg.columns);
    else if (el)    el._pendingColumns = msg.columns;
  });
})();
```

## Shared JS components

- `Blockr.Select.single(container, config)` / `.multi(container, config)` — dropdown / tags
- `Blockr.Input.create(container, config)` — code input with autocomplete
- `Blockr.icons` — SVG strings: `.x`, `.plus`, `.confirm`, `.code`, `.chevron`, `.remove`, `.gear`

See [../design-system/components/](../design-system/components/) for the full component specs.

## Shared CSS classes

| Class | Purpose |
|---|---|
| `.blockr-row` | Gray bordered row container (42px, flex) |
| `.blockr-row-content` | Flex-filling content area inside row |
| `.blockr-row-remove` | Remove button (hidden until row hover) |
| `.blockr-pill` | Click-through toggle button |
| `.blockr-add-row` | Footer bar with "Add" links |
| `.blockr-add-link` | "+ Add something" text link |
| `.blockr-add-link-expr` | Code icon button for adding expression rows |
| `.blockr-expr-confirm` | Enter / checkmark confirm button |
| `.blockr-num-input` | Number input inside rows |
| `.blockr-text-input` | Standalone text input (42px, matches dock) |
| `.blockr-label` | Input label (12px, 500 weight) |
| `.blockr-select--bordered` | Add to standalone Blockr.Select for 42px bordered style |
| `.blockr-gear-header` | Top-right gear icon container |
| `.blockr-gear-btn` | Gear settings button |
| `.blockr-popover` | Settings popover panel |
| `.blockr-popover-row/label/input/checkbox` | Popover content elements |

## Row divider principle

The first "key" input in a row gets `border-right`. Text separators (`=`, `→`, `of`) go between value-side elements.

## Expression building with `bbquote`

```r
make_my_expr <- function(param) {
  if (is_empty(param)) return(bbquote(.(data)))   # pass-through
  expr <- quote(dplyr::my_fn(.(data)))
  expr[["arg"]] <- as.name(param)
  bbquote(.(expr), list(expr = expr))
}
```

The `.(data)` placeholder is resolved by `blockr.core`'s `eval_impl` at evaluation time.

For namespaced function calls, use `str2lang()`:

```r
expr <- as.call(list(str2lang("dplyr::slice_head"), quote(.(data))))
```

## Registry

```r
register_blocks("new_my_block",
  arguments = list(
    structure(
      c(state = "Description of the state object"),
      examples = list(state = list(param1 = "example_value")),
      prompt = "Optional LLM guidance"
    )
  )
)
```

## Testing

JS-driven blocks need a different testing strategy than R-driven blocks. The expression-building layer is still pure R and testable with `testServer()`, but the JS UI itself needs `shinytest2` because `session$setInputs()` cannot drive custom input bindings.

Three tiers, in order of speed and frequency:

### Tier 1 — Unit tests for expression builders

Expression builders in `R/expr-builders.R` are pure functions. Test them directly. Use this helper to evaluate a `bquoted` expression against a data frame:

```r
eval_bquoted <- function(expr, df) {
  resolved <- do.call(bquote, list(expr, list(data = as.name("data"))))
  eval(resolved, envir = list(data = df))
}

test_that("make_filter_expr handles values condition", {
  conds <- list(list(type = "values", column = "cyl",
                     values = list("4", "6"), mode = "include"))
  expr <- make_filter_expr(conds, "&")
  result <- eval_bquoted(expr, mtcars)
  expect_true(all(result$cyl %in% c(4, 6)))
})
```

This covers the bulk of correctness — most bugs in JS-driven blocks live in the expression builder, not the UI.

### Tier 2 — `testServer()` for the R server logic

Use `testServer()` to verify that constructor state flows through to the right expression. You can inject state via the constructor (skipping the JS round-trip) or by setting `r_state` directly.

```r
test_that("filter block produces correct expression", {
  blk <- new_filter_block(
    state = list(
      conditions = list(list(type = "values", column = "cyl",
                             values = list("4"), mode = "include")),
      operator = "&"
    )
  )

  testServer(blk$expr_server, args = list(data = reactive(mtcars)), {
    session$flushReact()
    expr_result <- session$returned$expr()
    evaluated <- eval_bquoted(expr_result, mtcars)
    expect_true(all(evaluated$cyl == 4))
  })
})
```

`testServer()` cannot fire the JS input binding directly. To exercise state-change paths, push to `r_state` from inside the test or pass a new `state` to the constructor.

### Tier 3 — `shinytest2` for the JS round-trip

This is the part R-driven blocks don't need and JS-driven blocks can't do without. `shinytest2` is the only way to verify that the JS UI → input binding → R expression → evaluated result chain actually works end-to-end.

The trick is that custom JS input bindings aren't reachable via `set_inputs()`. Drive the block by calling its JS API directly through `run_js()`:

```r
set_block_state <- function(app, block_id, input_suffix, state) {
  input_id <- paste0("board-block_", block_id, "-expr-", input_suffix)
  state_json <- jsonlite::toJSON(state, auto_unbox = TRUE, null = "null")
  app$run_js(sprintf(
    "var el = document.getElementById('%s');
     el._block.setState(%s);
     el._block._submit();",
    input_id, state_json
  ))
  app$wait_for_idle()
}
```

Read results back with `app$get_values()$export$result$<block_id>` (the test app must call `exportTestValues()`). The reference test app is `blockr.dplyr/tests/testthat/apps/dplyr-e2e/app.R`, with one board pre-wired to all blocks against fixed datasets.

Run with `Sys.setenv(NOT_CRAN = "true"); testthat::test_file("tests/testthat/test-shinytest2.R")`. These are slow (~2-5s per test) — keep one happy-path test per block, push everything else into Tiers 1 and 2.

### What goes in which tier

| What you're testing | Tier |
|---|---|
| Expression construction from a fixed state | 1 |
| Pass-through / empty-state behavior | 1 |
| Constructor state → expression | 2 |
| Reactive state changes inside the server | 2 |
| Round-trip from JS UI to evaluated result | 3 |
| The JS class itself in isolation | (none — covered by Tier 3) |
