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
- `.autopsy/batch-{N}/bugs.md` (from Track 03)
- `.autopsy/batch-{N}/security.md` (from Track 03)
- `.autopsy/batch-{N}/errors.md` (from Track 03)
- `.autopsy/batch-{N}/performance.md` (from Track 03)
- `.autopsy/batch-{N}/stack.md` (from Track 03)
- `.autopsy/discovery.md` (from Track 02 — repo profile for context)

## Dependencies
- Track 01_plugin_scaffold: `agents/` directory must exist

<!-- END ARCHITECT CONTEXT -->

# Specification: Synthesizer Agent

## Overview

Define the synthesizer agent (`autopsy/agents/synthesizer.md`) — the Phase 3 agent that reads all findings from all batches, deduplicates them at three levels, spot-checks top findings, performs cross-cutting analysis, updates root AGENTS.md, and produces the final `REVIEW_REPORT.md`. This agent runs in foreground (blocking) with a fresh context.

## Design Decisions Applied

- **DD1 Dedup:** Three-level (exact match, semantic overlap, systemic patterns)
- **DD2 Spot-check:** Yes — re-read actual code for top 10 Critical/High findings
- **DD3 Analysis scope:** All four always (architecture, dependencies, testing gaps, secrets)
- **DD4 Report length:** Include all findings (no cap per severity)
- **DD5 Doc updates:** Root AGENTS.md only (no subdirectory updates)

## Functional Requirements

### FR-1: Agent Definition File

Create `autopsy/agents/synthesizer.md` (replacing placeholder) with:
- YAML frontmatter: `name: synthesizer`, `description`, `tools` (Read, Grep, Glob, Bash — needs Bash for dependency audit and secrets scan)
- Complete step-by-step instructions (Steps 1-5)
- Self-review checklist

### FR-2: Step 1 — Read All Findings

- Read every `.md` file in every `.autopsy/batch-*/` directory
- Parse findings using the standardized finding format (severity, confidence, file, line, category, description, impact, fix)
- Compile into a single master list

### FR-3: Step 2 — Three-Level Deduplication

**Level 1 — Exact match:** Same file + same line + same category → merge, keep most detailed description, note which agents flagged it

**Level 2 — Semantic overlap:** Overlapping line range + related category (e.g., "Missing Error Handling" and "Empty Catch" on adjacent lines) → merge into a single finding, note it spans multiple concerns

**Level 3 — Systemic patterns:** Same issue type appearing in 3+ files → create a single "systemic" finding that references all affected files, with severity based on the worst individual occurrence

**Confidence elevation:** When 2+ agents independently flag the same issue, elevate confidence (Low→Medium, Medium→High). Do NOT change severity — severity is based on impact.

### FR-4: Step 3 — Spot-Check Top Findings

For the top 10 Critical and High-confidence findings:
1. Re-read the actual source code file
2. Verify the finding is accurate (correct file, correct line, issue actually exists)
3. If a finding is a false positive, remove it and note the removal
4. If a finding is accurate but imprecise, refine the description

### FR-5: Step 4 — Cross-Cutting Analysis

Run all four analyses regardless of stack:

**Architecture:**
- Map module dependencies by grepping imports across the codebase
- Identify circular dependencies
- Identify inconsistent patterns (e.g., mixed error handling approaches)
- Identify god modules (too many responsibilities)
- Identify tight coupling between components that should be independent

**Dependencies:**
```bash
pip audit 2>/dev/null || safety check 2>/dev/null || echo "No Python audit tool"
npm audit 2>/dev/null || echo "No npm"
govulncheck ./... 2>/dev/null || echo "No govulncheck"
cargo audit 2>/dev/null || echo "No cargo-audit"
```
- Check for unused dependencies
- Check for version conflicts
- Handle missing tools gracefully (report as "audit tool not available")

**Testing gaps:**
- Compare modules/files reviewed against available tests
- Identify critical paths (auth, data mutations, payment, etc.) with no tests
- Note test quality issues from review agent findings

**Secrets scan:**
```bash
grep -rn -E '(password|secret|token|api_key|apikey|private_key|credential|passwd)\s*=' \
  --include='*.py' --include='*.js' --include='*.ts' --include='*.yaml' \
  --include='*.yml' --include='*.json' --include='*.toml' --include='*.cfg' \
  --include='*.ini' . 2>/dev/null | grep -v node_modules | grep -v venv | grep -v '.git/' | grep -v test | grep -v fixture | grep -v example | head -50
```
- Filter out test fixtures, documentation examples, and env var definitions
- Cross-reference with security-auditor findings

### FR-6: Step 5 — Update Root AGENTS.md and Write Report

**Root AGENTS.md update (root only):**
- Read existing root AGENTS.md
- Append or update "Known Issues" section with critical/high findings summary
- Append or update "Technical Debt" section with architectural issues
- Respect the idempotency marker (`<!-- generated by autopsy -->`) — if present, update in place; if not, append below a separator

**Write REVIEW_REPORT.md to repo root:**

```markdown
# Code Review Report

**Generated:** {date}
**Repository:** {name}
**Review method:** Automated multi-agent deep review (autopsy plugin)

---

## Executive Summary
{2-3 sentences: overall health assessment, biggest risk, top recommendation}

## Statistics
| Metric | Count |
|--------|-------|
| Files reviewed | {N} |
| Total issues | {N} |
| 🔴 Critical | {N} |
| 🟠 High | {N} |
| 🟡 Medium | {N} |
| 🔵 Low | {N} |
| AGENTS.md files generated | {N} |

---

## 🔴 Critical Issues — Fix Immediately

### {Issue title}
**File:** `{path}` **Line(s):** {N}
**Category:** {category}
**Found by:** {agent name(s)}
**Confidence:** {High|Medium|Low}
**Description:** {what's wrong}
**Impact:** {what could happen}
**Fix:** {how to fix}

---

## 🟠 High Priority Issues
{all high findings, grouped by category}

---

## 🟡 Medium Priority Issues
{all medium findings, grouped by category}

---

## 🔵 Low Priority Issues
{all low findings, grouped by category}

---

## Cross-Cutting Analysis

### Architecture
{findings}

### Dependencies
{audit results, vulnerabilities, unused deps}

### Testing Gaps
{untested critical paths}

### Configuration & Secrets
{any issues found}

---

## Recommendations
1. **{Highest priority}:** {action}
2. **{Second priority}:** {action}
3. **{Third priority}:** {action}

---

## Documentation Generated
This review created/updated the following AGENTS.md files:
{list all AGENTS.md file paths}

These files provide persistent context for future Claude Code sessions.
Run `/autopsy:maintain-docs` after making changes to keep them current.

---

## Appendix: Files Reviewed
{complete list of every file reviewed, grouped by batch}
```

### FR-7: Self-Review Checklist

- [ ] Read every findings file from every batch
- [ ] Deduplication applied at all 3 levels
- [ ] Confidence elevated for multi-agent consensus (NOT severity)
- [ ] Top 10 Critical/High findings spot-checked against actual code
- [ ] All 4 cross-cutting analyses performed
- [ ] Root AGENTS.md updated with Known Issues and Technical Debt
- [ ] REVIEW_REPORT.md written with all required sections
- [ ] Statistics table counts match actual findings
- [ ] Every file in the appendix was actually reviewed
- [ ] Executive summary is specific (not generic)

## Acceptance Criteria

- [ ] `autopsy/agents/synthesizer.md` exists with valid YAML frontmatter
- [ ] Agent has all 5 steps with complete instructions
- [ ] Three-level dedup algorithm is clearly documented
- [ ] Spot-check step with top 10 verification is present
- [ ] All 4 cross-cutting analysis sections with commands
- [ ] REVIEW_REPORT.md template with all sections
- [ ] Self-review checklist present and comprehensive
- [ ] Root AGENTS.md update respects idempotency marker

## Out of Scope

- Orchestrator logic for launching this agent (Track 05)
- Review agent definitions (Track 03)
