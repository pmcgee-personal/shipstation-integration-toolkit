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

| Fixture                                                                              | `status_code` | `carrier_detail_code` | Events | What it shows                                                                                               |
| ------------------------------------------------------------------------------------ | ------------- | --------------------- | ------ | ----------------------------------------------------------------------------------------------------------- |
| [`track-event-v2-pre-transit.json`](./track-event-v2-pre-transit.json)               | `NY`          | `MP`                  | 1      | Label created, UPS has not taken possession. `ship_date` is `null` but an `estimated_delivery_date` exists. |
| [`track-event-v2-multi-scan.json`](./track-event-v2-multi-scan.json)                 | `IT`          | `OD`                  | 10     | Mid-transit, loaded on the delivery vehicle. Ten scans accumulated.                                         |
| [`track-event-v2-out-for-delivery.json`](./track-event-v2-out-for-delivery.json)     | `IT`          | `OT`                  | 13     | Out for delivery. Note four near-identical `Preparing for Delivery` scans in a row.                         |
| [`track-event-v2-delivered.json`](./track-event-v2-delivered.json)                   | `DE`          | `FS`                  | 16     | Full lifecycle to delivery. `actual_delivery_date` matches the `Delivered` scan exactly.                    |
| [`track-event-v2-delivery-exception.json`](./track-event-v2-delivery-exception.json) | —             | —                     | —      | **Placeholder** — awaiting a real capture.                                                                  |

Each fixture is a different parcel (unlike the FedEx folder, where two
shipments are captured twice each). The `events` array is **cumulative** —
every delivery re-sends the full accumulated history, newest first.

## UPS uses `carrier_detail_code`, not `status_detail_code`

This is the biggest difference from the other two carriers, and the one most
likely to break shared code:

| Field                       | UPS                                   | FedEx / USPS                             |
| --------------------------- | ------------------------------------- | ---------------------------------------- |
| `status_detail_code`        | `null` — except on pre-transit        | Populated (`IN_TRANSIT`, `DELIVERED`, …) |
| `status_detail_description` | `null` — except on pre-transit        | Populated                                |
| `carrier_detail_code`       | Populated (`FS`, `OD`, `OT`, `MP`, …) | `null` everywhere                        |

So a receiver that branches on `status_detail_code` to tell _out for
delivery_ from _in transit_ — which works fine for FedEx and USPS — gets
`null` on UPS for both, and has to fall back to `carrier_detail_code` (`OT`
vs `OD`) or to `carrier_status_description`.

The single exception is
[`track-event-v2-pre-transit.json`](./track-event-v2-pre-transit.json),
where `status_detail_code` **is** `ELEC_ADVICE_RECD_BY_CARRIER` — the same
value USPS uses at that stage. So the field is not simply unsupported for
UPS; it is populated at label creation and then goes `null`. Don't infer
"UPS never sets this" from one fixture.

Event-level `carrier_detail_code` values seen across these captures: `AR`,
`DP`, `FS`, `OD`, `OR`, `OT`, `XD`, `YP`, `MP`. These are UPS's own codes
passed through — useful, but not a stable contract.

## Empty strings where you would expect `null`

UPS mixes `null` and `""` for absent values, and they are not
interchangeable:

- `postal_code` is `""` (empty string) on every event **except** the
  `Delivered` scan, which carries the real ZIP (`48067`).
- `signer` is `""` in
  [`track-event-v2-pre-transit.json`](./track-event-v2-pre-transit.json) but
  `null` in all other fixtures.

`if (event.postal_code)` and `if (event.postal_code !== null)` therefore
disagree on UPS data. Treat empty string and `null` as the same "absent"
case explicitly.

## Timestamps

**`carrier_occurred_at` is `Z`-suffixed but is not UTC.** It is UPS's local
scan time with a UTC marker glued on. Comparing it against `occurred_at`
across these fixtures yields offsets of 4, 5 **and** 7 hours _within a
single file_ — the parcel is crossing time zones and each scan carries its
own local clock. Parse `occurred_at` and ignore `carrier_occurred_at` for
anything but display. See
[`../README.md`](../README.md#carrier_occurred_at-is-not-utc).

[`track-event-v2-pre-transit.json`](./track-event-v2-pre-transit.json) is
the odd one out again: its `carrier_occurred_at` has **no** suffix at all
(`"2026-09-03T11:19:14"`), where the other three UPS fixtures use `Z`. Three
encodings of one field across four files.

`ship_date`, `estimated_delivery_date` and `actual_delivery_date` are
`Z`-suffixed here, where FedEx leaves them naive.

## Sanitization

Tracking numbers are masked mid-string with `XXX`, preserving the 18-character
UPS `1Z` length (`1ZXXXG2702XXXXX791`), and the long `wwwapps.ups.com`
`tracking_url` is kept in step. The `se-` label ID in `resource_url` is
masked the same way.

No names,signers, addresses or phone numbers appear (`signer` is `null`/`""`
throughout, `proof_of_delivery_url` is `null` throughout). See
[`../README.md`](../README.md#pii-in-tracking-payloads).

[`track-event-v2-multi-scan.json`](./track-event-v2-multi-scan.json) is
missing the `_captured_at` key its three siblings carry — see
[`../../../TODO.md`](../../../TODO.md).
