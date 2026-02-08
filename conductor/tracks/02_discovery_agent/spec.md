<!-- ARCHITECT CONTEXT | Track: 02_discovery_agent | Wave: 2 | CC: v1 -->

## Cross-Cutting Constraints
- AGENTS.md Quality Bar: specific to actual code, under 80 lines (root) / 60 lines (subdirectory), no file trees, three-tier boundary system
- Dual Documentation Output: AGENTS.md primary + CLAUDE.md companion with `@AGENTS.md`
- Batch Sizing Constraints: under 50 files or 5000 LOC per batch, group related modules, order by risk, target 30-40K tokens per agent
- Thoroughness Rules: read full contents, no skimming, exact file paths

## Interfaces

### Owns
- `.autopsy/discovery.md` (repo profile)
- `.autopsy/batch-plan.md` (batch list for review phase)
- `.autopsy/file-list.txt` (complete file inventory)
- `{dir}/AGENTS.md` files (module documentation)
- `{dir}/CLAUDE.md` companion files

### Consumes
- None (first phase — reads the target repo directly)

## Dependencies
- Track 01_plugin_scaffold: `agents/` directory must exist

<!-- END ARCHITECT CONTEXT -->

# Specification: Discovery Agent

## Overview

Define the discovery agent (`autopsy/agents/discovery.md`) — the Phase 1 agent that maps an entire repository, understands every module, generates AGENTS.md documentation throughout the codebase, and produces a batch plan for the orchestrator to coordinate Phase 2 review agents. This agent runs in foreground (blocking) with a fresh context. The quality of its output directly determines the quality of the entire review.

## Functional Requirements

### FR-1: Agent Definition File

Create `autopsy/agents/discovery.md` (replacing the placeholder) with:
- YAML frontmatter: `name: discovery`, `description`, `tools` list (Bash, Read, Write, Glob, Grep)
- Complete step-by-step instructions the agent must follow (Steps 1-5 below)
- Self-review checklist at the end

### FR-2: Step 1 — Repo Mapping

The agent must map the repository structure by running these commands:

```bash
# Generate file inventory (excluding common non-code dirs)
find . -type f | grep -v -E '(node_modules|\.git/|__pycache__|\.pyc|venv|\.venv|\.env$|dist/|build/|\.next|\.cache|coverage|\.idea|\.vscode|\.mypy_cache|\.pytest_cache|\.ruff_cache|egg-info|\.tox|target/|bin/|obj/|\.autopsy)' | sort > .autopsy/file-list.txt

# Count total files
wc -l .autopsy/file-list.txt

# File extension distribution
cat .autopsy/file-list.txt | sed 's/.*\.//' | sort | uniq -c | sort -rn | head -30

# Top-level directory distribution
cat .autopsy/file-list.txt | cut -d/ -f2 | sort | uniq -c | sort -rn

# Total LOC for code files
find . -type f \( -name '*.py' -o -name '*.js' -o -name '*.ts' -o -name '*.tsx' -o -name '*.jsx' -o -name '*.go' -o -name '*.rs' -o -name '*.java' -o -name '*.sql' \) | grep -v -E '(node_modules|\.git/|__pycache__|venv|dist|build)' | xargs wc -l 2>/dev/null | tail -1

# Read project metadata
ls -la
cat README.md 2>/dev/null || echo "No README"
cat pyproject.toml 2>/dev/null || cat package.json 2>/dev/null || cat Cargo.toml 2>/dev/null || cat go.mod 2>/dev/null || echo "No manifest"
cat requirements.txt 2>/dev/null || cat Pipfile 2>/dev/null || echo "No requirements"
cat Dockerfile 2>/dev/null || echo "No Dockerfile"
cat docker-compose.yml 2>/dev/null || cat docker-compose.yaml 2>/dev/null || echo "No docker-compose"
ls .github/workflows/ 2>/dev/null || echo "No CI"
```

Output: `.autopsy/file-list.txt` (one file path per line)

### FR-3: Step 2 — Module Understanding

For each top-level directory, the agent must understand the module by reading files:
- **Directories with <10 files:** Read ALL files completely
- **Directories with 10+ files:** Read 3-5 representative files (entry points, core logic, config)

For each directory, determine:
- Purpose (what does it do?)
- Key files (entry points, core logic)
- Internal dependencies (what other modules it imports)
- External dependencies (libraries, APIs, databases)
- Data flow (input → processing → output)
- Patterns and conventions used
- Configuration (env vars, config files)
- Gotchas (non-obvious behavior, complexity)
- Testing (where tests live, framework, gaps)

### FR-4: Step 3 — AGENTS.md Generation

**Trigger:** Generate an AGENTS.md for every directory with 3+ code files or complex logic.

**Template for subdirectory AGENTS.md** (must stay under 60 lines):
```markdown
# {Module Name}

## Purpose
{2-3 sentences}

## Key Files
| File | Purpose | Key Exports |
|------|---------|-------------|
| {f}  | {what}  | {functions/classes} |

## Data Flow
{input → processing → output}

## Dependencies
- Internal: {other modules}
- External: {libraries, APIs, databases}

## Patterns & Conventions
{module-specific patterns}

## Configuration
{env vars, config files}

## Gotchas
- {non-obvious behavior}
- {fragile areas}

## Testing
- Tests: {path}
- Run: {command}
- Gaps: {what's not tested}

## Recent Changes
{empty — will be populated by maintenance}
```

**Template for root AGENTS.md** (must stay under 80 lines):
- Project overview (2-3 sentences)
- Architecture diagram (text-based)
- Tech stack summary
- Directory map (purpose of each top-level dir, NOT a file tree)
- Key conventions
- Key commands (build, test, run)
- Three-tier boundaries: Always do / Ask first / Never do

**Companion CLAUDE.md files:**
At every level where an AGENTS.md is created, also create a `CLAUDE.md` containing only:
```markdown
@AGENTS.md
```

**Existing documentation handling (idempotent on re-runs):**

Every generated AGENTS.md must start with a marker comment:
```markdown
<!-- generated by autopsy -->
```

Strategy on subsequent runs:
- **No AGENTS.md exists:** Create from scratch with marker
- **AGENTS.md exists WITH marker** (previously generated by autopsy): Regenerate entirely — overwrite the whole file. This prevents duplication on repeated runs.
- **AGENTS.md exists WITHOUT marker** (manually written or from another tool): Preserve the manual content above, append autopsy content below a separator:
  ```markdown
  ---
  <!-- generated by autopsy — content below is auto-generated -->
  ```
- **CLAUDE.md handling:** Same pattern — if companion CLAUDE.md exists with `@AGENTS.md`, leave it. If missing, create it.
- Mark uncertainties with "TODO: verify"

**Quality bar (enforced via self-review):**
- Must contain information that saves 30+ minutes of code reading
- Must be specific to THIS code, not generic boilerplate
- File tables must list actual files with actual descriptions
- Data flow must describe actual data transformations
- Gotchas must describe actual non-obvious things
- "TODO: investigate" over guesses

### FR-5: Step 4 — Discovery Profile

Write `.autopsy/discovery.md`:
```markdown
# Repository Profile
- Name: {repo name}
- Stack: {languages, frameworks, databases}
- Architecture: {monolith/microservices/pipeline/library/etc}
- Total files: {count}
- Total LOC: {count}
- Modules: {count}
- AGENTS.md files created: {count}

## Architecture Diagram
{text-based diagram}

## Module Summary
| Module | Purpose | Files | LOC | Risk Level |
|--------|---------|-------|-----|------------|
| {dir}  | {what}  | {n}   | {n} | {high/med/low} |

## Risk Areas
1. {highest risk area and why}
2. {second highest}
3. {third highest}
```

### FR-6: Step 5 — Batch Plan Generation

Write `.autopsy/batch-plan.md`:
```markdown
# Review Batch Plan

Total batches: {N}
Batch sizing: {strategy chosen based on repo size}

## Batch 1: {descriptive name}
Risk level: {high/medium/low}
Directories:
- {path/to/dir1}
- {path/to/dir2}
Files: {count}
LOC: {count}
Focus areas: {what to look for specifically in this batch}
Stack-specific checks:
- {check 1 relevant to this batch}
- {check 2}

## Batch 2: {descriptive name}
...
```

**Batch grouping rules (directory-based):**
- Group files by directory tree — related files naturally co-locate
- Under 50 files or 5000 LOC per batch (whichever is smaller)
- Order batches by risk: highest-risk directories first
- Include stack-specific focus areas per batch based on what was learned in Steps 2-3

**Risk level assignment (pattern-based):**
- **High risk:** Directories containing auth logic, SQL/database queries, cryptographic operations, payment processing, user input handling, file upload, API endpoints exposed to the internet
- **Medium risk:** Business logic, data transformation, middleware, configuration management
- **Low risk:** Utilities, helpers, static assets, documentation, tests

### FR-7: Self-Review Checklist

The agent must verify before completing:
- [ ] `.autopsy/file-list.txt` exists and is non-empty
- [ ] `.autopsy/discovery.md` has all required sections (profile, diagram, module table, risk areas)
- [ ] `.autopsy/batch-plan.md` has at least 1 batch with all required fields
- [ ] Every batch has ≤50 files and ≤5000 LOC
- [ ] AGENTS.md created for every directory with 3+ code files
- [ ] Every AGENTS.md is under the line limit (80 root, 60 subdirectory)
- [ ] Every AGENTS.md has a companion CLAUDE.md with `@AGENTS.md`
- [ ] No generic boilerplate in any AGENTS.md — every statement references actual code
- [ ] Every file in file-list.txt appears in exactly one batch

## Non-Functional Requirements

- Agent definition must be a single Markdown file with YAML frontmatter
- All templates and instructions must be embedded in the agent definition (no external dependencies)
- Agent must work on any repository regardless of language/framework
- Agent must handle repos from small (<50 files) to huge (500+ files)
- The `.autopsy/` directory must be created if it doesn't exist

## Acceptance Criteria

- [ ] `autopsy/agents/discovery.md` exists with valid YAML frontmatter (name, description, tools)
- [ ] Agent definition contains all 5 steps with complete instructions
- [ ] Step 1 commands are complete and correct
- [ ] AGENTS.md template matches the quality bar from product guidelines
- [ ] Batch plan format matches the interface contract in `architect/interfaces.md`
- [ ] Self-review checklist is present and comprehensive
- [ ] Few-shot examples are NOT required for this agent (it produces docs and plans, not findings)

## Out of Scope

- Review agent definitions (Track 03)
- Synthesizer agent (Track 04)
- Orchestrator logic that launches this agent (Track 05)
- AGENTS.md maintenance/update logic (Track 06)
