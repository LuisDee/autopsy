# Track 09: Architecture Orchestration — Specification

<!-- ARCHITECT CONTEXT | Track: 09_architecture_orchestration | Wave: 5 | CC: v1 -->

## Cross-Cutting Constraints
- Compaction-Safe State Management (v1): Write architecture phase state to disk before launching architecture agents. After compaction, re-read state.json + progress.md to determine architecture agent status and resume.
- Error Recovery (v1): If architecture agents fail (output file missing), log failure, retry once, continue pipeline. Architecture phase failure must not block existing code review pipeline.
- Output Rendering (v1): All terminal output for architecture phases must follow references/output-rendering.md. Phase collapsing, symbols, colors, anti-patterns all apply to the new architecture phase lines.

## Interfaces

### Owns
- Phase 2A orchestration logic (launching architect + researcher agents in parallel alongside Phase 2B code review)
- Architecture-synthesizer launch in Phase 3 (in parallel with existing code review synthesizer)
- Updated final summary format (showing both REVIEW_REPORT.md and ARCHITECTURE_REPORT.md)

### Consumes
- `agents/architect.md` (from Track 08 — agent definition launched by orchestrator)
- `agents/researcher.md` (from Track 08 — agent definition launched by orchestrator)
- `agents/architecture-synthesizer.md` (from Track 08 — agent definition launched by orchestrator)
- `.deep-review/discovery.md` (from Track 02 — repo map passed to architecture agents)
- `.deep-review/architecture-analysis.md` (from Track 08 — output verified before launching architecture-synthesizer)
- `.deep-review/best-practices-research.md` (from Track 08 — output verified before launching architecture-synthesizer)
- `.deep-review/ARCHITECTURE_REPORT.md` or `ARCHITECTURE_REPORT.md` (from Track 08 — verified after architecture-synthesizer completes)
- `state.json` (from Track 05 — extended with architecture phase tracking)
- `progress.md` (from Track 05 — extended with architecture status)
- `references/output-rendering.md` (from Track 07 — governs all terminal output)

## Dependencies
- Track 08_architecture_agents: Agent definitions must exist (architect.md, researcher.md, architecture-synthesizer.md). This track launches them but does not define them.
- Track 05_orchestrator_command: COMPLETE. The orchestrator command (commands/full-review.md) is the file being modified. Must preserve all existing orchestration logic.
- Track 07_output_rendering: COMPLETE. Output rendering guide and conventions are established. Architecture output must follow the same design language.

<!-- END ARCHITECT CONTEXT -->

---

## Design Decisions

| ID | Decision | Choice | Rationale |
|----|----------|--------|-----------|
| DD-1 | Phase 2A execution model | Hybrid | Architecture agents (architect + researcher) launch as foreground tasks in the same message as the first batch's 5 review agents. All 7 tasks run concurrently as parallel foreground calls. Subsequent batches run normally with 5 agents. |
| DD-2 | Architecture phase failure handling | Non-blocking with fallback report | Architecture agent failure never blocks the code review pipeline. If agents fail, produce a minimal ARCHITECTURE_REPORT.md noting the failure with whatever partial data is available (e.g., "architect agent failed, researcher completed — here's what we got"). |
| DD-3 | State tracking granularity | Middle ground (phase-level) | Track `phase_2a_status` (running/complete/failed) and `phase_3a_status` (running/complete/failed) as separate state fields. Don't track individual agents within Phase 2A — if one fails, the synthesizer handles missing inputs gracefully per its agent definition. |
| DD-4 | Phase 2A/2B synchronization | Wait for all | Phase 3 starts only after ALL of Phase 2 completes (both code review batches AND architecture agents). Both synthesizers launch together in a single message as parallel foreground calls. Simpler, consistent with existing pattern. |
| DD-5 | Plugin.json registration | Flat list | Append 3 new agent entries to the existing agents array. No grouping or restructuring. |

---

## Functional Requirements

### FR-1: Modify `commands/full-review.md` — Phase 2A Integration

**What:** Add architecture agent launching to Phase 2, so architect and researcher agents run in parallel with the first batch of review agents.

**Where:** After Step 5a (recent changes context), before the existing Phase 2 batch loop (Step 6).

**Behavior:**
1. After discovery completes and batch plan is read, launch architect + researcher agents alongside the first batch's 5 review agents — all 7 in a single message with 7 Task calls.
2. Architect agent prompt: tell it to read `.deep-review/discovery.md` for repo context, then follow its agent definition (`agents/architect.md`), and write output to `.deep-review/architecture-analysis.md`.
3. Researcher agent prompt: tell it to read `.deep-review/discovery.md` for technology identification, then follow its agent definition (`agents/researcher.md`), and write output to `.deep-review/best-practices-research.md`.
4. After the first batch (all 7 agents) returns, verify architecture output files exist alongside the normal batch output verification.
5. If architecture output files are missing, log to `state.json` under `agent_failures`, retry once. If still missing after retry, continue — the architecture-synthesizer handles missing inputs gracefully.
6. For subsequent batches (batch 2+), only launch the 5 review agents as normal.

**State updates:**
- Before launching first batch: write `"phase_2a_status": "running"` to `state.json`
- After first batch completes: update `"phase_2a_status": "complete"` or `"phase_2a_status": "failed"` based on output file existence
- Add `architecture_analysis_exists` and `best_practices_research_exists` boolean flags to state.json

**Output rendering:**
- State 1 (Discovery) adds a pending architecture line:
  ```
  ⠹ Discovery · scanning repository...
  ◇ Review
  ◇ Architecture
  ◇ Synthesis
  ```
- State 2 (Review active) shows architecture status alongside review:
  ```
  ✓ Discovery · {N} files across {M} modules · {time}
  ──────────────────────────────────────────────────
  ⠹ Review · batch {N}/{total} · {files} files, ~{LOC} LOC
  ⠹ Architecture · analyzing...

  ◇ Next: synthesize → REVIEW_REPORT.md + ARCHITECTURE_REPORT.md
  ```
- When architecture completes (possibly before review finishes):
  ```
  ✓ Architecture · {time}
  ```
- When architecture fails:
  ```
  ✗ Architecture · failed (partial report available)
  ```

### FR-2: Modify `commands/full-review.md` — Phase 3 Integration

**What:** Launch the architecture-synthesizer alongside the existing code review synthesizer in Phase 3.

**Where:** Step 8 (synthesis launch).

**Behavior:**
1. Phase 3 waits for ALL of Phase 2 to complete (all code review batches AND architecture agents).
2. Launch both synthesizers in a single message as 2 parallel foreground Task calls:
   - Existing code review synthesizer (unchanged prompt from current Step 8)
   - Architecture-synthesizer: tell it to follow its agent definition (`agents/architecture-synthesizer.md`), read all inputs from `.deep-review/`, and write `ARCHITECTURE_REPORT.md` to the repo root
3. If `phase_2a_status` is "failed", still launch architecture-synthesizer but prepend to its prompt: "Some architecture analysis inputs may be missing. Follow your missing-input handling logic to produce a partial report."
4. After both synthesizers return, verify both output files exist (`REVIEW_REPORT.md` and `ARCHITECTURE_REPORT.md`).
5. If `ARCHITECTURE_REPORT.md` is missing, retry architecture-synthesizer once. If still missing, create a minimal fallback:
   ```
   # Architecture Report

   Architecture synthesis was unable to complete.

   ## Partial Results
   - architecture-analysis.md: {exists/missing}
   - best-practices-research.md: {exists/missing}

   ## Raw Data
   Available inputs are in `.deep-review/`. Run a full review again to regenerate.
   ```

**State updates:**
- Before launching synthesizers: write `"phase_3a_status": "running"` to state.json
- After synthesizers complete: update `"phase_3a_status": "complete"` or `"phase_3a_status": "failed"`

**Output rendering:**
- State 3 (Synthesis active) shows both synthesizers:
  ```
  ✓ Discovery · {N} files across {M} modules · {time}
  ✓ Review · 5 agents · {time}
  ✓ Architecture · {time}
  ──────────────────────────────────────────────────
  ⠹ Synthesis · deduplicating + architecture report
  ```

### FR-3: Modify `commands/full-review.md` — Final Summary Update

**What:** Update the final summary (Step 10) to include ARCHITECTURE_REPORT.md in the Reports section and architecture stats.

**Where:** Step 10 (completion summary).

**Behavior:**
1. Read statistics from both `REVIEW_REPORT.md` and `ARCHITECTURE_REPORT.md`.
2. Updated Reports section:
   ```
   ── Reports ───────────────────────────────────────

   → REVIEW_REPORT.md         full code review findings
   → ARCHITECTURE_REPORT.md   design assessment + recommendations
   → AGENTS.md (×{N})         module documentation
   → .deep-review/            raw findings + state
   ```
3. If `ARCHITECTURE_REPORT.md` doesn't exist (architecture failed), omit the line.
4. Updated Next steps:
   ```
   ── Next ──────────────────────────────────────────

   1. Fix critical issues first
   2. Read REVIEW_REPORT.md for code-level details
   3. Read ARCHITECTURE_REPORT.md for design-level assessment
   4. Run /deep-review:maintain-docs after changes
   ```
5. If architecture failed, replace step 3 with: "Re-run full review to regenerate architecture assessment"

### FR-4: Modify `commands/full-review.md` — Compaction Recovery Extension

**What:** Extend the compaction recovery logic to handle architecture agent state.

**Where:** Compaction Recovery section (after Step 10).

**Behavior:**
1. When recovering from compaction, read `phase_2a_status` and `phase_3a_status` from `state.json`.
2. Extended recovery logic:
   - If `phase_2a_status` is "running": check for output files (`architecture-analysis.md`, `best-practices-research.md`). If they exist, set to "complete". If missing, set to "failed" and continue.
   - If `phase_3a_status` is "running": check for `ARCHITECTURE_REPORT.md`. If exists, set to "complete". If missing, check if Phase 2 code review is also complete — if so, retry architecture-synthesizer.
3. Architecture state should NOT block code review recovery — if architecture state is unclear, mark it failed and continue with code review pipeline.

### FR-5: Modify `commands/full-review.md` — Error Handling Table Extension

**What:** Add architecture-specific error scenarios to the error handling table.

**Where:** Error Handling Summary table at the end of full-review.md.

**New rows:**
| Error | Action |
|-------|--------|
| Architect agent output missing | Log failure, retry once. If still missing, mark phase_2a as failed, continue code review |
| Researcher agent output missing | Log failure, retry once. If still missing, mark phase_2a as failed, continue code review |
| Architecture-synthesizer fails | Retry once. If still fails, create minimal fallback ARCHITECTURE_REPORT.md |
| ARCHITECTURE_REPORT.md missing | Retry architecture-synthesizer. If still missing, create fallback report |
| Architecture compaction recovery | Check output files exist, mark phase complete or failed accordingly |

### FR-6: Update `plugin.json`

**What:** Add 3 new agent entries to the agents array.

**Change:**
```json
{
  "agents": [
    "./agents/discovery.md",
    "./agents/bug-hunter.md",
    "./agents/security-auditor.md",
    "./agents/error-inspector.md",
    "./agents/performance-detector.md",
    "./agents/stack-reviewer.md",
    "./agents/synthesizer.md",
    "./agents/architect.md",
    "./agents/researcher.md",
    "./agents/architecture-synthesizer.md"
  ]
}
```

### FR-7: Update `README.md`

**What:** Document the architecture assessment capability.

**Changes:**
1. Update the "What It Does" section to mention both REVIEW_REPORT.md and ARCHITECTURE_REPORT.md.
2. Update the "What Gets Generated" table to add ARCHITECTURE_REPORT.md.
3. Update the Architecture Overview diagram to show Phase 2A (architect + researcher) and Phase 3A (architecture-synthesizer) alongside existing phases.
4. Update Cost Expectations table — add 3 to "Agents Spawned" column (architect, researcher, architecture-synthesizer are fixed overhead per run, like discovery and synthesis).
5. Add brief section explaining the two complementary reports: code-level (REVIEW_REPORT.md) and design-level (ARCHITECTURE_REPORT.md).

### FR-8: Update AGENTS.md files

**What:** Update AGENTS.md files in `deep-review/` and `deep-review/commands/` to reflect the new architecture orchestration.

**Changes:**
- `deep-review/AGENTS.md`: Update agent count (7→10), add architecture phases to data flow, mention ARCHITECTURE_REPORT.md
- `deep-review/commands/AGENTS.md`: Update full-review.md description to mention 4 phases (Discovery, Review+Architecture, Synthesis), update data flow

---

## State Schema Extension

The existing `state.json` is extended with these fields:

```json
{
  "phase_2a_status": "pending|running|complete|failed",
  "phase_3a_status": "pending|running|complete|failed",
  "architecture_analysis_exists": false,
  "best_practices_research_exists": false,
  "architecture_report_exists": false
}
```

The `progress.md` v2 format is extended:
```
<!-- progress v2 -->
phase: review
batch: {N}/{total}
completed: 1, 2, 3
critical: {N} | high: {N} | medium: {N} | low: {N}
failures: none
arch: complete|failed|running
```

---

## Acceptance Criteria

1. Running `/deep-review:full-review` launches architect + researcher agents alongside the first batch's review agents (7 parallel foreground tasks)
2. Phase 3 launches both synthesizers in parallel after all of Phase 2 completes
3. Final summary includes ARCHITECTURE_REPORT.md in the Reports section
4. Architecture agent failure does NOT block code review completion
5. Minimal fallback ARCHITECTURE_REPORT.md is created when architecture-synthesizer fails
6. `plugin.json` lists all 10 agents and parses as valid JSON
7. `README.md` documents both reports and updated architecture diagram
8. Compaction recovery handles architecture state correctly
9. All output follows `references/output-rendering.md` conventions
10. All existing orchestration logic is preserved — no regressions in code review pipeline
