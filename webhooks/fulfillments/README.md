# On Fulfillment Shipped (V2) Webhook

## Objective

To show what the ShipStation "On Fulfillment Shipped (V2)" webhook sends,
and what you get back when you call its `resource_url`.

A _fulfillment_ is a shipment marked as shipped **without** ShipStation
generating the label — an external tracking number was supplied instead. This
event is how you hear about that and it is a distinct code path from
[`../labels/`](../labels/): no label is created, so no label webhook fires.

## Step 1: Receive the webhook

```json
{
  "resource_url": "https://api.shipstation.com/v2/fulfillments?fulfillment_id=se-16731019&shipment_id=se-412669417",
  "resource_type": "FULFILLMENT_SHIPPED_V2"
}
```

See [`fulfillment-shipped-v2.json`](./fulfillment-shipped-v2.json).

Unlike the order and label webhooks, this `resource_url` is filtered by two
parameters — the specific `fulfillment_id` _and_ its `shipment_id` — so it
addresses exactly one record.

## Step 2: Call the `resource_url` with your API key

**Endpoint**

```
GET /v2/fulfillments?fulfillment_id=:fulfillment_id&shipment_id=:shipment_id
```

**Example request**

```bash
curl --request GET \
  --url 'https://api.shipstation.com/v2/fulfillments?fulfillment_id=se-16731019&shipment_id=se-412669417' \
  --header 'API-Key: YOUR_API_KEY'
```

**Example response**

A paginated `fulfillments` array plus `total`, `page`, `pages` and `links` —
see
[`fulfillment-shipped-v2-resource-url.json`](./fulfillment-shipped-v2-resource-url.json).

The fulfillment object is much smaller than a label or shipment: identifiers,
the tracking number, the provider, and a `ship_to`. There are no `items`, no
`packages`, no rate breakdown. Follow `shipment_id` to `/v2/shipments` for
the rest of the order.

## Scenarios

| Fixture                                                                                  | `total` | What it shows                                                                                        |
| ---------------------------------------------------------------------------------------- | ------- | ---------------------------------------------------------------------------------------------------- |
| [`fulfillment-shipped-v2.json`](./fulfillment-shipped-v2.json)                           | —       | The webhook body itself. Two keys, no fulfillment data.                                              |
| [`fulfillment-shipped-v2-resource-url.json`](./fulfillment-shipped-v2-resource-url.json) | `1`     | External fulfillment — a FedEx tracking number recorded against a shipment ShipStation didn't label. |

## Key fields

| Field                               | In the fixture | Meaning                                                                                         |
| ----------------------------------- | -------------- | ----------------------------------------------------------------------------------------------- |
| `fulfillment_provider_code`         | `external`     | Who shipped it. `external` = tracking supplied by the merchant, not bought through ShipStation. |
| `fulfillment_carrier_friendly_name` | `FedEx`        | The carrier the tracking number belongs to, for display.                                        |
| `fulfillment_fee`                   | `0.00 usd`     | ShipStation charged nothing — consistent with an externally-purchased label.                    |
| `order_source_notified`             | `false`        | Whether the selling channel has been told. Pair with `notification_error_message`.              |
| `void_requested` / `voided`         | `false`        | A fulfillment can be reversed; both are false on a fresh shipped event.                         |
| `delivered_at`                      | `null`         | Not populated at ship time. Delivery comes from tracking, not from this event.                  |

## Notes

Because `fulfillment_provider_code` is `external`, ShipStation is recording a
tracking number rather than owning it. There is no `label_download`, no
`rate_details` and no `shipment_cost` — if your receiver assumes every
shipped event has a label to fetch, this event will break it.

`order_source_notified` being `false` means the marketplace has not been
updated yet. If your integration is the thing that notifies the channel,
this is the flag to key off; check `notification_error_message` before
treating a `false` as "not yet attempted".

For the mechanics shared by every V2 webhook — auth, pagination, retries —
see [`../README.md`](../README.md).
