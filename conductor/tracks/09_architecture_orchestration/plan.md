# Track 09: Architecture Orchestration — Implementation Plan

> **Note:** This track modifies existing Markdown files. "Tests" are structural validations (JSON parsing, section presence, output format checks) rather than unit tests.

---

## Phase 1: Plugin Registration and Documentation

### [x] Task 1.1: Update plugin.json
- Add 3 new agent entries to the `agents` array in `deep-review/.claude-plugin/plugin.json`
- Append: `./agents/architect.md`, `./agents/researcher.md`, `./agents/architecture-synthesizer.md`
- Validate: `python3 -c "import json; d=json.load(open('deep-review/.claude-plugin/plugin.json')); assert len(d['agents']) == 10"`

### [x] Task 1.2: Update README.md
- Update "What It Does" to mention both REVIEW_REPORT.md and ARCHITECTURE_REPORT.md
- Add ARCHITECTURE_REPORT.md to the "What Gets Generated" table
- Update Architecture Overview diagram to show Phase 2A (architect + researcher) and Phase 3A (architecture-synthesizer)
- Update Cost Expectations: add 3 to "Agents Spawned" (fixed overhead: discovery + synthesis + architect + researcher + architecture-synthesizer = 5 fixed)
- Add brief section explaining the two complementary reports
- Follow output-rendering.md conventions in any example output shown

### [x] Task 1.3: Commit plugin registration and docs eb75af9
- Commit message: `feat(plugin): Register 3 architecture agents and update README documentation`

---

## Phase 2: Orchestrator Modifications — Phase 2A

### [x] Task 2.1: Add Phase 2A to full-review.md
- Insert new content after Step 5a (recent changes context), before existing Step 6 (batch processing)
- Add Step 5b: "Launch architecture agents alongside first batch"
  - Write `phase_2a_status: "running"` to state.json before launching
  - Modify Step 6c for the FIRST batch only: launch 7 agents (5 review + architect + researcher) in a single message
  - Add architect agent prompt (read discovery.md, follow agent definition, write to `.deep-review/architecture-analysis.md`)
  - Add researcher agent prompt (read discovery.md, follow agent definition, write to `.deep-review/best-practices-research.md`)
- Add Step 6d-arch: Verify architecture output files after first batch completes
  - If missing, log to agent_failures, retry once
  - Update `phase_2a_status` to "complete" or "failed"
  - Continue regardless — never block code review

### [x] Task 2.2: Update Phase 2 output rendering
- Update State 1 (Discovery) to show 4 pending lines: Discovery, Review, Architecture, Synthesis
- Update State 2 (Review active) to show architecture status alongside review progress
- Add collapsed architecture line (`✓ Architecture · {time}` or `✗ Architecture · failed`)
- Update the "Next" pending line to mention both reports

### [x] Task 2.3: Update state.json schema in full-review.md
- Extend the initial state (Step 2) with: `phase_2a_status`, `phase_3a_status`, `architecture_analysis_exists`, `best_practices_research_exists`, `architecture_report_exists`
- Extend progress.md v2 format with `arch:` field
- Update state writes throughout the document to maintain architecture fields

### [x] Task 2.4: Commit Phase 2A modifications 92d7b2b
- Commit message: `feat(orchestrator): Add Phase 2A architecture agents alongside first review batch`

---

## Phase 3: Orchestrator Modifications — Phase 3, Summary, Recovery

### [x] Task 3.1: Modify Phase 3 synthesis
- Update Step 8: launch BOTH synthesizers (code review + architecture) in a single message as 2 parallel foreground Task calls
- Add architecture-synthesizer prompt: follow agent definition, read all inputs from `.deep-review/`, write `ARCHITECTURE_REPORT.md` to repo root
- If `phase_2a_status` is "failed", prepend missing-input handling instruction to architecture-synthesizer prompt
- Update Step 9: verify both `REVIEW_REPORT.md` and `ARCHITECTURE_REPORT.md` exist
- Add fallback for missing ARCHITECTURE_REPORT.md (minimal report noting failure + listing available partial data)
- Update `phase_3a_status` in state.json after synthesis

### [x] Task 3.2: Update final summary (Step 10)
- Add ARCHITECTURE_REPORT.md to Reports section (omit if architecture failed)
- Update Next steps to include "Read ARCHITECTURE_REPORT.md for design-level assessment"
- Update State 3 (Synthesis active) output to show both synthesizers
- Update State 4 (Complete) output format
- Conditionally adjust next steps if architecture failed

### [x] Task 3.3: Extend compaction recovery
- Add `phase_2a_status` and `phase_3a_status` handling to compaction recovery section
- Recovery for phase_2a "running": check for output files, mark complete or failed
- Recovery for phase_3a "running": check for ARCHITECTURE_REPORT.md, retry if Phase 2 complete
- Architecture state never blocks code review recovery

### [x] Task 3.4: Extend error handling table
- Add 5 new rows for architecture-specific errors:
  - Architect agent output missing
  - Researcher agent output missing
  - Architecture-synthesizer fails
  - ARCHITECTURE_REPORT.md missing
  - Architecture compaction recovery

### [x] Task 3.5: Commit Phase 3 and recovery modifications c32338c
- Commit message: `feat(orchestrator): Add architecture synthesis, summary, and compaction recovery`

---

## Phase 4: AGENTS.md Updates and Validation

### [x] Task 4.1: Update AGENTS.md files
- Update `deep-review/AGENTS.md`: agent count 7→10, add architecture phases to data flow, mention ARCHITECTURE_REPORT.md
- Update `deep-review/commands/AGENTS.md`: update full-review.md description (4 phases), update data flow

### [x] Task 4.2: Cross-validation
- Verify full-review.md references all 3 architecture agent names correctly
- Verify state.json schema is consistent across initial state, updates, and recovery
- Verify progress.md v2 format includes arch field
- Verify output rendering follows output-rendering.md for all new output states
- Verify all existing Phase 1/2/3 logic is preserved (no regressions)

### [x] Task 4.3: Plugin loading validation
- Run `claude --plugin-dir ./deep-review` and verify no errors
- Validate plugin.json: `python3 -c "import json; d=json.load(open('deep-review/.claude-plugin/plugin.json')); assert len(d['agents']) == 10; print('OK:', d['agents'])"`
- Verify full-review.md is under a reasonable size (will grow from 449 lines but should stay manageable)

### [x] Task 4.4: Final commit and track completion
- Commit any validation fixes
- Commit message: `conductor(checkpoint): Complete Track 09 Architecture Orchestration`

### Phase Completion Verification
- plugin.json lists 10 agents and parses as valid JSON
- README.md documents both reports with updated architecture diagram
- full-review.md launches architect + researcher alongside first batch
- full-review.md launches both synthesizers in parallel in Phase 3
- Final summary includes ARCHITECTURE_REPORT.md in Reports section
- Compaction recovery handles architecture state
- Architecture failure never blocks code review pipeline
- All output follows output-rendering.md conventions
- AGENTS.md files reflect the new architecture orchestration
