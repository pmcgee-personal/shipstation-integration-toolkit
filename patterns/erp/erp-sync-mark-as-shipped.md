# Store Import to Shipment to Fulfillment — Multi-System Sync

## Overview

Orders are imported from a connected store (e.g. Shopify) into ShipStation. ShipStation sends a `"shipment_created_v2"` webhook to your ERP. Your ERP calls the `"resource_url"` to retrieve shipment details and creates an order record. When a user marks the order as shipped, ShipStation creates a fulfillment and sends a `"fulfillment_shipped_v2"` webhook to your ERP. Your ERP calls the `"resource_url"` to fetch fulfillment details (`"tracking_number"`, `"carrier"`, `"service_code"`) and updates the order.

## Flow

```mermaid
sequenceDiagram
    participant Store as Connected Store<br/>(e.g. Shopify)
    participant SS as ShipStation
    participant ERP as ERP

    Store->>SS: Order imported
    SS->>SS: Import & create shipment

    SS->>ERP: "shipment_created_v2" webhook<br/>("resource_url")
    ERP->>SS: GET "resource_url"
    SS-->>ERP: {"shipment_number", "shipment_id",<br/>order details}
    ERP->>ERP: Store shipment_number +<br/>shipment_id, create order record

    Note over SS,Store: User marks order as shipped in ShipStation UI
    SS->>Store: Update order with tracking number
    SS->>SS: Create fulfillment
    SS->>ERP: "fulfillment_shipped_v2" webhook<br/>("resource_url")

    ERP->>SS: GET "resource_url"
    SS-->>ERP: {"shipment_id",<br/>"tracking_number",<br/>"carrier", "service_code"}
    ERP->>ERP: Match shipment_id,<br/>update order with fulfillment details
```

## References

- Example webhook payload: [`webhooks/shipments/new-order-created-v2.json`](../../webhooks/orders/new-order-created-v2.json)
- Example resource_url payload: [`webhooks/shipments/new-order-created-v2-resource-url-response.json`](../../webhooks/orders/new-order-created-v2-resource-url-response.json)
- Example webhook payload: [`webhooks/fulfillments/fulfillment-shipped-v2.json`](../../webhooks/fulfillments/fulfillment-shipped-v2.json)
- Example resource_url payload: [`webhooks/fulfillments/fulfillment-shipped-v2-resource-url.json`](../../webhooks/fulfillments/fulfillment-shipped-v2-resource-url.json)

## Notes

- Orders originate in the connected store and ShipStation imports them. The integrator does **not** create shipments via API in this pattern.
- Store both `"shipment_number"` (the order number from the connected store) and `"shipment_id"` (ShipStation's internal identifier). A single order can spawn multiple shipments due to split shipments, backorders, or cancellations.
- Match incoming fulfillment webhooks using `"shipment_id"` from the `GET "resource_url"` response.
- A fulfillment record is created when an order is marked as shipped in ShipStation. The label has been created outside of ShipStation. It contains tracking and carrier information.
- Both `"shipment_created_v2"` and `"fulfillment_shipped_v2"` webhooks contain **pointers only** (`"resource_url"`). Always call `GET "resource_url"` to retrieve full details.
- ShipStation automatically syncs fulfillment and tracking info back to the connected store — your ERP does not need to handle this.
- Useful for keeping your ERP in sync with ShipStation's fulfillment state in near-real-time.
- This pattern can be combined with the [`erp-sync-label-print-ui.md`](../patterns/erp-sync-label-print-ui.md). Both `"label_created_v2"` and `"fulfillment_shipped_v2"` webhooks will fire, allowing your ERP to track label creation and fulfillment separately.
