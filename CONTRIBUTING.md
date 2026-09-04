# Contributing

Thanks for helping improve this toolkit! The most useful contributions are new or updated fixtures.

## Adding or updating fixtures

1. **Pull real data from production.** Capture the full webhook body or API response as-is, with no modifications.

2. **Sanitize carefully.** Replace:
   - Tracking numbers (use `9434650105497XXXXX0495` format)
   - Order/shipment/label IDs (use `se-184976XXX` format)
   - Names and addresses
   - ZIP codes
   - Phone numbers
   - Any customer-identifying data

   Keep (they're non-sensitive):
   - `store_id`, `warehouse_id`, `carrier_id`, `user_id`
   - Carrier/service codes
   - Status codes

3. **Save the fixture.** Use the naming convention from existing files (e.g., `track-event-v2-delivered.json`), under the appropriate folder.

4. **Add metadata:**
   - Remove any `_fixture_notes` key
   - Add `_captured_at` with the capture date (format: `YYYY-MM-DD`)

5. **For webhook pointer events,** capture both halves: the webhook body AND the response from calling its `resource_url`. Tracking webhooks are inline, so just the one capture.

6. **Update the README.** Add your new scenario to the Scenarios table in the relevant folder's `README.md`.

7. **Check the TODO.** See if there are known placeholders you're filling in or sanitization gaps you're addressing — update `TODO.md` if so.

## Reporting issues

- **Fixtures look stale or wrong?** Open an issue with the file path and what's incorrect.
- **Found real data (names, addresses, tracking)?** See `SECURITY.md`.
- **Carrier behavior difference?** Document it in the relevant folder's `README.md` (not the fixture).

## Questions?

Each folder has a `README.md` with carrier-specific notes and payload documentation. Start there, then check the root `README.md` for repo structure and the `TODO.md` for what's still needed.
