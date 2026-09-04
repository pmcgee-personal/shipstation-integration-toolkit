# USPS Tracking Webhook (`TRACK_EVENT_V2`)

## Objective

To hold real `TRACK_EVENT_V2` webhook bodies for USPS parcels at each stage
of the lifecycle, so a receiver can be built and tested without waiting on a
live shipment.

Unlike every other webhook in this repo, tracking delivers its data **inline
in the POST body** — there is no follow-up call to make. See
[`../README.md`](../README.md) for why, and for the fields shared by all
three carriers.

All fixtures here were captured with `carrier_code` `stamps_com` — the
carrier code USPS shipments report under — and `carrier_id` `1`.

## Scenarios

| Fixture                                                                              | `status_code` | `status_detail_code`          | Events | What it shows                                                                                   |
| ------------------------------------------------------------------------------------ | ------------- | ----------------------------- | ------ | ----------------------------------------------------------------------------------------------- |
| [`track-event-v2-pre-transit.json`](./track-event-v2-pre-transit.json)               | `AC`          | `ELEC_ADVICE_RECD_BY_CARRIER` | 1      | Label created, USPS awaiting the item. Every date field is `null`.                              |
| [`track-event-v2-multi-scan.json`](./track-event-v2-multi-scan.json)                 | `IT`          | `IN_TRANSIT`                  | 12     | Mid-transit across facilities. No `estimated_delivery_date` even this far along.                |
| [`track-event-v2-out-for-delivery.json`](./track-event-v2-out-for-delivery.json)     | `IT`          | `OUT_FOR_DELIVERY`            | 25     | Out for delivery. The richest fixture here, and the one with the format quirks described below. |
| [`track-event-v2-delivered.json`](./track-event-v2-delivered.json)                   | `DE`          | `DELIVERED`                   | 17     | Full lifecycle to delivery (`DELIVERED IN/AT MAILBOX`). Carries a **duplicate** delivery scan.  |
| [`track-event-v2-delivery-exception.json`](./track-event-v2-delivery-exception.json) | —             | —                             | —      | **Placeholder** — awaiting a real capture.                                                      |

Each fixture is a different parcel. The `events` array is **cumulative** —
every delivery re-sends the full accumulated history, newest first.

## The delivered fixture repeats itself

[`track-event-v2-delivered.json`](./track-event-v2-delivered.json) opens with
_two_ delivery scans one minute apart, identical in every other field:

| Index | `occurred_at`          | `event_code` | Description               |
| ----- | ---------------------- | ------------ | ------------------------- |
| 0     | `2026-09-03T13:52:00Z` | `01`         | `DELIVERED IN/AT MAILBOX` |
| 1     | `2026-09-03T13:51:00Z` | `01`         | `DELIVERED IN/AT MAILBOX` |

Same locality, same ZIP, same coordinates, same codes. This matters for
whichever de-duplication key you pick: `(tracking_number, occurred_at,
event_code)` keeps **both** — correct, since they really are two distinct
scans in the payload, but it means "parcel delivered" fires twice. Keying on
`(tracking_number, event_code)` collapses them but would also collapse the
legitimately repeated `Arrived at USPS Regional Facility` scans (`A1`
appears four times at four different facilities). If a delivery notification
must be sent exactly once, gate on the transition into `DE` rather than on
the event rows.

## Two event formats in one `events` array

This is the thing to know about USPS here, and
[`track-event-v2-out-for-delivery.json`](./track-event-v2-out-for-delivery.json)
shows it plainly. Its 25 events split cleanly into two groups with different
conventions:

|                  | Events 0–13 (newer)                                   | Events 14–24 (older)                                   |
| ---------------- | ----------------------------------------------------- | ------------------------------------------------------ |
| `description`    | `ALL CAPS` — `PROCESSED THROUGH USPS FACILITY`        | `Title Case` — `Arrived at USPS Regional Facility`     |
| Facility name in | `city_locality` — `"BILLINGS MT DISTRIBUTION CENTER"` | `company_name`, with `city_locality` just `"BILLINGS"` |
| `postal_code`    | Populated                                             | `null`                                                 |
| `country_code`   | `null`                                                | `US`                                                   |

So the same logical scan type appears twice in one array under two different
shapes. Anything that groups by `description`, or that reads the facility
from `city_locality`, will treat the two halves as unrelated. The switch
happens mid-history, so you cannot pick one convention per file either.

`track-event-v2-delivered.json` shows the same split (events 0–4 ALL-CAPS,
5–16 Title-Case). `track-event-v2-multi-scan.json` is entirely in the
ALL-CAPS group; `track-event-v2-pre-transit.json` is entirely in the
Title-Case group. Normalize `description` case and check both
`city_locality` and `company_name` for the facility.

The Title-Case group is not simply poorer data — in
`track-event-v2-delivered.json` those events carry a full `status_detail_code`
(`HUB_SCAN_IN`, `HUB_SCAN_OUT`, `RECEIVED_BY_CARRIER`) while still leaving
`postal_code` `null`. Which fields are populated depends on the group, not
on how far along the parcel is.

## Events are not sorted by `occurred_at`

The array is descending by `carrier_occurred_at`, **not** by `occurred_at`.
Because different events are converted to UTC with different offsets, the
two orderings disagree — in two of the three multi-event fixtures here:

| Fixture          | Index | `occurred_at`          | `carrier_occurred_at`  | Description                                   |
| ---------------- | ----- | ---------------------- | ---------------------- | --------------------------------------------- |
| out-for-delivery | 14    | `2026-08-31T06:39:00Z` | `2026-08-31T00:39:00Z` | Arrived at USPS Regional Facility             |
| out-for-delivery | 15    | `2026-08-31T07:00:00Z` | `2026-08-31T00:00:00Z` | In Transit to Next Facility, Arriving On Time |
| delivered        | 7     | `2026-09-02T01:41:00Z` | `2026-09-01T21:41:00Z` | Arrived at USPS Regional Facility             |
| delivered        | 8     | `2026-09-02T01:46:00Z` | `2026-09-01T18:46:00Z` | In Transit to Next Facility                   |

In both cases the offending row is an **`In Transit to Next Facility`** scan
(`event_code` `TL` or `NT`). Grouping every USPS event in this folder by its
implied offset makes the pattern exact: the 7-hour group contains _nothing
but_ `TL` and `NT` scans, while every other scan type in the same file
converts at 4, 5 or 6 hours. In the delivered fixture — a Texas-to-New-Jersey
parcel that never went near the Pacific — a 7-hour offset is not a plausible
local time at all.

So these events are converted with a fixed offset that ignores where the
parcel actually was, which drags their `occurred_at` forward past the scan
that follows them. **Sort by `occurred_at` yourself; do not trust array
position** — and be aware that even `occurred_at` is wrong by a couple of
hours on `TL`/`NT` rows specifically.

## `EX` appears on a parcel that is on time

The out-for-delivery event 15 above has `status_code` `EX` while its own
description reads _"Arriving On Time"_, and the parcel went out for
delivery normally. The
polled API shows the same pairing — see
[`api/tracking/usps/delivery-exception.json`](../../../api/tracking/usps/README.md).
An `EX` scan in USPS history is not by itself a problem worth alerting a
customer about; check `status_detail_code` (`CARRIER_DELAYS` vs
`RETURN_TO_SENDER`) before deciding.

Note also that `UN` (Unknown) appears as an event `status_code` six times
across these fixtures, on ordinary `IN TRANSIT TO NEXT FACILITY` scans.
Handle unrecognised codes by passing them through, not by dropping the event.

## Timestamps

**`carrier_occurred_at` is `Z`-suffixed but is not UTC** in the multi-scan,
out-for-delivery and delivered fixtures — it is USPS local scan time with a
UTC marker glued on, and the implied offset varies per event within a single
file (5, 6 and 7 hours in out-for-delivery; 4, 5 and 7 in delivered). Parse
`occurred_at` instead.

[`track-event-v2-pre-transit.json`](./track-event-v2-pre-transit.json) is
the exception: it carries a genuine offset (`"2026-08-19T15:44:46-07:00"`),
which is the only correct encoding of this field anywhere in the folder. See
[`../README.md`](../README.md#carrier_occurred_at-is-not-utc).

`ship_date` and `estimated_delivery_date` are `Z`-suffixed here, where FedEx
leaves them naive. USPS often leaves `estimated_delivery_date` `null` well
into transit — `track-event-v2-multi-scan.json` has twelve scans and still
no ETA.

## Sanitization

Tracking numbers are masked mid-string with `XXXXX`, preserving the 22-digit
USPS length (`94346362083XXXXX009034`), and `tracking_url` is kept in step.
The `se-` label ID in `resource_url` is masked the same way — though
`se-97410XXXX` in the pre-transit fixture uses a 5-character mask where the
rest of the tree uses 3, so its original length is not recoverable.

`company_name` is populated in out-for-delivery and delivered, but only ever
with USPS facility names (`TETERBORO NJ DISTRIBUTION CENTER`) — no customer
or business recipients. No names, signers, addresses or phone numbers appear
anywhere in this folder; `signer` and `proof_of_delivery_url` are `null`
across all 55 captured events, including both delivery scans. See
[`../README.md`](../README.md#pii-in-tracking-payloads).
