# UPS Tracking Webhook (`TRACK_EVENT_V2`)

## Objective

To hold `TRACK_EVENT_V2` webhook bodies for UPS parcels at each stage of the
lifecycle, so a receiver can be built and tested without waiting on a live
shipment.

Unlike every other webhook in this repo, tracking delivers its data **inline
in the POST body** — there is no follow-up call to make. See
[`../README.md`](../README.md) for why, and for the fields shared by all
three carriers.

All fixtures here use `carrier_code` `ups` and `carrier_id` `3`.

## Scenarios

| Fixture                                                                              | `status_code` | `status_detail_code`          | `carrier_detail_code` | Events | What it shows                                                                                                    |
| ------------------------------------------------------------------------------------ | ------------- | ----------------------------- | --------------------- | ------ | ---------------------------------------------------------------------------------------------------------------- |
| [`track-event-v2-pre-transit.json`](./track-event-v2-pre-transit.json)               | `NY`          | `ELEC_ADVICE_RECD_BY_CARRIER` | `MP`                  | 1      | Label created, UPS has not taken possession. `ship_date` is `null`.                                              |
| [`track-event-v2-multi-scan.json`](./track-event-v2-multi-scan.json)                 | `IT`          | `DELIVERY_SCHEDULED`          | `YP`                  | 13     | Across the country by hub, now processing at the destination facility. In transit, **not** yet out for delivery. |
| [`track-event-v2-out-for-delivery.json`](./track-event-v2-out-for-delivery.json)     | `IT`          | `OUT_FOR_DELIVERY`            | `OT`                  | 15     | Out for delivery. Two scans carry `OUT_FOR_DELIVERY`; this is the second.                                        |
| [`track-event-v2-delivered.json`](./track-event-v2-delivered.json)                   | `DE`          | `DELIVERED`                   | `FS`                  | 16     | Full lifecycle to delivery. `actual_delivery_date` matches the `Delivered` scan exactly.                         |
| [`track-event-v2-delivery-exception.json`](./track-event-v2-delivery-exception.json) | —             | —                             | —                     | —      | **Placeholder** — awaiting a real capture.                                                                       |

## One parcel, four snapshots

Unlike the earlier UPS captures (one parcel per file), these four files are
the **same parcel** (`1ZXXXH4303XXXXX632`, label `se-207301XXX`) at four
points in its life, like the FedEx folder's paired captures.

Only [`track-event-v2-delivered.json`](./track-event-v2-delivered.json) is a
real, as-received capture (2026-10-09). The other three are **derived from
it**: because `events` is cumulative and newest-first, the history at an
earlier moment is the delivered history with the newest scans cut off. Each
derived file carries a `_derived_from` key saying so. Strip that key if you
copy a file into a test suite that checks for exact shape.

What the derivation assumes, and has not observed:

- Top-level `status_code`, `status_detail_*`, `carrier_detail_code`,
  `status_description` and `carrier_status_*` mirror `events[0]`. This holds
  on the delivered capture.
- `estimated_delivery_date` is the same value at every stage.
- `ship_date` is the label-creation date (`2026-10-02T00:00:00`) on every
  derived file except pre-transit, where it is `null` — carried from the
  September pre-transit capture, which showed `ship_date: null` at that stage.
- Pre-transit top-level `status_description` is `Accepted` although the event
  says `Not Yet In System`. The September capture showed the same mismatch,
  so it is kept rather than "fixed".

If UPS behaves differently at any of these stages, replace the derived file
with a real capture and delete its `_derived_from` key.

The `events` array is **cumulative** — every delivery re-sends the full
accumulated history, newest first, sorted by `occurred_at`.

## `status_detail_code` is now populated on UPS

The September 2026 captures left `status_detail_code` and
`status_detail_description` `null` on every UPS event except pre-transit, and
told you to branch on `carrier_detail_code`. **The 2026-10-09 capture
populates both on every event, top-level and per scan**, alongside
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

So the FedEx/USPS pattern — branch on `status_detail_code` — now works for
UPS too. But note what the table shows:

- **`OUT_FOR_DELIVERY` fires twice, 68 minutes apart.** `Loaded on Delivery
Vehicle` (`OF`) and `Out For Delivery Today` (`OT`) carry the same
  `status_detail_code`. A receiver that notifies on the first sighting of
  `OUT_FOR_DELIVERY` sends at the _loaded_ scan, not the _out_ scan.
  `DELIVERY_SCHEDULED` (`YP`) comes just before and is still plain in transit.
- **`description` alone cannot classify a scan.** `Arrived at Facility`
  appears for both `RECEIVED_BY_CARRIER` (`OR`) and `HUB_SCAN_IN` (`AR`).
  Use the codes.
- **`status_code` is not monotonic.** Oldest to newest the sequence is `NY`,
  `IT` (`Drop-Off`), `AC` (`RFID Confirmed Pickup`), `IT`, … `DE`. The drop-off
  scan lands _before_ the pickup scan, so a state machine that only moves
  forward will reject the `AC` and stall.

`carrier_detail_code` is still populated and the two fields move together, but
the code vocabulary has shifted (see below), so prefer `status_detail_code`
for new code. Treat both as pass-through values, not a stable contract.

### Earlier shape (September 2026 captures)

Kept here for anyone replaying older bodies. The change follows deliberate
changes to the webhook and its field mapping, so the 2026-10-09 shape is the
current one. The original fixtures are in git history before that update.

|                           | September captures                                       | 2026-10-09 capture                                                        |
| ------------------------- | -------------------------------------------------------- | ------------------------------------------------------------------------- |
| `status_detail_code`      | `null` (except pre-transit)                              | Populated on every scan                                                   |
| Scan wording              | `On the Way`, `We Have Your Package`, `Out for Delivery` | `Arrived at Facility`, `Departed from Facility`, `Out For Delivery Today` |
| Loaded on vehicle         | `carrier_detail_code` `OD`                               | `OF`                                                                      |
| First possession scan     | `OR` — `We Have Your Package`                            | `PC` (`RFID Confirmed Pickup`) then `OR`                                  |
| Drop-off scan             | `Dropped off at The UPS Store by Customer`               | `Drop-Off`                                                                |
| Repeated scans            | Four `Preparing for Delivery` in a row                   | One `Processing at UPS Facility`                                          |
| `carrier_occurred_at`     | `Z`-suffixed, but local time                             | No suffix, genuinely local                                                |
| `signer`                  | `null` (`""` on pre-transit)                             | `""` on every scan                                                        |
| `ship_date`               | First scan timestamp, `Z`-suffixed                       | Label date at midnight, no suffix                                         |
| `estimated_delivery_date` | `Z`-suffixed                                             | No suffix                                                                 |
| `latitude` / `longitude`  | 2 decimals                                               | Full precision (normalized to 2 here)                                     |

A receiver that still has to read older bodies should key on codes and treat
`description` as display text.

## Codes seen

`carrier_detail_code` across the 2026-10-09 capture: `AR`, `DP`, `FS`, `MP`,
`OF`, `OR`, `OT`, `PC`, `XD`, `YP`. The September captures also used `OD`.
`event_code` is `M` (label), `P` (pickup), `I` (everything in transit), `D`
(delivered).

## Empty strings where you would expect `null`

UPS mixes `null` and `""` for absent values, and they are not
interchangeable:

- `postal_code` is `""` on every scan **except** `Delivered`, which carries
  the real ZIP (`06850`).
- `city_locality` and `state_province` are `""` (not `null`) on the
  label-created scan, which has no location.
- `signer` is `""` on every scan, including `Delivered` — delivery here was
  not signed for. `company_name` and `proof_of_delivery_url` are `null`.
- `latitude` and `longitude` are `null` on the label-created scan.

`if (event.postal_code)` and `if (event.postal_code !== null)` disagree on UPS
data. Treat empty string and `null` as the same "absent" case explicitly.

## Timestamps

**`carrier_occurred_at` is now genuine carrier-local time, with no suffix.**
Compared with `occurred_at` across the delivered capture, the offset is
exactly 7 hours for scans in Washington and Oregon (Pacific daylight time)
and exactly 4 hours for scans in Massachusetts and Connecticut (Eastern
daylight time). Nothing is glued on, so a parser will treat it as a naive
local timestamp rather than silently mistaking it for UTC.

This differs from the September behaviour (`Z` on local time, see
[`../README.md`](../README.md#carrier_occurred_at-is-not-utc)), but you still
cannot recover an instant from it without knowing the scan's time zone. Parse
`occurred_at` and use `carrier_occurred_at` for display only.

`actual_delivery_date` is `Z`-suffixed and equals the `Delivered` scan's
`occurred_at` exactly (`2026-10-09T22:21:51Z`). `ship_date` and
`estimated_delivery_date` are naive and date-only (midnight). So one payload
mixes suffixed and unsuffixed dates; a parser that requires an offset will
reject two of the three.

`ship_date` (`2026-10-02`) is the label date, not the date UPS took the
parcel. The first physical scan is three days later (`Drop-Off`,
`2026-10-05`). Don't use it as a pickup time.

## Sanitization

The tracking number is masked mid-string, preserving the 18-character UPS
`1Z` length (`1ZXXXH4303XXXXX632`), and the long `wwwapps.ups.com`
`tracking_url` is kept in step. The `se-` label ID in `resource_url` is
masked the same way (`se-207301XXX`).

Coordinates are normalized to 2 decimal places. The real `Delivered` scan
carried delivery-address precision (4 decimals) that differed from the
facility coordinates, which is the reason for the rule. The delivery ZIP
(`06850`) is kept, as the Royal Oak ZIP was in the earlier fixture.

No names, signers, addresses or phone numbers appear (`signer` is `""` and
`proof_of_delivery_url` is `null` throughout). See
[`../README.md`](../README.md#pii-in-tracking-payloads).
