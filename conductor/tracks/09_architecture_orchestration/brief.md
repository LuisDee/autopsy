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

# Track 09: Architecture Orchestration

## What This Track Delivers

Integration of the 3 new architecture assessment agents (architect, researcher, architecture-synthesizer) into the existing full-review orchestrator command. This track adds Phase 2A (architect and researcher running in parallel alongside the existing code review batches), adds the architecture-synthesizer to Phase 3 (running in parallel with the existing code review synthesizer), updates the final summary to show both reports, registers the new agents in plugin.json, and updates README.md documentation. After this track, running `/deep-review:full-review` produces two complementary outputs: REVIEW_REPORT.md (code-level bugs) and ARCHITECTURE_REPORT.md (design-level assessment).

## Scope

### IN
- Modify `commands/full-review.md` to add Phase 2A: launch architect and researcher agents in parallel alongside Phase 2B code review batches
- Modify `commands/full-review.md` to add architecture-synthesizer to Phase 3: launch alongside existing code review synthesizer
- Modify `commands/full-review.md` to update the final summary (State 4 output) to include ARCHITECTURE_REPORT.md in the Reports section and architecture stats
- Update `.claude-plugin/plugin.json` to register 3 new agent files (architect.md, researcher.md, architecture-synthesizer.md)
- Update `README.md` to document the architecture assessment capability and ARCHITECTURE_REPORT.md output
- Extend `state.json` schema with architecture phase tracking fields
- Extend `progress.md` output with architecture assessment status lines
- Compaction recovery logic updated to handle architecture agent state

### OUT
- Agent definition content (Track 08 defines what architect.md, researcher.md, and architecture-synthesizer.md contain)
- ARCHITECTURE_REPORT.md content and format (Track 08 -- architecture-synthesizer agent generates this)
- Existing review agent modifications (no changes to bug-hunter, security-auditor, etc.)
- Existing synthesizer modifications (no changes to synthesizer.md)
- REVIEW_REPORT.md format (stays exactly as-is)
- Discovery agent changes (no changes -- architecture agents consume discovery.md as-is)
- maintain-docs command (no architecture integration for incremental updates)

## Key Design Decisions

These should be resolved with the developer during spec generation:

1. **Phase 2A execution model:** Should the architect and researcher agents run as background tasks (maximizing parallelism with code review batches) or as foreground tasks (sequential but with full tool access)?
   - Background: maximum speed since architecture agents run truly in parallel with code review. But MCP tools are unavailable in background agents, and background task notifications may not fire reliably (known issue #21048).
   - Foreground: agents get full tool access including MCP. But they must either run before code review batches (adding latency) or interleave with batch processing somehow.
   - Hybrid: launch architecture agents as foreground tasks in the same message as the first batch's 5 review agents (all 7 tasks run concurrently as parallel foreground calls).

2. **Architecture phase failure handling:** Should architecture assessment failure block the entire review pipeline, or should the code review continue and complete independently?
   - Blocking: ensures the user always gets both reports or neither. Simpler mental model. But wastes completed code review work if architecture fails.
   - Non-blocking (architecture optional): code review always completes and produces REVIEW_REPORT.md. Architecture failure is logged but doesn't prevent a successful code review. But the user might expect both reports and be surprised by a partial result.
   - Fallback: architecture failure produces a minimal ARCHITECTURE_REPORT.md noting the failure, similar to the existing synthesizer fallback pattern.

3. **State tracking granularity for architecture agents:** Should state.json track architecture as a single phase ("architecture_phase": "running") or track each agent individually ("architect_status": "complete", "researcher_status": "running", "arch_synthesizer_status": "pending")?
   - Single phase: simpler state schema, less recovery complexity. But can't distinguish between architect failure and researcher failure during recovery.
   - Per-agent: precise recovery (retry only the failed agent). But adds 3 new state fields and more complex recovery logic.
   - Middle ground: track Phase 2A and Phase 3A (architecture synthesis) as separate phases, but not individual agents within them.

4. **Phase 2A and 2B synchronization:** Should Phase 3 wait for ALL of Phase 2 (both code review batches AND architecture agents) to complete before starting, or should the code review synthesizer launch as soon as code review batches finish (even if architecture agents are still running)?
   - Wait for all: simpler — Phase 3 starts once, launches both synthesizers together. But code review synthesis is blocked waiting for architecture agents that may take longer.
   - Independent: code review synthesizer launches as soon as batches finish. Architecture-synthesizer launches when architect + researcher finish. Both run concurrently but may finish at different times. More complex final summary logic.
   - Architecture-synthesizer reads code review findings: if the architecture-synthesizer optionally consumes code review findings for cross-referencing, it must wait for the code review synthesizer. This creates a dependency chain.

5. **Plugin.json registration:** Should the 3 new agents be added as a flat list extension to the existing agents array, or should plugin.json be restructured to group agents by phase (discovery agents, review agents, architecture agents, synthesis agents)?
   - Flat list: minimal change, just append 3 entries. But agents array grows to 10 with no grouping.
   - Grouped: clearer organization, self-documenting. But plugin.json doesn't currently support grouping, and Claude Code may not recognize non-standard schema extensions.
   - Comments: add JSON comments (not valid JSON) or a separate agents-manifest file for documentation. Adds complexity for minimal benefit.

## Architectural Notes

- `commands/full-review.md` is currently 449 lines and is the most complex file in the plugin. Modifications must be surgical additions that preserve all existing logic (Phase 1 discovery, Phase 2 batch processing, Phase 3 synthesis, compaction recovery, error handling). A rewrite would risk breaking the working pipeline.
- The existing orchestrator runs ALL agents as foreground tasks. Architecture agents running as background tasks would introduce a new execution pattern that breaks consistency and adds untested code paths.
- Phase 2A and Phase 2B are independent pipelines -- architecture agents read source code directly from discovery.md, not from code review findings. Only the architecture-synthesizer optionally reads code review findings for cross-referencing.
- State must be written to disk BEFORE launching architecture agents, consistent with the compaction-safe design constraint. The compaction recovery logic must be extended to handle the new architecture states.
- The output rendering guide (references/output-rendering.md) defines 4 output states. Architecture integration requires updating States 2, 3, and 4 to show architecture progress alongside code review progress. This must use the same visual grammar (checkmarks, spinners, thin dividers, no emoji).
- Strict report separation: REVIEW_REPORT.md contains code-level bugs and issues. ARCHITECTURE_REPORT.md contains design-level assessment. They may cross-reference each other but must not duplicate findings. The final summary must make this distinction clear.
- The plugin.json agents array currently has 7 entries. Adding 3 more brings it to 10. The plugin system uses this array for agent discovery -- all 10 agents will be available to the Task tool.
- README.md must explain to users that full-review now produces two reports and what each contains. This is a user-facing documentation change, not just an internal one.

## Test Strategy

- **No automated test framework** -- the plugin is markdown-based with no executable code to unit test
- **Plugin loading validation:** Run `claude --plugin-dir ./deep-review` and verify all 10 agents are recognized (7 existing + 3 new architecture agents)
- **Integration test:** Run `/deep-review:full-review` against a sample repository and verify:
  - Both REVIEW_REPORT.md and ARCHITECTURE_REPORT.md are generated
  - Architecture agents run during Phase 2A (not blocking code review batches)
  - Architecture-synthesizer runs during Phase 3
  - Final summary includes both reports in the Reports section
- **Error recovery test:** Simulate architecture agent failure (e.g., delete architecture-analysis.md before synthesizer runs) and verify:
  - Code review completes independently
  - Failure is logged to state.json and progress.md
  - Final summary reflects partial results gracefully
- **Compaction recovery test:** Trigger compaction during Phase 2A and verify orchestrator resumes correctly by re-reading state.json
- **Key test scenarios:**
  1. Happy path: both pipelines complete, two reports generated, final summary shows both
  2. Architecture agent failure: code review succeeds, architecture fails gracefully
  3. Compaction during architecture phase: state recovery works for both pipelines
  4. Plugin.json validity: JSON parses correctly, all 10 agents listed
  5. Output rendering compliance: architecture phase lines follow output-rendering.md patterns

## Complexity: M
## Estimated Phases: ~3
