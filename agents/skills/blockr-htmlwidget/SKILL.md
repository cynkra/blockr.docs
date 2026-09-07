---
name: blockr-htmlwidget
description: |
  Use when building a custom htmlwidget for a blockr package and wrapping it
  in a block: the file layout, the R payload, the JS factory, drawing over a
  filled area, label collisions, pointer handling, the block contract, and how
  to verify the result in a real board. Trigger on phrases like "create an
  htmlwidget", "write the widget", "wrap the widget in a block", "the widget
  renders blank".
argument-hint: "[widget name] [package]"
---

# blockr-htmlwidget

A widget is two halves that meet in a JSON payload: R builds a list, JS draws
from it. Everything below is the part that is the same whatever you are
drawing, so a prompt only has to say what this widget looks like.

## Files

```
R/<name>.R                     constructor + Shiny bindings
inst/htmlwidgets/<name>.js     HTMLWidgets.widget({ name, type: "output", factory })
inst/htmlwidgets/<name>.yaml   required, even when it declares no dependencies
inst/htmlwidgets/lib/          any JS library you vendor
```

The widget `name` in the JS has to match the file name and the `name` you
pass to `createWidget()`, or nothing binds.

`<name>Output()` wraps `htmlwidgets::shinyWidgetOutput(outputId, name,
width, height, package = "<pkg>")`, and `render<Name>()` wraps
`htmlwidgets::shinyRenderWidget(expr, outputFunction, env, quoted = TRUE)`.

**Bump `Version:` in DESCRIPTION after every edit under `inst/`.**
htmlDependency caches by package version, so without it the browser keeps
serving the old JS and you debug a file nobody is running.

Reference: <https://www.htmlwidgets.org/develop_intro.html>.

## The constructor

- Declare **every** argument the finished widget will take, from the first
  commit. An argument added later is an argument the block passes and the
  widget does not have, which surfaces as `unused argument`.
- Keep the payload JSON-simple: lists, atomic vectors, no S4, no factors.
  `NA` becomes `null`.
- **A data frame does not arrive as rows.** It is serialised in long form,
  an object of named vectors, so `x$riders$bib` is an array and there is no
  `x$riders[0]`. Either call `HTMLWidgets.dataframeToD3()` on the JS side,
  or build the list of rows in R and keep the JS dumb. Pick one and say so
  in the payload's documentation.
- A JS callback travels as `htmlwidgets::JS("function(x) {...}")`. Odd
  serialisation needs go through the `TOJSON_ARGS` attribute rather than a
  hand-rolled `toJSON()`.
- Thin anything long. A few hundred points is past screen resolution.

## The factory

`factory(el, width, height)` returns an object with `renderValue(x)` and,
for anything that has to re-lay-out, `resize(width, height)`.

- `renderValue` runs before anything of yours exists: **guard every dom
  reference**. htmlwidgets catches a throw in `renderValue` and logs it to
  the browser console, so a broken widget is a blank widget with a clean R
  console.
- Keep per-instance state in the factory closure, never in a global: the
  factory runs once per element, `renderValue` is called again on every
  Shiny update, and two widgets can share a page.
- **Implement `resize()`.** That is the framework's hook: it computes the
  size and hands it to you, rather than styling your drawing itself. Redraw
  or re-lay-out there; scaling an SVG blurs every label. Add a
  `ResizeObserver` only when the container can change size without the
  framework being told, which is the case inside a dock panel a user drags.
- **Style the container property by property.** `shinyWidgetOutput()` puts
  the width and height inline on that very element, so
  `el.style.cssText = "..."` replaces the whole attribute and takes them
  with it. The widget then draws correctly into a box 0 pixels tall, which
  is indistinguishable from never rendering.
- `Number.isFinite()`, not `isFinite()`. `isFinite(null)` is **true**,
  because `null` coerces to `0`, so a missing value prints as
  `"null m"` and a missing kilometre draws at zero.

Reference: <https://www.htmlwidgets.org/develop_sizing.html> and
<https://www.htmlwidgets.org/develop_advanced.html>.

## Drawing over a filled area

- Anything the colour of the fill is invisible. Give a marker a light face
  and a dark ring, or a dark face and a pale ring, so it reads on either.
- Draw the layer that moves **last**, over everything else.
- If you cannot count the markers in a screenshot, they are too small.

## Labels and collisions

- Measure a label with `getComputedTextLength()`. Characters times a
  constant is short for anything wide, and the next label lands on it.
- Test **box against box**. A "row" counted from a varying baseline is not a
  constant height, so two labels on different rows collide wherever the
  baseline between them moves by a row.
- Fixed furniture stays fixed and the moving layer gives way. A label that
  shifts because something rode past makes the scene look like it wanders.
- Near the right edge, flip a label to the left of its anchor, or it is cut.

## Pointer

- A press that does not travel has to stay a click. If a drag handler
  re-renders on `mouseup`, every node is replaced and the click that follows
  lands on an element no longer in the document: no handler runs and no panel
  opens. Wait until the pointer has moved a few pixels.
- Test with a **real press and release**. A dispatched `click` event skips
  `mousedown`/`mouseup` and hides exactly this bug.

## Wrapping it in a block

- `new_transform_block()`, `expr_type = "bquoted"`, one class of your own.
- The required input is the only one `dat_valid()` always checks. Optional
  inputs go in `allow_empty_state = list(input = TRUE, data = c("y", ...))`
  and are checked only when they carry rows.
- An unlinked input has no symbol at eval time. Substitute either
  `quote(.(y))` or an inline empty frame, and keep **one** expression
  skeleton rather than a branch per combination.
- **`NULL` cannot travel through a `bbquote()` substitution.** It is
  dropped, the placeholder survives, and the board says
  `object 'x' not found`. Unset parameters travel as empty frames, and empty
  strings for text.
- Anything that cannot change during a session is a constructor argument
  mirrored in `state`, not a link.
- `block_output.<class>()` returns `render<Name>(result)`;
  `block_ui.<class>()` returns `<name>Output(NS(id, "result"), height = ...)`.
- Helpers called from the expression must be **exported** and called
  namespaced. An unexported helper passes the tests and fails on the board.

## Verify

Against the board demo (`app.R`), never a standalone `serve()` of the block.
Drive it with the `shiny-chromote-inspect` skill and assert, rather than
eyeballing:

- the container has a non-zero height, and the drawing has nodes in it;
- the marker count matches the data, and label-label overlaps are zero;
- a real press and release opens the panel;
- the app's stderr is clean: a block whose expression failed prints
  `Warning: Error in eval: ...` there and nothing in the DOM.
