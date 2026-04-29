# Skills

A collection of optional Claude Code skills for the blockr ecosystem. Each is self-contained — install only what you want.

## Install

```sh
cp -r blockr.docs/agents/skills/<name> ~/.claude/skills/
```

Restart Claude Code (or wait for skill auto-discovery). To uninstall, delete the folder under `~/.claude/skills/`.

## Available

| Skill | When to use |
|---|---|
| [blockr-spec](./blockr-spec/) | Writing or continuing a design spec under `blockr.design/`. Enforces the four-phase flow (motivation → requirements → design → implementation). |
| [blockr-block](./blockr-block/) | Adding a new block to a blockr package. Walks through the R-driven vs JS-driven choice, scaffolds files, writes the matching tests, hands off to Playwright for verification. |

## Coming later

- **blockr-workflow** — drives the dashboard-building workflow (discover → add → configure → serve), currently shipped by `blockr.mcp::install_skill()`. Deferred until `blockr.mcp` is restructured around live Shiny app control.
