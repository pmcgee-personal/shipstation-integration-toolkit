# Contributing

Thanks for helping improve this toolkit! The most useful contributions are new or updated fixtures.

## Adding or updating fixtures

1. **Pull real data from production.** Capture the full webhook body or API response as-is, with no modifications. Fixtures are real captures: don't build one by trimming, editing or combining another capture. If a scenario hasn't been captured yet, leave its placeholder in place and list it in `TODO.md`.

2. **Sanitize carefully.** Replace:
   - Tracking numbers: mask the middle and preserve the original length. For example `940015020621XXXXX26585` (USPS, 22 digits), `1ZXXXH4303XXXXX632` (UPS, 18 characters) and `383369XXX473` (FedEx, 12 digits). The `tracking-v1` fixtures mask the whole number instead (for example `1Z################`, 18 characters), which is also accepted as long as the length is preserved. Keep any `tracking_url`, and the tracking number inside a v1 `resource_url`, in step with the masked number
   - Order/shipment/label IDs (use `se-184976XXX` format), including the `se-` ID inside a tracking `resource_url` or a v1 `label_url` (the `tracking-v1` fixtures mask it as `se-#########`, which is also accepted)
   - Names and addresses
   - ZIP codes in address fields. ZIP codes on tracking scans (carrier facilities and the delivery locality) are kept as received
   - Phone numbers
   - Any customer-identifying data
   - Latitude and longitude: round to 2 decimal places. Raw tracking bodies arrive at up to 4 decimals (UPS, USPS, FedEx and DHL eCommerce), and a `Delivered` scan can be at delivery-address precision

   Keep (they're non-sensitive):
   - `store_id`, `warehouse_id`, `carrier_id`, `user_id`
   - Carrier/service codes
   - Status codes

3. **Save the fixture.** Use the naming convention from existing files (e.g., `track-event-v2-delivered.json`, or `api-track-delivered.json` for v1), under the appropriate folder. Tracking fixtures go by `resource_type`: `TRACK_EVENT_V2` under `webhooks/tracking-v2/<carrier>/` and `API_TRACK` under `webhooks/tracking-v1/<carrier>/`. Carrier folder names are lowercase and hyphenated (for example `dhl-ecommerce`).

4. **Add metadata:**
   - Remove any `_fixture_notes` key
   - Add `_captured_at` with the date the payload was captured (format: `YYYY-MM-DD`)

5. **For webhook pointer events,** capture both halves: the webhook body AND the response from calling its `resource_url`. Tracking webhooks are inline, so just the one capture.

6. **Update the README.** Add your new scenario to the Scenarios table in the relevant folder's `README.md`. If the payload's shape differs from other fixtures in the folder (fields that were `null` and are now populated, or the reverse), record the differences in the README and mark which fixtures are the older shape.

7. **Check the TODO.** See if there are known placeholders you're filling in or sanitization gaps you're addressing — update `TODO.md` if so.

## Reporting issues

- **Fixtures look stale or wrong?** Open an issue with the file path and what's incorrect. Payload shapes change over time, so include the `_captured_at` date and which fields differ.
- **Found real data (names, addresses, tracking)?** See `SECURITY.md`.
- **Carrier behavior difference?** Document it in the relevant folder's `README.md` (not the fixture).

## Questions?

Each folder has a `README.md` with carrier-specific notes and payload documentation. Start there, then check the root `README.md` for repo structure and the `TODO.md` for what's still needed.
