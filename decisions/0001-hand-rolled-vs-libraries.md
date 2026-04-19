---
id: 0001
title: Hand-rolled UI components vs external libraries
status: Accepted
date: 2026-04-19
supersedes: none
superseded-by: none
---

# Hand-rolled UI components vs external libraries

## Context

blockr's production UI components (`Blockr.Select` dropdown, `Blockr.Input` code editor) are hand-rolled vanilla JS rather than built on external libraries like Tom Select, Choices.js, or CodeMirror. The original reasoning in `blockr.design/open/blockr.dplyr-rewrite/1-motivation.md` was: LLMs make it cheap to write good JS, so owning the DOM beats fighting a library's conventions.

After shipping the JS-driven rewrite + the `blockr.dplyr-picker-modernization` spec (label forwarding across all picker blocks), the empirical record supports that reasoning:

- Label forwarding across 11 dplyr blocks: ~1 hour.
- Bordered chrome modifier + CSS variants: small.
- `--bordered` variant adoption in dm pickers: trivial.
- Drag-to-reorder in multi mode: already working.

The one sustained pain point was the Dockview dropdown-stacking saga — a cascade of `contain: paint` / `isolation: isolate` / `overflow: hidden` ancestors that each needed a scoped override. A mature library (Tom Select, Floating UI) would have had portal-to-body built in and skipped this class of bug entirely. But the fix itself was small — roughly one hour of debugging + a ~50-line portal refactor in one file — and the lesson is now encoded.

## Decision

**Stay hand-rolled.** `Blockr.Select` and `Blockr.Input` remain our own components, built on `blockr.dplyr/inst/js/blockr-core.js` conventions.

**Freeze to current feature set.** No new *major* features land in `Blockr.Select` — bugfixes are fine. When a feature is needed that requires significant new capability (virtualization, grouped options, remote/async loading, creatable tags), re-evaluate against Tom Select at that point rather than grow the component further.

## Consequences

Pros of staying hand-rolled
- Zero runtime dependency footprint.
- Total control over styling — no `.selectize-*` specificity fights.
- Shiny integration is our thin wrapper, not a library's.
- The components we have solve the problems we have.

Cons of staying hand-rolled
- Each class of edge-case bug (stacking, accessibility regressions, focus traps under unusual host layouts) is ours to discover and fix. LLMs generate components that reproduce the same issues naïvely — they don't encode "accumulated edge-case wisdom" the way a mature library does.
- Future features like virtualization, grouped options, creatable tags, or remote loading cost more in Blockr.Select than they would in Tom Select, which has them as plugins.

### Cost of swapping to Tom Select (if we ever pull the trigger)

- **Bundle size — the newly decisive factor:**

  | Component | JS + CSS (gz) |
  |---|---|
  | Current Blockr.Select stack (blockr-core + blockr-select JS/CSS + blockr-blocks CSS) | ~9.2 KB |
  | Tom Select base build (no plugins) | ~18 KB |
  | Tom Select complete build (with plugins) | ~30 KB |

  Swapping to Tom Select is a **net +9 to +21 KB gzipped**, not a replacement. This contradicts the blockr ecosystem's established direction of **removing dependencies to shrink the bundle**. The Bootstrap-disable experiment produced a measurable startup-time improvement; Selectize removal is the next target. Adding Tom Select moves us in the opposite direction.

- **Migration labour:** ~3-4 focused days. ~50 call sites across `blockr.dplyr`, `blockr.dm`, `blockr.bi`, `blockr.extra`, `blockr.sandbox`. API shape differs (`ts.setValue` vs our `setOptions(opts, sel)`, render callbacks). Shiny binding wrapper rewritten.
- **Visual drift risk:** Tom Select ships its own structural CSS. ~100 lines of override CSS to match blockr's design system — which can silently break on Tom Select minor-version updates. Same pattern that bit us with selectize.
- **External dependency fragility:** selectize's own history (abandonment → community fork as Tom Select) can repeat. If it does, we swap again.
- **Regression surface:** keyboard-nav semantics, focus management in Shiny's DOM lifecycle, block re-render behaviour — each difference from today's component is a testing and support cost.
- **Test churn:** every testthat/Playwright test selecting `.blockr-select__*` needs updating to `.ts-*`. Grep-and-replace but non-trivial.
- **Opportunity cost:** 3-4 days not spent on new blocks, docs, or feature work.

The table of capabilities in the "Triggers" section may favour Tom Select on paper, but every `✓` in Tom Select's column for features we don't currently use is **dormant capability** — not a present benefit. And the cost of installing it (3-4 days of migration + 2-3× the bundle footprint) runs directly against a measured, prioritised performance direction. The math inverts only when one of those dormant features becomes a present need **and** cannot be added to Blockr.Select more cheaply than the whole-component swap.

## Triggers for revisit

The bar for a full swap is: (a) a real feature need fires, AND (b) adding that feature to Blockr.Select costs more than the full swap + migration + bundle cost. Given the small size of Blockr.Select (~9 KB gz, ~500 lines JS) and the bundle-shrink direction, most individual features are cheaper to add than to swap.

Candidate features worth re-evaluating at:

1. **Server-side remote loading** — "search 10k tickers as user types, only fetch matching N rows." For very large catalogs this is the *correct* architecture, not client-side virtualization. Adding a `load: function(query, callback)` mode to Blockr.Select is ~30 lines. Cheaper than swap.
2. **Client-side virtualization** — only relevant if server-side isn't an option. Non-trivial hand-roll (~100-150 lines, intersection observer + recycling). At this cost the swap math starts getting close.
3. **Grouped options** — section headers in the dropdown (e.g. dm column picker grouped by source table). ~50 lines in Blockr.Select (group headers as non-selectable options). Cheaper than swap.
4. **Creatable options** — user types a value that becomes a new tag. ~40 lines. Cheaper than swap.
5. **Accumulated complexity** — if Blockr.Select exceeds ~1200 LOC (currently ~500), revisit. By then the hand-rolled case is harder to justify.
6. **Variant proliferation** — today we already have `Blockr.Select` (plain) and `Blockr.SelectRich` (in `blockr.seasonal`, 680 LOC, rich two-line options with category badges/icons). A third distinct variant in a third package would be the signal that a single configurable component (Blockr.Select + optional `render` callbacks) is overdue. Before swapping to Tom Select, that unification is probably the cheaper step.
7. **A multi-day edge-case bug** of the same shape as the Dockview stacking saga, where the root cause is something a mainstream library has already debugged.

Two or more of these firing in the same quarter, or (2) alone at a size where the hand-roll is ~60% of the swap cost, warrants a new ADR.

### Related hand-rolled variants in the ecosystem

- `blockr.dplyr/inst/js/blockr-select.js` — canonical, ~500 LOC, 9 KB gz (JS + CSS + shared blocks CSS).
- `blockr.seasonal/inst/js/blockr-select-rich.js` — two-line rich option with category badges + icons, 680 LOC, 5 KB gz. Comment on the file says "extract to blockr.core when needed." The unification step (making Blockr.Select accept a `render` callback) would retire this variant without growing the bundle.

Unification-first, swap-later is the path this ADR endorses.

## Evaluated alternatives (2026-04-19 snapshot)

| Library | License | Size (gz) | Fit | Note |
|---|---|---|---|---|
| **Tom Select** | Apache 2.0 | ~30 KB | Drop-in shape | Modernized selectize fork, plain JS, plugins for virtualization / creatable / grouped. Main contender if we swap. |
| Choices.js | MIT | ~20 KB | Good | Similar vintage, wider install base, slightly older feel. |
| Downshift | MIT | ~6 KB | React-only | Doesn't fit blockr's vanilla-JS stack. |
| React Select | MIT | ~55 KB | React-only | Same issue. |
| Floating UI | MIT | ~15 KB | Partial | Positioning only; doesn't provide the select UI itself. Could pair with a bare `<select>` + custom UI, but we lose search/multi/reorder. |

## Related

- `blockr.design/open/blockr.dplyr-rewrite/` — original rewrite rationale ("JS is cheap in the LLM era").
- `blockr.design/done/blockr.dplyr-picker-modernization/dockview-stacking-analysis.md` — the one saga that tested this decision.
- `blockr.design/open/blockr-select-portal/` — the portal refactor that closed the stacking gap.

## Supporting artifact

`0001-assets/tom-select-demo.html` — self-contained look-and-feel demo covering six Tom Select configurations styled to the blockr design system: single + secondary label, multi + drag-to-reorder, grouped options, creatable tags, 10k-option virtualization, and seasonal-style rich two-line options with category badges. Open the file directly in a browser; no build step. Useful if someone re-opens this ADR and wants to feel what Tom Select actually delivers vs reading about it.
