<!-- ARCHITECT CONTEXT | Track: 08_architecture_agents | Wave: 5 | CC: v1 -->

## Cross-Cutting Constraints
- Self-Review Step (v1): Every agent must verify its own output before completing. The three new architecture agents must include the same self-review checklist pattern used by existing review agents.
- Few-Shot Examples (v1): Every agent definition must include 2-3 concrete output examples in the exact output format. Architecture agents need architecture-specific examples (not code-level finding examples).
- Thoroughness Rules (v1): Every agent must read full file contents, provide exact paths and references, never skim. The architect agent applies this to strategic file reading (entry points, configs, core modules) rather than exhaustive line-by-line review.
- Compaction-Safe State Management (v1): The architecture-synthesizer must write ARCHITECTURE_REPORT.md to disk as its final action. Intermediate files (.autopsy/architecture-analysis.md, .autopsy/best-practices-research.md) must be written to disk before the synthesizer is launched.
- Error Recovery (v1): If the architect or researcher agent fails, the orchestrator (Track 09) must log the failure and continue. The architecture-synthesizer must handle missing inputs gracefully (e.g., produce a partial report if only one input is available).
- Agent Output Format (v1): The existing finding format applies to code-level review agents. Architecture agents need their own output format specification -- this is a design decision for this track.

## Interfaces

### Owns
- `.autopsy/architecture-analysis.md` -- architect agent output (component map, interaction analysis, tooling evaluation, quality attributes, design gaps)
- `.autopsy/best-practices-research.md` -- researcher agent output (per-technology research, architectural pattern research, alternative tools research)
- `ARCHITECTURE_REPORT.md` -- architecture-synthesizer final report (7 sections + appendices)

### Consumes
- `.autopsy/discovery.md` (Track 02 -- repo profile, stats, module summary, risk areas)
- `{dir}/AGENTS.md` (Track 02 -- module-level documentation for architectural context)
- `{dir}/CLAUDE.md` (Track 02 -- project-level documentation)
- `.autopsy/batch-{N}/*.md` (Track 03 -- code review findings as optional supporting evidence)

## Dependencies
- Track 01_plugin_scaffold: COMPLETE. Provides `agents/` directory structure and plugin.json manifest.

<!-- END ARCHITECT CONTEXT -->

# Track 08: Architecture Agents

## What This Track Delivers

Three new agent definition markdown files that add an architecture assessment capability to the autopsy plugin. Unlike the existing review agents (which find code-level bugs, security issues, and performance problems), these agents analyze the system at a design level -- evaluating component structure, technology choices, interaction patterns, and quality attribute tradeoffs. The architect agent performs ATAM-inspired analysis of the codebase; the researcher agent looks up official documentation and best practices for every technology in the stack; and the architecture-synthesizer combines both outputs into a structured ARCHITECTURE_REPORT.md that lives alongside the existing REVIEW_REPORT.md. This creates a clear separation: REVIEW_REPORT.md covers code-level findings, ARCHITECTURE_REPORT.md covers design-level assessment.

## Scope

### IN
- `autopsy/agents/architect.md` -- ATAM-inspired 6-step analysis agent definition:
  1. Infer system goal from codebase structure and documentation
  2. Decompose components (modules, services, layers, boundaries)
  3. Analyze interactions (data flow, control flow, coupling patterns)
  4. Evaluate tooling choices (language, framework, libraries, infrastructure)
  5. Quality attribute tradeoff analysis (scalability, maintainability, security, performance, testability)
  6. Identify design gaps (missing abstractions, tight coupling, unclear boundaries)
- `autopsy/agents/researcher.md` -- technology research agent definition:
  - Per-technology official documentation lookup
  - Architectural pattern research (does the codebase follow recognized patterns?)
  - Alternative tools analysis (what else could be used, and why the current choice may or may not be optimal)
  - Interaction pattern research (how the chosen technologies are intended to work together)
- `autopsy/agents/architecture-synthesizer.md` -- cross-referencing and report generation:
  - Reads architect + researcher outputs from disk
  - Cross-references design analysis with best-practice research
  - Generates structured ARCHITECTURE_REPORT.md with sections and appendices
  - Produces design-level recommendations (not code-level fixes)
- YAML frontmatter for each agent (name, description, tools, model)
- Self-review verification step in each agent
- Output format specification for `.autopsy/architecture-analysis.md`
- Output format specification for `.autopsy/best-practices-research.md`
- Output format specification for `ARCHITECTURE_REPORT.md`
- Few-shot examples (2-3) in each agent definition

### OUT
- Orchestrator logic for launching these agents (Track 09_architecture_orchestration)
- Modifications to `commands/full-review.md` (Track 09)
- Plugin.json manifest updates (Track 09)
- README.md updates (Track 09)
- The existing REVIEW_REPORT.md synthesis (Track 04 -- unchanged)
- The existing 5 review agents (Tracks 03 -- unchanged)
- Discovery agent modifications (Track 02 -- unchanged)
- Any runtime code or scripts (this plugin is pure Markdown + JSON)

## Key Design Decisions

These should be resolved with the developer during spec generation:

1. **Architect agent tool access:** Should the architect agent have read-only tools (Read, Grep, Glob) or also include web tools (WebSearch, WebFetch) for cross-referencing architecture patterns externally?
   - Option A: Read-only keeps the architect focused purely on codebase analysis, producing faster results
   - Option B: Adding web tools lets the architect cross-reference patterns against external sources but risks distraction and context bloat
   - Trade-off: purity of analysis vs breadth of context

2. **Researcher agent tool strategy:** How should the researcher look up documentation -- Context7 MCP as primary with web search fallback, web search only, or always use both?
   - Option A: Context7 MCP provides structured, official documentation with high signal-to-noise ratio
   - Option B: WebSearch/WebFetch is universally available but produces noisier results
   - Option C: Both always gives the most thorough research but is slower and uses more context
   - Trade-off: research quality vs availability vs speed

3. **MCP availability in sub-agents:** MCP tools are NOT available in background sub-agents (known limitation). Should the researcher agent require foreground execution (to use Context7 MCP), or should it be designed for background execution (web search only)?
   - Option A: Foreground-only researcher can use Context7 MCP but blocks the orchestrator during execution
   - Option B: Background researcher with web-search-only mode runs in parallel with architect but may produce lower-quality research
   - Option C: Detect MCP availability at runtime and adapt (adds complexity to agent definition)
   - Trade-off: research quality vs parallelism vs agent complexity

4. **Architecture-synthesizer evidence scope:** Should the synthesizer only cross-reference architect + researcher outputs, or also incorporate code-level findings from existing review agents (batch-{N}/*.md)?
   - Option A: Clean separation -- architecture report is purely about design, code findings stay in REVIEW_REPORT.md
   - Option B: Incorporate code findings as supporting evidence for architectural observations (e.g., "the tight coupling identified by the architect is confirmed by 12 cross-module bug findings")
   - Trade-off: clean separation of concerns vs evidence-rich reporting

5. **Architect file reading strategy:** Should the architect read strategically (entry points, configs, core modules, boundaries) or exhaustively (all files like review agents)?
   - Option A: Strategic reading focuses on architecturally significant files, avoids context bloat, completes faster
   - Option B: Exhaustive reading ensures nothing is missed but may overwhelm with implementation details that obscure architecture
   - Trade-off: architectural focus vs completeness

6. **Intermediate file output format:** Should `.autopsy/architecture-analysis.md` and `.autopsy/best-practices-research.md` use strictly structured markdown (with defined section headers the synthesizer can parse) or freeform analysis?
   - Option A: Strict structure with mandatory sections enables reliable parsing by the synthesizer
   - Option B: Freeform analysis allows richer, more nuanced observations but makes synthesis harder
   - Trade-off: parseability vs analytical depth

7. **Agent model selection in YAML frontmatter:** Should all three agents specify `sonnet`, should architect/synthesizer use `opus` for deeper analysis, or should they use `inherit` to let the user decide?
   - Option A: All on `sonnet` -- cost-effective, consistent, fast
   - Option B: Architect and synthesizer on `opus` -- deeper architectural reasoning, but higher cost
   - Option C: All use `inherit` -- defers to the user's Claude Code model setting, most flexible
   - Trade-off: analysis depth vs cost vs user flexibility

## Architectural Notes

- The architect and researcher agents are designed to run in PARALLEL. They must be completely independent -- no shared state, no reading each other's output. The orchestrator (Track 09) launches both simultaneously and waits for both to complete before launching the synthesizer.
- The architecture-synthesizer runs AFTER both the architect and researcher complete. It reads their outputs from `.autopsy/architecture-analysis.md` and `.autopsy/best-practices-research.md`. This is the same sequential-after-parallel pattern used by the existing synthesizer (Track 04) with review agents.
- MCP tools are NOT available in background sub-agents (known platform limitation, see MEMORY.md). If the researcher needs Context7 MCP, it must run in foreground -- which means it runs sequentially with the architect, not in parallel. This is the core tension in DD-3.
- These agents follow the same markdown definition pattern as existing agents (`bug-hunter.md`, `security-auditor.md`, etc.): YAML frontmatter, structured instructions, checklists, few-shot examples, and a self-review step. Study the existing agents for the exact pattern.
- ARCHITECTURE_REPORT.md must be clearly distinct from REVIEW_REPORT.md. The architecture report covers design-level assessment (component structure, technology choices, quality attributes, design gaps). The review report covers code-level findings (bugs, security, errors, performance). They should cross-reference each other but not duplicate content.
- The architect agent needs to read strategically, not exhaustively. It focuses on architecturally significant files: entry points, configuration files, dependency manifests, core module boundaries, routing/API definitions, database schemas, and infrastructure definitions. This is fundamentally different from review agents which read every assigned file line-by-line.
- The `agents/` directory already contains 7 agent files (discovery, 5 reviewers, synthesizer). Adding 3 more brings the total to 10 agents. The plugin manifest (plugin.json) may need updating to recognize the new agents -- but that is Track 09's responsibility, not this track's.
- The ARCHITECTURE_REPORT.md proposed 7 sections should be specified in the synthesizer agent definition. A reasonable starting structure: Executive Summary, Component Architecture, Technology Assessment, Quality Attributes, Design Gaps, Recommendations, Appendices. The exact structure is a spec-level decision.

## Test Strategy

- **Framework:** None (plugin is Markdown-based, no executable code)
- **Validation approach:**
  - Load plugin with `claude --plugin-dir ./autopsy` and verify all 3 new agent definitions are recognized
  - Verify YAML frontmatter parses correctly (name, description, tools fields present)
  - Verify each agent definition includes a self-review step
  - Verify each agent definition includes 2-3 few-shot examples
  - Verify output format specifications are present and parseable
- **Manual integration test:**
  - Run the architect agent against a sample repository and verify `.autopsy/architecture-analysis.md` is produced with expected sections
  - Run the researcher agent and verify `.autopsy/best-practices-research.md` is produced
  - Run the architecture-synthesizer and verify `ARCHITECTURE_REPORT.md` is produced with expected structure
- **Prerequisites:** Track 01 must be complete (agents/ directory must exist)
- **Quality threshold:** All agent definitions must follow the established pattern (YAML frontmatter, checklists, few-shot examples, self-review step). Advisory coverage target: N/A (no executable code to cover).
- **Key test scenarios:**
  1. Architect agent produces component map and identifies at least 3 quality attribute tradeoffs for a non-trivial repository
  2. Researcher agent finds official documentation for each technology detected in the stack
  3. Architecture-synthesizer cross-references architect and researcher outputs into a coherent report
  4. Architecture-synthesizer handles missing researcher output gracefully (partial report)
  5. All three agent definitions load without errors in Claude Code plugin system

## Complexity: L
## Estimated Phases: ~4
