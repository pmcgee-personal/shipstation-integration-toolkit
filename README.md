# ShipStation Integration Toolkit

**Unofficial.** Real ShipStation webhook payloads, API
request/response examples and short diagram-first explainers for common
integration flows — a working companion to the
[official documentation](https://docs.shipstation.com/), for the cases
where it helps to see an actual payload.

## Why this exists

Started with USPS tracking, where sandbox directs callers to production and
so can't show you the real response shapes. The same need comes up
elsewhere: some scenarios are easier to understand from a concrete example
than from a schema, and a few flows get re-explained by email every time
they come up. This repo collects real examples and short reference docs so
integrators have something specific to link to and test against.

## Structure

```
webhooks/           Real webhook payloads, by resource
  orders/
  shipments/
  tracking/
api/                Real API request/response examples, by resource
  adjustments/      USPS shipping adjustment reports
  shipments/
    usps-single-payor/   USPS single payor labels (PCID compliance)
  tracking/
    usps/
patterns/           Short, diagram-first explanations of common flows
```

- **`webhooks/`** — what ShipStation actually sends to a webhook subscriber
  for a given event. Useful for building/testing your webhook receiver
  without needing to trigger the real event in sandbox.
- **`api/`** — what ShipStation actually returns when you call an endpoint
  directly. Useful for building/testing request handling without a live
  sandbox call, and for scenarios sandbox doesn't cover (e.g. USPS
  tracking today).
- **`patterns/`** — a diagram (Mermaid, renders natively on GitHub) for a
  commonly-requested flow, plus links to the official docs and any relevant
  fixture in this repo. No duplicated payloads or long prose — see
  [`patterns/README.md`](./patterns/README.md).

Webhooks and API responses for the same resource are _similar but not
identical_ — don't assume a webhook payload and an API response for the same
object share an exact schema. They're kept in separate trees on purpose.

## How to use

**Reference directly** — import a fixture JSON into your test suite as
expected/mock data for a given scenario.

**Serve from a mock endpoint** — point a local mock server (json-server,
WireMock, a small Express/Flask stub) at the relevant folder to stand in for
the live endpoint or to replay a webhook delivery.

## Adding a new fixture

1. Pull the real payload/response from production.
2. Sanitize it — replace tracking numbers, order IDs, names, addresses, and
   anything else identifying with clearly fake but correctly-formatted
   values.
3. Save it under the right resource folder with a self-explanatory filename
   (see existing files for the naming pattern).
4. Add a `_captured_at` key to the fixture with the capture date
   (`YYYY-MM-DD`), and remove the `_fixture_notes` key if one is present.
   `_fixture_notes` means "still a placeholder, real data needed" — a
   description of what the endpoint does belongs in the folder's
   `README.md` instead.

See [`TODO.md`](./TODO.md) for the current list of placeholders that still
need real data.

## Freshness

Fixtures are snapshots, not a guaranteed-current contract — ShipStation and
carrier schemas both change over time. Each filled-in fixture carries a
`_captured_at` date; fixtures still showing `_fixture_notes` are
placeholders awaiting real data. If something looks stale or wrong, open an
issue.

## Status

Unofficial, community-maintained. A practical supplement to
[ShipStation's own documentation](https://docs.shipstation.com/), not a
replacement for it — start there, and use this repo when you want a real
payload to build against.
