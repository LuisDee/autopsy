<!-- ARCHITECT CONTEXT | Track: 02_discovery_agent | Wave: 2 | CC: v1 -->

## Cross-Cutting Constraints
- AGENTS.md Quality Bar: specific to actual code, under 80 lines (root) / 60 lines (subdirectory), no file trees, three-tier boundary system
- Dual Documentation Output: AGENTS.md primary + CLAUDE.md companion with `@AGENTS.md`
- Batch Sizing Constraints: under 50 files or 5000 LOC per batch, group related modules, order by risk, target 30-40K tokens per agent
- Thoroughness Rules: read full contents, no skimming, exact file paths

## Interfaces

### Owns
- `.deep-review/discovery.md` (repo profile)
- `.deep-review/batch-plan.md` (batch list for review phase)
- `.deep-review/file-list.txt` (complete file inventory)
- `{dir}/AGENTS.md` files (module documentation)
- `{dir}/CLAUDE.md` companion files

### Consumes
- None (first phase — reads the target repo directly)

## Dependencies
- Track 01_plugin_scaffold: `agents/` directory must exist

<!-- END ARCHITECT CONTEXT -->

# Track 02: Discovery Agent

## What This Track Delivers

The discovery agent definition (`agents/discovery.md`) — the Phase 1 agent that maps an entire repository, understands every module, creates AGENTS.md documentation files throughout the codebase, and produces a batch plan that the orchestrator uses to coordinate Phase 2 review agents. This is the foundation for the entire review — the quality of the batch plan and documentation directly determines the quality of the review.

## Scope

### IN
- `agents/discovery.md` agent definition with YAML frontmatter
- Step-by-step instructions for: repo mapping (file inventory, tech stack detection, LOC counts), module understanding (reading representative files per directory), AGENTS.md generation (using the quality bar from product guidelines), batch plan generation (grouping related modules, sizing batches, ordering by risk)
- Discovery profile template (`.deep-review/discovery.md`)
- Batch plan template (`.deep-review/batch-plan.md`)
- AGENTS.md template for module documentation
- Root AGENTS.md template

### OUT
- Review agent definitions (Track 03)
- Synthesizer agent (Track 04)
- Orchestrator coordination logic (Track 05)
- AGENTS.md maintenance logic (Track 06)

## Key Design Decisions

These should be resolved with the developer during spec generation:

1. **Representative file sampling strategy:** Read 3-5 files per directory vs read all files under a threshold (e.g., directories with <10 files)?
   Trade-off: Sampling is faster and fits in context; reading all gives better understanding but risks context overflow
2. **AGENTS.md generation trigger:** Directories with 3+ code files vs directories with any code files vs only directories with complex logic?
   Trade-off: 3+ is the build plan default; lower threshold creates more docs; higher threshold reduces noise
3. **Batch grouping algorithm:** Directory-based (group by directory tree) vs import-based (analyze imports to find related modules) vs hybrid?
   Trade-off: Directory-based is simpler and more reliable; import-based groups semantically related code but requires parsing
4. **Risk level assignment:** Manual heuristics (file count, LOC, dependencies) vs pattern-based (presence of auth, SQL, crypto, etc.)?
   Trade-off: Heuristics are more general; pattern-based catches specific risk areas better
5. **Existing AGENTS.md handling:** Preserve entirely and only append vs merge sections intelligently?
   Trade-off: Preserve is safer; merge produces cleaner output but risks losing information

## Architectural Notes

- The discovery agent runs in FOREGROUND (blocking) — the orchestrator waits for it to complete before proceeding to Phase 2
- This agent creates the AGENTS.md files that ALL review agents read first for module context — quality here cascades to review quality
- The batch plan format is an interface contract consumed by the orchestrator — changes here require updating the orchestrator
- Agent should generate companion `CLAUDE.md` files containing `@AGENTS.md` at every level
- The agent must exclude `.deep-review/`, `node_modules`, `.git`, and other common non-code directories from scanning

## Complexity: M
## Estimated Phases: ~3
