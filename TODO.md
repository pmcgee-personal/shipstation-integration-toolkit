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

## webhooks/orders/
- [ ] order-created.json
- [ ] order-updated.json

## webhooks/shipments/
- [ ] shipment-created.json
- [ ] shipment-shipped.json
- [ ] shipment-cancelled.json

## webhooks/tracking/
- [ ] tracking-status-updated.json

## Once a section is filled in
- [ ] Confirm `_fixture_notes` is gone and `_captured_at` is set
- [ ] Double-check no real tracking numbers, order IDs, names, or addresses
      remain
- [ ] Confirm field names match exactly what the API/webhook actually sends
      (don't rely on the placeholder field names — they're guesses)

## Adding new resources beyond this list
Add new folders under `webhooks/` or `api/` as needed — new resource types,
additional carriers under `api/tracking/`, etc. Keep the type-first
(`webhooks/` vs `api/`) structure, then resource underneath.
