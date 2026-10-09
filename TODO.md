# Fill-in checklist

For each fixture: pull the real payload/response from production, sanitize
it (fake tracking/order numbers, fake names/addresses), paste it in over the
placeholder, delete the `_fixture_notes` key, and add a `_captured_at` key
with the capture date (`YYYY-MM-DD`).

`_fixture_notes` marks a fixture as a placeholder still awaiting real data —
if a file has it, it's unfinished. Descriptions of what an endpoint does
belong in the folder's `README.md`, not in the fixture.

## Coverage summary

**Fully covered:**

- ✅ `webhooks/orders/` — 3 fixtures (single shipment, split shipment, + resource_url response). Checked 2026-10-09: unchanged by the October webhook/mapping changes
- ✅ `webhooks/labels/` — 3 fixtures (single, split shipment, + resource_url response). Checked 2026-10-09: unchanged
- ✅ `webhooks/fulfillments/` — 2 fixtures (shipped + resource_url response). Checked 2026-10-09: unchanged
- ✅ `patterns/` — 5 patterns (root + erp/ + wms/ organization)

**Partly covered — older-shape fixtures awaiting recapture:**

- ⚠️ `webhooks/tracking/` — 15 fixtures (FedEx/UPS/USPS × 5 scenarios each): 12 real, 3 placeholders. FedEx was checked 2026-10-09 and is unchanged. UPS and USPS changed in October; only `ups/track-event-v2-delivered.json` and `usps/track-event-v2-multi-scan.json` are recaptured in the current shape (see below)
- ⚠️ `api/tracking/usps/` — 8 fixtures (full lifecycle). Not yet checked against the October changes

**Gaps:**

- ⚠️ Carrier exception scenarios (3 placeholder fixtures)
- ⚠️ Webhook events not yet captured (fulfillment cancelled, batch completed)

## webhooks/tracking/

Recapture in the current (October 2026) shape. These are real September
captures, still accurate for that shape and labelled as such in the carrier
READMEs. Replace each when a fresh capture is available:

- [ ] `ups/track-event-v2-pre-transit.json`
- [ ] `ups/track-event-v2-multi-scan.json`
- [ ] `ups/track-event-v2-out-for-delivery.json`
- [ ] `usps/track-event-v2-pre-transit.json`
- [ ] `usps/track-event-v2-out-for-delivery.json`
- [ ] `usps/track-event-v2-delivered.json`

Already in the current shape (captured 2026-10-09):
`ups/track-event-v2-delivered.json`, `usps/track-event-v2-multi-scan.json`.
FedEx fixtures were checked against a current payload and need no change.

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

- `carrier_occurred_at` is encoded four different ways across the tree. It
  equals `occurred_at` on FedEx and on the current USPS shape; it is naive
  carrier-local time (no suffix) on the current UPS shape; and it is carrier-
  local time with a `Z` glued on in the September UPS and USPS fixtures
- September USPS shape: `In Transit to Next Facility` scans (`TL`/`NT`)
  convert with a fixed 7-hour offset regardless of the parcel's location,
  leaving `usps/track-event-v2-out-for-delivery.json` (events 14/15) and
  `usps/track-event-v2-delivered.json` (events 7/8) out of `occurred_at`
  order — those arrays sort by `carrier_occurred_at`. The current USPS shape
  is sorted by `occurred_at`
- September USPS shape: two event formats within one `events` array —
  ALL-CAPS descriptions with the facility in `city_locality` and a populated
  `postal_code`, versus Title-Case ones with the facility in `company_name`
  and `postal_code` `null`. The current shape does not show this
- USPS duplicate scans: `usps/track-event-v2-delivered.json` carries two
  identical `DELIVERED IN/AT MAILBOX` scans a minute apart, and
  `usps/track-event-v2-multi-scan.json` carries two identical
  `Accepted at USPS Facility` scans 13 minutes apart
- Current USPS shape: `In Transit to Next Facility` scans have
  `city_locality`, `state_province`, `postal_code`, `latitude` and
  `longitude` all `null`, and `Accepted at USPS Facility` scans have a `null`
  `status_detail_code`
- FedEx `actual_delivery_date` and the `Delivered` scan's `occurred_at`
  disagree by 14 hours in `fedex/track-event-v2-delivered.json`
- Current UPS shape: `status_detail_code` is populated on every scan,
  alongside `carrier_detail_code`. The September UPS fixtures leave
  `status_detail_code` `null` (except pre-transit)
- Current UPS shape: `OUT_FOR_DELIVERY` appears on two scans (`Loaded on
Delivery Vehicle`, then `Out For Delivery Today`), `status_code` goes `IT`
  → `AC` → `IT` across the first scans, and `ship_date` is the label date at
  midnight rather than the first scan's timestamp
- UPS uses `""` rather than `null` for absent `postal_code`, and in the
  current shape for `signer`

## Sanitization completeness

All fixtures have been reviewed and sanitized:

- [x] Tracking numbers — masked (e.g. `3831XXXXX470`)
- [x] Order/shipment/label IDs — masked (e.g. `se-184976XXX`)
- [x] Names and addresses — replaced with fake values
- [x] ZIP codes on delivery scans — masked or normalized
- [x] Download token URLs — masked (`XXXXXXXXXXXXXXXXXXXX`)
- [x] Coordinates — normalized to 2 decimal places (privacy); raw UPS and USPS bodies now arrive at 4 decimals, so normalize on every new capture
- [x] Account identifiers — intentionally kept (`store_id`, `carrier_id`, etc. are non-personal)

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
