# CRM Sync

Order-to-delivery tracking synchronization with a CRM system. These patterns show how to capture shipment creation, label generation, and tracking updates as webhook events and maintain a complete audit trail in your CRM.

## Architecture Overview

A CRM receives real-time events about orders flowing through ShipStation: when shipments are created, when labels are generated (with carrier and tracking info), and when tracking updates arrive from carriers. By listening to webhooks and fetching resource details, the CRM becomes a centralized order intelligence hub.

## Patterns

- [`store-import-crm-sync.md`](./store-import-crm-sync.md) — Sync shipment creation, label generation, and tracking events to a CRM; create return labels via API

## Key Concepts

**Webhook pointers vs. inline data:**
- `shipment_created_v2` and `label_created_v2` webhooks contain only a `resource_url` pointer. Always fetch the full payload via `GET resource_url`.
- `track_event_v2` webhooks carry tracking data inline; no fetch needed.

**Key IDs for joining:**
- Store `shipment_id` when shipments are created.
- Use `shipment_id` to join label creation webhooks.
- Use `tracking_number` to link tracking events to the correct order.

**No cancellation webhook:**
- ShipStation does not fire a webhook when a shipment is cancelled.
- Poll `GET /v2/shipments/{shipment_id}` periodically to detect cancellations.

## Typical Flow

1. Order arrives in a connected store (Shopify, Amazon, etc.)
2. ShipStation imports it and fires `shipment_created_v2` webhook
3. CRM fetches the resource, creates an order record, and stores the `shipment_id`
4. User prints label in ShipStation UI (or label is created via API)
5. ShipStation fires `label_created_v2` webhook with carrier, service, and `tracking_number`
6. CRM fetches the resource, joins on `shipment_id`, and updates the order with tracking details
7. As the shipment travels, `track_event_v2` webhooks arrive with delivery events
8. CRM updates order status with each tracking update
9. For returns, CRM can create a return label via `POST /v2/labels/return-label` using the `shipment_id`

## Rate Limit Impact

- Shipment creation webhook: 1 fetch per shipment
- Label creation webhook: 1 fetch per label
- Tracking updates: inline, no fetch needed
- Cancellation polling: 1 call per shipment periodically

At typical e-commerce volume (50 orders/hour with 80% fulfillment rate and carrier tracking ~4 updates/shipment), this is moderate overhead (~100-150 API calls/hour including polling).

## Related Patterns

- [Order Lifecycle](../order%20lifecycle/) — High-level order import and fulfillment workflow
- [ERP Integration](../erp/) — If the order source is an ERP system
- [WMS Integration](../wms/) — If orders flow to a warehouse system for fulfillment

## References

- Official docs: [Webhooks Overview](https://docs.shipstation.com/webhooks)
- Official docs: [POST /v2/labels/return-label](https://docs.shipstation.com/apis/openapi/labels/create_return_label)
- Official docs: [GET /v2/shipments/{shipment_id}](https://docs.shipstation.com/apis/openapi/shipments/get_shipment_by_id)
- Webhook reference: [Shipment Created (V2)](../../webhooks/orders/README.md)
- Webhook reference: [Label Created (V2)](../../webhooks/labels/README.md)
- Webhook reference: [Track Event (V2)](../../webhooks/tracking/README.md)
