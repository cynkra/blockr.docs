# Skills

Optional Claude Code skills for building with blockr. Each is self-contained —
install only what you want.

## Install

```sh
cp -r blockr.docs/agents/skills/<name> ~/.claude/skills/
```

Restart Claude Code (or wait for skill auto-discovery). To uninstall, delete the
folder under `~/.claude/skills/`.

## Available

| Skill | When to use |
|---|---|
| [blockr-block](./blockr-block/) | Adding a new block to a blockr package. Walks through the R-driven vs JS-driven choice, starts new packages from the [scaffolds](../../scaffolds/), writes the matching tests, and verifies the result in a real board. |
| [blockr-htmlwidget](./blockr-htmlwidget/) | Building a custom htmlwidget for a blockr package and wrapping it in a block: file layout, the R payload, the JS factory, drawing and label collisions, pointer handling, the block contract, and what to assert when verifying. |
| [shiny-chromote-inspect](./shiny-chromote-inspect/) | Seeing what a running app actually rendered: driving a headless Chrome from R to read the DOM, htmlwidget state, Shiny input and output values, and the browser console. This is the browser verification `blockr-block` finishes with. |

Skills that automate a team's internal process rather than the act of building
a block don't belong here — they travel with whatever repo owns that process.

## Browser verification

`blockr-block` finishes by verifying the block in a real board, which needs a
skill that can drive a running Shiny app. Any of these works:

- [`shiny-chromote-inspect`](./shiny-chromote-inspect/), which ships here and
  needs nothing beyond `{chromote}` and a Chrome install.
- A Playwright MCP setup, driving the app the skill starts.
- `{chromote}` directly.

Whichever you use, verify against the **board demo** (`app.R` in the scaffold),
not a standalone `shiny::runApp()` of the block on its own. A block that works
in isolation and breaks in a board is the common failure, and only the board
catches it.

## Coming later

- **blockr-workflow** — drives the dashboard-building workflow (discover → add
  → configure → serve), currently shipped by `blockr.mcp::install_skill()`.
  Deferred until `blockr.mcp` is restructured around live Shiny app control.
