# Interface Contracts

> Track-to-track file contracts. Each interface has exactly one owner.
> In this plugin, "interfaces" are the file formats written by one component and read by another.
>
> Updated by: `/architect-decompose` (initial), `/architect-sync` (discovery-driven changes)

---

## File Contracts

### Discovery Agent → Orchestrator

**Owner:** Track 02_discovery_agent

| File | Format | Purpose |
|------|--------|---------|
| `.deep-review/discovery.md` | Markdown (repo profile template) | Repo stats, module summary, risk areas, architecture diagram |
| `.deep-review/batch-plan.md` | Markdown (batch plan template) | Batch list with directories, file counts, LOC, focus areas |
| `.deep-review/file-list.txt` | Plain text (one path per line) | Complete list of files to review |

**Consumed by:** Track 05_orchestrator_command (reads batch plan to launch review agents)

---

### Review Agents → Synthesizer

**Owner:** Track 03_review_agents

| File | Format | Purpose |
|------|--------|---------|
| `.deep-review/batch-{N}/bugs.md` | Markdown (finding format) | Bug-hunter findings for batch N |
| `.deep-review/batch-{N}/security.md` | Markdown (finding format) | Security-auditor findings for batch N |
| `.deep-review/batch-{N}/errors.md` | Markdown (finding format) | Error-inspector findings for batch N |
| `.deep-review/batch-{N}/performance.md` | Markdown (finding format) | Performance-detector findings for batch N |
| `.deep-review/batch-{N}/stack.md` | Markdown (finding format) | Stack-reviewer findings for batch N |

**Consumed by:** Track 04_synthesizer_agent (reads all findings for dedup + synthesis)

---

### Orchestrator → All Agents

**Owner:** Track 05_orchestrator_command

| Data | Passed Via | Purpose |
|------|-----------|---------|
| Batch assignment (directories, files) | Task tool `prompt` parameter | Tells each review agent which files to examine |
| Output file path | Task tool `prompt` parameter | Tells each agent where to write results |
| Discovery context path | Task tool `prompt` parameter | Points agents to AGENTS.md files and discovery.md |

---

### Synthesizer → User

**Owner:** Track 04_synthesizer_agent

| File | Format | Purpose |
|------|--------|---------|
| `REVIEW_REPORT.md` | Markdown (report template) | Final severity-graded review report |

**Consumed by:** User (reads directly)

---

### Discovery Agent → All Future Claude Sessions

**Owner:** Track 02_discovery_agent

| File | Format | Purpose |
|------|--------|---------|
| `{dir}/AGENTS.md` | Markdown (AGENTS.md standard) | Cross-LLM module documentation |
| `{dir}/CLAUDE.md` | Markdown (`@AGENTS.md` reference) | Claude Code compatibility |
| Root `AGENTS.md` | Markdown (AGENTS.md standard) | Project-wide documentation |

**Consumed by:** All LLM coding tools, all review agents (read AGENTS.md before reviewing)

---

### Architecture Agents → Architecture Synthesizer

**Owner:** Track 08_architecture_agents

| File | Format | Purpose |
|------|--------|---------|
| `.deep-review/architecture-analysis.md` | Markdown (structured sections) | Architect agent output: component map, interactions, tooling evaluation, quality attributes, design gaps |
| `.deep-review/best-practices-research.md` | Markdown (structured sections) | Researcher agent output: per-technology research, pattern research, alternative tools |

**Consumed by:** Track 08_architecture_agents (architecture-synthesizer reads both)

---

### Architecture Synthesizer → User

**Owner:** Track 08_architecture_agents

| File | Format | Purpose |
|------|--------|---------|
| `ARCHITECTURE_REPORT.md` | Markdown (report template — 7 sections + appendices) | Final architecture assessment report |

**Consumed by:** User (reads directly, alongside REVIEW_REPORT.md)

---

### Orchestrator → Architecture Agents

**Owner:** Track 09_architecture_orchestration

| Data | Passed Via | Purpose |
|------|-----------|---------|
| Discovery context path | Task tool `prompt` parameter | Points architecture agents to discovery.md and AGENTS.md files |
| Output file path | Task tool `prompt` parameter | Tells each architecture agent where to write results |

---

## Shared Data Schemas

### Finding Format (used by all review agents)

```markdown
## {SEVERITY_EMOJI} {SEVERITY}: {short description}
**File:** `{exact/file/path.ext}`
**Line(s):** {line number or range}
**Category:** {category name}
**Confidence:** {High|Medium|Low}
**Description:** {what's wrong and why}
**Impact:** {what breaks, data loss, crash, incorrect results}
**Fix:** {specific code change or approach}
```

**Owned by:** Cross-cutting (defined in cross-cutting.md v1)
**Used by:** All review agents, synthesizer (for parsing and dedup)

### Batch Plan Entry (used by discovery → orchestrator)

```markdown
## Batch {N}: {descriptive name}
Risk level: {high/medium/low}
Directories:
- {path/to/dir1}
- {path/to/dir2}
Files: {count}
LOC: {count}
Focus areas: {what to look for specifically}
Stack-specific checks:
- {check 1}
- {check 2}
```

**Owned by:** Track 02_discovery_agent
**Used by:** Track 05_orchestrator_command (parses to launch agents)

---

## Contract Change Protocol

When a file contract needs to change:

1. Owner proposes change in interfaces.md
2. All consumers listed under the interface are checked:
   - NOT_STARTED: auto-inherit via header regeneration
   - IN_PROGRESS: flag for developer review
   - COMPLETE: INTERFACE_MISMATCH discovery → patch phase if needed
3. Breaking changes require developer approval before applying
