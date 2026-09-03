# Fill-in checklist

For each fixture: pull the real payload/response from production, sanitize
it (fake tracking/order numbers, fake names/addresses), paste it in over the
placeholder, delete the `_fixture_notes` key, and add a `_captured_at` key
with the capture date (`YYYY-MM-DD`).

`_fixture_notes` marks a fixture as a placeholder still awaiting real data —
if a file has it, it's unfinished. Descriptions of what an endpoint does
belong in the folder's `README.md`, not in the fixture.

## api/tracking/usps/
- [ ] out-for-delivery.json

## webhooks/tracking/
- [ ] tracking-status-updated.json — unlike the other webhooks this one
      delivers its data inline, so the capture is the whole POST body and
      there is no `resource_url` response to pair with it
- [ ] Confirm the `resource_type` value and whether this event is versioned
      (`_V2`) like the others
- [ ] Confirm whether the body reports a single scan or the full
      accumulated `events` history

## Sanitization gaps in existing fixtures
These are already filled in with real data, but the sanitization is
inconsistent with the rest of the repo — same rules as above apply.

- [ ] `webhooks/orders/*` and `webhooks/labels/*` — FedEx tracking numbers
      are unmasked (e.g. `383144378470`). Elsewhere in the repo they're
      masked (`3831466XXXX3`, `9300110XXXXX3550425334`). Mask them, and
      keep `tracking_url` and `packages[].tracking_number` in step.
- [ ] `webhooks/labels/*` — `label_download` / `packages[].label_download`
      URLs contain live download tokens. Replace the token segment.
- [ ] Account/user identifiers left in place across the webhook fixtures
      (`store_id`, `warehouse_id`, `carrier_id`, `user_id`). Decide whether
      these are fine to publish or should be masked like the `se-` IDs that
      already are (e.g. `se-2077XXX`).

## Coverage gaps
Events with no fixture yet. Add a folder under `webhooks/` named for the
ShipStation UI event, following the existing pattern (webhook body +
`resource_url` response + `README.md`).

- [ ] Order/shipment **updated** (V2)
- [ ] Shipment **cancelled** (V2)
- [ ] Label **voided** — confirm whether this fires its own event or is
      only visible on a re-read of the `label_id`
- [ ] Rate/batch completed (V2)

## Once a section is filled in
- [ ] Confirm `_fixture_notes` is gone and `_captured_at` is set
      (`_captured_at` exactly — not `_redacted_at` or any other spelling)
- [ ] Double-check no real tracking numbers, order IDs, names, or addresses
      remain
- [ ] Confirm field names match exactly what the API/webhook actually sends
      (don't rely on the placeholder field names — they're guesses)
- [ ] For a pointer-style webhook, capture **both** halves: the webhook body
      and the response from calling its `resource_url` (tracking is inline,
      so it's a single capture)
- [ ] Add the fixture to the Scenarios table in the folder's `README.md`

## Adding new resources beyond this list
Add new folders under `webhooks/` or `api/` as needed — new resource types,
additional carriers under `api/tracking/`, etc. Keep the type-first
(`webhooks/` vs `api/`) structure, then resource underneath.
