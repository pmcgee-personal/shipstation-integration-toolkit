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

### Order Lifecycle

- `order lifecycle/order-creation-label-print-ui.md` — Order Creation (API), Label Print (UI)
- `order lifecycle/store-import-crm-sync.md` — Store Import CRM Sync

### WMS

- `wms/store-import-wms-label-print.md` — Store Import to WMS (API Label Creation)
- `wms/store-import-wms-mark-as-shipped.md` — Store Import to WMS Fulfillment (API Mark As Shipped)
- `wms/wms-rate-label-print.md` — WMS Rate Shop and Label Creation

### ERP

- `erp/store-import-erp-sync-label-print-ui.md` — Store Import ERP Sync (label print ShipStation)
- `erp/store-import-erp-sync-mark-as-shipped.md` — Store Import ERP Sync (mark as shipped ShipStation)

### Freight

- `freight/quote-book-print.md` — Freight Quote, Book, and Print
- `freight/pro-number-retrieval-tracking.md` — PRO Number Retrieval and Tracking
- `freight/track-freight.md` — Freight Tracking

### Webhook Setup

- `webhook setup/webhook-creation.md` — Creating Webhooks
- `webhook setup/webhook-update.md` — Updating Webhooks
- `webhook setup/webhook-delete.md` — Deleting Webhooks

### Exceptions

- `exceptions/update-shipment.md` — Shipment Update

### Reference

- `TEMPLATE.md` — starting point for new pattern docs
