# On New Order Created (V2) Webhook

## Objective

To show what the ShipStation "On New Order Created (V2)" webhook sends, and
what you get back when you call its `resource_url` — for both a single
shipment order and a split shipment order.

## Naming: order vs shipment

This folder is named for the **UI**, not the payload. The two disagree:

| Where              | What it's called                          |
| ------------------ | ----------------------------------------- |
| ShipStation UI     | **On New Order Created** (V2)             |
| Webhook payload    | `"resource_type": "SHIPMENT_CREATED_V2"`  |
| `resource_url`     | `GET /v2/shipments?import_batch_id=...`   |

An order arriving in ShipStation creates one or more *shipments*, and V2
models the event on the shipment side. Subscribe to the UI event called
"On New Order Created", then branch your receiver on
`resource_type == "SHIPMENT_CREATED_V2"`.

## Step 1: Receive the webhook

The body carries no order data — just the pointer:

```json
{
  "resource_url": "https://api.shipstation.com/v2/shipments?import_batch_id=b6ba2537-aef1-44b3-a219-34edc165c442",
  "resource_type": "SHIPMENT_CREATED_V2"
}
```

See [`new-order-created-v2.json`](./new-order-created-v2.json).

The `import_batch_id` is the important part: it scopes the follow-up call to
exactly the shipments this event created.

## Step 2: Call the `resource_url` with your API key

**Endpoint**

```
GET /v2/shipments?import_batch_id=:import_batch_id
```

**Example request**

```bash
curl --request GET \
  --url 'https://api.shipstation.com/v2/shipments?import_batch_id=b6ba2537-aef1-44b3-a219-34edc165c442' \
  --header 'API-Key: YOUR_API_KEY'
```

**Example response**

A paginated `shipments` array plus `total`, `page`, `pages` and `links`.
Each element is a full shipment object — `ship_to`, `ship_from`,
`return_to`, `packages`, `items`, `advanced_options`, buyer notes and
amounts paid.

## Scenarios

| Fixture                                                                                            | `total` | What it shows                                                                                             |
| -------------------------------------------------------------------------------------------------- | ------- | ----------------------------------------------------------------------------------------------------------- |
| [`new-order-created-v2.json`](./new-order-created-v2.json)                                         | —       | The webhook body itself. Two keys, no order data.                                                         |
| [`new-order-created-v2-resource-url-response.json`](./new-order-created-v2-resource-url-response.json) | `1`     | Single shipment. One order, one `shipment_id`, one package, `shipment_status` `pending`.                  |
| [`new-order-created-v2-split-shipment-resource-url.json`](./new-order-created-v2-split-shipment-resource-url.json) | `2`     | Split shipment. **One webhook, two shipments** under one `import_batch_id`, sharing a `shipment_number`.  |

## Split shipments

A split shipment does not fire two webhooks. It fires **one**, and the
`resource_url` returns `"total": 2`:

- Both shipments share the same `shipment_number` (`LW-TEST2`) and the same
  `ship_to`, but have distinct `shipment_id`s (`se-394496923`,
  `se-394496920`) and distinct `packages[].shipment_package_id`s.
- Order-level money is **not** duplicated across the split. The first
  shipment carries `amount_paid` `218.73`, `shipping_paid` `10.00` and
  `tax_paid` `5.00`; the second carries `0.0` for all three. Summing
  `amount_paid` across the array gives the order total — reading it off
  each shipment as if it were an order total does not.

A receiver that reads `shipments[0]` and stops will silently drop half of
every split order. Iterate the array, and follow `links.next` when
`pages > 1`.

## Notes

`shipment_status` is `pending` at this point — the order exists, nothing has
been rated or labelled. The label for these shipments arrives later on a
separate webhook, see [`../labels/`](../labels/).

`store_id` identifies the selling channel the order came from;
`warehouse_id` is the origin the `ship_from` was resolved against.

`items[].options` is a free-form name/value array from the source channel
(e.g. `Size` / `Large`) — don't assume a fixed set of keys.

The `_captured_at` key is a repo convention, not part of the API response.
For the mechanics shared by every V2 webhook — auth, pagination, retries —
see [`../README.md`](../README.md).
