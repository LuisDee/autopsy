<!-- ARCHITECT CONTEXT | Track: 04_synthesizer_agent | Wave: 2 | CC: v1 -->

## Cross-Cutting Constraints
- Agent Output Format: must parse the standardized finding format from review agents
- Self-Review Step: verify dedup completeness, cross-cutting analysis coverage, report structure
- Dual Documentation Output: update root AGENTS.md + CLAUDE.md with insights from review
- Thoroughness Rules: read ALL findings from ALL batches, no skipping

## Interfaces

### Owns
- `REVIEW_REPORT.md` (final report in repo root)
- Updated root `AGENTS.md` with architectural insights and known issues

### Consumes
- `.deep-review/batch-{N}/bugs.md` (from Track 03)
- `.deep-review/batch-{N}/security.md` (from Track 03)
- `.deep-review/batch-{N}/errors.md` (from Track 03)
- `.deep-review/batch-{N}/performance.md` (from Track 03)
- `.deep-review/batch-{N}/stack.md` (from Track 03)
- `.deep-review/discovery.md` (from Track 02 — repo profile for context)

## Dependencies
- Track 01_plugin_scaffold: `agents/` directory must exist

<!-- END ARCHITECT CONTEXT -->

# Track 04: Synthesizer Agent

## What This Track Delivers

The synthesizer agent definition (`agents/synthesizer.md`) — the Phase 3 agent that reads all findings from all batches, deduplicates them, performs cross-cutting analysis (architecture, dependencies, testing gaps, secrets scan), updates the root AGENTS.md with architectural insights, and produces the final `REVIEW_REPORT.md`. This is the agent that turns raw findings into an actionable, prioritized report.

## Scope

### IN
- `agents/synthesizer.md` agent definition with YAML frontmatter
- Step-by-step instructions for: reading all batch findings, three-level deduplication (exact match, semantic overlap, cross-file patterns), confidence elevation (multi-agent consensus), cross-cutting analysis (architecture, dependency audit, testing gaps, secrets scan), root AGENTS.md update, REVIEW_REPORT.md generation
- Report template with all sections: executive summary, statistics, findings by severity, cross-cutting analysis, recommendations, documentation generated, appendix
- Deduplication algorithm (same file + line + category = exact; overlapping line range + related category = semantic; same pattern across files = systemic)
- Dependency audit commands (pip audit, npm audit, govulncheck, cargo audit)
- Secrets scan grep pattern

### OUT
- Orchestrator logic for launching the synthesizer (Track 05)

## Key Design Decisions

These should be resolved with the developer during spec generation:

1. **Dedup strategy:** Strict dedup (same file + same line = always merge) vs fuzzy dedup (overlapping line ranges + related categories = merge if semantically similar)?
   Trade-off: Strict is safer (no accidental merges); fuzzy reduces noise but may lose distinct-but-related findings
2. **Verification step:** Synthesizer spot-checks top 10 Critical/High findings by re-reading the actual code vs trust agent findings as-is?
   Trade-off: Spot-checking improves confidence but costs context tokens; trusting is faster but may propagate false positives
3. **Cross-cutting analysis scope:** All four analyses (architecture, dependencies, testing gaps, secrets) always vs only those relevant to the detected stack?
   Trade-off: Always is more thorough; selective saves context and time on smaller repos
4. **Report length control:** Cap sections at a maximum number of findings per severity vs include everything?
   Trade-off: Capping keeps the report scannable; including everything ensures nothing is lost
5. **AGENTS.md update granularity:** Update root AGENTS.md only vs update root + affected subdirectory AGENTS.md files?
   Trade-off: Root-only is simpler and safer; updating subdirectories captures more specific insights but risks conflicts with existing content

## Architectural Notes

- The synthesizer runs in FOREGROUND (blocking) — the orchestrator waits for it to complete
- It must read ALL `.md` files in ALL `.deep-review/batch-*/` directories — this could be significant context for large repos
- The dependency audit commands (`pip audit`, `npm audit`, etc.) may not be available in all environments — the agent should handle missing tools gracefully
- The secrets scan pattern must avoid flagging test fixtures, documentation examples, and environment variable definitions
- Multi-agent consensus (3+ agents flagging same issue) should elevate confidence, not severity — severity is based on impact

## Complexity: M
## Estimated Phases: ~3
