# Freight Tracking Webhook (`FREIGHT_TRACK_EVENT_V2`)

## Objective

To hold real examples of the ShipStation freight (LTL) tracking webhook — the
event that fires when the carrier reports a new status for a freight shipment
booked through ShipStation API — at each stage of the lifecycle, so a receiver
can be built and tested without waiting on a live shipment.

This is the freight sibling of the parcel [`TRACK_EVENT_V2`](../README.md)
webhook, and it lives here because it shares the same shape of idea: tracking
data delivered **inline in the POST body**, with nothing to call back for. It is
a **different event with a different payload**, so don't reuse a parcel parser
or copy a fixture between the two.

The practical reason to subscribe is the **PRO number**. A booking response may
not contain one (see [PRO number timing](#pro-number-timing)); the webhook
delivers it when the carrier reports it, so you don't have to poll
`GET /freight/shipments/{freight_shipment_id}/tracking`. See
[`patterns/freight/pro-number-retrieval-tracking.md`](../../../patterns/freight/pro-number-retrieval-tracking.md)
for the integration flow.

## Event

|                          |                                                                         |
| ------------------------ | ----------------------------------------------------------------------- |
| Webhook `event`          | `freight_track_event_v2`                                                |
| `resource_type`          | `FREIGHT_TRACK_EVENT_V2`                                                |
| Created with             | `POST /v2/environment/webhooks` (API only)                              |
| `resource_url` points at | `https://api.shipengine.com/v2/freight/shipments/{freight_shipment_id}` |
| Second request needed    | **No** — everything is in `data`                                        |

## Scenarios

| Fixture                                                                                          | `status`     | `tracking_status` | `pro_number` | Events | What it shows                                                                                                                                         |
| ------------------------------------------------------------------------------------------------ | ------------ | ----------------- | ------------ | ------ | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
| [`freight-track-event-v2-booked.json`](./freight-track-event-v2-booked.json)                     | `booked`     | `pending_pickup`  | `null`       | 2      | Right after booking. No PRO, no carrier scans yet. `tracking_url` is the booking provider's link.                                                     |
| [`freight-track-event-v2-pro-assigned.json`](./freight-track-event-v2-pro-assigned.json)         | `in_transit` | `in_transit`      | populated    | 3      | The first carrier scan (pickup) and the PRO arrive **together**, about 16 hours after booking. `tracking_url` switches to the carrier's own link.     |
| [`freight-track-event-v2-in-transit.json`](./freight-track-event-v2-in-transit.json)             | `in_transit` | `in_transit`      | populated    | 5      | Mid-journey, newest scan is an arrival at a terminal.                                                                                                 |
| [`freight-track-event-v2-out-for-delivery.json`](./freight-track-event-v2-out-for-delivery.json) | `in_transit` | `in_transit`      | populated    | 20     | `AJ` "Tendered for Delivery". The full history to that point. Top-level status is still `in_transit`; there is no separate out-for-delivery status.   |
| [`freight-track-event-v2-delivered.json`](./freight-track-event-v2-delivered.json)               | `delivered`  | `delivered`       | populated    | 21     | `D1` "Completed Unloading at Delivery Location". `estimated_delivery_date` is unchanged from booking and three days earlier than the actual delivery. |

## One shipment, five captures

All five files are the **same freight shipment** captured at five points in its
journey, not five shipments: one XPO Logistics (`CNWY`) LTL shipment booked
through the `SPEEDSHIP` freight provider, from booking on 2026-11-10 to
delivery on 2026-11-16 (the delivered webhook arrived on 2026-11-17). Diffing
them shows exactly what a receiver sees between deliveries.

Everything here was observed on **one provider and one carrier**. Other
freight providers and carriers may differ, particularly in when the PRO
appears and in how descriptions and codes are populated.

## The body

```json
{
  "_captured_at": "2026-11-17",
  "resource_url": "https://api.shipengine.com/v2/freight/shipments/se-2XXX",
  "resource_type": "FREIGHT_TRACK_EVENT_V2",
  "data": {
    "event_id": "f5494e55-30e7-e6f6-17b6-919b96e7290e",
    "freight_shipment_id": "se-2XXX",
    "bol_number": "ATF444XXX80",
    "pro_number": "254XXX576",
    "status": "delivered",
    "tracking_status": "delivered",
    "events": ["…newest first"]
  }
}
```

`data` has 17 keys, identical in all five fixtures. `_captured_at` is a repo
convention, not something ShipStation sends — see the
[root README](../../../README.md).

| Field                                                   | Notes                                                                                                                |
| ------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------- |
| `event_id`                                              | Stable for a given update. A redelivery carries the same value. Not time-ordered, so don't sort on it.               |
| `freight_shipment_id`, `shipment_id`, `shipment_number` | ShipStation's freight shipment, the underlying shipment, and your own reference.                                     |
| `freight_provider_account_id`, `freight_provider_code`  | The freight provider connection used to book (`SPEEDSHIP` here).                                                     |
| `bol_number`, `pro_number`, `pickup_number`             | Bill of Lading, the carrier's PRO, and the pickup confirmation. `pro_number` is `null` until the carrier reports it. |
| `carrier_name`, `carrier_scac`                          | The LTL carrier.                                                                                                     |
| `status`, `tracking_status`                             | Two different fields. See [Status values](#status-values).                                                           |
| `tracking_url`                                          | **Changes during the shipment.** See below.                                                                          |
| `estimated_delivery_date`                               | A date. Never updated in this capture.                                                                               |
| `updated_at`                                            | When ShipStation recorded the update. Real UTC.                                                                      |
| `events`                                                | Cumulative history. Each event has four keys: `occurred_at`, `status`, `carrier_code`, `description`.                |

## How this differs from parcel `TRACK_EVENT_V2`

| Behaviour                  | Parcel `TRACK_EVENT_V2`                      | Freight `FREIGHT_TRACK_EVENT_V2`                             |
| -------------------------- | -------------------------------------------- | ------------------------------------------------------------ |
| Identifies the shipment by | `tracking_number`                            | `freight_shipment_id`, `bol_number`, `pro_number`            |
| De-duplicate on            | `(tracking_number, occurred_at, event_code)` | `data.event_id`                                              |
| `resource_url` host        | `api.shipstation.com`                        | `api.shipengine.com`                                         |
| `data` keys / event keys   | 16 / 20                                      | 17 / 4                                                       |
| Scan location fields       | City, state, postal code, coordinates        | **None**                                                     |
| `occurred_at`              | Trustworthy UTC                              | **Not UTC on carrier scans** — see [Timestamps](#timestamps) |
| Final delivery fields      | `actual_delivery_date`, signer               | **None** — take the time from the `delivered` event          |

## PRO number timing

In this capture the PRO was `null` at booking and appeared in the **first
webhook that carried a carrier scan**:

| Webhook        | `updated_at` (UTC)     | `pro_number` | Newest event                   |
| -------------- | ---------------------- | ------------ | ------------------------------ |
| `booked`       | `2026-11-10T16:30:27Z` | `null`       | `pending_pickup` (ShipStation) |
| `pro-assigned` | `2026-11-11T08:31:29Z` | populated    | `AF` pickup scan               |

There was no webhook in between. So:

- The webhook delivers the PRO **after pickup**, not before. It can't give a
  shipper a PRO to put on freight paperwork ahead of the driver arriving.
- Treat `pro_number` as nullable on every event, and keep the last non-null
  value you've seen.
- `tracking_url` changes at the same moment, from the booking provider's
  public link to the carrier's own link keyed on the PRO. Don't cache the
  first one.

This carrier and provider behaved as one that doesn't pre-assign a PRO at
booking. A carrier that supports e-PROs may return it in the booking response
and populate it on the very first webhook; that hasn't been captured.

## Status values

`status` and `tracking_status` are not duplicates. They differ at booking:

| Field             | Values observed                                        |
| ----------------- | ------------------------------------------------------ |
| `status`          | `booked`, `in_transit`, `delivered`                    |
| `tracking_status` | `pending_pickup`, `in_transit`, `delivered`            |
| Event `status`    | `created`, `pending_pickup`, `in_transit`, `delivered` |

No exception, delay or cancelled values have been captured, so don't treat
this list as the full enum.

## The `events` array

**Cumulative.** Every delivery re-sends the full history. Each new `event_id`
added scans to the previous one and kept every earlier event. One webhook is
sent per update that found changes, not per scan: a single delivery can carry
more than one new scan. In the capture set one update added two scans at once
(an arrival and a departure); it isn't one of the fixtures.

**Two kinds of event.** The two oldest events are generated by ShipStation
(`created`, `pending_pickup`) and have `null` `carrier_code` and `description`.
Everything else is a carrier scan with a code and a description.

**Carrier codes** seen, which look like X12 214 status codes:

| Code | Description                                     |
| ---- | ----------------------------------------------- |
| `AF` | Carrier Departed Pick-up Location with Shipment |
| `P1` | Departed Terminal Location                      |
| `X4` | Arrived at Terminal Location                    |
| `AJ` | Tendered for Delivery                           |
| `D1` | Completed Unloading at Delivery Location        |

**No locations.** The descriptions are generic. The carrier's own public
tracking page shows the city and state of every scan; the webhook carries
none of it.

**Departures are extra.** Checked against the carrier's public tracking page
for this PRO, the webhook's 8 `X4` arrivals match the page's 8 interim service
center arrivals, and pickup, out for delivery and delivery match as well. The
8 `P1` departure scans are in the webhook but not on the public page. The
delivered fixture's 21 events are therefore 19 carrier scans plus the 2
ShipStation events.

**Order.** The array is newest first. Don't re-sort it by `occurred_at`; see
the next section.

## Timestamps

**Observed, not confirmed with ShipStation.** On carrier scans, `occurred_at`
is 8 hours behind the true UTC instant: the value is the Pacific wall-clock
time with a `Z` on it. This was checked against the carrier's public tracking
page for this PRO, where every one of the 11 history entries lined up with a
webhook scan at exactly that offset:

| Carrier page                                 | True UTC            | Webhook `occurred_at`  |
| -------------------------------------------- | ------------------- | ---------------------- |
| Picked up, 11/10 3:48 PM EST                 | `2026-11-10T20:48Z` | `2026-11-10T12:48:00Z` |
| Arrived at service center, 11/11 6:04 AM EST | `2026-11-11T11:04Z` | `2026-11-11T03:04:00Z` |
| Arrived at service center, 11/12 5:23 PM CST | `2026-11-12T23:23Z` | `2026-11-12T15:23:00Z` |
| Delivered, 11/16 12:40 PM CST                | `2026-11-16T18:40Z` | `2026-11-16T10:40:00Z` |

What this means for a receiver:

- The two ShipStation-generated events and `updated_at` appear to be real UTC,
  so **one `events` array mixes two meanings of the same `Z`**. Sorting by
  `occurred_at` puts the pickup scan before the booking event; the array's own
  order is the correct one.
- A parser that trusts `occurred_at` as UTC will place carrier scans 8 hours
  too early. Don't compare a carrier scan's `occurred_at` with `updated_at`
  or with the ShipStation events.
- Whether the offset follows an account time zone, a server default or a fixed
  -8 can't be told from one November shipment. A capture during daylight
  saving time would show whether it moves.

If you need an instant for display, treat the carrier's own page as the
reference until this is confirmed.

## Latency and the late delivered event

`updated_at` lands 0.2 to 2.3 hours after the true time of the newest carrier
scan in every update after the pickup scan (the pickup took about 12 hours to
be reported), apart from delivery. That is consistent with ShipStation polling the carrier
roughly hourly and firing when it finds changes. The exception is delivery:
`D1` happened at `2026-11-16T18:40Z` and the delivered webhook's `updated_at`
is `2026-11-17T16:43:58Z`, about 22 hours later. That `updated_at` is exactly
24 hours after the previous webhook's, to the second. It might be a slower
poll cadence at the end of a journey, or the carrier posting the delivery late;
one shipment can't say which. Don't build anything that needs the delivered
event in real time.

## `estimated_delivery_date` is not maintained

It reads `2026-11-13` in every one of the five fixtures, including the
`delivered` one where the shipment arrived on 2026-11-16. It is set at
booking and not updated as the carrier slips.

## What the payload does not carry

- **Proof of delivery.** The carrier's page has a Documents tab; the webhook
  has no `documents`. Call `GET /freight/shipments/{freight_shipment_id}/documents`.
- **A delivery date field.** There is no `actual_delivery_date` or signer. Take
  the delivery time from the event with `status` `delivered`, noting the
  timestamp caveat above.
- **Scan locations**, as above.

## Redelivery

Each unique update was delivered **three times** in this capture: the first
attempt plus two retries. The capture receiver returned `404` throughout, which
is why. All copies of an `event_id` are **byte-identical**, including
`updated_at`, so the fixtures use one copy of each.

What this does not tell you is how ShipStation behaves when the receiver
returns `2xx`, so duplicate delivery on success is unobserved. Two practical
points:

- De-duplicate on `data.event_id`.
- If a receiver is down and recovers, an older update's retry can land after a
  newer update. Don't let an older redelivery overwrite newer state: compare
  `updated_at`, or the length of `events`.

## Sanitization

No names, addresses, phone numbers or emails appear in these payloads, only
identifiers. Following the repo convention, these are masked mid-string with
`XXX`/`X` and the length preserved, and kept in step wherever they repeat:

- `freight_shipment_id` and the copy in `resource_url`, `shipment_id`
- `shipment_number`, `bol_number`, `pickup_number`
- `pro_number` and the PRO inside the carrier `tracking_url`
- the token in the booking provider's `tracking_url`, which is a link to the
  provider's public tracking page for the shipment

`event_id` is kept as received, because masking it would hide the property
that makes it useful. Account-level identifiers (`freight_provider_account_id`,
`carrier_scac`) are kept, as elsewhere in this repo. Each `_captured_at` is the
date of the first delivery of that update.

## Not yet captured

- A carrier that returns a PRO at booking (e-PRO)
- Any exception, delay, cancelled or rescheduled status
- Other freight providers or carriers
- A shipment during daylight saving time, to test the timestamp offset
- A webhook with `estimated_delivery_date` of `null`

## Related

The same data is reachable by polling:
[`patterns/freight/pro-number-retrieval-tracking.md`](../../../patterns/freight/pro-number-retrieval-tracking.md)
covers both the webhook and the polling fallback.
Freight booked outside ShipStation can only be tracked with
`GET /freight/tracking`; see
[`patterns/freight/track-freight.md`](../../../patterns/freight/track-freight.md).
