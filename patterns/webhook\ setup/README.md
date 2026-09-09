# Webhook Setup

Webhook lifecycle management — creation, updating, and deletion via ShipStation's webhook API. These patterns cover registering, modifying, and deactivating webhooks to receive real-time events.

## Architecture Overview

Webhooks are the primary mechanism for receiving real-time events from ShipStation. Before using any webhook-driven pattern (orders, labels, tracking, fulfillment), you must register the webhook endpoints with ShipStation. These patterns show how to manage webhooks programmatically, ensuring you receive the events your integration needs without duplicates or stale configurations.

## Patterns

- [`webhook-creation.md`](./webhook-creation.md) — Register new webhooks and check for existing ones
- [`webhook-update.md`](./webhook-update.md) — Modify webhook configuration (event type, target URL, active status)
- [`webhook-delete.md`](./webhook-delete.md) — Deactivate or remove webhooks

## Key Concepts

**Webhook registration:**
- Use `POST /v2/webhooks` to register a new webhook endpoint and event type.
- ShipStation returns a `webhook_id` for future reference.
- Always call `GET /v2/webhooks` first to check for existing webhooks and avoid duplicates.

**Event types:**
- Common events: `shipment_created_v2`, `label_created_v2`, `track_event_v2`, `fulfillment_shipped_v2`
- Each webhook registers for a single event type (or multiple types can be combined in some cases).
- Multiple webhooks can listen to the same event type on different URLs.

**Active status:**
- Webhooks are active by default when created.
- Set `active: false` to register a webhook without receiving events initially.
- Update `active: true` later when you're ready to receive events.

**Webhook retrieval:**
- `GET /v2/webhooks` lists all registered webhooks for the account.
- Use this to check for existing webhooks before creating new ones.
- Each webhook entry includes `webhook_id`, `event_type`, `target_url`, and status.

## Typical Flow

1. Before integrating, call `GET /v2/webhooks` to see what's already configured
2. For each event type your integration needs, create a webhook via `POST /v2/webhooks`
3. ShipStation confirms with `webhook_id` and status
4. During development, you can toggle `active` via `PUT /v2/webhooks/{webhook_id}` to enable/disable
5. If you need to change the URL or event type, update via `PUT /v2/webhooks/{webhook_id}`
6. When decommissioning, delete via `DELETE /v2/webhooks/{webhook_id}`

## Rate Limit Impact

Webhook management is low-volume:
- Initial setup: 1 call to list webhooks, 1 call per event type to register
- Maintenance updates: occasional `PUT` or `DELETE` calls
- Negligible impact on the 200 req/min parcel API limit.

## Related Patterns

All webhook-driven patterns require webhook setup:
- [Order Lifecycle](../order%20lifecycle/)
- [ERP Integration](../erp/)
- [WMS Integration](../wms/)
- [CRM Sync](../crm/)
- [Inventory Management](../inventory/)

## References

- Official docs: [GET /v2/webhooks](https://docs.shipstation.com/apis/openapi/webhooks/list_webhooks)
- Official docs: [POST /v2/webhooks](https://docs.shipstation.com/apis/openapi/webhooks/create_webhook)
- Official docs: [PUT /v2/webhooks/{webhook_id}](https://docs.shipstation.com/apis/openapi/webhooks/update_webhook)
- Official docs: [DELETE /v2/webhooks/{webhook_id}](https://docs.shipstation.com/apis/openapi/webhooks/delete_webhook)
- Webhook reference: [Event Types](../../webhooks/README.md)

## Notes

- Always check for existing webhooks before registering new ones to avoid duplicates.
- Store the `webhook_id` if you may need to update or delete the webhook later.
- Webhook URLs must be publicly accessible and respond with a 2xx HTTP status.
- Webhooks are account-wide; a single registered webhook is visible to all API users.
- For development/testing, use a staging URL first, then update to production when ready.
