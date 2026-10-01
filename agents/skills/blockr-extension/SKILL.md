---
name: blockr-extension
description: |
  Use when writing a blockr.dock extension: a dock panel whose server sees
  the whole board, such as a DAG view, an assistant, an inspector or a
  control bridge. Covers deciding between a block and an extension, the
  constructor and server contract, reading block results, keeping off-screen
  blocks evaluated, changing the board through `update`, external control,
  a block driving another block, layout, tests and verification in a real
  board. Trigger on phrases like "create an extension", "write a dock
  extension", "should this be an extension or a block", "a panel that
  changes the board", "new_dock_extension".
argument-hint: "[extension name] [package]"
---

# blockr-extension

An extension is a dock panel with a server that receives the board: every
block, link and result, plus `update`, which can add, remove and relink
blocks and set any block's state. A block server receives only its inputs.

Checked against blockr.dock 0.1.3.9000 and blockr.core 0.1.4. Read
`?blockr.dock::new_dock_extension` in the installed version before relying
on a detail here.

## Block or extension?

**A block computes from its inputs, however rich its view. An extension
acts on the board.** Board access is a privilege: give it only to what needs
it.

| it needs to... | write |
|---|---|
| read linked data and show it, however elaborate the view | a block |
| hand its result to other blocks | a block |
| read the board's structure: which blocks and links exist, the layout | an extension |
| add, remove or relink blocks | an extension |
| set another component's state on the user's behalf | an extension (or reuse `blockr.viz::new_ctrl_bridge_extension()`) |

A profile, a dashboard card or a report is a block, even with a sidebar and
stacked charts: link its data in, return something useful (the selected
entity's rows, for instance), draw the rest in its UI. A DAG view, an
assistant, a board inspector or a control bridge is an extension.

Signs you picked wrong:

- A block that reads board options or looks other blocks up by class to get
  its data: link the data in instead.
- A block whose only way to change another block is a workaround: use the
  control channel (`blockr.viz::ctrl_send()` plus the bridge extension)
  rather than turning it into an extension.
- An extension that only reads two blocks' results and draws them: it has
  board write access it never uses. Make it a block with those two inputs.

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
    list(list(set = c("a", "b"))), session$ns("sources")
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
update(list(blocks = list(mod = list(a = list(n = 10L)))))
```

- `blocks$mod` only sets arguments the target block declares in its
  `external_ctrl`; anything else is rejected.
- Updates apply on a later flush. React to `board$board` or
  `board$last_update`, not to the value you just wrote.
- `blockr.core::validate_board_update(delta, board)` errors early if you
  want to check a delta before sending it.

## External control

State other parts of the board may set goes in `external_ctrl`. Each name
must be a formal of the constructor, and the server must return it as a
`reactiveVal` in `state`:

```r
new_inspect_extension <- function(selected = NULL, ...) {
  blockr.dock::new_dock_extension(
    server = inspect_server(selected),
    ui = inspect_ui,
    name = "Inspect",
    class = "inspect_extension",
    external_ctrl = "selected",
    ...
  )
}

inspect_server <- function(selected) {
  force(selected)
  function(id, board, update, ...) {
    moduleServer(id, function(input, output, session) {
      r_selected <- reactiveVal(selected)
      # ...
      list(state = list(selected = r_selected))
    })
  }
}
```

The board then writes it through
`update(list(extensions = list(mod = list(inspect = list(selected = "a")))))`,
from another extension or from the assistant. That path is validated and the
value is saved with the board. Never write the `reactiveVal` from outside.

## A block driving another block

A block server gets `(id, data)`, never `update`, so it cannot change
another block directly. Give the target block `external_ctrl` names, add
`blockr.viz::new_ctrl_bridge_extension()` to the board, and send from the
source block:

```r
blockr.viz::ctrl_send(target, rider = 23L, at = "16:30:00")
```

The bridge is an extension precisely because it needs `update`; everything
else stays a block. Board access then lives in one small, auditable place.

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

- **Don't make a view an extension** to read data. Link the data into a
  block; an extension gets write access to the whole board.
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
