# Inventory Management

ERP-driven inventory synchronization with ShipStation. The ERP is the source of truth for inventory levels; these patterns show how to keep ShipStation in sync while respecting API rate limits and accepting operational latency.

## Architecture Overview

ShipStation deducts inventory automatically when labels are created (in the UI or via API), but the ERP doesn't learn about this until reconciliation runs. The patterns below balance real-time visibility, rate limit constraints, and eventual consistency.

**Key constraints:**
- **Rate limit:** 200 req/min (shared across all operations: orders, labels, inventory)
- **Acceptable latency:** 5 minutes on inventory updates
- **Model:** ERP is source of truth until fulfillment; SS becomes authoritative during label creation; reconciliation corrects the gap

## Patterns

### Foundation

- [`erp-inventory-warehouse-setup.md`](./erp-inventory-warehouse-setup.md) — Create warehouse/location mapping (prerequisite for all patterns below)

### Sync Operations

- [`erp-inventory-initial-upload.md`](./erp-inventory-initial-upload.md) — Bulk upload all SKUs; migration, disaster recovery, or full resync
- [`erp-inventory-real-time-sync.md`](./erp-inventory-real-time-sync.md) — Queue + batch updates every 5 minutes; continuous operational sync
- [`erp-inventory-periodic-reconciliation.md`](./erp-inventory-periodic-reconciliation.md) — Daily off-peak comparison; catches deductions and drift

### Monitoring (Optional)

- [`erp-inventory-pending-shipment-visibility.md`](./erp-inventory-pending-shipment-visibility.md) — Forecast inventory deductions; overselling alerts

## Key Decisions

**Why not reactive sync on label creation?** The `label_created_v2` webhook doesn't include SKU/item data. Fetching it would require 2-3 extra API calls per label (GET label details, GET shipment details), adding ~60 req/min overhead during peak fulfillment. Reconciliation is simpler and more efficient.

**Why 5-minute batching for real-time sync?** Batches 10-50 SKU changes per POST call, reducing API usage by 10x. At high volume (100 SKU changes/min), this becomes 3 calls per batch (1 per warehouse) every 5 minutes = 36 calls/hour, leaving 150+ req/min for order/label/rating operations.

**Why is reconciliation mandatory?** It catches inventory deductions from UI label creation (which the ERP doesn't see until reconciliation runs), API failures that weren't retried successfully, and manual adjustments. It's the safety net that guarantees eventual consistency.

**Model: ERP as source of truth, except during fulfillment.** Before labels are created, ERP inventory is authoritative. Once SS creates labels, it deducts inventory and becomes the source for that moment. Reconciliation re-syncs, and the cycle repeats. This is pragmatic and handles real-world workflows.

## Implementation Order

1. **Warehouse Setup** — prerequisite; one-time
2. **Initial Upload** — onboarding or migration
3. **Real-time Sync** — continuous operations (queue + batch)
4. **Periodic Reconciliation** — daily safety net
5. **Pending Shipment Visibility** — optional monitoring

## References

- Official docs: [Inventory Management](https://docs.shipstation.com/manage-inventory-levels)
- Official docs: [Inventory API](https://docs.shipstation.com/apis/openapi/inventory)
- Webhook reference: [label_created_v2](../labels/README.md) (pointer-only; no SKU data)
- API endpoints:
  - `POST /v2/inventory_warehouses` — create warehouse
  - `POST /v2/inventory_locations` — create location
  - `GET /v2/inventory` — list SKU inventory
  - `POST /v2/inventory` — update SKU quantities
  - `GET /v2/shipments?status=pending` — list pending shipments (for forecasting)
