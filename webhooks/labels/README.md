# On Labels Created (V2) Webhook

## Objective

To show what the ShipStation "On Labels Created (V2)" webhook sends, and
what you get back when you call its `resource_url` — for both a single
shipment label and a split shipment that produces two labels.

This is the event to hang label download, tracking-number capture, and
"notify the buyer" logic off. It is the first point at which a tracking
number exists.

## Step 1: Receive the webhook

```json
{
  "resource_url": "https://api.shipstation.com/v2/labels?batch_id=se-64616278",
  "resource_type": "LABEL_CREATED_V2"
}
```

See [`label-created-v2.json`](./label-created-v2.json).

Note the filter here is `batch_id` (a `se-` prefixed ShipStation ID), not
the `import_batch_id` UUID used by the order webhook — see
[`../orders/README.md`](../orders/README.md).

## Step 2: Call the `resource_url` with your API key

**Endpoint**

```
GET /v2/labels?batch_id=:batch_id
```

**Example request**

```bash
curl --request GET \
  --url 'https://api.shipstation.com/v2/labels?batch_id=se-64616278' \
  --header 'API-Key: YOUR_API_KEY'
```

**Example response**

A paginated `labels` array plus `total`, `page`, `pages` and `links`. Each
element carries the `tracking_number`, the costs, the `label_download`
links, and a trimmed `ship_to` — but *not* the full shipment (no
`ship_from`, no `items`). Follow `shipment_id` back to `/v2/shipments` if
you need the rest.

## Scenarios

| Fixture                                                                                          | `total` | What it shows                                                                                     |
| -------------------------------------------------------------------------------------------------- | ------- | --------------------------------------------------------------------------------------------------- |
| [`label-created-v2.json`](./label-created-v2.json)                                               | —       | The webhook body itself. Two keys, no label data.                                                 |
| [`label-created-v2-resource-url.json`](./label-created-v2-resource-url.json)                     | `1`     | Single shipment. One `label_id`, one tracking number, one set of download links.                  |
| [`label-created-v2-split-shipment-resource-url.json`](./label-created-v2-split-shipment-resource-url.json) | `2`     | Split shipment. **One webhook, two labels** under one `batch_id`, each with its own tracking number. |

## Split shipments

As with the order event, a split shipment produces **one** webhook whose
`resource_url` returns `"total": 2`:

- The two labels share a `batch_id` (`se-64620276`) and an
  `external_shipment_id` (`SEAuto-A9QmfQO4D0CEBEm3Q8Otvg`), but have
  distinct `label_id`s, distinct `shipment_id`s, and — importantly —
  **distinct `tracking_number`s** (`383144914556`, `383144913218`).
- Each label has its own `label_download` URL set and its own
  `tracking_url`. There is no combined document.
- `shipment_cost` is stated **per label** (`21.96` on each), not split
  across them. Two labels on one order means two charges.

If you push tracking back to a marketplace, a split order needs both
numbers. Iterate the `labels` array; don't read `[0]`.

## Downloading the label

`label_download` offers the same label in three formats plus an `href`
alias for the PDF:

```json
"label_download": {
  "pdf": "https://api.shipstation.com/v2/downloads/14/.../label-184976210.pdf",
  "png": "https://api.shipstation.com/v2/downloads/14/.../label-184976210.png",
  "zpl": "https://api.shipstation.com/v2/downloads/14/.../label-184976210.zpl",
  "href": "https://api.shipstation.com/v2/downloads/14/.../label-184976210.pdf"
}
```

`packages[].label_download` is a *separate* per-package document
(`labelpackage-...`) from the shipment-level one. For a one-package
shipment they show the same label; for a multi-package shipment the
per-package links are what you print. Treat these URLs as
credential-bearing — don't log them or hand them to a browser you don't
control.

## Notes

`rate_details` breaks the `shipment_cost` down into carrier and product
charges, each tagged with a `billing_source` (`carrier` vs `product`). This
is the pre-adjustment cost — the carrier can later disagree, which shows up
in [`api/adjustments/`](../../api/adjustments/README.md).

`tracking_status` is already populated at label creation (`in_transit` in
these fixtures) and is not a substitute for real tracking. For actual scan
history call the tracking endpoint — see
[`api/tracking/usps/`](../../api/tracking/usps/README.md).

`carrier_weight` (`7.2 pound`) is what the carrier rated, while
`packages[].weight` (`18 ounce`) is what was declared. They legitimately
differ; the gap is what drives adjustments.

`voided`, `voided_at` and `refund_details` are all null/false here — a
voided label is a different state you'd see on a later read of the same
`label_id`, not a separate creation webhook.

For the mechanics shared by every V2 webhook — auth, pagination, retries —
see [`../README.md`](../README.md).
