# ERP Integration

Order import, label creation, and fulfillment synchronization between an ERP system and ShipStation. These patterns show how to maintain a complete order lifecycle in the ERP as shipments are created, labels are generated, and fulfillment is marked.

## Architecture Overview

An ERP system is typically the source of truth for orders, inventory, and financial records. ShipStation handles fulfillment logistics (label creation, carrier selection, tracking). These patterns synchronize order state between the two systems: the ERP receives webhooks when ShipStation processes shipments and labels, maintains order-to-shipment mappings, and can update fulfillment status back when labels are created or shipments are marked as shipped.

## Patterns

- [`store-import-erp-sync-label-print-ui.md`](./store-import-erp-sync-label-print-ui.md) — Sync orders imported from store; react to label creation in ShipStation UI
- [`store-import-erp-sync-mark-as-shipped.md`](./store-import-erp-sync-mark-as-shipped.md) — Sync fulfillment state when labels are generated or orders are marked shipped

## Key Concepts

**Shipment vs. Order:**
- ShipStation models shipments, not orders. A single order may spawn multiple shipments (split shipments, backorders, returns).
- Always store both `shipment_number` (the order number from your source system) and `shipment_id` (ShipStation's internal identifier).
- Join webhooks using `shipment_id`.

**Label as trigger:**
- `label_created_v2` fires when a label is printed in the ShipStation UI or created via API.
- This is the point at which a tracking number is generated and shipping cost is calculated.
- The ERP receives the label details and can update the order with carrier, service, and cost information.

**Webhook pointers:**
- `shipment_created_v2` and `label_created_v2` webhooks contain only a `resource_url`.
- Always fetch the full payload via `GET resource_url` before processing.

## Typical Flow

1. Order arrives in a connected store (Shopify, Amazon, custom)
2. ShipStation imports it and fires `shipment_created_v2` webhook
3. ERP fetches the resource, stores `shipment_id`, and creates an order record
4. User prints label in ShipStation UI
5. ShipStation fires `label_created_v2` webhook
6. ERP fetches the resource and updates the order with label details (carrier, tracking, cost)
7. Optionally, ERP marks the order as fulfilled when `fulfillment_shipped_v2` webhook fires
8. ERP syncs fulfillment state back to the order source system if needed

## Rate Limit Impact

- Shipment creation: 1 API call per shipment (webhook fetch)
- Label creation: 1 API call per label (webhook fetch)
- Fulfillment updates: optional; 0-1 API calls depending on pattern

At typical volume (50 orders/hour with 80% fulfillment rate), overhead is ~40-60 API calls/hour, leaving ample room for inventory sync, order updates, and other operations.

## Related Patterns

- [Inventory Management](../inventory/) — If the ERP needs to sync inventory levels with ShipStation
- [Order Lifecycle](../order%20lifecycle/) — High-level overview of order-to-delivery workflow
- [WMS Integration](../wms/) — If orders flow through a warehouse system before ShipStation
- [CRM Sync](../crm/) — If orders also sync to a CRM for customer tracking

## References

- Official docs: [Shipments API](https://docs.shipstation.com/apis/openapi/shipments)
- Official docs: [Labels API](https://docs.shipstation.com/apis/openapi/labels)
- Webhook reference: [Shipment Created (V2)](../../webhooks/orders/README.md)
- Webhook reference: [Label Created (V2)](../../webhooks/labels/README.md)
- Webhook reference: [Fulfillment Shipped (V2)](../../webhooks/fulfillments/README.md)

## Notes

- Orders originate in the connected store; the ERP does not create shipments via API in these patterns.
- Both `shipment_created_v2` and `label_created_v2` webhooks are pointer-style. Always call `GET resource_url`.
- Use `shipment_id` to join incoming webhooks to existing order records.
- ShipStation automatically syncs label and tracking info back to the connected store — the ERP does not need to.
- For split shipments or backorders, the ERP will receive multiple webhooks for a single order; use `shipment_id` to distinguish them.
