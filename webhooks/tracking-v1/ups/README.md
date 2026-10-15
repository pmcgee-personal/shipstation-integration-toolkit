# UPS Tracking Webhook v1 (`API_TRACK`)

## Objective

To hold real `API_TRACK` webhook bodies for a UPS parcel at each stage of the
lifecycle, so a receiver can be built and tested without waiting on a live
shipment.

Tracking data is delivered **inline in the POST body** — there is no
follow-up call to make. See [`../README.md`](../README.md) for why, and for
the fields shared by all four carriers. The v2 UPS fixtures are in
[`../../tracking-v2/ups/`](../../tracking-v2/ups/README.md).

All fixtures here have `carrier_code=ups` in the `resource_url`. The body
itself carries no `carrier_code` or `carrier_id`.

## Scenarios

| Fixture                                                                | `status_code` | `status_detail_code`          | `carrier_detail_code` | Events | What it shows                                                                                     |
| ---------------------------------------------------------------------- | ------------- | ----------------------------- | --------------------- | ------ | ------------------------------------------------------------------------------------------------- |
| [`api-track-label-created.json`](./api-track-label-created.json)       | `NY`          | `ELEC_ADVICE_RECD_BY_CARRIER` | `MP`                  | 1      | Label created, UPS has not taken possession. `ship_date` is `null` but an ETA exists.             |
| [`api-track-in-transit.json`](./api-track-in-transit.json)             | `IT`          | `HUB_SCAN_IN`                 | `AR`                  | 4      | First hub scans. `ship_date` is now set.                                                          |
| [`api-track-out-for-delivery.json`](./api-track-out-for-delivery.json) | `IT`          | `OUT_FOR_DELIVERY`            | `OF`                  | 8      | Loaded on the delivery vehicle. Ends at the `OF` scan; there is no `Out For Delivery Today` scan. |
| [`api-track-delivered.json`](./api-track-delivered.json)               | `DE`          | `DELIVERED`                   | `FS`                  | 10     | Full lifecycle to delivery. `actual_delivery_date` matches the `Delivered` scan exactly.          |

The four fixtures are **one parcel at four points in its journey**. Each
fixture's events appear unchanged in the next. The events array is
cumulative and newest first.

## Codes seen

| `status_code` | `status_detail_code`            | `carrier_detail_code` | `event_code` | `description`                                                    |
| ------------- | ------------------------------- | --------------------- | ------------ | ---------------------------------------------------------------- |
| `NY`          | `ELEC_ADVICE_RECD_BY_CARRIER`   | `MP`                  | `M`          | `Shipper created a label, UPS has not received the package yet.` |
| `IT`          | `RECEIVED_BY_CARRIER`           | `OR`                  | `I`          | `Arrived at Facility` (first hub)                                |
| `IT`          | `HUB_SCAN_IN`                   | `AR`                  | `I`          | `Arrived at Facility`                                            |
| `IT`          | `HUB_SCAN_OUT`                  | `DP`                  | `I`          | `Departed from Facility`                                         |
| `IT`          | `DELIVERY_SCHEDULED`            | `YP`                  | `I`          | `Processing at UPS Facility`                                     |
| `IT`          | `RECEIVED_LOCAL_DELIVERY_DEPOT` | `DS`                  | `I`          | `Processing at UPS Facility`                                     |
| `IT`          | `OUT_FOR_DELIVERY`              | `OF`                  | `I`          | `Loaded on Delivery Vehicle`                                     |
| `DE`          | `DELIVERED`                     | `FS`                  | `D`          | `Delivered`                                                      |

- **`description` alone does not identify a scan.** `Arrived at Facility`
  is both `RECEIVED_BY_CARRIER` and `HUB_SCAN_IN`, and `Processing at UPS
Facility` is both `DELIVERY_SCHEDULED` and `RECEIVED_LOCAL_DELIVERY_DEPOT`.
  Use the codes.
- **`status_code` does not only move forward.** The label scan is `NY`, and
  the scans after it are `IT`.
- **Top-level `carrier_status_code` is the coarse `event_code`.** It is `I`
  for every in-transit scan, so the detail is in `carrier_detail_code` and
  `status_detail_code`.

## Differences from UPS v2

|                                   | v1                                                                                  | v2                                                                                   |
| --------------------------------- | ----------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------ |
| Event `status_description`        | **`null` on every event**                                                           | Populated (`In Transit`, `Delivered`, …)                                             |
| Event `event_description`         | Present, equals `description`                                                       | Absent                                                                               |
| Event `carrier_status_code`       | Absent                                                                              | Present                                                                              |
| `carrier_detail_code` values seen | `AR`, `DP`, `DS`, `FS`, `MP`, `OF`, `OR`, `YP` (`DS` is not in the v2 UPS fixtures) | `AR`, `DP`, `FS`, `MP`, `OF`, `OR`, `OT`, `PC`, `XD`, `YP` in the 2026-10-09 capture |

Top-level `status_description` is populated in v1 (`Delivered`, `In
Transit`, `Accepted`), so it is the event-level field that is `null`. On the
label-created fixture the top level pairs `status_code` `NY` with
`status_description` `Accepted`; FedEx's `NY` scan says `Not Yet In System`.

## Empty strings where you would expect `null`

UPS mixes `null` and `""` for absent values, and they are not
interchangeable:

- `postal_code` is `""` on every scan **except** `Delivered`, which carries
  the real ZIP.
- `signer` is `""` on every scan, including `Delivered`.
- `city_locality` and `state_province` are `""` (not `null`) on the label
  scan, which has no location. `latitude` and `longitude` are `null` there.
- `company_name` and `proof_of_delivery_url` are `null` throughout.
- City casing differs by scan: `Norwalk`, `Parsippany` and `Blauvelt` are
  title case, and the delivery scan is upper case (`NEW CITY`).

`if (event.postal_code)` and `if (event.postal_code !== null)` disagree on
UPS data. Treat empty string and `null` as the same "absent" case
explicitly.

## Timestamps and dates

`carrier_occurred_at` has **no suffix** and is carrier-local time. In these
fixtures it is 4 hours behind `occurred_at` on every scan, because the
parcel only moves through Connecticut, New Jersey and New York. Across time
zones the offset would vary by scan, so the scan's time zone is needed to
turn it into an instant. Parse `occurred_at` and use `carrier_occurred_at`
for display only.

The top-level dates mix suffixed and unsuffixed values in one payload:

- `ship_date` and `estimated_delivery_date` are the date at midnight with no
  suffix. `ship_date` is the label date (`2026-10-13`), not the first
  physical scan, which is on `2026-10-14`.
- `actual_delivery_date` is `Z`-suffixed and equals the `Delivered` scan's
  `occurred_at`.
- `ship_date` is `null` on the label-created fixture while
  `estimated_delivery_date` is populated. The ETA stays on the delivered
  fixture.

## Sanitization

The tracking number is masked as `1Z################` (18 characters, length
preserved) and the `resource_url` matches. `label_url` is `null`, so there
is no label ID to mask. There is no `tracking_url` in v1.

Coordinates are rounded to 2 decimal places. The delivery ZIP is kept as
received. No names, addresses or phone numbers appear. See
[`../README.md`](../README.md#sanitization).
