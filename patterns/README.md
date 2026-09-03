# patterns/

Short, diagram-first explanations of common integration flows — the kind of
thing you'd otherwise re-explain by email each time it comes up.

## When to add one

If you've explained the same flow by email more than once, it's a
candidate. This folder is for recurring, commonly-requested explanations —
not a place to document every possible flow or edge case.

## Format

Use [`TEMPLATE.md`](./TEMPLATE.md) as the starting point for a new pattern
doc. Keep the diagram and walkthrough focused on the **happy path** — the
flow as it normally goes, not every branch. Edge cases go in the `Notes`
section at the bottom, kept brief.

Diagrams use [Mermaid](https://mermaid.js.org/), which GitHub renders
natively in markdown — no image export needed. A `sequenceDiagram` usually
fits request/response and webhook flows best; use a `flowchart` instead when
the pattern is more about branching/decision logic than a timeline.

## Index

<!-- Add a line here each time you add a pattern doc, so this folder is
     browsable without opening every file. -->

- `TEMPLATE.md` — starting point for new pattern docs
