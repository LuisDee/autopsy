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
- `.autopsy/batch-{N}/*.md` (Track 03 -- code review findings as supporting evidence)

## Dependencies
- Track 01_plugin_scaffold: COMPLETE. Provides `agents/` directory structure and plugin.json manifest.

<!-- END ARCHITECT CONTEXT -->

# Track 08: Architecture Agents — Specification

## Overview

Create three new agent definition markdown files that add architecture assessment to the autopsy plugin. These agents analyze codebases at the design level — evaluating component structure, technology choices, interaction patterns, and quality attribute tradeoffs — producing `ARCHITECTURE_REPORT.md` as a companion to the existing code-level `REVIEW_REPORT.md`.

### Design Decisions (Resolved)

| # | Decision | Choice | Rationale |
|---|----------|--------|-----------|
| DD-1 | Architect tool access | Read-only (Read, Grep, Glob) | Researcher handles web lookups; architect stays focused on code |
| DD-2 | Researcher tool strategy | WebSearch/WebFetch (background mode) | Context7 MCP unavailable in background agents |
| DD-3 | MCP availability | Background execution, web-search-only | Parallelism > research quality; targeted web queries find official docs |
| DD-4 | Synthesizer evidence scope | Incorporate code findings as evidence | Richer report with concrete code evidence supporting design observations |
| DD-5 | Architect reading strategy | Strategic reading | Focus on architecturally significant files; avoid context bloat |
| DD-6 | Intermediate file format | Strict structure with mandatory sections | Reliable parsing by synthesizer |
| DD-7 | Agent model selection | All `inherit` | Defer to user's Claude Code model setting |

---

## Functional Requirements

### FR-1: Architect Agent (`autopsy/agents/architect.md`)

**Role:** Senior/principal architect evaluating system design. NOT a code reviewer — thinks at the design level.

**YAML Frontmatter:**
```yaml
---
name: architect
description: "Evaluates system architecture: components, interactions, tooling choices, quality attributes, and design gaps. Inspired by ATAM (Architecture Tradeoff Analysis Method)."
tools:
  - Read
  - Grep
  - Glob
---
```

**Inputs:**
- `.autopsy/discovery.md` (repo profile)
- Root `CLAUDE.md` and all module-level `CLAUDE.md` files
- Key source files read strategically: entry points, configs, core business logic, API definitions, data models, orchestration files (DAGs/workflows/pipelines), infrastructure configs

**Output:** `.autopsy/architecture-analysis.md` (strict structure)

**6-Step Analysis Process:**

1. **Infer the System Goal** — What the system exists to do, who it serves, key value, operating context. Inferred from README, CLAUDE.md, docs/, entry points, API routes, config files.

2. **Decompose into Components** — Map system into logical components. For each: purpose, responsibility, implementation (directories/files), interfaces, internal dependencies, external dependencies, data owned. Components map to actual code boundaries. Flag directories doing multiple unrelated things or monolithic god files.

3. **Analyze Component Interactions** — For each interaction: mechanism (HTTP, function call, shared DB, queue, filesystem), data exchanged, coupling assessment (tight/moderate/loose), correctness of coupling level. Look for: circular dependencies, hub components, unnecessary coupling, missing abstractions, data flow bottlenecks, shared mutable state, leaky abstractions. Produce mermaid diagram.

4. **Evaluate Tooling Choices** — For each significant technology: what it's asked to do, fit score (Good Fit/Acceptable/Poor Fit/Wrong Tool), where it excels, where it falls short, red flags, alternative worth considering, migration difficulty. Evaluate against: purpose alignment, scale appropriateness, operational complexity, ecosystem maturity, team capability fit, redundancy, integration friction.

5. **Quality Attribute Tradeoff Analysis (Adapted ATAM)** — Build quality attribute matrix covering: Performance, Scalability, Reliability, Security, Maintainability, Testability, Operability, Extensibility. For each: evidence of priority, current support level (Strong/Moderate/Weak/None). Identify sensitivity points and tradeoff points.

6. **Identify Where the Design Falls Short** — For each gap: component(s) affected, intent, current implementation, why it falls short, impact, evidence from code, suggested approach (design-level), effort estimate (small/medium/large/rewrite), priority (critical/high/medium/low). 10 gap categories: responsibility misplacement, missing component, wrong abstraction level, temporal coupling, data model mismatch, missing boundaries, over-engineering, under-engineering, stale architecture, impedance mismatch.

**Self-Review Checklist** (must be included in agent definition):
- [ ] All 6 steps completed
- [ ] System goal is stated (not assumed)
- [ ] Every identified component has all 7 fields documented
- [ ] Component interaction diagram produced
- [ ] Every significant technology evaluated with fit score
- [ ] Quality attribute matrix has 8 rows with evidence
- [ ] At least 3 design gaps identified with evidence
- [ ] Output written to `.autopsy/architecture-analysis.md`

**Few-Shot Examples:** 2-3 architecture-specific examples (component analysis, tooling evaluation, design gap) — NOT code-level finding examples.

### FR-2: Researcher Agent (`autopsy/agents/researcher.md`)

**Role:** Look up official docs and best practices for every major technology and pattern in the codebase. Provide evidence-based backing.

**YAML Frontmatter:**
```yaml
---
name: researcher
description: "Researches official documentation, best practices, and recommended patterns for every major technology in the stack. Provides evidence-based backing for architecture assessment."
tools:
  - Read
  - Grep
  - Glob
  - WebSearch
  - WebFetch
---
```

**Inputs:** `.autopsy/discovery.md` and root `CLAUDE.md`

**Output:** `.autopsy/best-practices-research.md` (strict structure)

**5-Step Research Process:**

1. **Identify Research Targets** — From discovery profile, list every significant technology (languages, frameworks, databases, message brokers, orchestrators, cloud services, key libraries, CI/CD, monitoring) and the overall architectural pattern.

2. **Research Each Technology** — For each: official documentation consulted (URLs), current recommended patterns, deprecated features/patterns found in this codebase, official migration guides (if version outdated), known gotchas, how top projects use this tool.

3. **Research the Architectural Pattern** — Industry best practices for this system type (with sources), common anti-patterns, reference architectures. Research by system type: data pipeline, API service, event-driven, trading/finance, ML system, etc.

4. **Research Alternative Tools** — Only where there's a genuine case for switching: current tool strengths/limitations, alternative strengths/trade-offs, migration complexity, source documentation.

5. **Research Interaction Patterns** — For each component interaction type: recommended integration pattern, does current approach match, problems at scale.

**CRITICAL:** Be specific and cite real documentation. No generic "follow best practices" advice. Every recommendation must reference a specific official doc, migration guide, or reference architecture.

**Self-Review Checklist:**
- [ ] All technologies from discovery.md covered
- [ ] Every recommendation cites a specific URL or documentation source
- [ ] No generic "follow best practices" statements without specific references
- [ ] Architectural pattern research completed for the system type
- [ ] Alternative tools only suggested where genuinely warranted
- [ ] Output written to `.autopsy/best-practices-research.md`

**Few-Shot Examples:** 2-3 research-specific examples (technology evaluation with real URL citations, deprecated pattern identification, reference architecture comparison).

### FR-3: Architecture Synthesizer (`autopsy/agents/architecture-synthesizer.md`)

**Role:** Combine architect's analysis with researcher's findings into a strategic, actionable `ARCHITECTURE_REPORT.md`.

**YAML Frontmatter:**
```yaml
---
name: architecture-synthesizer
description: "Phase 3 agent: cross-references architecture analysis with best-practice research and code review findings to produce the final ARCHITECTURE_REPORT.md."
tools:
  - Read
  - Grep
  - Glob
  - Bash
---
```

**Inputs:**
- `.autopsy/architecture-analysis.md` (architect output)
- `.autopsy/best-practices-research.md` (researcher output)
- `.autopsy/discovery.md` (reference)
- `.autopsy/batch-{N}/*.md` (code review findings — as supporting evidence, NOT duplicated)

**Output:** `ARCHITECTURE_REPORT.md` (in repo root)

**4-Step Synthesis Process:**

1. **Cross-Reference** — For each design gap: does research support it? Do official docs recommend different? Is there a known pattern? Do code-level findings confirm it? For each tooling evaluation: does research confirm fit? Migration guides available? Official docs on usage patterns?

2. **Generate Design Recommendations** — For each significant finding: problem, evidence (architect + research + code findings), recommended approach (design-level, NOT line-level), implementation sketch, impact, effort (S/M/L/XL), risk of not doing this, priority (Critical/High/Medium/Low).

3. **Write ARCHITECTURE_REPORT.md** — Structured report:
   - Executive Summary (3-5 sentences)
   - Section 1: System Purpose & Goals
   - Section 2: Component Architecture (mermaid diagram, summary table, interaction analysis, strengths, concerns)
   - Section 3: Tooling Assessment (stack table, per-tool analysis, deprecated patterns)
   - Section 4: Quality Attribute Analysis (matrix, tradeoffs, sensitivity points)
   - Section 5: Design Gaps & Recommendations (Critical → High → Medium → Low)
   - Section 6: Implementation Roadmap (Phase 1: quick wins, Phase 2: structural improvements, Phase 3: strategic migrations)
   - Section 7: Comparison with Best Practices
   - Appendix A: Research Sources
   - Appendix B: Component Inventory

4. **Handle Missing Inputs Gracefully** — If researcher output is missing, produce report with "[Research data unavailable]" placeholders in sections that require it. If architect output is missing, produce minimal report noting the failure. Never fail silently.

**Self-Review Checklist:**
- [ ] All 7 sections + appendices present in ARCHITECTURE_REPORT.md
- [ ] Executive summary is 3-5 sentences, not generic
- [ ] Every recommendation includes evidence from at least 2 sources
- [ ] Code findings used as supporting evidence where relevant (not duplicated from REVIEW_REPORT.md)
- [ ] No generic advice without specific evidence
- [ ] Implementation roadmap has concrete phases
- [ ] ARCHITECTURE_REPORT.md written to repo root

**Few-Shot Examples:** 2-3 synthesis examples (cross-referencing architect finding with research, generating a design recommendation with multi-source evidence).

---

## Intermediate File Output Formats

### `.autopsy/architecture-analysis.md` (Mandatory Sections)

```markdown
# Architecture Analysis

## 1. System Goal
{inferred goal, audience, key value, operating context}

## 2. Component Decomposition
### Component: {name}
- **Purpose:** ...
- **Responsibility:** ...
- **Implementation:** ...
- **Interfaces:** ...
- **Internal Dependencies:** ...
- **External Dependencies:** ...
- **Data Owned:** ...

## 3. Component Interactions
### Interaction: {Component A} → {Component B}
- **Mechanism:** ...
- **Data Exchanged:** ...
- **Coupling:** {tight/moderate/loose}
- **Assessment:** ...

### Interaction Diagram
{mermaid diagram}

## 4. Tooling Evaluation
### Tool: {name}
- **Used For:** ...
- **Fit Score:** {Good Fit/Acceptable/Poor Fit/Wrong Tool}
- **Excels At:** ...
- **Falls Short:** ...
- **Red Flags:** ...
- **Alternative:** ...
- **Migration Difficulty:** ...

## 5. Quality Attribute Matrix
| Quality Attribute | Evidence of Priority | Current Support Level |
|---|---|---|
| Performance | ... | Strong/Moderate/Weak/None |
{8 rows}

### Sensitivity Points
{list}

### Tradeoff Points
{list}

## 6. Design Gaps
### Gap: {short description}
- **Component(s):** ...
- **Intent:** ...
- **Current Implementation:** ...
- **Why It Falls Short:** ...
- **Impact:** ...
- **Evidence:** ...
- **Suggested Approach:** ...
- **Effort:** {small/medium/large/rewrite}
- **Priority:** {critical/high/medium/low}
```

### `.autopsy/best-practices-research.md` (Mandatory Sections)

```markdown
# Best Practices Research

## 1. Research Targets
{list of technologies and architectural pattern identified}

## 2. Technology Research
### Technology: {name}
- **Version in Codebase:** ...
- **Official Documentation:** {URLs}
- **Current Recommended Patterns:** ...
- **Deprecated Patterns Found:** ...
- **Migration Guides:** ...
- **Known Gotchas:** ...
- **Top Project Usage:** ...

## 3. Architectural Pattern Research
- **System Type:** ...
- **Industry Best Practices:** {with sources}
- **Common Anti-Patterns:** ...
- **Reference Architectures:** {with sources}

## 4. Alternative Tools Research
### {Current Tool} vs {Alternative}
- **Current Strengths:** ...
- **Current Limitations:** ...
- **Alternative Strengths:** ...
- **Alternative Trade-offs:** ...
- **Migration Complexity:** ...
- **Source:** {URL}

## 5. Interaction Pattern Research
### {Interaction Type}
- **Recommended Pattern:** ...
- **Current Approach Match:** {yes/partial/no}
- **Scale Concerns:** ...
```

---

## Non-Functional Requirements

- **Agent file size:** Each agent definition should be 200-400 lines (consistent with existing agents)
- **Pattern consistency:** Follow the same markdown structure as existing agents: YAML frontmatter, role introduction, numbered steps, checklists, few-shot examples, self-review
- **Strict separation:** ARCHITECTURE_REPORT.md covers design-level assessment only. No code-level bug reports (those belong in REVIEW_REPORT.md). Cross-references are allowed.
- **No runtime dependencies:** Pure markdown definitions, no scripts or external tools needed

---

## Acceptance Criteria

1. Three agent files exist: `autopsy/agents/architect.md`, `autopsy/agents/researcher.md`, `autopsy/agents/architecture-synthesizer.md`
2. Each agent has valid YAML frontmatter with name, description, and tools
3. Each agent includes 2-3 few-shot examples in architecture-specific format
4. Each agent includes a self-review checklist
5. Architect agent has 6 analysis steps with exhaustive checklists
6. Researcher agent has 5 research steps with citation requirements
7. Architecture-synthesizer has 4 synthesis steps and the full ARCHITECTURE_REPORT.md template
8. Architecture-synthesizer handles missing inputs gracefully (partial report)
9. Output format specifications are defined for both intermediate files
10. Plugin loads without errors: `claude --plugin-dir ./autopsy`

---

## Out of Scope

- Orchestrator integration (Track 09)
- plugin.json updates (Track 09)
- README.md updates (Track 09)
- full-review.md modifications (Track 09)
- Changes to existing agents or synthesizer
- Automated testing framework
