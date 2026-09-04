# Fill-in checklist

For each fixture: pull the real payload/response from production, sanitize
it (fake tracking/order numbers, fake names/addresses), paste it in over the
placeholder, delete the `_fixture_notes` key, and add a `_captured_at` key
with the capture date (`YYYY-MM-DD`).

`_fixture_notes` marks a fixture as a placeholder still awaiting real data —
if a file has it, it's unfinished. Descriptions of what an endpoint does
belong in the folder's `README.md`, not in the fixture.

## webhooks/tracking/

Missing fixtures:

- [ ] `fedex/track-event-v2-delivery-exception.json` — placeholder
- [ ] `ups/track-event-v2-delivery-exception.json` — placeholder
- [ ] `usps/track-event-v2-delivery-exception.json` — placeholder
- [ ] No carrier has a capture where `EX` is the **top-level** status. The
      existing `EX` occurrences are historical scans on parcels that
      recovered — capture a genuinely stuck parcel
- [ ] FedEx has no standalone pre-transit fixture (the `NY` scan only
      appears at the tail of longer histories)

**Confirmed production behaviour — do not "fix" these in the fixtures.**
They are documented in the carrier READMEs. Listed here so a future capture
that reproduces them isn't mistaken for a bad sanitize:

- `carrier_occurred_at` is encoded four different ways across the tree and
  is **not** UTC on UPS/USPS despite the `Z` suffix
- USPS `In Transit to Next Facility` scans (`TL`/`NT`) convert with a fixed
  7-hour offset regardless of the parcel's location, leaving
  `usps/track-event-v2-out-for-delivery.json` (events 14/15) and
  `usps/track-event-v2-delivered.json` (events 7/8) out of `occurred_at`
  order — the arrays sort by `carrier_occurred_at`
- USPS mixes two event formats within one `events` array — ALL-CAPS
  descriptions with the facility in `city_locality` and a populated
  `postal_code`, versus Title-Case ones with the facility in `company_name`
  and `postal_code` `null`
- `usps/track-event-v2-delivered.json` carries two identical
  `DELIVERED IN/AT MAILBOX` scans a minute apart
- FedEx `actual_delivery_date` and the `Delivered` scan's `occurred_at`
  disagree by 14 hours in `fedex/track-event-v2-delivered.json`
- UPS reports detail in `carrier_detail_code` and leaves `status_detail_code`
  `null` (except pre-transit); FedEx and USPS do the opposite
- UPS uses `""` rather than `null` for absent `postal_code`

## Sanitization gaps in existing fixtures

These are already filled in with real data, but the sanitization is
inconsistent with the rest of the repo — same rules as above apply.

- [x] `webhooks/labels/*` — FedEx tracking numbers masked; download token
      URLs masked (`XXXXXXXXXXXXXXXXXXXX`)
- [ ] Account/user identifiers left in place across the webhook fixtures
      (`store_id`, `warehouse_id`, `carrier_id`, `user_id`). These are
      non-sensitive (store/account level, not personal). No masking needed.

## Coverage gaps

Events with no fixture yet. Add a folder under `webhooks/` named for the
ShipStation UI event, following the existing pattern (webhook body +
`resource_url` response + `README.md`).

- [ ] Fulfillment **cancelled** (V2)
- [ ] Batch completed (V2)

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
