<!-- ARCHITECT CONTEXT | Track: 05_orchestrator_command | Wave: 3 | CC: v1 -->

## Cross-Cutting Constraints
- Compaction-Safe State Management: write state to disk before every risk point
- Error Recovery: log failures, optionally retry (max 1), continue with remaining batches
- Batch Sizing Constraints: adaptive sizing based on repo size

## Dependencies
- Track 02: discovery agent defined
- Track 03: all 5 review agents defined
- Track 04: synthesizer agent defined

<!-- END ARCHITECT CONTEXT -->

# Specification: Orchestrator Command

## Design Decisions Applied
- DD1: Background parallel agents (5 per batch, run_in_background: true)
- DD2: Sequential batch processing (one batch at a time)
- DD3: Auto-compaction + state writes (write state.json before every batch)
- DD4: Full instructions in each agent's Task prompt
- DD5: Detailed progress output (summary after each batch)
- DD6: Continue on failure + note gaps (retry once, then continue)
- DD7: Per-phase timing breakdown

## Acceptance Criteria
- [ ] `commands/full-review.md` has valid YAML frontmatter
- [ ] Setup phase creates .autopsy/, updates .gitignore, writes state.json
- [ ] Phase 1 launches discovery agent and reads batch plan
- [ ] Phase 2 launches 5 agents in parallel per batch with full instructions
- [ ] Batch sizing strategy matches repo size table
- [ ] Error handling with retry logic documented
- [ ] Compaction recovery logic documented
- [ ] Phase 3 launches synthesizer and verifies REVIEW_REPORT.md
- [ ] Final summary matches build plan format with per-phase timing
- [ ] state.json and progress.md updated at every step
