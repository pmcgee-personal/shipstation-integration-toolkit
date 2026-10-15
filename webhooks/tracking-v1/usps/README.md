# USPS Tracking Webhook v1 (`API_TRACK`)

## Objective

To hold real `API_TRACK` webhook bodies for a USPS parcel at each stage of
the lifecycle, so a receiver can be built and tested without waiting on a
live shipment.

Tracking data is delivered **inline in the POST body** — there is no
follow-up call to make. See [`../README.md`](../README.md) for why, and for
the fields shared by all four carriers. The v2 USPS fixtures are in
[`../../tracking-v2/usps/`](../../tracking-v2/usps/README.md).

All fixtures here have `carrier_code=usps` in the `resource_url`. That
differs from v2, where USPS reports as `stamps_com`. The body itself carries
no `carrier_code` or `carrier_id`.

## Scenarios

| Fixture                                                                | `status_code` | `status_detail_code`          | `carrier_status_code` | Events | What it shows                                                                                            |
| ---------------------------------------------------------------------- | ------------- | ----------------------------- | --------------------- | ------ | -------------------------------------------------------------------------------------------------------- |
| [`api-track-label-created.json`](./api-track-label-created.json)       | `AC`          | `ELEC_ADVICE_RECD_BY_CARRIER` | `GX`                  | 1      | Label created, USPS awaiting the item. `ship_date` is `null`; an ETA exists.                             |
| [`api-track-in-transit.json`](./api-track-in-transit.json)             | `IT`          | `INFORMATION`                 | `TL`                  | 10     | Mid-transit across Louisiana, Arizona and a location-less scan. The newest scan has no location.         |
| [`api-track-out-for-delivery.json`](./api-track-out-for-delivery.json) | `IT`          | `OUT_FOR_DELIVERY`            | `OF`                  | 16     | Out for delivery. The description carries a delivery window.                                             |
| [`api-track-delivered.json`](./api-track-delivered.json)               | `DE`          | `DELIVERED`                   | `01`                  | 17     | Full lifecycle to delivery (`Delivered, In/At Mailbox`). `actual_delivery_date` does not match the scan. |

The four fixtures are **one parcel at four points in its journey**. Each
fixture's events appear in the next, with one change: the `Out for
Delivery` scan reads `Out for Delivery, Expected Delivery Between 11:15am and
1:15pm` in the out-for-delivery fixture and `Out for Delivery` in the
delivered one. The events array is cumulative and newest first.

## Codes seen

From the delivered fixture, oldest to newest:

| `status_code` | `status_detail_code`            | `event_code` | `description`                 | Count |
| ------------- | ------------------------------- | ------------ | ----------------------------- | ----- |
| `AC`          | `ELEC_ADVICE_RECD_BY_CARRIER`   | `GX`         | `Shipping Label Created`      | 1     |
| `AC`          | `PICKED_UP`                     | `03`         | `USPS picked up item`         | 1     |
| `IT`          | `RECEIVED_BY_CARRIER`           | `SF`         | `Departed Post Office`        | 1     |
| `IT`          | `HUB_SCAN_IN`                   | `10`         | `Arrived at USPS Facility`    | 2     |
| `IT`          | `HUB_SCAN_OUT`                  | `T1`         | `Departed USPS Facility`      | 4     |
| `IT`          | `HUB_SCAN_OUT`                  | `U1`         | `Arrived at USPS Facility`    | 1     |
| `IT`          | `INFORMATION`                   | `TL`         | `In Transit to Next Facility` | 2     |
| `IT`          | `HUB_SCAN_IN`                   | `A1`         | `Arrived at USPS Facility`    | 2     |
| `IT`          | `RECEIVED_LOCAL_DELIVERY_DEPOT` | `07`         | `Arrived at Post Office`      | 1     |
| `IT`          | `OUT_FOR_DELIVERY`              | `OF`         | `Out for Delivery`            | 1     |
| `DE`          | `DELIVERED`                     | `01`         | `Delivered, In/At Mailbox`    | 1     |

- **`U1` is mislabelled.** The scan reads `Arrived at USPS Facility` but
  maps to `HUB_SCAN_OUT`. Don't derive direction from `description`.
- **Arrival scans use two codes.** `10` and `A1` both map to `HUB_SCAN_IN`.
- **`status_code` is `AC` for the label and the pickup scans**, then `IT`
  until delivery.
- **`INFORMATION` is a real top-level state.** The in-transit fixture's
  newest scan is `TL`, so its top-level `status_detail_code` is
  `INFORMATION`, not a hub scan. A receiver that only recognises hub codes
  will see an unknown state here.
- **Top-level `carrier_status_description` is a sentence**, such as `Your
item was delivered in or at the mailbox at 12:24 pm on October 1, 2026 in
LOS ANGELES, CA 90025.`, where each event has a short label. The label
  fixture's sentence includes `This does not indicate receipt by USPS or the
actual mailing date.`

## Differences from USPS v2

|                                | v1                            | v2                                              |
| ------------------------------ | ----------------------------- | ----------------------------------------------- |
| `resource_url` carrier         | `usps`                        | n/a (`stamps_com` in `carrier_code`)            |
| Event `event_description`      | Present, equals `description` | Absent                                          |
| Event `carrier_status_code`    | Absent                        | Present                                         |
| `carrier_occurred_at` encoding | `-07:00` offset               | Several (see v2 README)                         |
| Duplicate scans                | None                          | Present in two fixtures (delivered, multi-scan) |
| Out-of-order events            | None                          | Present in two September fixtures               |
| Event `status_description`     | Populated                     | Populated                                       |

## Location fields

- **Facility names in `city_locality`.** On the two Baton Rouge scans
  `city_locality` is `BATON ROUGE LA PROCESSING CENTER`, `state_province` is
  `null` and `company_name` is `null`. On the Lafayette and Los Angeles
  distribution-centre scans the city and state are normal and the facility is
  in `company_name`. The two shapes can appear in the same file.
- **Location-less scans.** `In Transit to Next Facility` (`TL`) has `null`
  `city_locality`, `state_province`, `postal_code`, `latitude` and
  `longitude`, while `country_code` is still `US`.
- **`country_code`** is `US` on every scan, including the location-less
  ones.
- **`signer` and `proof_of_delivery_url`** are `null` throughout.

## Timestamps and dates

`carrier_occurred_at` is a real offset, and it is the **same instant** as
`occurred_at` on every scan. But the offset is `-07:00` on every scan,
whether the scan was in Louisiana, Arizona or California, so it is a fixed
Pacific-daylight rendering rather than the scan's own local time. A scan at
`2026-09-27T15:42:27Z` in Lafayette, Louisiana reads
`2026-09-27T08:42:27-07:00`. Use it for display only, and parse
`occurred_at`.

The top-level dates are naive (no suffix):

- `ship_date` is the pickup scan's time in the same UTC−7 rendering
  (`2026-09-25T13:27:58` for a scan at `20:27:58Z`). It is `null` on the
  label-created fixture, before any pickup exists.
- `actual_delivery_date` is `2026-10-01T12:24:49`, in UTC−7 and unsuffixed.
  The `Delivered` scan's `occurred_at` is `19:24:49Z`, so a parser that
  treats the value as UTC places it 7 hours early.
- `estimated_delivery_date` comes and goes between snapshots: midnight on
  the label fixture (`2026-09-29T00:00:00`), `null` while in transit,
  `2026-09-30T17:00:00` on the out-for-delivery fixture, and `null` again
  once delivered.

## Sanitization

The tracking number is masked as 22 `X` characters (length preserved) and
the `resource_url` matches. `label_url` is `null`, so there is no label ID
to mask. There is no `tracking_url` in v1.

Coordinates are rounded to 2 decimal places. ZIP codes on scans, including
the delivery ZIP, are kept as received, and so is the delivery city in the
top-level `carrier_status_description` sentence. No names, addresses or
phone numbers appear. See [`../README.md`](../README.md#sanitization).
