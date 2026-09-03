# Tracking Webhook (`TRACK_EVENT_V2`)

## Objective

To hold real examples of the ShipStation tracking webhook — the event that
fires as a parcel moves, rather than at order or label time — across the
three carriers it has been captured from.

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

A tracking body *does* still carry a `resource_url`
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
    "events": [ "…newest first" ]
  }
}
```

`resource_type` is `TRACK_EVENT_V2` for all three carriers. `data` holds the
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

| Folder                | Carrier | `carrier_code`   | `carrier_id` | Fixtures |
| --------------------- | ------- | ---------------- | ------------ | -------- |
| [`fedex/`](./fedex/)  | FedEx   | `fedex_walleted` | `194`        | 4 real, 1 placeholder |
| [`ups/`](./ups/)      | UPS     | `ups`            | `3`          | 4 real, 1 placeholder |
| [`usps/`](./usps/)    | USPS    | `stamps_com`     | `1`          | 4 real, 1 placeholder |

Each folder has its own README covering that carrier's quirks. Read them
before writing carrier-specific logic — the differences below are real and
none of them are documented by ShipStation.

## Coverage

| Scenario           | FedEx | UPS | USPS |
| ------------------ | ----- | --- | ---- |
| Pre-transit        | —     | ✅  | ✅   |
| In transit         | ✅    | ✅  | ✅   |
| Out for delivery   | ✅    | ✅  | ✅   |
| Delivered          | ✅    | ✅  | ✅   |
| Delivery exception | 🚧 placeholder | 🚧 placeholder | 🚧 placeholder |

FedEx has no dedicated pre-transit fixture, but its in-transit and delivered
fixtures both include the `Shipment information sent to FedEx` scan (`NY`)
at the tail of `events`, which is the same moment.

No carrier has a fixture where `EX` is the **top-level** status. FedEx's
multi-scan and USPS's out-for-delivery both contain `EX` scans in their
history on parcels that recovered — useful, but not the same thing as a
parcel that is actually stuck.

## Carrier differences that will break shared code

The envelope is identical across carriers — same 16 `data` keys in the same
order, same 20 event keys in the same order, in every one of the 12 real
fixtures. The **values** are not consistent:

| Behaviour                          | FedEx                        | UPS                                | USPS                             |
| ---------------------------------- | ---------------------------- | ---------------------------------- | -------------------------------- |
| `status_detail_code`               | Populated                    | `null` (except pre-transit)        | Populated                        |
| `carrier_detail_code`              | `null`                       | Populated (`FS`, `OD`, `OT`, …)    | `null`                           |
| `postal_code` when absent          | `null`                       | `""` empty string                  | `null`                           |
| `country_code`                     | Always `US`                  | Always `US`                        | Often `null`                     |
| `ship_date` / delivery dates       | Naive, no suffix             | `Z`-suffixed                       | `Z`-suffixed                     |
| `actual_delivery_date` vs scan     | **Disagree by 14h**          | Match exactly                      | Match exactly                    |
| Events sorted by `occurred_at`     | Yes                          | Yes                                | **No** — see below               |

The first two rows are the ones that bite: telling *out for delivery* from
*in transit* works off `status_detail_code` on FedEx and USPS, and off
`carrier_detail_code` on UPS. There is no single field that does it for all
three.

## `carrier_occurred_at` is not UTC

Every event carries two timestamps. `occurred_at` is trustworthy UTC.
`carrier_occurred_at` is not, and it is encoded three different ways across
this tree:

| Encoding                          | Where                                              | Actually UTC? |
| --------------------------------- | -------------------------------------------------- | ------------- |
| `2026-08-28T19:24:32Z`            | All FedEx fixtures                                 | Yes — identical to `occurred_at` |
| `2026-09-03T12:18:41Z`            | UPS and USPS in-transit / delivered fixtures       | **No** — carrier-local time with a `Z` glued on |
| `2026-08-19T15:44:46-07:00`       | `usps/track-event-v2-pre-transit.json`             | Yes — a real offset, the only correct one here |
| `2026-09-03T11:19:14`             | `ups/track-event-v2-pre-transit.json`              | Unspecified — no suffix at all |

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
addresses, emails or phone numbers appear in any fixture here. But they are
not free of identifying data, and two fields deserve attention before you
publish a capture or hand a body to a third party:

**`latitude` / `longitude`.** Present on nearly every event at 4-decimal
(~11 m) precision, and **not masked in any fixture**. On an in-transit scan
that is a carrier facility and harmless. On the `Delivered` scan it is the
delivery point:

| Fixture                               | `Delivered` coordinates | Locality      | ZIP on that scan |
| ------------------------------------- | ----------------------- | ------------- | ---------------- |
| `fedex/track-event-v2-delivered.json` | `28.6622, -81.4895`     | Apopka, FL    | `null`           |
| `ups/track-event-v2-delivered.json`   | `42.4905, -83.1379`     | Royal Oak, MI | `48067`          |
| `usps/track-event-v2-delivered.json`  | `40.9865, -74.3831`     | Butler, NJ    | `07405`          |

**`postal_code` on the delivery scan.** UPS leaves this `""` on every event
except `Delivered`, where it carries the recipient ZIP. USPS carries the
destination ZIP on its delivery scans too. Paired with the coordinates
above, that locates a destination fairly precisely.

USPS additionally states the **delivery method** in the description
(`DELIVERED IN/AT MAILBOX`), which the other two carriers do not.

Neither is a name, and for a fixture repo of self-shipped test parcels this
may well be acceptable — but decide it deliberately rather than by omission,
and round or null the coordinates on the final scan if these were real
customer deliveries.

Two more fields are `null` in every fixture captured so far and would need
handling if they ever populate:

- `signer` — a recipient's name, on signature-required deliveries.
- `proof_of_delivery_url` — typically a signature image, and
  credential-bearing. Treat like the `label_download` URLs in
  [`../labels/`](../labels/README.md): don't log it, don't hand it to a
  browser you don't control.

## Related

The same tracking data is also reachable by polling — see
[`api/tracking/usps/`](../../api/tracking/usps/README.md), which has a
fixture per lifecycle status including the two exception cases (carrier
delay and return-to-sender) that the webhook folders are still missing.
Webhook and API shapes for the same resource are similar but not guaranteed
identical, so don't copy one into the other.

## Filling in the gaps

Capture the real POST body — the whole thing, since here the body *is* the
payload. See the root [README](../../README.md) for the sanitize-and-date
steps, and [`../../TODO.md`](../../TODO.md) for the current checklist.
Tracking numbers, the `se-` label ID in `resource_url`, and `tracking_url`
all need masking together.

Add a sibling fixture per distinct status rather than overloading one file,
matching how [`api/tracking/usps/`](../../api/tracking/usps/README.md) is
organised, and add a row to the carrier folder's Scenarios table.
