# Implementation Guide

A well-written spec should make implementation a one-shot operation. The four phases did the thinking — motivation, requirements, design, and implementation detail. The code just follows. If you find yourself making major decisions during implementation, the spec wasn't detailed enough.

## 1. Write the code

Follow the phase 4 implementation document. If something doesn't work as specced during debugging, adapt — don't stop to ask unless the deviation is fundamental (e.g. the approach is wrong, not just a detail).

## 2. Write tests

Two layers, both using testthat.

### Unit tests for pure functions

Test helpers, parsers, validators — anything that takes input and returns output without Shiny. Focus on edge cases like invalid input and boundary conditions. Don't test expression structure (e.g. checking `expr[[1]]`) — that's fragile and breaks when the wrapping changes. Behavioral coverage belongs in testServer tests.

### testServer tests for blocks

This is the main layer. Every block needs testServer tests that verify the **result** — the actual data the block produces.

```r
test_that("filter block filters by selected values", {
  block <- new_filter_block(
    conditions = list(list(column = "cyl", values = c(4, 6), mode = "include"))
  )

  testServer(
    blockr.core:::get_s3_method("block_server", block),
    {
      session$flushReact()
      result <- session$returned$result()

      expect_true(all(result$cyl %in% c(4, 6)))
      expect_false(any(result$cyl == 8))
    },
    args = list(x = block, data = list(data = function() mtcars))
  )
})
```

The pattern:

- `blockr.core:::get_s3_method("block_server", block)` to get the server method
- `args = list(x = block, data = list(data = function() df))` for input data
- `session$flushReact()` before reading anything
- `session$returned$result()` — **this is the thing you're testing**
- `session$returned$state` for reactive state values

### What to test

- **Every constructor argument.** If a block constructor takes `column`, `values`, `mode`, write tests that vary each one and verify the result changes accordingly.
- **State changes.** Modify state mid-test and confirm the result updates:

```r
state <- session$returned$state
state$conditions(list(
  list(column = "Species", values = "virginica", mode = "include")
))
session$flushReact()
expect_equal(nrow(session$returned$result()), 50)
```

- **Edge cases** from the spec's edge cases section.

## 3. Verify

Run `devtools::check()` in the package directory. Must be clean: 0 errors, 0 warnings, 0 notes.

## 4. Playwright

For any UI changes, use the `/blockr-playwright` skill to visually verify the app works. Launch the app, take screenshots, interact with the new UI, confirm it renders and behaves correctly. This catches things that testServer can't — layout issues, JS errors, widget rendering.
