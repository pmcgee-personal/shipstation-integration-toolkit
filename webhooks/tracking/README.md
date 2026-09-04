# Tracking Webhook (`TRACK_EVENT_V2`)

## Objective

To hold real examples of the ShipStation tracking webhook — the event that
fires as a parcel moves, rather than at order or label time — across USPS, UPS and FedEx.

## Tracking is the exception

Every other webhook in this repo is a **notification**: a `resource_url` and
a `resource_type`, and you call that URL with your API key to get the actual
record. **Tracking is not.** The tracking webhook carries the tracking data
inline in the POST body — there is nothing to call back for, and no API key
is needed to read it.

|                        | Other webhooks                   | Tracking                 |
| ---------------------- | -------------------------------- | ------------------------ |
| Body contains          | `resource_url` + `resource_type` | The tracking data itself |
| Second request needed  | Yes                              | **No**                   |
| API key needed to read | Yes, for the follow-up call      | No                       |

A tracking body _does_ still carry a `resource_url`
(`https://api.shipstation.com/v2/labels/se-454868XXX/track`), which makes
this easy to get wrong — it points at the label's tracking endpoint, not at
a resource you need to fetch to understand the event. Everything is already
in `data`. Switch on `resource_type` first, then decide whether to read the
body or go fetch. See [`../README.md`](../README.md) for the pointer pattern
the other events use.

## The body

```json
{
  "_captured_at": "2026-09-01",
  "resource_url": "https://api.shipstation.com/v2/labels/se-454868XXX/track",
  "resource_type": "TRACK_EVENT_V2",
  "data": {
    "tracking_number": "383369XXX473",
    "status_code": "DE",
    "status_description": "Delivered",
    "events": ["…newest first"]
  }
}
```

`resource_type` is `TRACK_EVENT_V2` for all carriers. `data` holds the
current status at the top level plus an `events` array of every scan so far.
`_captured_at` is a repo convention, not something ShipStation sends — see
the [root README](../../README.md).

The `events` array is **cumulative**: every webhook delivery re-sends the
full accumulated history, not just the scan that fired it. Combined with
at-least-once delivery, that means de-duplicating on `(tracking_number,
occurred_at, event_code)` rather than treating each POST as one new scan.

Note that even that key does not give you one row per real-world happening:
`usps/track-event-v2-delivered.json` contains two `DELIVERED IN/AT MAILBOX`
scans a minute apart, identical in every other field. If a customer-facing
notification must fire exactly once, gate it on the transition into a status
rather than on the arrival of an event row.

## Carriers

| Folder               | Carrier | `carrier_code`   | `carrier_id` | Fixtures              |
| -------------------- | ------- | ---------------- | ------------ | --------------------- |
| [`fedex/`](./fedex/) | FedEx   | `fedex_walleted` | `194`        | 4 real, 1 placeholder |
| [`ups/`](./ups/)     | UPS     | `ups`            | `3`          | 4 real, 1 placeholder |
| [`usps/`](./usps/)   | USPS    | `stamps_com`     | `1`          | 4 real, 1 placeholder |

Each folder has its own README covering that carrier's quirks.

## Coverage

| Scenario           | FedEx          | UPS            | USPS           |
| ------------------ | -------------- | -------------- | -------------- |
| Pre-transit        | —              | ✅             | ✅             |
| In transit         | ✅             | ✅             | ✅             |
| Out for delivery   | ✅             | ✅             | ✅             |
| Delivered          | ✅             | ✅             | ✅             |
| Delivery exception | 🚧 placeholder | 🚧 placeholder | 🚧 placeholder |

## Carrier differences that will break shared code

The envelope is identical across carriers — same 16 `data` keys in the same
order, same 20 event keys in the same order, in every one of the 12 real
fixtures. The **values** are not consistent:

| Behaviour                      | FedEx               | UPS                             | USPS               |
| ------------------------------ | ------------------- | ------------------------------- | ------------------ |
| `status_detail_code`           | Populated           | `null` (except pre-transit)     | Populated          |
| `carrier_detail_code`          | `null`              | Populated (`FS`, `OD`, `OT`, …) | `null`             |
| `postal_code` when absent      | `null`              | `""` empty string               | `null`             |
| `country_code`                 | Always `US`         | Always `US`                     | Often `null`       |
| `ship_date` / delivery dates   | Naive, no suffix    | `Z`-suffixed                    | `Z`-suffixed       |
| `actual_delivery_date` vs scan | **Disagree by 14h** | Match exactly                   | Match exactly      |
| Events sorted by `occurred_at` | Yes                 | Yes                             | **No** — see below |

The first two rows are the ones that bite: telling _out for delivery_ from
_in transit_ works off `status_detail_code` on FedEx and USPS, and off
`carrier_detail_code` on UPS. There is no single field that does it for all
three.

## `carrier_occurred_at` is not UTC

Every event carries two timestamps. `occurred_at` is trustworthy UTC.
`carrier_occurred_at` is not, and it is encoded three different ways across
this tree:

| Encoding                    | Where                                        | Actually UTC?                                   |
| --------------------------- | -------------------------------------------- | ----------------------------------------------- |
| `2026-08-28T19:24:32Z`      | All FedEx fixtures                           | Yes — identical to `occurred_at`                |
| `2026-09-03T12:18:41Z`      | UPS and USPS in-transit / delivered fixtures | **No** — carrier-local time with a `Z` glued on |
| `2026-08-19T15:44:46-07:00` | `usps/track-event-v2-pre-transit.json`       | Yes — a real offset, the only correct one here  |
| `2026-09-03T11:19:14`       | `ups/track-event-v2-pre-transit.json`        | Unspecified — no suffix at all                  |

In the `Z`-but-local case the implied offset varies **per event within one
file** — `usps/track-event-v2-out-for-delivery.json` mixes 5, 6 and 7 hours
as the parcel crosses time zones. A parser that trusts the `Z` will place
scans hours from where they happened.

Worse, on USPS the conversion into `occurred_at` is itself inconsistent:
`In Transit to Next Facility` scans (`event_code` `TL`/`NT`) are converted
with a fixed 7-hour offset regardless of where the parcel is, which pushes
them past the scan that follows and leaves two USPS fixtures **out of
chronological order** in array position. Sort by `occurred_at` rather than
trusting the array, and treat `TL`/`NT` timestamps as approximate. See
[`usps/README.md`](./usps/README.md).

**Parse `occurred_at`. Use `carrier_occurred_at` for display only, and sort
by `occurred_at` rather than trusting array position.**

## PII in tracking payloads

Tracking bodies are lower-risk than order or label payloads — no names,
addresses, emails or phone numbers appear in any fixture here.

## Related

The same tracking data is also reachable by polling — see
[`api/tracking/usps/`](../../api/tracking/usps/README.md), which has a
fixture per lifecycle status including the two exception cases (carrier
delay and return-to-sender) that the webhook folders are still missing.
Webhook and API shapes for the same resource are similar but not guaranteed
identical, so don't copy one into the other.
