# WMS Integration

Warehouse management system integration with ShipStation covering order receipt, label creation, and fulfillment tracking. These patterns show how a WMS receives orders from a connected store, creates labels via API with preferred carriers, and syncs fulfillment state.

## Architecture Overview

A WMS (Warehouse Management System) is responsible for picking, packing, and shipping orders. ShipStation provides the logistics layer (label creation, carrier selection, tracking). These patterns show how a WMS receives orders from store imports, queries shipping rates to select the best carrier, creates labels via API, and tracks fulfillment through completion.

## Patterns

- [`store-import-wms-label-print.md`](./store-import-wms-label-print.md) — Receive orders from store; create labels via API with optional rate shopping
- [`store-import-wms-mark-as-shipped.md`](./store-import-wms-mark-as-shipped.md) — Track fulfillment state when labels are created or shipments are marked shipped
- [`wms-rate-label-print.md`](./wms-rate-label-print.md) — Query rates before label creation for carrier comparison

## Key Concepts

**Shipment vs. Order:**
- ShipStation models shipments, not orders. A single order may spawn multiple shipments (split shipments, backorders, partial fulfillment).
- Always store both `shipment_number` (the order number from your store) and `shipment_id` (ShipStation's internal identifier).
- Join webhooks and API responses using `shipment_id`.

**API-driven label creation:**
- Unlike ERP/CRM patterns (which react to UI label creation), WMS patterns create labels via API.
- This gives the WMS full control over timing, carrier selection, and service level.
- The WMS can query rates before committing to a carrier.

**Webhook pointers:**
- `shipment_created_v2` webhooks contain only a `resource_url`.
- Always fetch the full payload via `GET resource_url` before creating a label.

## Typical Flow

1. Order arrives in a connected store (Shopify, Amazon, custom)
2. ShipStation imports it and fires `shipment_created_v2` webhook
3. WMS fetches the resource and creates a shipment record
4. WMS calls `POST /v2/rates` to query shipping rates across carriers (optional)
5. WMS selects a carrier and calls `POST /v2/labels/shipment/{shipment_id}` to create the label
6. ShipStation returns label details (`label_download` URL, `tracking_number`, carrier info)
7. WMS stores label and tracking information in its system
8. ShipStation automatically updates the connected store with tracking
9. WMS marks the shipment as fulfilled when label is created or when `fulfillment_shipped_v2` webhook fires

## Rate Limit Impact

- Shipment creation webhook: 1 fetch per shipment
- Rate query (optional): 1 call per shipment before label creation
- Label creation: 1 API call per label
- Fulfillment updates: optional; 0-1 API calls

At typical volume (50 orders/hour with 80% fulfillment rate, 80% rate queries):
- Without rate queries: ~80 calls/hour
- With rate queries: ~110 calls/hour

Well within the 200 req/min (12,000 req/hour) limit.

## Related Patterns

- [Order Lifecycle](../order%20lifecycle/) — High-level order import and fulfillment workflow
- [ERP Integration](../erp/) — If orders flow through an ERP before the WMS
- [Inventory Management](../inventory/) — If the WMS needs inventory visibility

## References

- Official docs: [Shipments API](https://docs.shipstation.com/apis/openapi/shipments)
- Official docs: [Rates API](https://docs.shipstation.com/apis/openapi/rates/calculate_rates)
- Official docs: [Labels API](https://docs.shipstation.com/apis/openapi/labels/create_label_from_shipment)
- Webhook reference: [Shipment Created (V2)](../../webhooks/orders/README.md)
- Webhook reference: [Fulfillment Shipped (V2)](../../webhooks/fulfillments/README.md)

## Notes

- Orders originate in the connected store and ShipStation imports them. The WMS does not create shipments via API.
- The `shipment_created_v2` webhook contains a pointer only (`resource_url`). Always fetch full details before creating a label.
- Calling `POST /v2/rates` is optional. Your WMS can skip it and go straight to label creation with a pre-selected carrier.
- Use `shipment_id` to join incoming webhooks to existing shipment records in your WMS.
- For split shipments or backorders, you'll receive multiple webhooks for a single order; use `shipment_id` to distinguish them.
- ShipStation automatically syncs label and tracking info back to the connected store — your WMS does not need to.
