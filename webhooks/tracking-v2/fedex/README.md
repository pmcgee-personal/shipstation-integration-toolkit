# FedEx Tracking Webhook (`TRACK_EVENT_V2`)

## Objective

To hold real `TRACK_EVENT_V2` webhook bodies for FedEx parcels at each stage
of the lifecycle, so a receiver can be built and tested without waiting on a
live shipment.

Unlike every other webhook in this repo, tracking delivers its data **inline
in the POST body** — there is no follow-up call to make. See
[`../README.md`](../README.md) for why, and for the fields shared by all
three carriers.

All fixtures here were captured with `carrier_code` `fedex_walleted` and
`carrier_id` `194`.

## Scenarios

| Fixture                                                                                      | `status_code` | `status_detail_code` | Events | What it shows                                                                                               |
| -------------------------------------------------------------------------------------------- | ------------- | -------------------- | ------ | ----------------------------------------------------------------------------------------------------------- |
| [`track-event-v2-in-transit-single-scan.json`](./track-event-v2-in-transit-single-scan.json) | `IT`          | `IN_TRANSIT`         | 3      | Early movement — label created, picked up, arrived at the origin hub. Carries an `estimated_delivery_date`. |
| [`track-event-v2-multi-scan.json`](./track-event-v2-multi-scan.json)                         | `IT`          | `IN_TRANSIT`         | 13     | Mid-transit with a `EX` **delivery exception** already in the history. Top-level status is still `IT`.      |
| [`track-event-v2-out-for-delivery.json`](./track-event-v2-out-for-delivery.json)             | `IT`          | `OUT_FOR_DELIVERY`   | 15     | On the vehicle. Same parcel as the multi-scan fixture, two scans later.                                     |
| [`track-event-v2-delivered.json`](./track-event-v2-delivered.json)                           | `DE`          | `DELIVERED`          | 8      | Full lifecycle to delivery. `actual_delivery_date` is set.                                                  |
| [`track-event-v2-delivery-exception.json`](./track-event-v2-delivery-exception.json)         | —             | —                    | —      | **Placeholder** — awaiting a capture where `EX` is the _top-level_ status, not just a historical scan.      |

## Two parcels, four captures

These five files cover **two** shipments, not five:

| Parcel         | Label          | Fixtures                           |
| -------------- | -------------- | ---------------------------------- |
| `383369XXX473` | `se-454868XXX` | in-transit-single-scan → delivered |
| `383144XXX630` | `se-452094XXX` | multi-scan → out-for-delivery      |

Each pair is the same tracking number captured at two points in time, so
diffing them shows exactly what a receiver sees between two deliveries of
the same event. The `events` array is **cumulative** — every delivery
re-sends the full accumulated history, newest first, not just the scan that
fired the webhook. De-duplicate on `(tracking_number, occurred_at,
event_code)` rather than assuming one webhook means one new scan.

## `IT` can hide an exception

[`track-event-v2-multi-scan.json`](./track-event-v2-multi-scan.json) is the
one to read carefully. Its top-level `status_code` is `IT` / `IN_TRANSIT`,
but `events[1]` is a `Delivery exception` scan with `status_code` `EX`:

```json
{
  "occurred_at": "2026-08-17T09:00:23Z",
  "description": "Delivery exception",
  "status_code": "EX",
  "city_locality": "CADILLAC",
  "state_province": "MI"
}
```

The parcel recovered and went on to be delivered. A receiver that only reads
the top-level `status_code` will never see the exception; one that scans the
`events` array for `EX` and alerts on it will cry wolf on a parcel that is
fine. Neither is wrong, but pick deliberately — and note that top-level
`exception_description` is `null` in every fixture here, including this one,
so it is not a reliable exception signal for FedEx.

## FedEx quirks

**`carrier_occurred_at` equals `occurred_at`.** In all four FedEx captures
the two timestamps are byte-identical and both `Z`-suffixed. FedEx is the
only carrier here where that holds — UPS and USPS put carrier-_local_ time
in that field. Don't write shared parsing logic that infers a rule from the
FedEx fixtures; see [`../README.md`](../README.md#carrier_occurred_at-is-not-utc).

**Dates are naive.** `ship_date`, `estimated_delivery_date` and
`actual_delivery_date` carry no timezone suffix at all
(`"2026-08-24T17:00:00"`), where UPS and USPS `Z`-suffix the same fields. A
strict ISO-8601 parser configured to require an offset will reject these.

**`actual_delivery_date` disagrees with the Delivered scan.** In
[`track-event-v2-delivered.json`](./track-event-v2-delivered.json):

| Field                                 | Value                  |
| ------------------------------------- | ---------------------- |
| `data.actual_delivery_date`           | `2026-08-28T05:24:32`  |
| `events[0].occurred_at` (`Delivered`) | `2026-08-28T19:24:32Z` |

Same seconds, 14 hours apart. Whatever the cause, the two are not
interchangeable — if you display a delivery time, take it from the
`Delivered` event and note which one you chose. UPS does not show this gap
(both read `2026-09-03T16:18:41Z`).

**`postal_code` is always `null`.** FedEx populates `city_locality` and
`state_province` but never `postal_code` in any captured event, where USPS
and UPS sometimes do. Don't key a geo lookup off it.

**`ship_date` precedes the first scan.** In the delivered fixture
`ship_date` is `2026-08-24T17:00:00` but the earliest event
(`Shipment information sent to FedEx`) is `2026-08-25T17:14:00Z`. `ship_date`
is the intended ship date from the label, not an observed scan.

## Sanitization

Tracking numbers are masked mid-string with `XXX`, preserving the 12-digit
FedEx length (`383369XXX473`), and `tracking_url` is kept in step. The `se-`
label ID in `resource_url` is masked the same way.

No names, signers, addresses, phone
numbers or `proof_of_delivery_url` values appear in any fixture here
(`signer` and `proof_of_delivery_url` are `null` throughout). See
[`../README.md`](../README.md#pii-in-tracking-payloads) before adding a
capture with either populated.

[`track-event-v2-out-for-delivery.json`](./track-event-v2-out-for-delivery.json)
is missing the `_captured_at` key its three siblings carry — see
[`../../../TODO.md`](../../../TODO.md).
