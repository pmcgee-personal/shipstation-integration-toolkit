# Order Lifecycle

Order creation and fulfillment tracking via ShipStation's order and label APIs. These patterns cover the happy path from order import through label printing and tracking.

## Patterns

- [`order-creation-label-print-ui.md`](./order-creation-label-print-ui.md) — Create shipment via API, print label in UI, sync tracking

## Key Concepts

**Order vs. Shipment:** ShipStation models shipments, not orders. When you push an order, ShipStation creates one or more shipments (single shipment is common; split shipments happen for backorders or partial fulfillment).

**Label as trigger:** Label creation is the point at which ShipStation generates a tracking number and shipping cost. The `label_created_v2` webhook fires when a user prints the label (in ShipStation UI) or when a label is created via API.

**UI-driven label creation:** In this pattern, the label is printed manually in ShipStation (not API-driven). Your integration doesn't control when the label is created; it reacts to the webhook.

## Typical Flow

1. Order arrives in order source (store, marketplace, custom system)
2. Your integration creates a shipment in ShipStation via `POST /v2/shipments`
3. ShipStation confirms: shipment created, status is `pending`
4. User prints label in ShipStation UI (or label created via separate API call)
5. ShipStation fires `label_created_v2` webhook
6. Your integration fetches label details and syncs back to order source (tracking number, cost, carrier)

## Rate Limit Impact

- Order creation: 1 API call per order (`POST /v2/shipments`)
- Label creation: 1 API call per label (if via API); 0 if UI-driven
- Label webhook: 1-2 API calls to fetch details (depends on your implementation)

At typical e-commerce volume (50 orders/hour with 80% fulfillment rate), this is low overhead (~40-50 API calls/hour).

## Related Patterns

- [Inventory Management](../inventory/) — If you're syncing inventory when orders are created
- [WMS Integration](../wms/) — If you're pushing orders to a warehouse system
- [ERP Integration](../erp/) — If the order source is an ERP system
- [CRM Sync](../crm/) — If orders also flow to a CRM for customer tracking

## References

- Official docs: [Shipments API](https://docs.shipstation.com/apis/openapi/shipments)
- Official docs: [Labels API](https://docs.shipstation.com/apis/openapi/labels)
- Webhook docs: [On Labels Created (V2)](../labels/README.md)
