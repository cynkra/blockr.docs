# Spacing and sizing

## Control heights

Three tiers, and controls of the same tier line up on a row.

| Token | Height | Use |
|---|---|---|
| `--blockr-control-h` | **42px** | standalone inputs, bordered selects — the universal "input" height |
| `--blockr-control-h-sm` | **30px** | nested inputs inside rows, number inputs, add-row bar |
| `--blockr-control-h-xs` | **26px** | icon buttons (gear, remove), pill toggles, confirm buttons |

`Blockr.Select` is transparent and unsized by default so it inherits the parent
row's 30px slot; the `--bordered` modifier promotes it to the 42px standalone
size.

There is no popover tier. Settings used to open in a floating popover with its
own 38px control height; that popover is gone, replaced by the in-flow settings
band, which uses the standard tiers like everything else. See
[components/blockr-settings.md](./components/blockr-settings.md).

> **Known drift.** The 26px gear is canon, but blockr.viz renders it at 30px so
> it can share the block's top-right corner with search and download, and
> blockr.io at 32px through an unscoped selector that leaks onto every block in
> the app. Whether that corner should become a proper 30px control row — one
> that could hold search and download beside the gear — is an open question.
> **26px remains canon until it is decided; do not bump a gear to match a
> neighbour.**

## Border radius

| Token | Radius | Use |
|---|---|---|
| `--blockr-radius-lg` | **8px** | inputs, rows, dropdowns, settings bands |
| `--blockr-radius-md` | **6px** | internal card sub-sections (separate, bind-rows, pivot-longer, unite) |
| `--blockr-radius-sm` | **4px** | pills, small buttons |

### Data marks

A bar, a box or a swimlane segment is not chrome, and rounding it answers to a
different rule. There are **two** radii on data marks and confusing them is the
mistake to avoid:

| | Value | Meaning |
|---|---|---|
| `--blockr-mark-radius` | **2px** | **Nothing.** Cosmetic. Keeps a mark from reading as a raw div. |
| capsule | `height / 2`, written `999px` | **"This boundary is an estimate."** The point-range fence and inner range. |

Keeping them apart is what makes the second one legible, so the cosmetic radius
must never approach half a mark's thickness. Clamp it as
`min(2px, thickness / 4)` wherever thickness is computed at render time — a
3px-tall bar with a flat 2px radius is a capsule by accident, and now claims
something about the data.

Two rules for the cosmetic radius:

**It applies to the silhouette, never to sub-marks.** A median tick, a fence cap
and a whisker are 1–2px; a radius turns them into dots.

**An end stays square if it sits on an axis, or if it abuts a sibling by
construction.** A bar grows from zero to its value: the value end is a
measurement and rounds, the zero end is the axis, shared by every bar in the
column, and rounding it lifts the bar off its baseline.

The abutment half of that is about *construction*, not about the mark type, and
the distinction that matters is **stacks versus timelines**:

- A **stack tiles by construction** — segments always share edges, because that
  is what stacking means, and they compose one quantity. A seam between them
  would read as a gap in that quantity. Inner joins stay square; only the
  outermost segment has a value end.
- A **timeline does not tile.** Its segments are per-event intervals that
  overlap, leave gaps, or only incidentally touch. Where two of them do touch,
  the seam is *true*: two events, not one long one. Both ends round.

| Mark | Ends | Why |
|---|---|---|
| Bar, value end | round | free end, a measurement |
| Bar, zero end | square | the axis |
| Stack, inner joins | square | tiles by construction |
| Diverging bar | round away from the tick | zero is in the middle, so neither rail end is an axis |
| Box / IQR body | round both | free-standing, neither end on an axis |
| Interval, swimlane, gantt | round both | a timeline is not a stack |
| Heatmap cell | round | separated by its border, so it abuts nothing |
| Ticks, caps, whiskers | square | too thin to carry a radius |

The token is CSS, but two renderers cannot reach it: echarts bars set
`itemStyle.borderRadius`, and custom `renderItem` marks are JS strings built in
R. Both mirror the value as a constant and carry a `CANONICAL SOURCE:` comment.
A stacked bar's value end also has to be set **per datum** rather than per
series, because which segment is outermost varies by group when a category is
missing.

## Gaps and padding

- Row padding: 5px vertical, flex gap 6–12px between fields.
- Dropdown shadow: `--blockr-shadow-dropdown` (`0 4px 12px rgba(0, 0, 0, 0.1)`).
- Focus ring: `--blockr-focus-ring` — one ring for everything focusable, see [colors.md](./colors.md).
- Hover transitions: `--blockr-transition` (`0.15s ease`). No other motion.

## Responsive layout (flex-wrap, not media queries)

Block width is unknown at author time — stacks range from ~300px (narrow sidebar) to the full workspace width, and resize at runtime. **Do not use `@media` or `@container` queries** for block internals. The idiom is a wrapping flex row where each field claims an equal share above a soft minimum, then wraps:

```css
.xxb-grid  { display: flex; flex-wrap: wrap; align-items: flex-end; gap: 6px 12px; }
.xxb-field { flex: 1 1 160px; min-width: 0; display: flex; flex-direction: column; }
```

Rules:

- `flex: 1 1 160px` — fields grow equally, shrink to the ~160px soft minimum, then wrap. Tune the basis per block (short labels can go ~120px; long inputs need 200px+).
- `min-width: 0` is **mandatory** on flex children that contain `Blockr.Select` or other overflowing content — otherwise the intrinsic content width prevents shrinking.
- `align-items: flex-end` keeps inputs bottom-aligned when labels span two lines on one field but not another.
- For a field that should always occupy its own row (e.g. a multi-select with many tags), add a `--full` modifier: `.xxb-field--full { flex: 1 1 100%; }`.

Canonical examples: `blockr.dplyr/inst/css/pivot-longer-block.css` (`.plb-input-row` / `.plb-field`), `blockr.viz/inst/css/summary-table-block.css` (`.stb-grid` / `.stb-field` / `.stb-field--full`).
