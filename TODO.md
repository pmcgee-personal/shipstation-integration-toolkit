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
- ✅ `webhooks/tracking-v1/` — 16 real fixtures (UPS/USPS/FedEx/DHL eCommerce × label-created, in-transit, out-for-delivery, delivered), all captured 2026-10-14 in the current shape. No delivery-exception captures yet (see below)
- ✅ `patterns/` — 5 patterns (root + erp/ + wms/ organization)

**Partly covered — older-shape fixtures awaiting recapture:**

- ⚠️ `webhooks/tracking-v2/` — 15 fixtures (FedEx/UPS/USPS × 5 scenarios each): 12 real, 3 placeholders. FedEx was checked 2026-10-09 and is unchanged. UPS and USPS changed in October; only `ups/track-event-v2-delivered.json` and `usps/track-event-v2-multi-scan.json` are recaptured in the current shape (see below)
- ⚠️ `api/tracking/usps/` — 8 fixtures (full lifecycle). Not yet checked against the October changes

**Gaps:**

- ⚠️ Carrier exception scenarios (3 placeholder fixtures in `webhooks/tracking-v2/`, none yet for `webhooks/tracking-v1/`)
- ⚠️ Webhook events not yet captured (fulfillment cancelled, batch completed)

## webhooks/tracking-v2/

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

## webhooks/tracking-v1/

All 16 fixtures are real captures from 2026-10-14, after the October webhook
and mapping changes, so they are in the current shape. Each carrier's four
fixtures are one parcel at four points in its journey, not four parcels.

Missing fixtures:

- [ ] Delivery exception for each carrier (UPS, USPS, FedEx, DHL eCommerce) —
      no capture and no placeholder file yet. Name it
      `api-track-delivery-exception.json` in the carrier folder
- [ ] No v1 fixture has an `EX` status at the top level or on any scan —
      capture a genuinely stuck parcel

**Confirmed production behaviour — do not "fix" these in the fixtures.**
They are documented in the carrier READMEs:

- DHL eCommerce: the scans at `2026-10-09T06:34:37Z` and `06:51:25Z` are out
  of `occurred_at` order in three fixtures, and are 17 minutes apart between
  a Texas ZIP and a Washington ZIP, so at least one timestamp is not a true
  UTC instant
- DHL eCommerce: `status_detail_code`, `status_detail_description` and
  `carrier_detail_code` are `null` at the top level and on every event;
  `city_locality` and `state_province` are `null` on every scan; and
  `estimated_delivery_date` is still populated (two days after the actual
  delivery) on the delivered fixture
- FedEx: the `Picked up` scan (`2026-10-13T00:00:00Z`) is timestamped before
  the label-created scan, so `status_code` goes `AC` → `NY` → `IT`;
  `actual_delivery_date` is 14 hours before the `Delivered` scan; and
  `postal_code` is `null` on every scan
- USPS: `carrier_occurred_at` is `-07:00` on every scan regardless of
  location; the `U1` scan reads `Arrived at USPS Facility` but maps to
  `HUB_SCAN_OUT`; the Baton Rouge scans carry the facility name in
  `city_locality` with a `null` `state_province`; and the `Out for Delivery`
  description differs between the out-for-delivery and delivered fixtures
- UPS: event-level `status_description` is `null` on every event while the
  top level is populated, the label-created fixture pairs `NY` with
  `Accepted`, and `carrier_occurred_at` is naive carrier-local time

## Sanitization completeness

All fixtures have been reviewed and sanitized:

- [x] Tracking numbers — masked (e.g. `3831XXXXX470`). The `tracking-v1` fixtures mask the whole number with length preserved (e.g. `1Z################`), which is accepted
- [x] Order/shipment/label IDs — masked (e.g. `se-184976XXX`; `se-#########` in the `tracking-v1` `label_url` values)
- [x] Names and addresses — replaced with fake values
- [x] ZIP codes on delivery scans — masked or normalized
- [x] Download token URLs — masked (`XXXXXXXXXXXXXXXXXXXX`)
- [x] Coordinates — normalized to 2 decimal places (privacy); raw tracking bodies from UPS, USPS, FedEx and DHL eCommerce arrive at up to 4 decimals, so normalize on every new capture
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
