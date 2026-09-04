---
name: Fixture Issue
about: Report stale, incorrect, or incomplete fixture data
title: "[FIXTURE] "
labels: fixture
---

## Which fixture?

Path: `webhooks/` or `api/` / ...

## What's the issue?

- [ ] Fixture doesn't match current ShipStation behavior
- [ ] Real customer data found (see SECURITY.md)
- [ ] Schema mismatch (field missing, type wrong, etc.)
- [ ] Data looks outdated (`_captured_at` date?)
- [ ] Other:

## Details

Describe what you expected vs. what the fixture shows. If comparing to current behavior, include the ShipStation API response or webhook body you're testing against.

## How to fix

(Optional) What should the fixture show instead?
