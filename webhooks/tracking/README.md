# Tracking Status Updated Webhook

## Objective

To hold a real example of the ShipStation tracking webhook — the event that
fires as a parcel moves, rather than at order or label time.

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

Practically: a receiver that pattern-matches on `resource_url` and fetches
it will do nothing at all on a tracking event. Switch on `resource_type`
first, then decide whether to read the body or go fetch. See
[`../README.md`](../README.md) for the pointer pattern the other events use.

**The fixture here is still a placeholder.** It carries `_fixture_notes` and
has not been captured from production, so don't build against its field
names — they're guesses. See [`../../TODO.md`](../../TODO.md).

## Scenarios

| Fixture                                                        | Status          | What it shows                                            |
| ---------------------------------------------------------------- | --------------- | ------------------------------------------------------------ |
| [`tracking-status-updated.json`](./tracking-status-updated.json) | **Placeholder** | Awaiting a real capture. `resource_type` is not yet confirmed. |

## In the meantime

The response *shape* you get for a tracking lookup is already documented
with real data — see
[`api/tracking/usps/`](../../api/tracking/usps/README.md), which has a
fixture per lifecycle status (no scans, pre-transit, in transit, delivered,
exception, return to sender).

If you need delivery state today and can tolerate polling, calling
`GET /v1/tracking` on the tracking numbers you collected from
[`../labels/`](../labels/) and [`../fulfillments/`](../fulfillments/) is the
path with real fixtures behind it.

## Filling this in

Capture the real POST body — the whole thing, since here the body *is* the
payload. See the root [README](../../README.md) for the sanitize-and-date
steps; tracking numbers in particular need masking.

Add a sibling fixture per distinct status rather than overloading one file,
matching how [`api/tracking/usps/`](../../api/tracking/usps/README.md) is
organised. Worth confirming while capturing:

- [ ] The `resource_type` value, and whether this event is versioned (`_V2`)
      like the others
- [ ] Whether the body reports a single scan event or the full accumulated
      `events` history
- [ ] Whether the field names match the `GET /v1/tracking` response in
      [`api/tracking/usps/`](../../api/tracking/usps/README.md) or differ —
      webhook and API shapes for the same resource are similar but not
      guaranteed identical
