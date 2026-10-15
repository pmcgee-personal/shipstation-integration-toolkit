# FedEx Tracking Webhook v1 (`API_TRACK`)

## Objective

To hold real `API_TRACK` webhook bodies for a FedEx parcel at each stage of
the lifecycle, so a receiver can be built and tested without waiting on a
live shipment.

Tracking data is delivered **inline in the POST body** — there is no
follow-up call to make. See [`../README.md`](../README.md) for why, and for
the fields shared by all four carriers. The v2 FedEx fixtures are in
[`../../tracking-v2/fedex/`](../../tracking-v2/fedex/README.md).

All fixtures here have `carrier_code=fedex` in the `resource_url`. That
differs from v2, where FedEx reports as `fedex_walleted`. The body itself
carries no `carrier_code` or `carrier_id`.

## Scenarios

| Fixture                                                                | `status_code` | `status_detail_code` | `carrier_status_code` | Events | What it shows                                                                  |
| ---------------------------------------------------------------------- | ------------- | -------------------- | --------------------- | ------ | ------------------------------------------------------------------------------ |
| [`api-track-label-created.json`](./api-track-label-created.json)       | `NY`          | `SHIPMENT_CREATED`   | `OC`                  | 1      | Label created. Every location field is `null`. `ship_date` and ETA are `null`. |
| [`api-track-in-transit.json`](./api-track-in-transit.json)             | `IT`          | `IN_TRANSIT`         | `AR`                  | 3      | Picked up, label scan, first arrival. ETA is set.                              |
| [`api-track-out-for-delivery.json`](./api-track-out-for-delivery.json) | `IT`          | `OUT_FOR_DELIVERY`   | `OD`                  | 5      | On the vehicle for delivery.                                                   |
| [`api-track-delivered.json`](./api-track-delivered.json)               | `DE`          | `DELIVERED`          | `DL`                  | 6      | Delivered. `actual_delivery_date` is 14 hours **before** the `Delivered` scan. |

The four fixtures are **one parcel at four points in its journey**. Each
fixture's events appear unchanged in the next. The events array is
cumulative and newest first.

## Codes seen

Oldest to newest, from the delivered fixture:

| `status_code` | `status_description` | `status_detail_code` | `event_code` | `description`                        |
| ------------- | -------------------- | -------------------- | ------------ | ------------------------------------ |
| `AC`          | `Accepted`           | `PICKED_UP`          | `PU`         | `Picked up`                          |
| `NY`          | `Not Yet In System`  | `SHIPMENT_CREATED`   | `OC`         | `Shipment information sent to FedEx` |
| `IT`          | `In Transit`         | `IN_TRANSIT`         | `AR`         | `Arrived at FedEx location`          |
| `IT`          | `In Transit`         | `IN_TRANSIT`         | `AR`         | `At local FedEx facility`            |
| `IT`          | `In Transit`         | `OUT_FOR_DELIVERY`   | `OD`         | `On FedEx vehicle for delivery`      |
| `DE`          | `Delivered`          | `DELIVERED`          | `DL`         | `Delivered`                          |

- **The pickup is timestamped before the label scan.** `Picked up` has
  `occurred_at` `2026-10-13T00:00:00Z`, midnight exactly, and the `Shipment
information sent to FedEx` scan is at `11:27:00Z` the same day. Oldest to
  newest, `status_code` goes `AC`, `NY`, `IT`, so a forward-only state
  machine will reject the `NY`. The midnight timestamp looks like a date
  with no time attached.
- **`AR` covers two differently worded scans** that carry the same
  `status_detail_code`. Use the codes, not `description`.
- **`carrier_detail_code` is `null` throughout**, as in FedEx v2. The
  `status_detail_code` field is the one to branch on.

## Differences from FedEx v2

|                             | v1                            | v2                                       |
| --------------------------- | ----------------------------- | ---------------------------------------- |
| `resource_url` carrier      | `fedex`                       | n/a (`fedex_walleted` in `carrier_code`) |
| `label_url`                 | Populated (masked)            | Absent                                   |
| Event `event_description`   | Present, equals `description` | Absent                                   |
| Event `carrier_status_code` | Absent                        | Present                                  |
| `carrier_occurred_at`       | `Z`, equals `occurred_at`     | `Z`, equals `occurred_at`                |
| Event `status_description`  | Populated                     | Populated                                |

## Location fields

- **`postal_code` is `null` on every scan**, including `Delivered`. FedEx is
  the only carrier here that never carries a ZIP.
- **City casing is inconsistent.** `MESQUITE` is upper case and
  `Farmersville` (the delivery scan) is title case.
- **The label scan has no location.** `city_locality`, `state_province`,
  `postal_code`, `latitude` and `longitude` are all `null`, while
  `country_code` is `US`.
- `signer`, `company_name` and `proof_of_delivery_url` are `null`
  throughout.

## Timestamps and dates

`carrier_occurred_at` is `Z`-suffixed and identical to `occurred_at` on
every scan, so FedEx is the one carrier here where it can be trusted as UTC.

The top-level dates are naive (no suffix) and do not line up with the scans:

- `ship_date` (`2026-10-12T17:00:00`) is the `Picked up` scan expressed in
  UTC−7. The scan itself is `2026-10-13T00:00:00Z`.
- `estimated_delivery_date` (`2026-10-13T10:00:00`) is set on the in-transit
  and out-for-delivery fixtures, and `null` on the label-created and
  delivered ones.
- `actual_delivery_date` is `2026-10-14T05:46:29`. The `Delivered` scan is
  `2026-10-14T19:46:29Z`, so the date is 14 hours earlier than the scan. The
  same gap appears in the v2 FedEx fixtures. Take the delivery time from the
  scan.

## Sanitization

The tracking number is masked as 12 `#` characters (length preserved) and
the `resource_url` matches. `label_url` is a URL whose `se-` ID is masked
(`se-#########`). There is no `tracking_url` in v1.

Coordinates are rounded to 2 decimal places. No names, addresses or phone
numbers appear. See [`../README.md`](../README.md#sanitization).
