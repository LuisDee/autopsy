# Track 10: Rebrand deep-review to Autopsy — Plan

## Phase 1: Structural Renames
- [x] `git mv deep-review/ autopsy/`
- [x] `git mv deep-review-plugin-build-plan.md autopsy-plugin-build-plan.md`
- [x] `rm -rf .deep-review/`

## Phase 2: Bulk Text Replacement
- [x] Replace all text references using ordered patterns (most-specific first)

## Phase 3: Verification
- [x] Zero remaining "deep-review" references (grep)
- [x] Valid plugin.json with name "autopsy"
- [x] All idempotency markers say "autopsy"
- [x] All slash commands use `/autopsy:` format

## Phase 4: Conductor Track
- [x] Create track metadata, brief, and plan

## Phase 5: Commit
- [x] Single atomic commit
