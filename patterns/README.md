# patterns/

Short, diagram-first explanations of common integration flows with ShipStation.

## When to add one

If you've explained the same flow more than once, it's a
candidate. This folder is for recurring, commonly-requested explanations —
not a place to document every possible flow or edge case.

## Format

Use [`TEMPLATE.md`](./TEMPLATE.md) as the starting point for a new pattern
doc. Keep the diagram and walkthrough focused on the **happy path** — the
flow as it normally goes, not every branch. Edge cases go in the `Notes`
section at the bottom, kept brief.

Diagrams use [Mermaid](https://mermaid.js.org/), which GitHub renders
natively in markdown. A `sequenceDiagram` usually
fits request/response and webhook flows best; use a `flowchart` instead when
the pattern is more about branching/decision logic than a timeline.

## Index

<!-- Add a line here each time you add a pattern doc, so this folder is
     browsable without opening every file. -->

- `order-creation-api-label-print-ui.md` — Order to Shipment to Label — UI-Driven Label Print
- `order-import-wms-label-print.md` — Store Import to Shipment to WMS Label Creation — API-Driven
- `erp-sync-label-print-ui.md` — Store Import to Shipment to Label — Multi-System Sync
- `erp-sync-mark-as-shipped.md` — Store Import to Shipment to Fulfillment — Multi-System Sync
- `update-shipment.md` — Shipment Update — Full Payload Required

### Reference

- `TEMPLATE.md` — starting point for new pattern docs
