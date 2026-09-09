# Real-time Inventory Sync (Queue + Batch)

## Overview

Inventory changes in the ERP (sales, receipts, adjustments) are queued and batched for efficient pushing to ShipStation every 5 minutes (or when batch threshold is reached). This pattern respects the 200 rpm API rate limit while keeping inventory visibility within a 5-minute latency window. ERP inventory is updated immediately; ShipStation sync is buffered.

## Flow

```mermaid
sequenceDiagram
    participant ERP as ERP System
    participant Queue as Inventory Queue
    participant Worker as Batch Worker
    participant SS as ShipStation

    ERP->>ERP: Item sold (qty: 5)
    ERP->>ERP: Deduct locally; inventory now 95
    ERP->>Queue: Add change: {sku: "ABC-001", delta: -5, warehouse_id: "w-12345"}

    Note over Queue: Collect for 5 min or until batch threshold

    ERP->>Queue: Item received (qty: 10)
    ERP->>Queue: Add change: {sku: "ABC-002", delta: +10, warehouse_id: "w-12345"}

    ERP->>Queue: Item adjusted (qty: -2)
    ERP->>Queue: Add change: {sku: "ABC-001", delta: -2, warehouse_id: "w-12345"}

    Note over Queue: 5 minutes elapsed (or 50 SKUs queued)

    Worker->>Queue: Batch timer fires
    Queue-->>Worker: All pending changes
    Worker->>Worker: Group by warehouse<br/>Merge duplicates (ABC-001: -5 + -2 = -7)
    Worker->>SS: POST /v2/inventory<br/>(warehouse: w-12345, updates: [ABC-001: -7, ABC-002: +10])
    SS-->>Worker: Success
    Worker->>ERP: Log batch: 2 SKUs, 1 warehouse, timestamp
    Worker->>Queue: Clear processed changes
    Queue->>Queue: Reset timer
```

## References

- Official docs: [Manage Inventory Levels](https://docs.shipstation.com/manage-inventory-levels)
- API docs: [POST /v2/inventory](https://docs.shipstation.com/apis/openapi/inventory/update_inventory)
- Related pattern: [`erp-inventory-periodic-reconciliation.md`](./erp-inventory-periodic-reconciliation.md) (catches failures)

## Notes

- **Batch window:** Collect changes for 5 minutes (aligned to clock: :00, :05, :10, etc.) OR until batch size threshold (e.g., 50 SKUs), whichever comes first. This prevents holding changes too long during quiet periods or creating huge payloads during bursts.

- **Rate limit math:** At peak volume (100 SKU changes/min), a 5-minute batch = 500 changes. Grouped by warehouse (3 sites) = ~170 per site = 1 POST per site = 3 API calls per batch. With batches every 5 minutes, that's 36 calls/hour = 0.6 req/min. Well within the 200 rpm budget (leaves 160+ rpm for orders, labels, rating).

  ```
  Real-time inventory: 0.6 req/min
  Order creation:      50 req/min
  Label rating:        30 req/min
  Label creation:      30 req/min
  ─────────────────────────────
  Total during peak:   ~111 req/min ✅ (under 200)
  ```

- **Deduplication:** If the same SKU is updated twice before batch fires, merge the deltas:
  ```
  T=0:00: SKU-001 sells 5 units → queue: {sku: SKU-001, delta: -5}
  T=2:00: SKU-001 receives 2 units → queue: {sku: SKU-001, delta: +2}
  
  T=5:00 (batch):
    Merged: SKU-001 delta = -5 + 2 = -3
    POST /v2/inventory: {sku: SKU-001, quantity_change: -3}
  ```

- **Multi-warehouse:** If SKU-001 is in both DC-Chicago (w-12345) and DC-LA (w-67890), and both get updates in the same batch:
  ```
  Queue:
    {sku: SKU-001, delta: -3, warehouse: w-12345}
    {sku: SKU-001, delta: -1, warehouse: w-67890}
  
  Batch:
    POST /v2/inventory: {warehouse: w-12345, updates: [{sku: SKU-001, delta: -3}]}
    POST /v2/inventory: {warehouse: w-67890, updates: [{sku: SKU-001, delta: -1}]}
  ```
  Send 1 POST per warehouse; both go in the same batch window (same timestamp).

- **Dead-letter queue:** If a POST fails (rate limit, bad warehouse_id, malformed request):
  ```
  Worker: POST /v2/inventory fails → 429 Too Many Requests
  → Move batch to dead-letter queue
  → Log error with batch details
  → Retry at next batch boundary + exponential backoff (next :05, then :10, then :15, etc.)
  → After 3 failures: alert ops, flag as manual review
  → ERP inventory is already deducted locally; SS is out of sync until retry succeeds
  ```

- **Monitoring queue depth:** Track the queue size (number of pending changes). Alert if it grows unbounded (indicates system is slow or batches aren't clearing). Example thresholds:
  - Queue depth > 1000 for 10+ minutes → alert: "inventory sync backlog growing"
  - Queue depth never drops → alert: "batch worker may be stuck"

- **Timezone awareness:** Log all timestamps in UTC or with explicit timezone. If ERP and ShipStation have clock drift, timestamps may not align; document this in logs for troubleshooting.

- **Transaction semantics:** ERP inventory is committed immediately (local); ShipStation sync is best-effort buffered. If the batch worker crashes, queued changes may be lost. Mitigation: persist the queue to durable storage (database, message broker) and replay on restart. Don't rely on in-memory queues alone.

- **Concurrency:** If multiple ERP instances or nodes are pushing changes, ensure only one batch worker is active at a time (mutex/lock). Or use a message broker (RabbitMQ, SQS) to serialize batches.

- **Threshold tuning:** "Batch every 5 minutes OR when 50 SKUs queued" is a starting point. Adjust based on:
  - Lower threshold (e.g., 20 SKUs) → more frequent batches, tighter latency
  - Higher threshold (e.g., 100 SKUs) → fewer batches, lower API cost
  - Shorter window (e.g., 2 minutes) → tighter latency, more batches
  - Longer window (e.g., 10 minutes) → lower cost, acceptable latency

- **Fallback:** If batching is too complex for your first implementation, consider immediate-push (one change = one POST). Accept that you may hit the 200 rpm limit during peak; reconciliation will catch any missed updates. Add batching later for optimization.

- **Audit trail:** Log each batch:
  ```
  {
    "event": "inventory_batch_sync",
    "timestamp": "2026-09-08T15:05:00Z",
    "batch_id": "batch-2026-09-08-15-05",
    "sku_count": 23,
    "warehouse_count": 2,
    "changes": [
      {sku: "ABC-001", delta: -3, warehouse: "w-12345"},
      {sku: "ABC-002", delta: +10, warehouse: "w-12345"},
      ...
    ],
    "status": "success",
    "duration_ms": 145
  }
  ```
  Reference during reconciliation and troubleshooting.

- **Reconciliation dependency:** This pattern assumes [`erp-inventory-periodic-reconciliation.md`](./erp-inventory-periodic-reconciliation.md) runs daily to catch failures and ShipStation deductions (labels created). Real-time sync + reconciliation together provide data integrity; real-time sync alone does not.
