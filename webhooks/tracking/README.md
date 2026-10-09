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
scans a minute apart, and `usps/track-event-v2-multi-scan.json` contains two
`Accepted at USPS Facility` scans 13 minutes apart, each pair identical in
every other field. If a customer-facing
notification must fire exactly once, gate it on the transition into a status
rather than on the arrival of an event row.

## Carriers

| Folder               | Carrier | `carrier_code`   | `carrier_id` | Fixtures                                                    |
| -------------------- | ------- | ---------------- | ------------ | ----------------------------------------------------------- |
| [`fedex/`](./fedex/) | FedEx   | `fedex_walleted` | `194`        | 4 real, 1 placeholder                                       |
| [`ups/`](./ups/)     | UPS     | `ups`            | `3`          | 1 real, 3 derived from it, 1 placeholder                    |
| [`usps/`](./usps/)   | USPS    | `stamps_com`     | `1`          | 3 real (1 current shape, 2 older), 1 derived, 1 placeholder |

Each folder has its own README covering that carrier's quirks.

## October 2026 update

Following changes to the webhook and its field mapping, UPS and USPS bodies
changed after the September captures. FedEx was checked by hand against a
current payload and did not change, so its fixtures stand.

- **UPS** — all four non-placeholder fixtures were rebuilt from one parcel
  captured 2026-10-09. Only the delivered fixture is as received; the other
  three are earlier snapshots cut from its cumulative `events` array and carry
  a `_derived_from` key.
- **USPS** — `track-event-v2-multi-scan.json` is a new real capture and
  `track-event-v2-pre-transit.json` is cut from it. `out-for-delivery` and
  `delivered` are still the **September shape** and are labelled as such in
  [`usps/README.md`](./usps/README.md) until they are recaptured.

The September UPS fixtures and the September USPS pre-transit are in git
history. Wherever this file says "September shape" it refers to payloads that
the current webhook no longer produces.

## Coverage

| Scenario           | FedEx          | UPS            | USPS             |
| ------------------ | -------------- | -------------- | ---------------- |
| Pre-transit        | —              | 🧪 derived     | 🧪 derived       |
| In transit         | ✅             | 🧪 derived     | ✅               |
| Out for delivery   | ✅             | 🧪 derived     | ✅ (older shape) |
| Delivered          | ✅             | ✅             | ✅ (older shape) |
| Delivery exception | 🚧 placeholder | 🚧 placeholder | 🚧 placeholder   |

🧪 means a snapshot cut from a real capture of the same parcel, not an
independent capture. Its top-level fields are inferred from the newest scan;
see the carrier README for exactly what was assumed.

## Carrier differences that will break shared code

The envelope is identical across carriers — same 16 `data` keys in the same
order, same 20 event keys in the same order, in every one of the 12
non-placeholder fixtures. The **values** are not consistent:

| Behaviour                      | FedEx               | UPS                                   | USPS                                         |
| ------------------------------ | ------------------- | ------------------------------------- | -------------------------------------------- |
| `status_detail_code`           | Populated           | Populated on every scan               | Populated, except `null` on `OB` scans       |
| `carrier_detail_code`          | `null`              | Populated (`FS`, `OF`, `OT`, …)       | `null`                                       |
| `postal_code` when absent      | `null`              | `""` empty string                     | `null` (only on scans with no location)      |
| `country_code`                 | Always `US`         | Always `US`                           | Always `US` (often `null` in older fixtures) |
| `signer`                       | `null`              | `""` on every scan                    | `null`                                       |
| `ship_date` / delivery dates   | Naive, no suffix    | `ship_date` and ETA naive; actual `Z` | `Z`-suffixed                                 |
| `actual_delivery_date` vs scan | **Disagree by 14h** | Match exactly                         | Match exactly                                |
| Events sorted by `occurred_at` | Yes                 | Yes                                   | Yes (older fixtures: **No** — see below)     |

`status_detail_code` now works as the single field for telling _out for
delivery_ from _in transit_ on all three carriers, which it did not for UPS in
September. It does not map one event to one state:

- **UPS** reports `OUT_FOR_DELIVERY` on two scans about an hour apart
  (`Loaded on Delivery Vehicle`, then `Out For Delivery Today`), and
  `DELIVERY_SCHEDULED` just before them is still in transit.
- **USPS** leaves it `null` on `Accepted at USPS Facility` scans.
- **Both** can move `status_code` backwards or sideways (UPS `IT` → `AC` →
  `IT`), so don't build a forward-only state machine on it.

Location fields are not uniformly present either. A USPS
`In Transit to Next Facility` scan has `city_locality`, `state_province`,
`postal_code`, `latitude` and `longitude` all `null` while `country_code` is
still `US`; a UPS label-created scan has `""` for city, state and postal code
and `null` coordinates. Test each field for absence, and treat `""` and
`null` as the same case.

## `carrier_occurred_at` is not UTC

Every event carries two timestamps. `occurred_at` is trustworthy UTC.
`carrier_occurred_at` is not reliably so, and is encoded three different ways
across this tree:

| Encoding               | Where                                                 | Actually UTC?                                            |
| ---------------------- | ----------------------------------------------------- | -------------------------------------------------------- |
| `2026-08-28T19:24:32Z` | All FedEx fixtures; the current USPS shape            | Yes — identical to `occurred_at`                         |
| `2026-10-09T18:21:51`  | All UPS fixtures                                      | **No** — naive carrier-local time; offset varies by scan |
| `2026-09-03T12:18:41Z` | USPS `out-for-delivery` and `delivered` (older shape) | **No** — carrier-local time with a `Z` glued on          |

The UPS value carries no suffix and is carrier-local time. In the delivered
fixture the offset from `occurred_at` is 7 hours for scans in Washington and
Oregon and 4 hours for scans in Massachusetts and Connecticut, so the scan's
time zone is needed to turn it into an instant. In the older USPS fixtures
the `Z` suffix is present but the value is also local, and the implied offset
varies **per event within one file** —
`usps/track-event-v2-out-for-delivery.json` mixes 5, 6 and 7 hours as the
parcel crosses time zones — and a parser that trusts the `Z` will place scans
hours from where they happened.

In the older USPS shape the conversion into `occurred_at` was itself
inconsistent: `In Transit to Next Facility` scans (`event_code` `TL`/`NT`)
were converted with a fixed 7-hour offset regardless of where the parcel was,
which pushed them past the scan that follows and left two USPS fixtures **out
of chronological order** in array position. The current shape does not show
this, but its `TL` scans have no location to check against, so keep sorting.
See [`usps/README.md`](./usps/README.md).

**Parse `occurred_at`. Use `carrier_occurred_at` for display only, and sort
by `occurred_at` rather than trusting array position.**

## PII in tracking payloads

Tracking bodies are lower-risk than order or label payloads — no names,
addresses, emails or phone numbers appear in any fixture here.

Raw bodies do carry fine-grained coordinates, though: the 2026-10-09 UPS
`Delivered` scan arrived at 4-decimal precision at the delivery address, which
differs from the facility coordinates. All fixtures normalize `latitude` and
`longitude` to 2 decimals, and tracking numbers and `se-` label IDs are
masked.

## Related

The same tracking data is also reachable by polling — see
[`api/tracking/usps/`](../../api/tracking/usps/README.md), which has a
fixture per lifecycle status including the two exception cases (carrier
delay and return-to-sender) that the webhook folders are still missing.
Webhook and API shapes for the same resource are similar but not guaranteed
identical, so don't copy one into the other.
