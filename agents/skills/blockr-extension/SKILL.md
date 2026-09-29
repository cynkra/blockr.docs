---
name: blockr-extension
description: |
  Use when writing a blockr.dock extension: a dock panel whose server sees
  the whole board, such as a DAG view, an assistant, an inspector or an
  entity profile. Covers deciding between a block and an extension, the
  constructor and server contract, reading block results, keeping off-screen
  blocks evaluated, changing the board through `update`, external control,
  letting a block drive the extension, layout, tests and verification in a
  real board. Trigger on phrases like "create an extension", "write a dock
  extension", "this should be an extension, not a block", "a panel that
  reads other blocks", "new_dock_extension".
argument-hint: "[extension name] [package]"
---

# blockr-extension

An extension is a dock panel with a server that receives the board. A block
is a step in the pipeline: it returns code and other blocks build on its
result. An extension is a view of the board or a control over it, and
nothing links to it.

Checked against blockr.dock 0.1.3.9000 and blockr.core 0.1.4. Read
`?blockr.dock::new_dock_extension` in the installed version before relying
on a detail here.

## Block or extension?

Write an extension when any of these hold. Two or more and it is not a
close call.

| sign | what it looks like in a block |
|---|---|
| the result is not the product | the expr is `identity(data)` or a filter nobody downstream reads |
| it hides its own output | a `block_output()` method that returns an empty or hidden tag |
| it needs the board | it reads board options, or looks up other blocks by class or id |
| it reads several blocks | it has inputs only so it can display them, not transform them |
| it wants to stay put | the layout pins it in a rail next to other extensions |

Stay with a block when other blocks need its result, or when the board
should save and replay what it computes.

## On invocation

1. **Identify the package, what the panel shows, which blocks it reads and
   what it may change.** Ask only if unclear.
2. **Pick a class name ending in `_extension`** and a constructor
   `new_<name>_extension()`. Tell the user both in your first message.
3. **Write constructor, UI, server and tests as one unit**, then a board
   demo, then verify it in a browser.

## Files

```
R/<name>-extension.R            constructor, ui, server, helpers
tests/testthat/test-<name>-extension.R
inst/examples/<name>/app.R      the board demo you verify against
```

`blockr.dock` goes in `Imports`: the constructor calls
`blockr.dock::new_dock_extension()`. No registration step exists for
extensions, unlike blocks.

## The contract

```r
new_peek_extension <- function(...) {
  blockr.dock::new_dock_extension(
    server = peek_server,
    ui = peek_ui,
    name = "Peek",                 # panel title
    class = "peek_extension",      # exactly one class, ending in _extension
    ...
  )
}

peek_ui <- function(id, board, ...) {
  tagList(
    selectInput(NS(id, "blk"), "Block", board_block_ids(board)),
    textOutput(NS(id, "info"))
  )
}

peek_server <- function(id, board, update, ...) {
  moduleServer(id, function(input, output, session) {
    ids <- reactive(board_block_ids(board$board))
    observeEvent(ids(), updateSelectInput(session, "blk", choices = ids()))

    observeEvent(input$blk, {
      update(list(sustain = stats::setNames(
        list(list(set = input$blk)), session$ns("peek")
      )))
    })

    output$info <- renderText({
      req(input$blk, board$blocks[[input$blk]])
      res <- board$blocks[[input$blk]]$server$result()
      paste(input$blk, ":", NROW(res), "rows")
    })

    list(state = list())
  })
}
```

Rules the validator enforces:

- `class` is one string ending in `_extension`. A longer class vector aborts.
- The UI is `function(id, board, ...)`. The id is already namespaced, so use
  `NS(id, ...)` inside it.
- The server is `function(id, board, update, ...)` returning a
  `moduleServer()`. It returns a list with a named `state` list;
  `list(state = list())` is the minimum, and `NULL` is coerced to it.
- Always take `...` in both. More arguments arrive than you use.

The extension's key is its class without `_extension` (`peek`), unless the
board names it: `extensions = list(inspect = new_peek_extension())`. The
server's module id is `ext_<key>`.

## What the server receives

| argument | what it is |
|---|---|
| `board` | read-only reactive values. `board$board` is the board object (`board_block_ids()`, `board_blocks()`, `board_links()`). `board$blocks[[id]]$server$result()` is a block's evaluated data. `board$eval[[id]]` is its status. |
| `update` | a `reactiveVal`. Writing a delta to it is the only way to change the board. Observing it shows every delta, whoever sent it. |
| `view_data` | the live layout, `NULL` until every view has reported. `req()` it. |
| `actions` | the board's action triggers. |
| `extensions` | an environment of the other extensions' results, keyed by id. |

## Reading block data

```r
result_of <- function(blk) {
  req(blk %in% names(board$blocks))
  res <- tryCatch(board$blocks[[blk]]$server$result(),
                  error = function(e) NULL)
  req(is.data.frame(res), nrow(res) > 0L)
  res
}
```

- **Only visible blocks are evaluated.** A block whose panel is closed or in
  a background tab has a `NULL` or stale result. Ask the board to keep the
  blocks you read running, keyed by an id you own:

  ```r
  update(list(sustain = stats::setNames(
    list(list(set = c("riders", "streams"))), session$ns("sources")
  )))
  ```

  Send `set = character()` to release them. Use
  `update(list(evaluate = ids))` for a one-off run.
- **The extension starts before any block server exists.** `board$blocks`
  is empty at init. Read it inside a reactive and `req()` the block; never
  `isolate()` it at start.
- **`result()` can throw** while a block's inputs are unset. Wrap it.

## Changing the board

```r
update(list(blocks = list(add = blocks(h = new_head_block()), rm = "a")))
update(list(links = list(add = links(ab = new_link("a", "b")), rm = "cd")))
update(list(extensions = list(mod = list(profile = list(rider = 23L)))))
```

- Updates apply on a later flush. React to `board$board` or
  `board$last_update`, not to the value you just wrote.
- `blockr.core::validate_board_update(delta, board)` errors early if you
  want to check a delta before sending it.

## External control

State other parts of the board may set goes in `external_ctrl`. Each name
must be a formal of the constructor, and the server must return it as a
`reactiveVal` in `state`:

```r
new_profile_extension <- function(rider = NULL, ...) {
  blockr.dock::new_dock_extension(
    server = profile_server(rider),
    ui = profile_ui,
    name = "Profile",
    class = "profile_extension",
    external_ctrl = "rider",
    ...
  )
}

profile_server <- function(rider) {
  force(rider)
  function(id, board, update, ...) {
    moduleServer(id, function(input, output, session) {
      r_rider <- reactiveVal(rider)
      # ...
      list(state = list(rider = r_rider))
    })
  }
}
```

The board then writes it through
`update(list(extensions = list(mod = list(<key> = list(rider = 23L)))))`.
That path is validated, the value is saved with the board, and the
assistant can set it too. Never write the `reactiveVal` from outside.

## Letting a block drive the extension

A block server gets `(id, data)`, never `update`, so it cannot write that
delta itself. The extension leaves a sender where the block can find it:

```r
# in the extension server
key <- sub("^ext_", "", id)
ctrl <- session$userData$blockr_ext_ctrl
if (is.null(ctrl)) {
  ctrl <- new.env(parent = emptyenv())
  session$userData$blockr_ext_ctrl <- ctrl
}
ctrl[[key]] <- function(...) {
  update(list(extensions = list(mod = stats::setNames(list(list(...)), key))))
}

# in the block server, e.g. on a click
to_ext <- session$userData$blockr_ext_ctrl[[target]]
if (is.function(to_ext)) to_ext(rider = bib)
```

The block takes the extension key as a parameter (`ctrl_target`), so
nothing is hard-coded. For one block driving another block, use
`blockr.viz::ctrl_send()` with `blockr.viz::new_ctrl_bridge_extension()` on
the board instead.

## Layout

With no `views`/`grids`, every extension lands in a left rail. To place it:

```r
new_dock_board(
  extensions = list(dag = new_dag_extension(), peek = new_peek_extension()),
  views = list(main = c("dag", "a", "peek")),
  grids = list(main = dock_grid(group(ext("dag"), ext("peek")), blk("a")))
)
```

An extension key that equals a block id breaks bare-id grids. Keep them
distinct.

## Testing

- **Constructor**: `expect_s3_class()` on the class, `is_dock_extension()`,
  and `blockr.core::external_ctrl_vars()` for the controllable names.
- **Server**: `shiny::testServer()` on the server function with a mocked
  board. A block result only needs `list(server = list(result = reactive(df)))`:

  ```r
  board <- shiny::reactiveValues(
    board = blockr.core::new_board(blocks = c(a = new_dataset_block("iris"))),
    blocks = list(a = list(server = list(result = shiny::reactive(iris))))
  )
  update <- shiny::reactiveVal()
  shiny::testServer(peek_server, args = list(board = board, update = update), {
    session$setInputs(blk = "a")
    expect_equal(update()$sustain[[1]]$set, "a")
    expect_equal(output$info, "a : 150 rows")
  })
  ```

  Assert on what the server wrote to `update()` and on its returned
  `state`, not on internals.
- Put plotting and formatting in plain functions and test them without
  Shiny.

## Verification

Write `inst/examples/<name>/app.R`: a dock board holding the blocks the
extension reads, the extension placed in the grid, and whatever drives it.
Launch it with `shiny::runApp("<path>")`, confirm `Listening on`, read
stderr, then drive it with the `shiny-chromote-inspect` skill (or Playwright,
or `{chromote}`). Check that:

- the panel renders with no `.shiny-output-error` and nothing on the console;
- it shows data from blocks whose panels are **not** open (proves `sustain`);
- a change from outside (a click, or `update()` from another extension)
  moves its state.

Don't install or reinstall packages for the demo. If a dependency is
missing, say so and stop.

## Don'ts

- **Don't turn a view into a block** to reach the board. If the block needs
  a hidden output or a pass-through expr, it is an extension.
- **Don't read `board$blocks` at init**, and don't assume a block's result
  exists because the block does.
- **Don't write another component's `reactiveVal`.** Go through `update`.
- **Don't build board-dependent choices in the UI.** The UI is built once;
  refresh from the server with `update*Input()` or `renderUI()`.
- **Don't forget `...`** in the UI and server signatures.

## When you're done

Tell the user how to run the demo (`shiny::runApp("<path>")`), which blocks
the extension reads and what it can change, and which names are externally
controllable.
