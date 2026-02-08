<!-- ARCHITECT CONTEXT | Track: 05_orchestrator_command | Wave: 3 | CC: v1 -->

## Cross-Cutting Constraints
- Compaction-Safe State Management: write state to disk before every risk point; re-read after compaction; use state.json + progress.md + batch-plan.md for resume
- Error Recovery: log failures, optionally retry (max 1), continue with remaining batches
- Batch Sizing Constraints: adaptive sizing based on repo size (small/medium/large/huge)

## Interfaces

### Owns
- `.deep-review/` working directory structure
- `.deep-review/state.json` (machine-readable progress)
- `.deep-review/progress.md` (human-readable progress)
- `.gitignore` update (add `.deep-review/`)
- Final terminal summary output

### Consumes
- `.deep-review/batch-plan.md` (from discovery agent via Track 02)
- `.deep-review/discovery.md` (from discovery agent via Track 02)
- `.deep-review/batch-{N}/*.md` (from review agents via Track 03 — existence check only)
- `REVIEW_REPORT.md` (from synthesizer via Track 04 — existence check only)

## Dependencies
- Track 02_discovery_agent: discovery agent must be defined (orchestrator launches it)
- Track 03_review_agents: all 5 review agents must be defined (orchestrator launches them)
- Track 04_synthesizer_agent: synthesizer must be defined (orchestrator launches it)

<!-- END ARCHITECT CONTEXT -->

# Track 05: Orchestrator Command

## What This Track Delivers

The main orchestrator command (`commands/full-review.md`) — the most complex and critical file in the plugin. When the user runs `/deep-review:full-review`, this command coordinates the entire 3-phase review: launches the discovery agent, reads the batch plan, launches 5 review agents in parallel per batch, tracks progress, handles context pressure, launches the synthesizer, and prints a final summary. The orchestrator never reads source code directly — it stays light by delegating all heavy work to sub-agents and using disk for coordination.

## Scope

### IN
- `commands/full-review.md` command definition
- Setup phase: create `.deep-review/`, update `.gitignore`, note start time, run `/effort max`
- Phase 1 orchestration: launch discovery agent (foreground), wait, read batch plan, print Phase 1 summary
- Phase 2 orchestration: for each batch — create batch directory, launch 5 agents in parallel (background), poll for output files, update progress, print batch summary
- Phase 3 orchestration: launch synthesizer (foreground), wait, confirm REVIEW_REPORT.md exists, print Phase 3 summary
- Batch sizing strategy (adaptive: small/medium/large/huge repos)
- Context pressure management: never read file contents, use `/compact` between batches for large repos, write state before compaction
- Compaction recovery: after any compaction, re-read state.json + batch-plan.md + progress.md to resume
- Error handling: agent failure detection (missing output files), retry logic (max 1), graceful continuation
- Final summary output (formatted terminal output with statistics)

### OUT
- Nothing — this is the top-level command that consumes everything else

## Key Design Decisions

These should be resolved with the developer during spec generation:

1. **Agent launch mode:** All review agents as background tasks with file polling vs foreground sequential per agent within a batch?
   Trade-off: Background parallel is faster but harder to debug; foreground sequential is simpler but slower
2. **Batch processing strategy:** Process all batches sequentially vs overlap batches (start batch N+1 while batch N agents finish)?
   Trade-off: Sequential is simpler and predictable; overlapping is faster but increases context pressure
3. **Compaction trigger:** Manual `/compact` between batches vs rely on auto-compaction at 95%?
   Trade-off: Manual gives control over what's preserved; auto is hands-off but may lose important instructions
4. **Agent prompting strategy:** Full instructions in the Task prompt vs reference agent definition files?
   Trade-off: Full instructions make agents self-contained; references are more maintainable but agents may not read the files correctly
5. **Progress output verbosity:** Print detailed progress after each batch vs minimal progress (only errors)?
   Trade-off: Detailed keeps the user informed if watching; minimal is cleaner for walk-away usage
6. **Failure handling granularity:** Fail the entire review on any critical agent failure vs continue and note gaps in the final report?
   Trade-off: Fail-fast ensures complete coverage; continue ensures partial results are always available
7. **Duration tracking:** Wall-clock time from start to finish vs per-phase timing breakdown?
   Trade-off: Wall-clock is simpler; per-phase helps identify bottlenecks

## Architectural Notes

- This is a COMMAND file (`commands/full-review.md`), not an agent — it runs in the main conversation context, not as a sub-agent
- The orchestrator must NEVER read source code files — only batch plans, progress files, and existence checks on output files
- Sub-agents are launched via the Task tool — each gets a detailed prompt with batch assignment, output file path, and agent instructions
- The build plan specifies the final summary format with emoji severity indicators — match it exactly
- Context pressure is the #1 risk: if the orchestrator's context fills, the review stalls. Every design decision should minimize orchestrator context usage
- The orchestrator must handle the case where it's resumed after a crash or session end — `state.json` is the recovery mechanism

## Complexity: XL
## Estimated Phases: ~5
