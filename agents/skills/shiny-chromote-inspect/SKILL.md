---
name: shiny-chromote-inspect
description: |
  Use when you need to see what a running Shiny app actually rendered:
  verifying a block or widget in a board demo, reading htmlwidget state,
  checking DOM attributes or CSS classes, or reproducing a UI bug. Drives a
  headless Chrome from R via {chromote}, reads the DOM and the browser
  console, and takes screenshots. Trigger on phrases like "check the app
  runs", "verify the block in the board", "inspect the widget", "why is the
  widget blank".
argument-hint: "[what to verify] [app path or URL]"
---

# shiny-chromote-inspect

Headless inspection of a running Shiny app from R. Tests passing is not the
same as the app working: this is how you find out which one you have.

## When to use

- Verifying a block or widget end to end in a board demo (`app.R`), the
  deliverable `blockr-block` hands back.
- Reading htmlwidget internals (`HTMLWidgets.find(...)`, then the library's
  own API) rather than guessing from the R payload.
- Checking that a UI fix changed the rendered DOM, not only the R code.
- Regression-testing reactive behaviour: set an input, wait, read an output.
- Reproducing a user-reported visual bug reproducibly.

Do **not** use for:

- Pixel-perfect visual regression. Use `{shinytest2}`.
- Tests you want to commit. Port them to `{shinytest2}` or `testServer()`.

## Prerequisites

```r
install.packages(c("chromote", "shiny"))
```

Chrome or Chromium must be installed; `{chromote}` finds it. On macOS
`/Applications/Google Chrome.app` is enough.

## The pattern

1. Start the app in a background R process on a known port.
2. Poll the port instead of sleeping a fixed amount.
3. Run an inspection script: navigate, evaluate JS, print values.
4. Kill the background process.

```bash
R --quiet -e 'shiny::runApp("<pkg>", port = 4978, host = "127.0.0.1", launch.browser = FALSE)' \
  &>/tmp/shiny.log &
SHINY_PID=$!

until curl -s http://127.0.0.1:4978/ -o /dev/null 2>&1; do sleep 1; done

Rscript inspect.R

kill $SHINY_PID 2>/dev/null
```

`shiny::runApp("<pkg>")` sets the working directory to the package root, so
the `pkgload::load_all(".")` inside `app.R` loads that package wherever you
launched from.

```r
# inspect.R
library(chromote)

b <- ChromoteSession$new(width = 1500, height = 900)
on.exit(b$close(), add = TRUE)

b$Runtime$enable()
b$Runtime$exceptionThrown(callback = function(m) {
  message("JS throw: ", m$exceptionDetails$exception$description)
})
b$Runtime$consoleAPICalled(callback = function(m) {
  if (m$type %in% c("error", "warning")) {
    message("console.", m$type, ": ", m$args[[1]]$value)
  }
})

b$Page$navigate("http://127.0.0.1:4978/")
Sys.sleep(20)   # a board that reads real data needs longer than a toy app

ev <- function(js) b$Runtime$evaluate(js, returnByValue = TRUE)$result$value

ev('document.title')
b$screenshot(filename = "app.png")
```

Details that matter:

- `returnByValue = TRUE` gives you an R value in `$result$value`. Without it
  you get a remote object handle and another round trip.
- Wrap JS in an IIFE (`(function () { ... })()`) so early returns and locals
  work.
- Return only JSON-serialisable values. A DOM node serialises as `{}`, so
  read the attributes you want on the JS side.
- Hook **both** `exceptionThrown` and `consoleAPICalled`. htmlwidgets
  catches a throw inside `renderValue()` and logs it to the console, so a
  widget that fails to render says nothing through `exceptionThrown`.

## Recipes

### Is the block's output actually there, with a size?

An element that rendered into a box 0 pixels tall looks exactly like one
that never rendered. Measure it.

```r
ev('(function () {
  const w = document.querySelector(".myWidget");
  if (!w) return "no widget";
  const r = w.getBoundingClientRect();
  return Math.round(r.width) + "x" + Math.round(r.height);
})()')

ev('document.querySelectorAll(".myWidget svg path, .myWidget svg circle").length')
```

A zero height usually means the widget's own JS overwrote the inline style
htmlwidgets set on the output element (`el.style.cssText = ...` replaces the
whole attribute, `width`/`height` included).

### htmlwidget internal state

```r
ev('(function () {
  const w = HTMLWidgets.find("#my_output_id");
  const inst = w && w.getWidget ? w.getWidget() : w;
  return inst ? Object.keys(inst) : "not found";
})()')
```

### The widget payload

Under Shiny the payload arrives over the websocket, so there is no
`script[data-for]` tag to read: that exists only in static rendering
(R Markdown, `saveWidget()`). Inspect the payload in R (`str(w$x)`) and use
the browser to check what the JS did with it. If the widget keeps it, read
it back:

```r
ev('(function () {
  const w = HTMLWidgets.find("#my_output_id");
  const inst = w && w.getWidget ? w.getWidget() : w;
  return inst && inst.x ? Object.keys(inst.x) : "not exposed";
})()')
```

### Set an input, read an output

```r
ev('Shiny.setInputValue("my_input", 42, { priority: "event" })')
Sys.sleep(1.5)
ev('document.querySelector("#my_output").textContent')
```

### Every Shiny input value

```r
ev('Object.fromEntries(
  Object.entries(Shiny.shinyapp.$inputValues)
    .filter(([k]) => !k.startsWith(".clientdata"))
)')
```

### Board-specific: which panels came up

```r
ev('[...document.querySelectorAll(".dv-default-tab-content")]
     .map(t => t.textContent.trim()).join(" | ")')
ev('[...document.querySelectorAll(".shiny-output-error, .alert")]
     .map(e => e.textContent.trim().slice(0, 200)).join(" | ") || "none"')
```

Also read the app's stderr (`/tmp/shiny.log`): a block whose expression
fails prints `Warning: Error in eval: ...` there and shows nothing in the
DOM.

### Trigger a handler without a real pointer

Many JS libraries attach handlers to nodes. Dispatch the event yourself:

```r
ev('document.querySelector(".item").dispatchEvent(
     new MouseEvent("click", { bubbles: true }))')
```

This verifies handler logic, not real pointer-event plumbing.

## Screenshots

`b$screenshot(filename = "x.png")` captures the viewport;
`b$screenshot(selector = ".myWidget")` captures one element.

A full-page capture of a Shiny app can catch a mid-transition frame, which
reads as a layout bug that is not there. When a measurement and a screenshot
disagree, the DOM measurement wins.

## Cleanup checklist

- Kill the background R process. An orphan holds the port, and the next run
  silently measures the *old* app.
- Pair `ChromoteSession$new()` with `$close()` via `on.exit()`.
- Delete the throwaway script. If it earned its keep, port it to
  `{shinytest2}`.

## Anti-patterns

- A fixed `sleep 10` before connecting. Poll the port.
- Returning DOM nodes from `Runtime$evaluate`.
- Forgetting `returnByValue = TRUE`.
- Trusting a screenshot over the DOM.
- Committing the inspection script as if it were a test.
