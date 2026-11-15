# Scale map — board-level level→aesthetic convention

Study-wide discrete scales ("PD is dark red in every view") carried as the
`"scale_map"` board option. Mechanism lives in **blockr.theme** (the single
home — no vendored copies); renderers consume it via `Suggests:
blockr.theme` behind a `requireNamespace()` guard and fall back to their
standard colors when the package is absent.

## The contract

- **Option id:** `"scale_map"`. Value: a named list keyed by *variable*
  name; each binding is a named list of channels (`color`, `shape`,
  `linetype`).
- **Channel vectors:** *named* = fixed values per level (names matched as
  character; stated in display order), *unnamed* = pool for stable-hash
  auto-assignment. `shape` is integer (`pch` codes).
- **Board palette:** optional, carried as `attr(map, "palette")`,
  serialized as the reserved `.palette` entry (variable names starting with
  `.` are rejected). Fallback-pool order at resolution: binding's own pool →
  board palette → caller's `palette` argument (the renderer's default).
- **Hash assignment:** `blockr.theme:::scale_map_hash_pick()` — a pure
  function of the level name, so the same level gets the same color in
  every view regardless of which other levels are shown. Pinned: saved
  boards rely on stable assignment across versions (test
  `PINNED: hash assignment` in blockr.theme).
- **Serialization:** fixed channels travel as named *lists* (jsonlite drops
  names on atomic vectors); the option value wraps under the `map` ctor arg
  (`blockr_ser.scale_map_option`).
- **Forward compatibility:** readers tolerate unknown entry names inside a
  binding (a future `ramp` kind for continuous variables; see the
  blockr.theme spec, phase 2). `scale_binding()` itself stays strict.

## Consuming (renderer side)

```r
map <- blockr.core::get_board_option_or_null("scale_map", session)  # core only
res <- if (requireNamespace("blockr.theme", quietly = TRUE)) {
  blockr.theme::resolve_scales(map, var, levels, palette = MY_DEFAULTS)
} else NULL   # no theme -> standard colors
```

`resolve_scales()` returns `color`/`shape`/`linetype` (named vectors over
the supplied levels) plus `order` (fixed levels in binding order first), or
`NULL` for an unregistered variable. Pass factor `levels()` when available —
level order lives in the data as factors. Reference consumers:
`blockr.viz/R/scale-map-resolve.R` (drilldown chart, applies at render via
the JS payload), `blockr.ggplot/R/scale-map.R` (injects literal
`scale_*_manual()` into the block expr — exported code reproduces the shown
colors; only complete assignments, since `scale_*_manual()` errors on
gaps). A block that draws several charts from one variable should resolve
the binding once and hand the same colors to each chart, so they cannot
disagree.
The cross-renderer guarantee is the point: the same scale map drives echarts
and ggplot to hex-identical colors, so a level does not change appearance when
a view switches renderer.

## Authoring (template/study side)

Writers and the sidebar editor live in blockr.theme; clinical defaults are
*template content* in the app's template package, serialized into each
study's board file and amended per study (deck edit, not a package change):

```r
options = c(
  dock_board_options(),
  new_board_options(new_scale_map_option(new_scale_map(
    template_scale_map(),                   # template catalog
    scale_binding("TRT", color = c(Placebo = "#6D8196", ...))  # study delta
  )))
)
```

Never put semantic value→color pinning on *continuous* variables: continuous
color encodes relative magnitude and must recompute against the data in
frame (renderer-native `visualMap` / `scale_*_gradientn`). Only the
variable→ramp choice will become bindable (phase 2).
