# USPS Tracking Webhook (`TRACK_EVENT_V2`)

## Objective

To hold `TRACK_EVENT_V2` webhook bodies for USPS parcels at each stage of the
lifecycle, so a receiver can be built and tested without waiting on a live
shipment.

Unlike every other webhook in this repo, tracking delivers its data **inline
in the POST body** — there is no follow-up call to make. See
[`../README.md`](../README.md) for why, and for the fields shared by all
three carriers.

All fixtures here were captured with `carrier_code` `stamps_com` — the
carrier code USPS shipments report under — and `carrier_id` `1`.

## Two shapes in this folder

USPS payloads changed between the September 2026 captures and the
2026-10-09 capture. The folder currently holds both:

| Fixture                                                                              | Shape     | `status_code` | `status_detail_code`          | Events | What it shows                                                                                                                |
| ------------------------------------------------------------------------------------ | --------- | ------------- | ----------------------------- | ------ | ---------------------------------------------------------------------------------------------------------------------------- |
| [`track-event-v2-pre-transit.json`](./track-event-v2-pre-transit.json)               | September | `AC`          | `ELEC_ADVICE_RECD_BY_CARRIER` | 1      | Label created, USPS awaiting the item. Every date field is `null`.                                                           |
| [`track-event-v2-multi-scan.json`](./track-event-v2-multi-scan.json)                 | October   | `IT`          | `HUB_SCAN_OUT`                | 12     | Mid-transit across four facilities. Has an `estimated_delivery_date`, three location-less scans and a duplicated acceptance. |
| [`track-event-v2-out-for-delivery.json`](./track-event-v2-out-for-delivery.json)     | September | `IT`          | `OUT_FOR_DELIVERY`            | 25     | Out for delivery. The richest fixture, and the one with the two-format quirks described below.                               |
| [`track-event-v2-delivered.json`](./track-event-v2-delivered.json)                   | September | `DE`          | `DELIVERED`                   | 17     | Full lifecycle to delivery (`DELIVERED IN/AT MAILBOX`). Carries a **duplicate** delivery scan.                               |
| [`track-event-v2-delivery-exception.json`](./track-event-v2-delivery-exception.json) | —         | —             | —                             | —      | **Placeholder** — awaiting a real capture.                                                                                   |

Each fixture is a different parcel. The `events` array is **cumulative** —
every delivery re-sends the full accumulated history, newest first.

All five files except the placeholder are real, as-received captures.
`track-event-v2-multi-scan.json` is the only one in the October shape; the
other three are September captures from before the webhook and mapping
changes.

The change follows deliberate changes to the webhook and its field mapping,
so the October shape is the current one. Until `pre-transit`,
`out-for-delivery` and `delivered` are recaptured, treat those three as the
older shape — a receiver should be tested against the October file, and
tolerate the older shape only if it still has September-era bodies to
replay.

## What changed: null before, populated now

Field population, September captures versus 2026-10-09:

| Field                                  | September                                                               | 2026-10-09                                                                      |
| -------------------------------------- | ----------------------------------------------------------------------- | ------------------------------------------------------------------------------- |
| `country_code`                         | `null` on about half of scans (11 of 12 in multi-scan)                  | `US` on **every** scan                                                          |
| `postal_code`                          | `null` on the Title-Case scans                                          | Populated on every scan that has a location                                     |
| `state_province`                       | `null` on distribution-centre scans (state folded into `city_locality`) | Populated on every scan that has a location                                     |
| `company_name`                         | Facility name, but only on the Title-Case scans                         | Facility name on the two Birmingham distribution-centre scans, `null` elsewhere |
| `estimated_delivery_date`              | `null` through twelve in-transit scans                                  | `2026-10-13T00:00:00Z` while in transit                                         |
| Top-level `carrier_status_description` | Equal to the scan description (long sentence on pre-transit only)       | A full sentence naming facility, ZIP and local time, even in transit            |
| Event `carrier_status_description`     | Varies (`PROCESSED THROUGH USPS FACILITY`, `Shipping Label Created, …`) | Always identical to `description`                                               |
| `latitude` / `longitude`               | 2 decimals                                                              | 4 decimals in the raw body (normalized to 2 here)                               |

And the reverse — **populated before, `null` now**:

| Field                                                                                                             | September                  | 2026-10-09                                      |
| ----------------------------------------------------------------------------------------------------------------- | -------------------------- | ----------------------------------------------- |
| `city_locality`, `state_province`, `postal_code`, `latitude`, `longitude` on `In Transit to Next Facility` (`TL`) | Populated (a town and ZIP) | **All five `null`** — three of the twelve scans |
| `status_detail_code`, `status_detail_description` on `Accepted at USPS Facility` (`OB`)                           | Populated on every scan    | `null` on both `OB` scans (`status_code` `UN`)  |

Still `null` throughout: `carrier_detail_code`, `signer`,
`proof_of_delivery_url`, `exception_description` and (until delivery)
`actual_delivery_date`.

Two consequences for receivers:

- **`country_code` and `postal_code` can now be relied on for located scans,
  but not for `TL` scans.** `if (event.postal_code)` is the right test; a
  scan with no location has `city_locality`, `state_province`, `postal_code`,
  `latitude` and `longitude` all `null` at once, while `country_code` is still
  `US`.
- **`status_detail_code` is not always set.** A receiver that assumes it is
  populated on every USPS scan will encounter `null` on `OB`.

The facility name is no longer carried inside `city_locality`: the
September `"NORTH HOUSTON TX DISTRIBUTION C"` style is gone, and
`city_locality` is a city name only (`BIRMINGHAM`), with the facility, when
present, in `company_name`.

## Duplicate scans

Both shapes repeat themselves. The October multi-scan opens its acceptance
with two `Accepted at USPS Facility` scans 13 minutes apart, identical in
every other field:

| Index | `occurred_at`          | `event_code` | Description                 |
| ----- | ---------------------- | ------------ | --------------------------- |
| 9     | `2026-10-06T03:26:26Z` | `OB`         | `Accepted at USPS Facility` |
| 10    | `2026-10-06T03:13:22Z` | `OB`         | `Accepted at USPS Facility` |

The September
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
legitimately repeated arrival scans (`A1` appears at several different
facilities in the September fixtures, and twice at different facilities in
the October multi-scan). If a delivery notification must fire exactly
once, gate on the transition into `DE` rather than on the event rows.

## Events are sorted by `occurred_at` in the October shape

The October multi-scan is strictly descending by `occurred_at`, and
`carrier_occurred_at` equals `occurred_at` on every scan. All three `TL`
scans fall in sequence rather than ahead of the scan that follows them.

That cannot be fully trusted yet. The September problem was specific to `TL`
and `NT` scans, and the October `TL` scans have no location, so there is no
local clock to compare against. The one cross-check available is the
top-level text: `departed … on October 9, 2026 at 4:35 pm` at a Birmingham
(Central) facility matches `2026-10-09T21:35:23Z` exactly, so `occurred_at` is
genuine UTC for that scan. **Sort by `occurred_at` yourself regardless**, as
the September shape below does not.

## September shape: two event formats in one `events` array

This section applies to
[`track-event-v2-out-for-delivery.json`](./track-event-v2-out-for-delivery.json)
and [`track-event-v2-delivered.json`](./track-event-v2-delivered.json). The
October multi-scan does not show it: its descriptions are consistently
mixed-case and its `city_locality` is always a plain city.

The 25 events in out-for-delivery split cleanly into two groups with
different conventions:

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
5–16 Title-Case). Normalize `description` case and check both
`city_locality` and `company_name` for the facility.

The Title-Case group is not simply poorer data — in
`track-event-v2-delivered.json` those events carry a full `status_detail_code`
(`HUB_SCAN_IN`, `HUB_SCAN_OUT`, `RECEIVED_BY_CARRIER`) while still leaving
`postal_code` `null`. Which fields are populated depends on the group, not
on how far along the parcel is.

## September shape: events are not sorted by `occurred_at`

In the September captures the array is descending by `carrier_occurred_at`,
**not** by `occurred_at`. Because different events are converted to UTC with
different offsets, the two orderings disagree — in two of the three
multi-event September fixtures:

| Fixture          | Index | `occurred_at`          | `carrier_occurred_at`  | Description                                   |
| ---------------- | ----- | ---------------------- | ---------------------- | --------------------------------------------- |
| out-for-delivery | 14    | `2026-08-31T06:39:00Z` | `2026-08-31T00:39:00Z` | Arrived at USPS Regional Facility             |
| out-for-delivery | 15    | `2026-08-31T07:00:00Z` | `2026-08-31T00:00:00Z` | In Transit to Next Facility, Arriving On Time |
| delivered        | 7     | `2026-09-02T01:41:00Z` | `2026-09-01T21:41:00Z` | Arrived at USPS Regional Facility             |
| delivered        | 8     | `2026-09-02T01:46:00Z` | `2026-09-01T18:46:00Z` | In Transit to Next Facility                   |

In both cases the offending row is an **`In Transit to Next Facility`** scan
(`event_code` `TL` or `NT`). Grouping every September event by its implied
offset makes the pattern exact: the 7-hour group contains _nothing but_ `TL`
and `NT` scans, while every other scan type in the same file converts at 4, 5
or 6 hours. In the delivered fixture — a Texas-to-New-Jersey parcel that never
went near the Pacific — a 7-hour offset is not a plausible local time at all.

So these events were converted with a fixed offset that ignores where the
parcel actually was, which drags their `occurred_at` forward past the scan
that follows them. **Sort by `occurred_at` yourself; do not trust array
position** — and be aware that even `occurred_at` is wrong by a couple of
hours on `TL`/`NT` rows specifically.

## `EX` appears on a parcel that is on time

The September out-for-delivery event 15 above has `status_code` `EX` while
its own description reads _"Arriving On Time"_, and the parcel went out for
delivery normally. The polled API shows the same pairing — see
[`api/tracking/usps/delivery-exception.json`](../../../api/tracking/usps/README.md).
An `EX` scan in USPS history is not by itself a problem worth alerting a
customer about; check `status_detail_code` (`CARRIER_DELAYS` vs
`RETURN_TO_SENDER`) before deciding.

Note also that `UN` (Unknown) appears as an event `status_code` on ordinary
scans in both shapes — `IN TRANSIT TO NEXT FACILITY` in September, and
`Accepted at USPS Facility` in October, where it comes with a `null`
`status_detail_code`. Handle unrecognised codes by passing them through, not
by dropping the event.

## Timestamps

**October shape:** `carrier_occurred_at` is `Z`-suffixed and **equal to
`occurred_at`** on every scan, as it is for FedEx. It is real UTC, not local
time. See [`../README.md`](../README.md#carrier_occurred_at-is-not-utc).

**September shape:** `carrier_occurred_at` is `Z`-suffixed but is not UTC in
the out-for-delivery and delivered fixtures — it is USPS local scan time with
a UTC marker glued on, and the implied offset varies per event within a
single file (5, 6 and 7 hours in out-for-delivery; 4, 5 and 7 in delivered).
[`track-event-v2-pre-transit.json`](./track-event-v2-pre-transit.json)
carries a genuine offset (`"2026-08-19T15:44:46-07:00"`), the only fixture
here with a real UTC offset in this field.

Parse `occurred_at` in every case, and use `carrier_occurred_at` for display
only.

`ship_date` and `estimated_delivery_date` are `Z`-suffixed here, where FedEx
leaves them naive. In the October multi-scan `ship_date` equals the
`occurred_at` of the first `Arrived at USPS Facility` scan (`event_code`
`10`) to the second — not the two earlier `Accepted` scans — and
`estimated_delivery_date` is set at midnight. In September, USPS often left
`estimated_delivery_date` `null` well into transit.

## Sanitization

Tracking numbers are masked mid-string with `XXXXX`, preserving the 22-digit
USPS length (`940015020621XXXXX26585`), and `tracking_url` is kept in step.
The `se-` label ID in `resource_url` is masked the same way
(`se-207416XXX`) — though `se-97410XXXX` in the pre-transit fixture uses a
5-character mask where the rest of the tree uses 3, so its original length is
not recoverable.

Coordinates are normalized to 2 decimal places; the October body arrives at 4.
The top-level `carrier_status_description` in the October multi-scan names a
USPS facility, its ZIP and a local scan time. These are public facility
details and are kept.

`company_name` is only ever populated with USPS facility names
(`BIRMINGHAM AL DISTRIBUTION CENTER`, `TETERBORO NJ DISTRIBUTION CENTER`) —
no customer or business recipients. No names, signers, addresses or phone
numbers appear anywhere in this folder; `signer` and `proof_of_delivery_url`
are `null` across every captured event, including both September delivery
scans. See [`../README.md`](../README.md#pii-in-tracking-payloads).
