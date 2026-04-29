# Patterns

Durable "how we build X" guides for the blockr ecosystem.

## Contents

- [r-driven-blocks.md](./r-driven-blocks.md) — pure-R Shiny module returning `expr` + `state`. The default path; what most blocks should be.
- [js-driven-blocks.md](./js-driven-blocks.md) — block whose UI is a custom JS class wired through a Shiny input binding. Reach for this when you need rich interaction (drag/drop, autocomplete, multi-row builders) that stock Shiny inputs can't deliver cleanly.

## Choosing between R-driven and JS-driven

These are two valid patterns, not a default and a fallback. The right choice depends on the polish you need and the effort you can spend.

**R-driven** is the easier on-ramp:

- Faster to write, easier to debug, `testServer()` is sufficient to verify it.
- Right for simple blocks, internal tools, prototypes, and anyone learning the framework.
- `blockr.core`'s built-in blocks (`new_head_block()`, `new_select_block()`, `new_merge_block()`) follow this pattern.

**JS-driven** is what mature, user-facing blocks tend to become:

- The UX is meaningfully better — multi-row builders, inline expression editors with autocomplete, drag handles, instant client-side feedback. Stock Shiny inputs can't deliver these cleanly.
- Required when you need bidirectional state sync where the JS owns the source of truth between submits, or when you want to use the shared `Blockr.Select` / `Blockr.Input` components.
- All of `blockr.dplyr` is JS-driven — that's the deliberate choice for a polished, production-quality block library.

If you're building blocks that real users will spend hours in, plan for JS-driven from the start. If you're sketching something out or building a one-off, R-driven is fine and you can always rewrite later.

## Testing implication

The two patterns have different testing strategies:

- **R-driven blocks**: `testServer()` covers everything. UI inputs can be simulated with `session$setInputs()`, reactive state can be inspected directly, expressions can be evaluated. `shinytest2` is not needed and was deliberately removed from `blockr.dplyr` when that package was R-driven (~50× faster, easier to debug, no headless browser).
- **JS-driven blocks**: `testServer()` still covers expression building and pure R logic, but it can't drive a custom JS input binding — `session$setInputs()` doesn't fire on bindings Shiny doesn't know about. `shinytest2` becomes necessary for end-to-end coverage of the JS UI → binding → R expression round-trip.

See the "Testing" section in each guide for the patterns.
