# Store Import to Shipment to Label — Multi-System Sync

## Short answer

Orders are imported from a connected store (e.g. Shopify) into ShipStation. ShipStation sends a `"shipment_created_v2"` webhook to your ERP. Your ERP calls the `"resource_url"` to retrieve shipment details and creates an order record. When a user prints the label, ShipStation updates the connected store and sends a `"label_created_v2"` webhook to your ERP. Your ERP calls the `"resource_url"` to fetch label details (`"carrier"`, `"service_code"`, `"tracking_number"`, `"shipment_cost"`) and updates the order.

## Flow

```mermaid
sequenceDiagram
    participant Store as Connected Store<br/>(e.g. Shopify)
    participant SS as ShipStation
    participant ERP as ERP

    Store->>SS: New order
    SS->>SS: Import & create shipment

    SS->>ERP: "shipment_created_v2" webhook<br/>("resource_url")
    ERP->>SS: GET "resource_url"
    SS-->>ERP: {"shipment_number", order details}
    ERP->>ERP: Create order record

    Note over SS,Store: User prints label in ShipStation UI
    SS->>Store: Update order with label/tracking
    SS->>ERP: "label_created_v2" webhook<br/>("resource_url")

    ERP->>SS: GET "resource_url"
    SS-->>ERP: {"shipment_number",<br/>"carrier", "service_code",<br/>"tracking_number", "shipment_cost", ...}
    ERP->>ERP: Update order with label details
```

## References

- Example webhook payload: [`webhooks/shipments/new-order-created-v2.json`](../webhooks/orders/new-order-created-v2.json)
- Example resource_url payload: [`webhooks/shipments/new-order-created-v2-resource-url-response.json`](../webhooks/orders/new-order-created-v2-resource-url-response.json)
- Example webhook payload: [`webhooks/labels/label-created-v2.json`](../webhooks/labels/label-created-v2.json)
- Example resource_url payload [`webhooks/labels/label-created-v2-resource-url.json](../webhooks/labels/label-created-v2-resource-url.json)

## Notes

- Orders originate in the connected store and ShipStation imports them. The integrator does **not** create shipments via API in this pattern.
- `"shipment_number"` is the order number from the connected store
- Both `"shipment_created_v2"` and `"label_created_v2"` webhooks contain **pointers only** (`"resource_url"`). Always call `GET "resource_url"` to retrieve full details.
- ShipStation automatically syncs label and tracking info back to the connected store — your ERP does not need to handle this.
- Useful for keeping your ERP in sync with ShipStation's shipment and label state in near-real-time.
- Different from the integrator-push pattern: here, ShipStation is the order source; the integrator is listening for downstream sync needs.
