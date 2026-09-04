# Security Policy

## Reporting a Vulnerability

If you discover real customer data (names, addresses, real tracking numbers, API tokens, email addresses, phone numbers) in any fixture, please **do not open a public issue**.

Instead, email your findings to [maintainer email] with:
- The file path and line numbers
- What the real data is (e.g., "real tracking number", "real phone number")
- How you discovered it (if contributing a new fixture, etc.)

All fixture data should be sanitized according to `CONTRIBUTING.md`. Real data in a fixture is considered a bug that should be fixed before public visibility.

## What's safe to discuss publicly

- **Wrong/stale schema.** If a fixture doesn't match current ShipStation behavior, open an issue.
- **Missing carriers or event types.** File an issue requesting coverage.
- **Documentation clarity.** Typos, unclear explanations, etc.

## What data is already public

Fixtures include:
- ShipStation's actual JSON schema (structure, field names, types)
- Carrier codes and status codes
- Store/account-level identifiers (`store_id`, `carrier_id`, etc.)
- Example formatting for dates, amounts, and structures

This is intentional — the whole point is to show real payload shapes for integration testing.
