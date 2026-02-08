# Track 10: Rebrand deep-review to Autopsy

## Scope
Pure rename/rebrand of the plugin from "deep-review" to "Autopsy". No functionality changes.

## Key Changes
- Rename `deep-review/` directory to `autopsy/`
- Rename `deep-review-plugin-build-plan.md` to `autopsy-plugin-build-plan.md`
- Delete transient `.deep-review/` output directory
- Bulk replace ~595 text references across 62 files
- Update plugin.json name field

## Design Decisions
- Ordered sed replacements (most-specific first) to avoid partial matches
- Single atomic commit for the entire rebrand
- No functionality changes — pure text substitution

## Dependencies
None — this is a standalone cosmetic change.
