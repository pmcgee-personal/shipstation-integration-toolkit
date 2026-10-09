# UPS Tracking Webhook (`TRACK_EVENT_V2`)

## Objective

To hold real `TRACK_EVENT_V2` webhook bodies for UPS parcels at each stage of
the lifecycle, so a receiver can be built and tested without waiting on a
live shipment.

Unlike every other webhook in this repo, tracking delivers its data **inline
in the POST body** — there is no follow-up call to make. See
[`../README.md`](../README.md) for why, and for the fields shared by all
three carriers.

All fixtures here were captured with `carrier_code` `ups` and `carrier_id`
`3`.

## Scenarios

| Fixture                                                                              | Shape     | `status_code` | `status_detail_code`          | `carrier_detail_code` | Events | What it shows                                                                                               |
| ------------------------------------------------------------------------------------ | --------- | ------------- | ----------------------------- | --------------------- | ------ | ----------------------------------------------------------------------------------------------------------- |
| [`track-event-v2-pre-transit.json`](./track-event-v2-pre-transit.json)               | September | `NY`          | `ELEC_ADVICE_RECD_BY_CARRIER` | `MP`                  | 1      | Label created, UPS has not taken possession. `ship_date` is `null` but an `estimated_delivery_date` exists. |
| [`track-event-v2-multi-scan.json`](./track-event-v2-multi-scan.json)                 | September | `IT`          | `null`                        | `OD`                  | 10     | Mid-transit, loaded on the delivery vehicle. Ten scans accumulated.                                         |
| [`track-event-v2-out-for-delivery.json`](./track-event-v2-out-for-delivery.json)     | September | `IT`          | `null`                        | `OT`                  | 13     | Out for delivery. Four near-identical `Preparing for Delivery` scans in a row.                              |
| [`track-event-v2-delivered.json`](./track-event-v2-delivered.json)                   | October   | `DE`          | `DELIVERED`                   | `FS`                  | 16     | Full lifecycle to delivery. `actual_delivery_date` matches the `Delivered` scan exactly.                    |
| [`track-event-v2-delivery-exception.json`](./track-event-v2-delivery-exception.json) | —         | —             | —                             | —                     | —      | **Placeholder** — awaiting a real capture.                                                                  |

Each fixture is a different parcel. The `events` array is **cumulative** —
every delivery re-sends the full accumulated history, newest first.

## Two shapes in this folder

UPS payloads changed between the September 2026 captures and the 2026-10-09
capture, following changes to the webhook and its field mapping. The October
shape is the current one. Only `track-event-v2-delivered.json` has been
recaptured in it; the other three real fixtures are the September shape and
are due for recapture.

The sections below describe the current shape first, then the differences
from the September fixtures that remain in this folder.

## Current shape: `status_detail_code` is populated

The 2026-10-09 capture populates `status_detail_code` and
`status_detail_description` on every event, top-level and per scan, alongside
`carrier_detail_code`:

| `status_detail_code`          | `carrier_detail_code` | Scan description                  | `status_code` |
| ----------------------------- | --------------------- | --------------------------------- | ------------- |
| `ELEC_ADVICE_RECD_BY_CARRIER` | `MP`                  | `Shipper created a label, …`      | `NY`          |
| `IN_TRANSIT`                  | `XD`                  | `Drop-Off`                        | `IT`          |
| `PICKED_UP`                   | `PC`                  | `RFID Confirmed Pickup`           | `AC`          |
| `RECEIVED_BY_CARRIER`         | `OR`                  | `Arrived at Facility` (first hub) | `IT`          |
| `HUB_SCAN_IN`                 | `AR`                  | `Arrived at Facility`             | `IT`          |
| `HUB_SCAN_OUT`                | `DP`                  | `Departed from Facility`          | `IT`          |
| `DELIVERY_SCHEDULED`          | `YP`                  | `Processing at UPS Facility`      | `IT`          |
| `OUT_FOR_DELIVERY`            | `OF`                  | `Loaded on Delivery Vehicle`      | `IT`          |
| `OUT_FOR_DELIVERY`            | `OT`                  | `Out For Delivery Today`          | `IT`          |
| `DELIVERED`                   | `FS`                  | `Delivered`                       | `DE`          |

So the FedEx/USPS pattern — branch on `status_detail_code` — applies to UPS
too. Note what the table shows:

- **`OUT_FOR_DELIVERY` appears on two scans, 68 minutes apart.**
  `Loaded on Delivery Vehicle` (`OF`) and `Out For Delivery Today` (`OT`)
  carry the same `status_detail_code`. A receiver that notifies on the first sighting of
  `OUT_FOR_DELIVERY` sends at the _loaded_ scan. `DELIVERY_SCHEDULED` (`YP`)
  comes just before and is still in transit.
- **`description` alone does not identify a scan.** `Arrived at Facility`
  appears for both `RECEIVED_BY_CARRIER` (`OR`) and `HUB_SCAN_IN` (`AR`).
  Use the codes.
- **`status_code` does not only move forward.** Oldest to newest the
  sequence is `NY`, `IT` (`Drop-Off`), `AC` (`RFID Confirmed Pickup`), `IT`, …
  `DE`. The drop-off scan is timestamped before the pickup scan, so a state
  machine that only moves forward will reject the `AC`.

`carrier_detail_code` is still populated and the two fields move together.
Treat both as pass-through values, not a stable contract.

### Differences from the September fixtures

|                           | September fixtures                                       | 2026-10-09 capture                                                        |
| ------------------------- | -------------------------------------------------------- | ------------------------------------------------------------------------- |
| `status_detail_code`      | `null` (except pre-transit)                              | Populated on every scan                                                   |
| Scan wording              | `On the Way`, `We Have Your Package`, `Out for Delivery` | `Arrived at Facility`, `Departed from Facility`, `Out For Delivery Today` |
| Loaded on vehicle         | `carrier_detail_code` `OD`                               | `OF`                                                                      |
| First possession scan     | `OR` — `We Have Your Package`                            | `PC` (`RFID Confirmed Pickup`) then `OR`                                  |
| Drop-off scan             | `Dropped off at The UPS Store by Customer`               | `Drop-Off`                                                                |
| Repeated scans            | Four `Preparing for Delivery` in a row                   | One `Processing at UPS Facility`                                          |
| `carrier_occurred_at`     | `Z`-suffixed, but local time (pre-transit: no suffix)    | No suffix, local time                                                     |
| `signer`                  | `null` (`""` on pre-transit)                             | `""` on every scan                                                        |
| `ship_date`               | First scan timestamp, `Z`-suffixed (pre-transit: `null`) | Label date at midnight, no suffix                                         |
| `estimated_delivery_date` | `Z`-suffixed (pre-transit: no suffix)                    | No suffix                                                                 |
| `latitude` / `longitude`  | 2 decimals                                               | 4 decimals in the raw body (normalized to 2 here)                         |

A receiver that still has to read older bodies should key on codes and treat
`description` as display text.

## Codes seen

`carrier_detail_code` in the 2026-10-09 capture: `AR`, `DP`, `FS`, `MP`,
`OF`, `OR`, `OT`, `PC`, `XD`, `YP`. The September fixtures use `AR`, `DP`,
`FS`, `MP`, `OD`, `OR`, `OT`, `XD`, `YP`. `event_code` is `M` (label), `P`
(pickup, October only), `I` (everything in transit) and `D` (delivered).

## Empty strings where you would expect `null`

UPS mixes `null` and `""` for absent values, and they are not
interchangeable:

- `postal_code` is `""` on every scan **except** `Delivered`, which carries
  the real ZIP (`06850` in the October capture, `48067` in September).
- In the October capture `signer` is `""` on every scan, including
  `Delivered`. In the September fixtures it is `null` except on pre-transit,
  where it is `""`.
- `city_locality` and `state_province` are `""` (not `null`) on the October
  label-created scan, which has no location, and `latitude` and `longitude`
  are `null` there.
- `company_name` and `proof_of_delivery_url` are `null` throughout.

`if (event.postal_code)` and `if (event.postal_code !== null)` disagree on UPS
data. Treat empty string and `null` as the same "absent" case explicitly.

## Timestamps

**October capture:** `carrier_occurred_at` has no suffix and is carrier-local
time. Compared with `occurred_at`, the offset is 7 hours for scans in
Washington and Oregon (Pacific daylight time) and 4 hours for scans in
Massachusetts and Connecticut (Eastern daylight time). The scan's time zone
is needed to turn it into an instant.

**September fixtures:** `carrier_occurred_at` is `Z`-suffixed but is not UTC.
It is UPS's local scan time with a UTC marker glued on. Comparing it against
`occurred_at` yields offsets of 4, 5 **and** 7 hours _within a single file_ —
the parcel is crossing time zones and each scan carries its own local clock.
[`track-event-v2-pre-transit.json`](./track-event-v2-pre-transit.json) has
**no** suffix at all (`"2026-09-03T11:19:14"`), where the other September
fixtures use `Z`.

In every case, parse `occurred_at` and use `carrier_occurred_at` for display
only. See [`../README.md`](../README.md#carrier_occurred_at-is-not-utc).

`actual_delivery_date` is `Z`-suffixed in the October capture and equals the
`Delivered` scan's `occurred_at` exactly (`2026-10-09T22:21:51Z`). Its
`ship_date` and `estimated_delivery_date` are naive and date-only (midnight),
so one payload mixes suffixed and unsuffixed dates; a parser that requires an
offset rejects two of the three. In the September fixtures all three are
`Z`-suffixed (pre-transit: naive).

The October `ship_date` (`2026-10-02`) is the label date, not the date UPS
took the parcel; the first physical scan is three days later (`Drop-Off`,
`2026-10-05`). In the September fixtures `ship_date` equals the first scan's
timestamp.

## Sanitization

Tracking numbers are masked mid-string, preserving the 18-character UPS `1Z`
length (`1ZXXXH4303XXXXX632` in the October capture, `1ZXXXG2702XXXXX791` in
September), and the long `wwwapps.ups.com` `tracking_url` is kept in step.
The `se-` label ID in `resource_url` is masked the same way (`se-207301XXX`).

Coordinates are normalized to 2 decimal places. The real October `Delivered`
scan arrived at delivery-address precision (4 decimals), which differed from
the facility coordinates. The delivery ZIP is kept in both shapes.

No names, addresses or phone numbers appear (`signer` is `null` or `""` and
`proof_of_delivery_url` is `null` throughout). See
[`../README.md`](../README.md#pii-in-tracking-payloads).
