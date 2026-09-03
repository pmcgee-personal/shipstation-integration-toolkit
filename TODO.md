# Fill-in checklist

For each fixture: pull the real payload/response from production, sanitize
it (fake tracking/order numbers, fake names/addresses), paste it in over the
placeholder, and delete the `_fixture_notes` key.

## api/tracking/usps/
- [ ] no-scans-yet.json
- [ ] pre-transit.json
- [ ] in-transit-single-scan.json
- [ ] in-transit-multi-scan.json
- [ ] out-for-delivery.json
- [ ] delivered.json
- [ ] delivery-exception.json
- [ ] delivery-attempt-failed.json
- [ ] returned-to-sender.json

## api/orders/
- [ ] get-order-response.json
- [ ] list-orders-response.json

## api/shipments/
- [ ] create-shipment-response.json
- [ ] get-shipment-response.json
- [ ] create-label-response.json

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
- [ ] Fill in the capture date for that fixture (add a `_captured_at` note or
      track in the README — pick one convention and stay consistent)
- [ ] Double-check no real tracking numbers, order IDs, names, or addresses
      remain
- [ ] Confirm field names match exactly what the API/webhook actually sends
      (don't rely on the placeholder field names — they're guesses)

## Adding new resources beyond this list
Add new folders under `webhooks/` or `api/` as needed — new resource types,
additional carriers under `api/tracking/`, etc. Keep the type-first
(`webhooks/` vs `api/`) structure, then resource underneath.
