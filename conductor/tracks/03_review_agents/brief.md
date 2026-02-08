<!-- ARCHITECT CONTEXT | Track: 03_review_agents | Wave: 2 | CC: v1 -->

## Cross-Cutting Constraints
- Agent Output Format: severity, confidence, exact file path, line number, category, description, impact, fix
- Self-Review Step: every agent verifies its own output completeness before finishing
- Few-Shot Examples: 2-3 concrete finding examples per agent in the exact output format
- Thoroughness Rules: read full file contents, no skimming, no "looks fine", exact paths + lines, re-read 100+ line files with zero findings

## Interfaces

### Owns
- `.autopsy/batch-{N}/bugs.md` (bug-hunter findings)
- `.autopsy/batch-{N}/security.md` (security-auditor findings)
- `.autopsy/batch-{N}/errors.md` (error-inspector findings)
- `.autopsy/batch-{N}/performance.md` (performance-detector findings)
- `.autopsy/batch-{N}/stack.md` (stack-reviewer findings)

### Consumes
- `{dir}/AGENTS.md` (from Track 02 — read for module context before reviewing)
- `.autopsy/discovery.md` (from Track 02 — repo profile for stack context)

## Dependencies
- Track 01_plugin_scaffold: `agents/` directory must exist

<!-- END ARCHITECT CONTEXT -->

# Track 03: Review Agents

## What This Track Delivers

Five specialized Phase 2 review agent definitions — the core value of the plugin. Each agent runs in its own context window with a single focus area, reading every file in its assigned batch and producing severity-graded findings with exact file paths, line numbers, and fix recommendations. These agents are: bug-hunter, security-auditor, error-inspector, performance-detector, and stack-reviewer.

## Scope

### IN
- `agents/bug-hunter.md` — logic errors, null access, race conditions, off-by-one, type confusion, dead branches, boolean logic, date/time, collections
- `agents/security-auditor.md` — injection (SQL, XSS, command, template), auth/authz, data exposure, input handling (path traversal, SSRF, file upload), secrets, dependencies
- `agents/error-inspector.md` — missing error handling, broken error handling (empty catch, swallow-all), resilience (timeouts, circuit breakers, graceful shutdown), resource cleanup, logging/observability
- `agents/performance-detector.md` — database (N+1, unbounded queries, missing indexes), memory (unbounded growth, event listener leaks), CPU/IO (sync in async, O(n²), repeated computation), concurrency, resources, network
- `agents/stack-reviewer.md` — dynamic checks based on detected stack: Python, Airflow, BigQuery/SQL, FastAPI/Flask, Docker/K8s, JS/TS, React, Go, Rust
- YAML frontmatter for each agent (name, description, tools, model)
- Few-shot finding examples (2-3 per agent)
- Self-review verification step in each agent

### OUT
- Orchestrator logic for launching these agents (Track 05)
- Synthesizer logic for parsing their output (Track 04)

## Key Design Decisions

These should be resolved with the developer during spec generation:

1. **Agent model selection:** All agents use `sonnet` vs `inherit` from parent vs mix (e.g., security-auditor on `opus` for higher accuracy)?
   Trade-off: Sonnet is cost-effective; opus is more thorough but 5x cost; inherit lets the user decide
2. **Stack-reviewer scope:** Static checklist covering all known stacks vs dynamic approach where the agent reads AGENTS.md to determine which checks to apply?
   Trade-off: Static is more thorough (nothing skipped); dynamic avoids false positives from irrelevant checks
3. **Few-shot example source:** Generic examples vs stack-specific examples vs examples from real open-source code reviews?
   Trade-off: Generic is universally applicable; stack-specific is more relevant; real examples are most convincing but harder to craft
4. **Agent tool access:** Read + Grep + Glob (read-only) vs also include Bash (for running linters, type checkers)?
   Trade-off: Read-only is safer and focused; Bash enables automated checks but agents might run arbitrary commands
5. **AGENTS.md update scope:** Review agents can update module AGENTS.md if they find undocumented gotchas vs read-only (only discovery/synthesizer update docs)?
   Trade-off: Updating during review captures insights immediately; read-only keeps clear ownership boundaries
6. **Confidence score calibration:** Let each agent self-assess confidence vs define explicit rules (e.g., "High confidence if you can show the exact code path")?
   Trade-off: Self-assessment is flexible; explicit rules are more consistent across agents

## Architectural Notes

- All 5 agents run in PARALLEL as BACKGROUND tasks — they must be completely independent with no cross-agent communication
- Each agent reads the AGENTS.md file in its assigned directories FIRST for module context
- Each agent reads the FULL contents of EVERY file in its batch — no skimming
- The output file path is passed in the Task tool `prompt` parameter by the orchestrator
- MCP tools are NOT available in background sub-agents — agents use only Read, Grep, Glob (and optionally Bash)
- The 5 agent definitions share the same output format but have completely different checklists — the checklists are the core value and must be exhaustive

## Complexity: L
## Estimated Phases: ~4
