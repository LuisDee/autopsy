# Track 08: Architecture Agents — Implementation Plan

> **Note:** This track produces pure Markdown agent definitions (no executable code). The standard TDD workflow is adapted: "tests" are structural validation checks (YAML frontmatter parsing, section presence, plugin loading) rather than unit tests. Coverage requirements (>80%) do not apply to Markdown files.

---

## Phase 1: Architect Agent

### [x] Task 1.1: Create architect agent definition
- Create `autopsy/agents/architect.md` with:
  - YAML frontmatter (name: architect, tools: Read/Grep/Glob)
  - Role introduction (ATAM-inspired, NOT a code reviewer)
  - Step 1: Infer the System Goal (what, who, value, context)
  - Step 2: Decompose into Components (7 fields per component, decomposition rules)
  - Step 3: Analyze Component Interactions (mechanism, coupling, mermaid diagram, 7 patterns to detect)
  - Step 4: Evaluate Tooling Choices (fit score, 7 evaluation criteria)
  - Step 5: Quality Attribute Tradeoff Analysis (8-row matrix, sensitivity/tradeoff points)
  - Step 6: Identify Design Gaps (10 gap categories, 9 fields per gap)
  - Self-review checklist
  - 2-3 few-shot examples (architecture-specific)
- Reference: `new-request.md` (lines 28-160) for detailed step specifications
- Reference: `agents/bug-hunter.md` for structural pattern (YAML, steps, checklist, examples)
- Output format: must match the mandatory section headers defined in spec.md

### [x] Task 1.2: Validate architect agent
- Verify YAML frontmatter parses correctly: `python3 -c "import yaml; yaml.safe_load(open('autopsy/agents/architect.md').read().split('---')[1])"`
- Verify agent includes all 6 analysis steps
- Verify self-review checklist is present with all items
- Verify 2-3 few-shot examples are included
- Verify agent is under 400 lines
- Verify plugin loads: `claude --plugin-dir ./autopsy` (check for errors)

### [x] Task 1.3: Commit architect agent dc43920
- Commit message: `feat(agents): Add architect agent for ATAM-inspired architecture analysis`

---

## Phase 2: Researcher Agent

### [x] Task 2.1: Create researcher agent definition
- Create `autopsy/agents/researcher.md` with:
  - YAML frontmatter (name: researcher, tools: Read/Grep/Glob/WebSearch/WebFetch)
  - Role introduction (evidence-based research, cite real documentation)
  - Step 1: Identify Research Targets (technologies + architectural pattern)
  - Step 2: Research Each Technology (6 fields per technology, URLs required)
  - Step 3: Research the Architectural Pattern (by system type: pipeline, API, event-driven, etc.)
  - Step 4: Research Alternative Tools (only where genuinely warranted)
  - Step 5: Research Interaction Patterns (recommended vs actual)
  - Self-review checklist
  - 2-3 few-shot examples (research-specific with real URL citation patterns)
- CRITICAL: Agent uses WebSearch/WebFetch (not Context7 MCP) — designed for background execution
- Reference: `new-request.md` (lines 163-208) for detailed step specifications

### [x] Task 2.2: Validate researcher agent
- Verify YAML frontmatter parses correctly
- Verify agent includes all 5 research steps
- Verify self-review checklist is present
- Verify 2-3 few-shot examples with URL citation patterns
- Verify agent is under 400 lines
- Verify no references to Context7 MCP as required tool (it's optional/noted, not required)

### [x] Task 2.3: Commit researcher agent fb7dba1
- Commit message: `feat(agents): Add researcher agent for technology best-practices lookup`

---

## Phase 3: Architecture Synthesizer Agent

### [x] Task 3.1: Create architecture-synthesizer agent definition
- Create `autopsy/agents/architecture-synthesizer.md` with:
  - YAML frontmatter (name: architecture-synthesizer, tools: Read/Grep/Glob/Bash)
  - Role introduction (cross-references, produces ARCHITECTURE_REPORT.md)
  - Step 1: Read All Inputs (architecture-analysis.md, best-practices-research.md, discovery.md, batch-{N}/*.md)
  - Step 2: Cross-Reference (design gaps vs research, tooling vs research, code findings as evidence)
  - Step 3: Generate Design Recommendations (8 fields per recommendation)
  - Step 4: Write ARCHITECTURE_REPORT.md (7 sections + 2 appendices, full template)
  - Graceful handling of missing inputs (partial report with placeholders)
  - Self-review checklist
  - 2-3 few-shot examples (synthesis-specific: cross-referencing, recommendation generation)
- CRITICAL: Include full ARCHITECTURE_REPORT.md template structure in agent definition
- Reference: `new-request.md` (lines 212-294) for detailed specifications
- Reference: `agents/synthesizer.md` for Phase 3 agent structural pattern

### [x] Task 3.2: Validate architecture-synthesizer agent
- Verify YAML frontmatter parses correctly
- Verify agent includes all 4 synthesis steps
- Verify ARCHITECTURE_REPORT.md template has all 7 sections + 2 appendices
- Verify missing-input handling logic is present
- Verify self-review checklist
- Verify 2-3 few-shot examples
- Verify agent is under 400 lines

### [x] Task 3.3: Commit architecture-synthesizer agent 44c352d
- Commit message: `feat(agents): Add architecture-synthesizer agent for ARCHITECTURE_REPORT.md generation`

---

## Phase 4: Integration Validation

### [x] Task 4.1: Cross-agent validation
- Verify architect output format (architecture-analysis.md sections) matches what synthesizer expects to read
- Verify researcher output format (best-practices-research.md sections) matches what synthesizer expects to read
- Verify all three agents follow the same structural pattern as existing agents (YAML, steps, checklist, examples)
- Verify no agent exceeds 400 lines
- Verify total agent count would be 10 (7 existing + 3 new) when plugin.json is updated in Track 09

### [x] Task 4.2: Plugin loading validation
- Run `claude --plugin-dir ./autopsy` and verify no errors from the 3 new agent files
- Verify `python3 -c "import json; json.load(open('autopsy/.claude-plugin/plugin.json'))"` passes (existing agents still valid)

### [x] Task 4.3: Final commit and phase completion
- Commit any validation fixes
- Commit message: `conductor(checkpoint): Complete Track 08 Architecture Agents`

### Phase Completion Verification
- Verify all 3 agent files exist in `autopsy/agents/`
- Verify each has valid YAML frontmatter, self-review checklist, few-shot examples
- Verify output format specifications align between producer and consumer agents
- Manual verification: review each agent definition for completeness against spec acceptance criteria
