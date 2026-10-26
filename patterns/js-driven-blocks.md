# JS-driven blocks

The advanced approach: a block whose UI is a custom JavaScript class, registered as a Shiny input binding, that sends a JSON state to R. R turns that state into a `dplyr` (or other) expression via `bbquote()`.

Use this only when [r-driven-blocks.md](./r-driven-blocks.md) doesn't fit — see the decision guidance in [README.md](./README.md). The canonical reference implementation is `blockr.dplyr` (≥ 0.2.0, the factory-based rewrite).

Most of the pattern lives in two factories, so a new block is ~30 lines of R and a JS class:

- R: `new_js_transform_block()` in `blockr.dplyr/R/js-block.R` generates the server (state reactive, column push, bidirectional sync) and UI. Blocks with non-standard data arity compose the same pieces via `js_block_sync()` / `js_block_ui()` / `js_block_dep()`.
- JS: `Blockr.registerBlock()` in `blockr.dplyr/inst/js/blockr-core.js` generates the input binding and message handlers.

> **Where the factories live.** The shared controls (`Blockr.Select`, `Blockr.Input`) and their CSS ship in `blockr.ui` as `blockr.ui::controls_dep()`. The factories are still internal to `blockr.dplyr`: importing it gets you the exported dependency functions (`blockr_core_js_dep()`, which loads `blockr-core.js` on top of `controls_dep()`) and helpers, but NOT `new_js_transform_block()` / `js_block_sync()` / `js_block_ui()`. `blockr_select_dep()` and `blockr_input_dep()` still exist and now return the same thing as `blockr_core_js_dep()`. A new package should start from **`blockr.docs/scaffolds/jsblock`**: it vendors an adapted `R/js-block.R` (including a `new_js_plot_block()` variant) while loading all shared JS/CSS through the exported dependency functions, so nothing else gets copied. Keep the vendored file unedited so that moving the factories out of blockr.dplyr later is a deletion.

## Anatomy

```
inst/js/<name>-block.js     # JS class + Blockr.registerBlock() call
inst/css/<name>-block.css   # Block-specific CSS
R/<name>_block.R            # ~30-line constructor via the factory
R/expr-builders.R           # Expression builder (shared file, one function per block)
```

One naming convention drives everything. From `name = "pivot-longer"`:

| derived | value |
|---|---|
| input id | `pivot_longer_input` |
| container class | `pivot-longer-block-container` |
| Shiny binding | `blockr.pivot-longer` |
| messages | `pivot-longer-columns`, `pivot-longer-block-update` |
| assets | `inst/js/pivot-longer-block.js`, `inst/css/pivot-longer-block.css` |

References: `blockr.dplyr/R/mutate_block.R` (minimal), `R/filter_block.R` (with extras), `R/join_block.R` (bespoke arity).

## R constructor

Flat formals — one constructor argument per state field, assembled into the `state` list. `names(state)` must equal the formals, exactly and in count: `blockr.core` restores a block by calling the constructor with the serialized fields.

```r
new_my_block <- function(param1 = default1, param2 = NULL, ...) {
  new_js_transform_block(
    class = "my_block",
    name = "my",
    state = list(param1 = param1, param2 = param2),
    expr_fn = function(s) make_my_expr(s$param1 %||% default1, s$param2),
    ...
  )
}
```

Factory arguments beyond the required four:

- `columns_meta` — function of the data frame producing what the `<name>-columns` message carries. Default `build_column_picker_meta()` (name + label). The filter block passes `build_column_summary()` (adds type + hasNA) and loads unique values lazily (see Performance).
- `setup` — `function(input, session, ns, data, input_name)` for block-specific observers or one-shot init messages (filter's on-demand value requests, summarize's function list).
- `normalize_state` — applied to the state before the R→JS update message; see the auto-unbox rule under Protocol contract.
- `shared_deps` — which shared components to load: `"select"` (default), `c("select", "input")`, or `character(0)`.

Rules the factory enforces so you don't have to remember them:

- R holds the state as one `reactiveVal` per field (what external control and serialization need); the JS side sees it recombined as a single blob. The factory does both directions — don't reimplement the sync.
- `expr_type = "bquoted"`, `external_ctrl = TRUE`, `allow_empty_state = TRUE` (every field may legitimately be empty on a fresh block).
- No `req()` — an empty state returns the `.(data)` pass-through from the expression builder (transform), or a friendly placeholder plot (plot blocks — see the scaffold). An unconfigured block must be a no-op, never an error and never a data-destroying default (an empty summarize must not collapse to one row).

Blocks whose server signature isn't `function(id, data)` (join's `function(id, x, y)`, bind's `function(id, ...args)`) keep a bespoke `new_transform_block()` call but compose `js_block_sync(input, session, name, input_name, r_state)` for the state sync and `js_block_ui(name, shared_deps)` for the UI.

## JS class

The class owns the DOM and the in-progress state; it implements a small contract:

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

    _autoSubmit() {                      // debounce user edits
      clearTimeout(this._debounceTimer);
      this._debounceTimer = setTimeout(() => this._submit(), 300);
    }

    _buildDOM() { /* build UI, attach listeners */ }
    _compose()  { return { /* state JSON */ }; }
    _submit()   { this._submitted = true; this._callback?.(true); }
    getValue()  { return this._submitted ? this._compose() : null; }
    setState(state)     { /* rebuild UI from state; never fire _callback */ }
    updateColumns(cols) { /* refresh dropdowns with new column metadata */ }
  }

  Blockr.registerBlock({
    name: 'my',
    Block: MyBlock,
    messages: {
      'my-columns':      (block, msg) => block.updateColumns(msg.columns),
      'my-block-update': (block, msg) => block.setState(msg.state)
    }
  });
})();
```

`Blockr.registerBlock()` generates the input binding (find/getValue/setValue/subscribe/initialize/receiveMessage) and registers the message handlers with id-dispatch and a pending queue — do not hand-write either.

## Lifecycle

The part that breeds bugs when reinvented per block. The factories implement it once:

```
constructor(state)                      R server init
        │                                    │
        │   r_state <- reactiveVal(state)    │
        │   sync observer fires ON INIT ─────┼──► "<name>-block-update" {id, state}
        │                                    │
        │            element not in DOM yet ─┼──► Blockr queues message by id
        │                                    │
   Shiny binds the container element         │
        │  initialize: new Block(el),        │
        │  replay queued messages in         │
        │  arrival order                     │
        ▼                                    ▼
   user edits → debounce → _submit() → input$<name>_input → r_state → expr()
```

- **Initial state reaches JS through the sync observer's init fire.** `js_block_sync()` deliberately does *not* use `ignoreInit = TRUE`; the first `block-update` message is how a constructor or board-restore state arrives in the UI. (Earlier versions of this guide showed `ignoreInit = TRUE` — that variant never delivers initial state.)
- **Messages can arrive before the element exists.** `Blockr.registerBlock()` queues them per element id and replays in arrival order on `initialize`; queue entries expire after 30s. No polling, no `_pendingState` expandos.
- **The `self_write` guard breaks the JS→R→JS echo**: a state change originating from the UI must not be sent back down as an update message.

### Teardown rules

Blocks get removed from boards; anything you attach outside the block's own subtree outlives it and leaks.

- **Never call `document.addEventListener` / `window.addEventListener` from a block.** For close-on-outside-click popovers use `Blockr.onDocClick(this.el, (e) => {...})` — one shared document listener whose entries self-remove when the anchor element leaves the DOM.
- Listeners on the block's own elements are fine — they die with the subtree.
- Components handed to you (`Blockr.Select`, `Blockr.Input`) have `destroy()`; call it whenever you remove a row that owns one.
- A pending debounce timer firing after removal is harmless (the binding nulls `_callback` on unsubscribe).

## Protocol contract

The state JSON is the interface between R and JS. The machine-checkable version lives in `blockr.dplyr/inst/js/types.d.ts` (column metadata shapes, filter conditions, the block-class contract, `Blockr.Select`/`Blockr.Input` config and handle types). Rules that aren't expressible in types:

- **JS guarantees:** `setState()` never fires the callback; `getValue()` returns `null` until the first user submit; `_compose()` output round-trips through `setState()` losslessly.
- **Type-tag values.** Anything the user picks in JS arrives in R as a string. Don't guess the R type by coercibility — `"007"` from a character column must stay `"007"`, not become `7`. Send the column's type alongside (the filter block's `colType` on each condition) and convert explicitly in the builder.
- **The auto-unbox rule.** `sendCustomMessage` serializes with `auto_unbox = TRUE`: a length-1 character vector becomes a JSON scalar, and JS code iterating it gets characters. Any state field that is conceptually an array must be wrapped with `as.list()` before sending — that's what the factory's `normalize_state` hook is for (see `normalize_filter_state_for_js()`).
- **R guarantees:** messages carry `{id: ns(input_name), ...}` and are dispatched by element id; the state sent in `block-update` equals what the constructor would receive on restore.

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

**Build language objects, never paste-and-parse.** `as.name()` handles any column name (spaces, quotes, backticks); string assembly breaks on the first odd name and tends to fail silently. For namespaced functions use `str2lang("dplyr::slice_head")` — that's a fixed developer string, not user input. Validate anything enum-like from JS (operators, function names) against a whitelist before splicing it into a call.

## Shared JS components

- `Blockr.Select.single(container, config)` / `.multi(container, config)` — dropdown / tags
- `Blockr.Input.create(container, config)` — code input with autocomplete
- `Blockr.icons` — SVG strings; `Blockr.onDocClick` — leak-safe document clicks

Config and handle types — including the `setOptions` (reconciles selection) vs `updateOptions` (preserves selection, for lazy loading) distinction and the `maxRendered` cap — are documented in `types.d.ts`. How the components look and behave: [Controls](https://bristolmyerssquibb.github.io/blockr.ui/articles/design-system.html#controls) in blockr.ui's design system.

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
| `.blockr-settings` | Settings band — in flow under the gear header, not a floating panel |
| `.blockr-settings__field` / `--full` | One field in the band's wrapping grid |
| `.blockr-checkbox` | Plain on/off data option (not a self-labelling pill) |

Row divider principle: the first "key" input in a row gets `border-right`; text separators (`=`, `→`, `of`) go between value-side elements.

## Performance checklist

Lessons blockr.dplyr paid for. The measurement behind them: eager metadata made
a 50K-row board 5.6× slower to start, and the cost was JSON serialization, not R
compute.

- **Send light metadata eagerly, values lazily.** On data change push names/types/labels only; fetch a column's unique values on dropdown-open via a request message (`filter-column-values` + `setLoading`/`onOpen` on `Blockr.Select`).
- **Cap what you render** (`maxRendered`, default 200) — and cap what you *send*: above a distinct-count threshold (`blockr.dplyr.max_filter_values`, default 1000) switch to server-side search. The server returns a capped page plus `total` and a sticky `truncated` flag; `Blockr.Select` enters search mode via `setSearchInfo()` and re-queries through its `onSearch` hook as the user types. Old servers that never send `truncated` keep the pure client-side behavior — the modes are wire-compatible. See `build_column_values(limit =, query =)` and the filter block's `_search_values` input for the reference wiring.
- **Debounce submits** (300ms) so each keystroke doesn't re-evaluate the board.
- **Bump the package `Version:` after editing `inst/js` or `inst/css`** — `htmlDependency` versions cache by package version; without a bump the browser keeps the old asset.

## Typed JS

The JS layer is type-checked without a build step: `tsconfig.json` at the package root (`allowJs`, `noEmit`, `strictNullChecks`), files opt in with a leading `// @ts-check`, and shared types live in `inst/js/types.d.ts`. Run `npx -p typescript tsc`; CI runs it as a `typecheck-js` job. Annotations are JSDoc (`@param`, `@type`, inline `/** @type {X} */ (expr)` casts) — the shipped files stay plain JS.

When adding a block: write the state interfaces in `types.d.ts` first (they're the protocol contract), `// @ts-check` the new file, and keep `tsc` at zero errors.

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
                     values = list("4", "6"), mode = "include",
                     colType = "numeric"))
  expr <- make_filter_expr(conds, "&")
  result <- eval_bquoted(expr, mtcars)
  expect_true(all(result$cyl %in% c(4, 6)))
})
```

This covers the bulk of correctness — most bugs in JS-driven blocks live in the expression builder, not the UI. Make sure tier 1 includes:

- empty / partially-edited states (must pass through, not error or collapse),
- the value-type matrix: numeric-looking strings on character columns (`"007"`), logicals, factors, and the no-type-tag legacy fallback,
- non-syntactic column names (spaces, quotes, backticks),
- whitelist rejection for enum-like inputs (operators, join types).

If the package migrates serialized state formats, test the migration paths with fixture payloads mirroring the old JSON (see `blockr.dplyr/tests/testthat/test-legacy-deser.R`) — that code decides whether saved boards still open.

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

The trick is that custom JS input bindings aren't reachable via `set_inputs()`. Drive the block by calling its JS API directly through `run_js()`. The element id follows from the naming convention: `<board_id>-block_<block_id>-expr-<name>_input` (board id `"board"`, block id from the `blocks = c(...)` name, `<name>_input` from the block's kebab-case name with `-` → `_`):

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

Read results back with `app$get_values()$export$result$<block_id>` (the test app must call `exportTestValues()`). The reference test app is `blockr.dplyr/tests/testthat/apps/dplyr-e2e/app.R`, with one board pre-wired to all blocks against fixed datasets. Skip the whole file on CRAN (`skip_on_cran()` in the shared fixture setup).

Run with `Sys.setenv(NOT_CRAN = "true"); testthat::test_file("tests/testthat/test-shinytest2.R")`. These are slow (~2-5s per test) — keep one happy-path test per block, push everything else into Tiers 1 and 2.

### What goes in which tier

| What you're testing | Tier |
|---|---|
| Expression construction from a fixed state | 1 |
| Pass-through / empty-state behavior | 1 |
| Value type conversion / odd column names | 1 |
| Legacy state migration | 1 |
| Constructor state → expression | 2 |
| Reactive state changes inside the server | 2 |
| Round-trip from JS UI to evaluated result | 3 |
| The JS class itself in isolation | (none — covered by Tier 3 + `tsc`) |
