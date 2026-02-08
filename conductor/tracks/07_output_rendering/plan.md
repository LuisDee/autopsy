# Plan: Output Rendering Overhaul [checkpoint: 8009368]

## Phase 1: Output Rendering Guide (Reference Document)

- [x] Task 1.1: Create output rendering guide reference document
  - [ ] Create `deep-review/references/` directory
  - [ ] Write `deep-review/references/output-rendering.md` based on full-output-spec.md
  - [ ] Adapt content for LLM instruction context (DD-1: spinner as preference hint, DD-5: named colors)
  - [ ] Include all 4 output states with exact template text
  - [ ] Include symbol set, color palette (named), anti-patterns list
  - [ ] Include final summary rules (severity line, critical-only display, 3-line format)
  - [ ] Include DD-4 fallback: top 3 high-severity when zero criticals
  - [ ] Include DD-2: batch-level progress with agent pending/complete dots (no per-agent bars)

- [x] Task 1.2: Validate guide against full-output-spec.md
  - [ ] Cross-reference every section of full-output-spec.md against the new guide
  - [ ] Verify all core rules present (no filler, phase collapsing, symbols, colors, agent display)
  - [ ] Verify all 4 states match (Discovery, Review, Synthesis, Complete)
  - [ ] Verify anti-patterns list complete
  - [ ] Verify final summary rules match (severity line, critical issues, next steps, reports)

- [x] Task: Conductor - User Manual Verification 'Phase 1: Output Rendering Guide' (Protocol in workflow.md)

## Phase 2: full-review.md Terminal Output Overhaul

- [x] Task 2.1: Replace phase transition output in full-review.md
  - [ ] Add reference instruction to output rendering guide at top of file
  - [ ] Replace Step 3 output (setup) — remove any filler, use dense format
  - [ ] Replace Step 5 output (discovery complete) — collapsed discovery line with ✓
  - [ ] Replace Step 6 batch output — active review line with ⠹ spinner, batch info
  - [ ] Add agent status display after batch agents complete (✓/✗ dots, no per-agent bars)
  - [ ] Replace Step 7 output (review complete) — collapsed review line
  - [ ] Replace Step 8 output (synthesis start) — collapsed phases + active synthesis line

- [x] Task 2.2: Replace final summary output in full-review.md (Step 10)
  - [ ] Remove double-line box (═══) entirely
  - [ ] Replace with: three collapsed ✓ lines + thin divider + "Deep Review Complete — {time} total"
  - [ ] Add severity counts line: `{N} critical · {N} high · {N} medium · {N} low     {total} issues`
  - [ ] Add named color instructions for severity counts (red, orange, yellow, dim)
  - [ ] Remove emoji severity markers (🔴🟠🟡🔵) from all terminal output
  - [ ] Remove duration breakdown section (already in collapsed lines)
  - [ ] Remove post-completion prose paragraph

- [x] Task 2.3: Add critical-only terminal display logic (FR-5)
  - [ ] Add instruction to read REVIEW_REPORT.md Critical Issues section after synthesis
  - [ ] Add `── Critical ──` section header (thin divider)
  - [ ] Display each critical issue in 3-line format: number + title, impact, → file paths
  - [ ] Add DD-4 fallback: if zero criticals, show top 3 high issues under `── High ──`
  - [ ] Add fallback: if zero criticals AND zero high, print "No critical or high issues found."

- [x] Task 2.4: Replace next steps and reports sections
  - [ ] Replace next steps with numbered, imperative format (no "Consider" or "You should")
  - [ ] Replace reports section with → arrows + filename + size/count
  - [ ] Remove any remaining filler text or narration

- [x] Task 2.5: Update progress.md template (FR-4)
  - [ ] Replace progress.md template in Step 6b with v2 dense format
  - [ ] Add `<!-- progress v2 -->` version marker
  - [ ] Use key-value format: phase, batch, completed, severity counts, failures
  - [ ] Remove emoji from progress.md template (🔴🟠🟡🔵)

- [x] Task 2.6: Update compaction recovery for dual-format progress.md
  - [ ] Add version marker detection to Compaction Recovery section
  - [ ] Add parsing logic for v2 format (key-value)
  - [ ] Preserve existing v1 parsing logic (Markdown with emoji)
  - [ ] Document both formats in recovery instructions

- [x] Task: Conductor - User Manual Verification 'Phase 2: full-review.md Overhaul' (Protocol in workflow.md)

## Phase 3: maintain-docs.md Output Overhaul

- [x] Task 3.1: Replace maintain-docs summary output
  - [ ] Add reference instruction to output rendering guide
  - [ ] Replace double-line box (═══) with: `Docs Maintenance Complete`
  - [ ] Use thin dividers (`──`) for section headers
  - [ ] Remove all emoji from terminal output
  - [ ] Replace verbose summary with dense line: `{N} updated · {N} created · {N} missing`
  - [ ] Replace file lists with → arrow format
  - [ ] Replace next steps with imperative format

- [x] Task: Conductor - User Manual Verification 'Phase 3: maintain-docs.md Overhaul' (Protocol in workflow.md)

## Phase 4: Validation and Cross-Reference

- [x] Task 4.1: Cross-reference all changes against full-output-spec.md
  - [ ] Verify full-review.md output matches all 4 states in full-output-spec.md
  - [ ] Verify final summary matches STATE 4 example exactly
  - [ ] Verify all anti-patterns are eliminated from full-review.md and maintain-docs.md
  - [ ] Verify no emoji severity markers remain in any terminal output instruction
  - [ ] Verify no double-line boxes remain
  - [ ] Verify no filler text patterns remain
  - [ ] Verify agent definitions, batch findings, and REVIEW_REPORT.md are NOT modified (NFR-1, NFR-3)

- [x] Task 4.2: Verify plugin structure and loading
  - [ ] Run `find deep-review/ -type f | sort` to verify file structure
  - [ ] Validate plugin.json: `python3 -c "import json; json.load(open('deep-review/.claude-plugin/plugin.json'))"`
  - [ ] Verify references/ directory is properly placed within plugin

- [x] Task: Conductor - User Manual Verification 'Phase 4: Validation' (Protocol in workflow.md)
