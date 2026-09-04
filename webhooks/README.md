# ShipStation V2 Webhooks

## Objective

To document what ShipStation actually POSTs to a webhook subscriber, and
what you get back when you follow the payload up with an API call.

Most ShipStation webhooks are **notifications, not data**. The POST body
tells you _something happened_ and _where to go and read about it_ — it does
not contain the order, label, or fulfillment itself. Those fixture folders
are therefore a pair: the webhook body, and the response from calling the
`resource_url` it points at.

**Tracking is the exception.** The tracking webhook delivers the tracking
data inline in the POST body, so there is no second request to make. It does
still carry a `resource_url`, but that URL is not how you read the event —
everything is already in `data`. See [`tracking/`](./tracking/).

| Event                        | Body contains                    | Follow-up call needed |
| ---------------------------- | -------------------------------- | --------------------- |
| On New Order Created (V2)    | `resource_url` + `resource_type` | Yes                   |
| On Labels Created (V2)       | `resource_url` + `resource_type` | Yes                   |
| On Fulfillment Shipped (V2)  | `resource_url` + `resource_type` | Yes                   |
| Tracking status updated (V2) | The tracking data itself         | **No**                |

Everything below describes the pointer pattern — i.e. all of the above
except tracking.

## The webhook body

For the pointer events, the body is the same two keys:

```json
{
  "resource_url": "https://api.shipstation.com/v2/shipments?import_batch_id=b6ba2537-aef1-44b3-a219-34edc165c442",
  "resource_type": "SHIPMENT_CREATED_V2"
}
```

| Field           | Description                                                                                  |
| --------------- | -------------------------------------------------------------------------------------------- |
| `resource_type` | Which event fired. Note this does **not** always match the name shown in the ShipStation UI. |
| `resource_url`  | A fully-formed, pre-filtered `GET` URL for the resource(s) the event covers. Call it as-is.  |

`_captured_at` in the fixtures is a repo convention, not something
ShipStation sends — see the [root README](../README.md).

## Step 1: Receive the webhook

Return a `2xx` promptly. Don't do the `resource_url` fetch inline if that
risks a timeout — acknowledge first, queue the fetch.

## Step 2: Call the `resource_url` with your API key

The `resource_url` is not public. It's an ordinary ShipStation API endpoint
with query parameters already applied, and it needs the same `API-Key`
header as any other call:

```bash
curl --request GET \
  --url 'https://api.shipstation.com/v2/shipments?import_batch_id=b6ba2537-aef1-44b3-a219-34edc165c442' \
  --header 'API-Key: YOUR_API_KEY'
```

Use the URL exactly as sent. Don't rebuild it from the ID — the filter it
carries (`import_batch_id`, `batch_id`, `fulfillment_id`) is what scopes the
response to just this event.

## Step 3: Expect a list, not a single object

Every `resource_url` response is a paginated collection —
`shipments` / `labels` / `fulfillments` plus `total`, `page`, `pages` and
`links`. **One webhook can describe more than one record.** A split shipment
fires a single webhook whose `resource_url` returns two shipments, and later
a single label webhook whose `resource_url` returns two labels.

Always iterate the array and honour `links.next` rather than reading
`[0]` — see the split-shipment fixtures in
[`orders/`](./orders/) and [`labels/`](./labels/).

## Folders

| Folder                             | Event                                                  | `resource_url` points at |
| ---------------------------------- | ------------------------------------------------------ | ------------------------ |
| [`orders/`](./orders/)             | On New Order Created (V2) — `SHIPMENT_CREATED_V2`      | `/v2/shipments`          |
| [`labels/`](./labels/)             | On Labels Created (V2) — `LABEL_CREATED_V2`            | `/v2/labels`             |
| [`fulfillments/`](./fulfillments/) | On Fulfillment Shipped (V2) — `FULFILLMENT_SHIPPED_V2` | `/v2/fulfillments`       |
| [`tracking/`](./tracking/)         | Tracking status updated — `TRACK_EVENT_V2`             | n/a — data is inline     |

`tracking/` is split by carrier ([`fedex/`](./tracking/fedex/),
[`ups/`](./tracking/ups/), [`usps/`](./tracking/usps/)) because the envelope
is identical across the three but the values are not — see
[`tracking/README.md`](./tracking/README.md).

## Notes

Don't write a receiver that assumes a `resource_url` always means "go
fetch". The tracking event carries one
(`https://api.shipstation.com/v2/labels/se-454868XXX/track`) but has already
given you the data, so a receiver that pattern-matches on `resource_url` and
fetches will make a pointless extra call — and, because that endpoint reads
live, may get a _different_ state back than the event described. Switch on
`resource_type` first, then decide whether to read the body or go fetch.

The UI name and the `resource_type` are not the same string, and in the
orders case they don't even use the same noun — see
[`orders/README.md`](./orders/README.md). Branch on `resource_type`, and
treat the UI name as documentation only.

Webhook delivery is at-least-once. For the pointer events the body carries
no state of its own, so a duplicate delivery just means a duplicate fetch —
de-duplicate on the IDs in the `resource_url` response (`shipment_id`,
`label_id`, `fulfillment_id`), not on the webhook body. For tracking, the
body _is_ the state, so de-duplicate on the tracking number plus the event
that moved it.

The `resource_url` reads live data at the time you call it, not a snapshot
from when the event fired. If you retry a queued fetch much later, the
record may have moved on. The tracking body doesn't have this problem —
it's a snapshot by construction, which is the flip side of the same trade:
you can't re-read it to get something fresher.
