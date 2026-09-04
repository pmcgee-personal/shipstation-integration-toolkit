# ShipStation Integration Toolkit

**Unofficial.** A practical companion to the
[official ShipStation documentation](https://docs.shipstation.com/). Diagram-first explanations of common integration flows, with production-captured fixture examples and technical references.

## Start Here: Common Integration Workflows

Browse the pattern that matches your integration:

- **[Order Lifecycle](./patterns/order%20lifecycle/)** — Order creation, label printing via UI
- **[WMS Integration](./patterns/wms/)** — Store import to WMS, label creation, fulfillment marking
- **[ERP/Store Sync](./patterns/erp/)** — Multi-system sync between store, ShipStation and ERP
- **[Freight Handling](./patterns/freight/)** — LTL quote, booking, printing, and tracking
- **[Webhook Setup](./patterns/webhook%20setup/)** — Creating, updating, and managing webhooks
- **[Exception Handling](./patterns/exceptions/)** — Handling special cases (shipment updates)
- **[Inventory Management](./patterns/inventory/)** — Inventory sync patterns

Each pattern includes:

- A Mermaid sequence or flow diagram of the happy path
- Step-by-step walkthrough with relevant API calls
- Links to the official docs for deep dives
- References to fixture examples in this repo

See [`patterns/README.md`](./patterns/README.md) for the full index and contributor guidelines.

## Reference: Fixture Examples

Real webhook payloads and API responses, captured from production and sanitized for sharing. Use these to:

- **Test webhook receivers** — import fixtures to test without triggering real events
- **Mock ShipStation responses** — seed your test suite with realistic data
- **Understand carrier nuances** — see how different carriers structure tracking, exceptions, etc.

```
patterns/               Diagram-driven integration flow guides (start here)
  [7 subdirectories with pattern docs by workflow type]
webhooks/               Real webhook payloads and resource_url responses
  orders/               On New Order Created (V2)
  labels/               On Labels Created (V2)
  fulfillments/         On Fulfillment Shipped (V2)
  tracking/             On New Track Event (V2)
    fedex/, ups/, usps/ [carrier-specific tracking events]
api/                    Real API request/response examples
  adjustments/          USPS shipping adjustment reports
  shipments/            Shipment data (including USPS PCID single payor)
  tracking/             USPS tracking API responses
```

### Technical Notes: Webhooks vs API Responses

Webhooks and API responses for the same resource are _similar but not identical_ — don't assume they share an exact schema. They're kept in separate trees on purpose.

- **Webhook bodies** are mostly pointers: you receive a `resource_url` and `resource_type`, then call that URL with your API key to fetch the actual record. Our webhook folders hold both the pointer _and_ the resource_url response.
- **Tracking is the exception**: the webhook body carries tracking data inline, with no follow-up call required. See [`webhooks/README.md`](./webhooks/README.md) for details.
- **Event names vs. resource_type**: Webhook folders are named after the ShipStation UI event name, which may differ from the `resource_type` in the payload. For example, `webhooks/orders/` holds the "On New Order Created (V2)" event, but the payload says `SHIPMENT_CREATED_V2`. Always branch your code on `resource_type`.

## How to Use Fixtures

**Import into tests** — Copy a fixture JSON into your test suite as mock/expected data:

```javascript
const orderPayload = require("./fixtures/webhooks/orders/new-order-created-v2-resource-url-response.json");
expect(myIntegration.parseOrder(orderPayload)).toEqual({
  /* ... */
});
```

**Serve from a mock endpoint** — Point a local mock server (json-server, WireMock, Express/Flask) at a fixture folder to replay webhook deliveries or mock API responses without touching production.

## Contributing Fixtures

Found a stale fixture or missing carrier/event? Pull requests welcome. See [`CONTRIBUTING.md`](./CONTRIBUTING.md) for the submission process and sanitization guidelines. Check [`TODO.md`](./TODO.md) first to see what's on the roadmap.

**Found real customer data in a fixture?** Report it privately at [`SECURITY.md`](./SECURITY.md).

## Fixture Freshness

Fixtures are point-in-time snapshots. ShipStation and carrier schemas evolve; each fixture carries a `_captured_at` date. If something looks outdated, [open an issue](https://github.com/pmcgee-personal/shipstation-integration-toolkit/issues).

## Status

Unofficial, community-maintained. A practical supplement to [ShipStation's official docs](https://docs.shipstation.com/) — not a replacement. Start with the official docs for authoritative specs; use this repo for practical flow guidance and real-world fixture examples.
