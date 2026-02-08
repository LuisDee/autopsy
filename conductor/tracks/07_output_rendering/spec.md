<!-- ARCHITECT CONTEXT | Track: 07_output_rendering | Wave: 4 | CC: v1, v2 -->

## Cross-Cutting Constraints
- Agent Output Format (v1): Finding format uses emoji severity markers and specific header format. This track proposes replacing emoji markers with text-only severity labels in terminal output while preserving structured data for file output.
- Agent Output Format (v2 -- introduced by this track): Terminal-facing output uses text-only severity with color codes. File-persisted findings (batch-*/\*.md, REVIEW_REPORT.md) may retain or remove emoji per design decision DD-3.
- Compaction-Safe State Management (v1): progress.md format changes must remain parseable by the orchestrator's compaction recovery logic.
- Self-Review Step (v1): Agents must still verify output completeness. Output rendering changes must not break self-review file checks.

## Interfaces

### Owns
- Output Rendering Guide (new reference document: `deep-review/references/output-rendering.md`)
- Terminal output format for all 4 states (Discovery, Review, Synthesis, Complete)
- Final summary format (replaces current double-line box format in full-review.md)
- Progress bar format for review agents
- Severity line format ("N critical . N high . N medium . N low")

### Consumes
- `commands/full-review.md` (Track 05 -- modifies terminal output sections)
- `commands/maintain-docs.md` (Track 06 -- modifies summary output section)
- `.deep-review/state.json` (Track 05 -- reads finding counts for severity line)
- `.deep-review/progress.md` (Track 05 -- modifies format for dense display)
- Finding format shared schema (interfaces.md -- severity emoji removal in terminal)
- `REVIEW_REPORT.md` (Track 04 -- critical-only terminal display, full report to file)

## Dependencies
- Track 05_orchestrator_command: COMPLETE. Contains the primary output to be overhauled (terminal progress, batch summaries, final summary). Must be modified.
- Track 06_secondary_commands: COMPLETE. Contains maintain-docs summary output. Must be modified.
- Track 03_review_agents: COMPLETE. Agent finding format uses emoji severity markers. Terminal display of findings changes; file format may change per DD-3.
- Track 04_synthesizer_agent: COMPLETE. REVIEW_REPORT.md format and terminal summary. Critical-only display in terminal.

<!-- END ARCHITECT CONTEXT -->

# Spec: Output Rendering Overhaul

## Overview

Replace all terminal output in the deep-review plugin with a clean, dense CLI dashboard following Claude Code's native design language. The user's terminal shows a compact, professional dashboard — not a chatbot narrating its work. Completed phases collapse to a single line, the active phase shows a braille spinner, severity counts are color-coded text (no emoji), and only critical issues appear inline. Everything else goes to REVIEW_REPORT.md.

## Design Decisions (Resolved)

| # | Decision | Resolution |
|---|----------|------------|
| 1 | Spinner representation | LLM preference hint — specify braille character set as preferred output; LLM picks one frame per print, no animation |
| 2 | Progress bar granularity | Batch-level only — no per-agent progress bars. Agents shown as pending/complete status dots |
| 3 | Emoji in persisted files | Terminal-only change — keep emoji in batch findings and REVIEW_REPORT.md |
| 4 | Critical-only threshold | Fallback: show top 3 high-severity issues when zero criticals exist |
| 5 | Color specification | Named color instructions in Markdown (e.g., "print checkmark in green") |
| 6 | progress.md backward compat | Version marker — add format version, handle both old and new in recovery logic |
| 7 | Maintain-docs parity | Full parity — same design language across all commands |

## Functional Requirements

### FR-1: Output Rendering Guide Reference Document

Create `deep-review/references/output-rendering.md` containing the full output specification. Both `full-review.md` and `maintain-docs.md` reference this document to avoid duplicating format rules.

The guide defines:
- **Core rules:** No filler text, phase collapsing, symbol set, color palette (named), agent display format
- **4 output states:** Discovery, Review, Synthesis, Complete — with exact template text for each
- **Final summary format:** Collapsed phase lines → severity line → critical issues (3 lines each) → next steps → reports section
- **Anti-patterns:** Explicit list of banned patterns (double-line boxes, emoji severity, narration, process descriptions)

### FR-2: full-review.md Terminal Output Overhaul

Replace all terminal output instructions in `commands/full-review.md`:

**Phase transitions (all 4 states):**
- **Discovery (active):** `⠹ Discovery · scanning repository...` + pending markers for Review and Synthesis
- **Review (active):** Collapsed Discovery line + active Review with batch info + agent status (pending/complete dots, batch-level only) + pending Synthesis
- **Synthesis (active):** Collapsed Discovery + collapsed Review + active Synthesis with current step
- **Complete:** All three collapsed + final summary

**Batch progress (Step 6):**
- Print batch header: `⠹ Review · batch {N}/{total} · {files} files, ~{LOC} LOC`
- After agents complete: show 5 agents each with ✓ (complete) or ✗ (failed) status dot
- After batch: collapse to batch count update

**Final summary (Step 10):**
- Severity counts on one line: `{N} critical · {N} high · {N} medium · {N} low     {total} issues`
- Colors: critical in red, high in orange, medium in yellow, low in dim
- Show critical issues inline (3 lines each: number + title, impact, file paths with →)
- If zero criticals: show top 3 high-severity issues instead with `── High ──` header
- If zero criticals AND zero high: print `No critical or high issues found.`
- Next steps: numbered, imperative verbs, no "Consider" or "You should"
- Reports section: arrow + filename + size/count

**Remove entirely:**
- Double-line box (`═══`) around completion header
- Emoji severity markers (🔴🟠🟡🔵) in terminal
- Phase duration breakdown (already in collapsed ✓ lines)
- Agent descriptions / focus area lists
- Filler sentences ("Excellent!", "Let me provide you with a status update", etc.)
- Post-completion prose paragraphs re-explaining findings

### FR-3: maintain-docs.md Output Overhaul

Replace the summary output in `commands/maintain-docs.md` with the same design language:

- Replace double-line box with: `Docs Maintenance Complete — {time} total`
- Use thin dividers (`──`) not double-line (`═══`)
- No emoji in terminal output
- Dense format: `{N} updated · {N} created · {N} missing`
- File list with → arrows
- Imperative next steps

### FR-4: progress.md Format Update

Update the progress.md template in `full-review.md` (Step 6b) to use a versioned, dense format:

```
<!-- progress v2 -->
phase: review
batch: {N}/{total}
completed: {list}
critical: {N} | high: {N} | medium: {N} | low: {N}
failures: {list or none}
```

Add recovery logic in the Compaction Recovery section to detect and parse both v1 (current Markdown format with emoji) and v2 (dense key-value format with version marker).

### FR-5: Orchestrator Reading Critical Findings for Terminal Display

The orchestrator currently only counts findings via grep. To display critical issues in the terminal summary, the orchestrator (or synthesizer) must extract critical issue summaries.

Add to Step 9 (after synthesizer completes): Read REVIEW_REPORT.md, extract the Critical Issues section, and print each critical issue in the 3-line format (number + title, impact, file paths).

If zero criticals: extract top 3 High issues instead and display under `── High ──` header.

### FR-6: Reference Integration

Add to both `full-review.md` and `maintain-docs.md` a reference instruction at the top of their output sections:

> Follow the output rendering guide at `references/output-rendering.md` for all terminal output formatting. Use the exact symbols, colors, and patterns defined there. Never use banned patterns.

## Non-Functional Requirements

- **NFR-1:** All output changes are terminal-only. Persisted file formats (batch findings, REVIEW_REPORT.md, state.json) are unchanged.
- **NFR-2:** Compaction recovery must work with both old and new progress.md formats (version marker detection).
- **NFR-3:** No agent definition files are modified (agents still produce findings in the same format with emoji).
- **NFR-4:** Output rendering guide is the single source of truth for format — commands reference it, not duplicate it.

## Acceptance Criteria

1. `deep-review/references/output-rendering.md` exists and contains the full output specification (4 states, symbols, colors, anti-patterns)
2. `full-review.md` references the output rendering guide and uses its format for all terminal output
3. Terminal output during full-review uses ✓ for completed phases, ⠹ for active, ◇ for pending — no emoji
4. Final summary shows severity counts on one line without emoji, with named color instructions
5. Only critical issues (or top 3 high if zero criticals) appear in terminal — all others go to report only
6. Each critical issue in terminal is exactly 3 lines: title, impact, file path(s)
7. `maintain-docs.md` uses the same design language (thin dividers, no emoji, dense format)
8. `progress.md` uses v2 format with version marker; compaction recovery handles both v1 and v2
9. No double-line boxes (`═══`), no emoji severity markers, no filler text in any terminal output
10. Agent definitions, batch finding files, and REVIEW_REPORT.md are NOT modified

## Out of Scope

- Agent definition file changes (agents still produce emoji-marked findings)
- REVIEW_REPORT.md content structure changes
- Synthesizer deduplication/analysis logic
- Batch plan format
- state.json schema
- Runtime spinner animation (Claude Code is an LLM, not a UI framework)
- Per-agent progress bars (resolved as batch-level only)
