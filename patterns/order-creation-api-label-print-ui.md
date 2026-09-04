# Order to Shipment to Label — UI-Driven Label Print

## Short answer

Create a shipment in ShipStation via API, store the `"shipment_id"` from the response. When a user prints the label in the ShipStation UI, ShipStation sends a `"label_created_v2"` webhook. Your integrator calls the `"resource_url"` in the webhook to fetch label details (`"carrier"`, `"service_code"`, `"tracking_number"`) and syncs them back to the order source.

## Flow

```mermaid
sequenceDiagram
    participant OrderSrc as Order Source
    participant Int as Integrator
    participant SS as ShipStation

    OrderSrc->>Int: New order
    Int->>SS: POST /v2/shipments
    SS-->>Int: {"shipment_id", ...}
    Int->>Int: Store "shipment_id"

    Note over Int,SS: User prints label in ShipStation UI
    SS->>Int: "label_created_v2" webhook<br/>("resource_url", "resource_type")

    Int->>SS: GET "resource_url"<br/>(with API key)
    SS-->>Int: {"carrier", "service_code",<br/>"tracking_number", "cost", ...}

    Int->>OrderSrc: Sync label details back
```

## References

- Official docs: [POST /v2/shipments](https://docs.shipstation.com/docs/create-shipment-label)
- Official docs: [On Labels Created (V2) webhook](https://docs.shipstation.com/docs/on-labels-created)
- Example webhook payload: [`webhooks/labels/label_created_v2.json`](../webhooks/labels/label_created_v2.json)

## Notes

- Webhook payload contains a **pointer** (`"resource_url"`), not the full label data. You must call `GET "resource_url"` with your API key to retrieve `"carrier"`, `"tracking_number"`, and `"cost"`.
- UI event name is "On Labels Created (V2)" but the webhook `"resource_type"` field will say `"LABEL_CREATED_V2"` — branch your code on `"resource_type"`.
- The label print is **user-initiated in the UI**, not API-driven. Integrator has no control over _when_ it's printed.
- Useful for syncing tracking numbers and label costs back to the order source in near-real-time.
