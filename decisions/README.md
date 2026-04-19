# Architecture Decision Records

Numbered, dated, immutable records of architectural choices. Filenames follow `NNNN-slug.md`.

When a decision changes, write a **new** ADR and mark the previous one `Superseded`. Don't edit historical ADRs — the whole point is the durable record.

## Index

| # | Title | Status | Date |
|---|---|---|---|
| [0001](./0001-hand-rolled-vs-libraries.md) | Hand-rolled UI components vs external libraries | Accepted | 2026-04-19 |

## Format

Each ADR follows this skeleton:

```markdown
---
id: NNNN
title: <one-line summary>
status: Proposed | Accepted | Superseded
date: YYYY-MM-DD
supersedes: (id or none)
superseded-by: (id or none)
---

# <Title>

## Context
Why this was on the table.

## Decision
What we chose, succinctly.

## Consequences
What this buys us, what it costs.

## Triggers for revisit
Conditions under which we'd reopen this decision.
```
