## What's this PR for?

- [ ] New fixture(s)
- [ ] Updated/fixed fixture
- [ ] Documentation update
- [ ] Other:

## Fixture checklist

If adding/updating fixtures, confirm:

- [ ] Fixture is properly sanitized (see `CONTRIBUTING.md`)
- [ ] `_captured_at` date is set (format: `YYYY-MM-DD`)
- [ ] `_fixture_notes` key is removed (if it was a placeholder)
- [ ] Folder's `README.md` updated with new scenario
- [ ] Related `TODO.md` item marked as done (if applicable)
- [ ] No real tracking numbers, names, addresses, or customer data
- [ ] Tracking numbers and `se-` IDs masked with length preserved, and `tracking_url` / `resource_url` match the masked values
- [ ] `latitude` / `longitude` rounded to 2 decimal places
- [ ] Fixture is a real capture, not trimmed or edited from another payload
- [ ] If the shape differs from other fixtures in the folder, the folder's `README.md` says which are older
- [ ] For webhook pointer events: both webhook body + `resource_url` response included

## Details

What changed, and why? For new fixtures, which scenario/event does this cover?

## Reference

Link to related issue, if any: #

## Notes for reviewers

Anything reviewers should know (carrier quirks, schema differences, etc.)?
