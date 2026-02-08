---
description: "Orchestrates a multi-agent, exhaustive code review of the entire repository across three phases: Discovery, Review, and Synthesis."
---

# Deep Review: Full Review

You are the orchestrator for a deep, multi-agent code review and architecture assessment. You coordinate three phases:
1. **Discovery** — Map the repo, generate documentation, create a batch plan
2. **Review + Architecture** — Launch 5 code review agents per batch in parallel; architect and researcher agents run alongside the first batch
3. **Synthesis** — Deduplicate code review findings and produce REVIEW_REPORT.md; architecture-synthesizer cross-references analysis and produces ARCHITECTURE_REPORT.md

**CRITICAL RULES:**
- You NEVER read source code files directly — only batch plans, progress files, and output existence checks
- All heavy work happens in sub-agents with fresh context windows
- Disk (`.deep-review/` directory) is the coordination layer between phases
- Write state to disk before every batch — this makes the review resumable after compaction
- Follow the output rendering guide at `references/output-rendering.md` for ALL terminal output formatting. Use the exact symbols, colors, and patterns defined there. Never use banned patterns.

---

## Setup

### Step 1: Initialize

```bash
mkdir -p .deep-review
```

Add `.deep-review/` to `.gitignore` if not already present:

```bash
grep -q '.deep-review/' .gitignore 2>/dev/null || echo '.deep-review/' >> .gitignore
```

### Step 2: Record start time

Write initial state:

```json
{
  "phase": "setup",
  "started_at": "{ISO timestamp}",
  "phase_times": {},
  "batches_total": 0,
  "batches_completed": 0,
  "findings": {"critical": 0, "high": 0, "medium": 0, "low": 0},
  "agent_failures": [],
  "false_positives_removed": 0,
  "phase_2a_status": "pending",
  "phase_3a_status": "pending",
  "architecture_analysis_exists": false,
  "best_practices_research_exists": false,
  "architecture_report_exists": false
}
```

Write this to `.deep-review/state.json`.

### Step 3: Set effort level

Run `/effort max` to ensure maximum reasoning depth for all sub-agents.

---

## Phase 1: Discovery

### Step 4: Launch discovery agent

Update `state.json`: set `"phase": "discovery"`.

Print (State 1 — Discovery active):
```
  ⠹ Discovery · scanning repository...
  ◇ Review + Architecture
  ◇ Synthesis
```

Launch the discovery agent using the **Task tool** in FOREGROUND (blocking):

**Prompt for the discovery agent:**
> You are the discovery agent for a deep code review. Your job is to map the entire repository, understand every module, create AGENTS.md documentation files, and produce a batch plan for the review phase.
>
> Follow the instructions in your agent definition file (`agents/discovery.md`) exactly. Complete all 5 steps:
> 1. Map the repository (file inventory, stats, metadata)
> 2. Understand every module (read files, analyze each directory)
> 3. Generate AGENTS.md files (with companion CLAUDE.md) for directories with 3+ code files
> 4. Write the discovery profile to `.deep-review/discovery.md`
> 5. Generate the batch plan to `.deep-review/batch-plan.md`
>
> Run your self-review checklist before completing.

**Agent configuration:**
- `subagent_type`: Use the discovery agent
- Do NOT run in background — wait for completion

### Step 5: Read discovery results

After the discovery agent completes:

1. Check if `.deep-review/discovery.md` exists — if NOT, print "Discovery failed: discovery.md not found. Cannot proceed." and abort.
2. Check if `.deep-review/batch-plan.md` exists — if NOT, print "Discovery failed: batch-plan.md not found. Cannot proceed." and abort.
3. Read `.deep-review/batch-plan.md` to determine:
   - Total number of batches
   - Files per batch
   - Total files to review
4. Read `.deep-review/discovery.md` for repo stats (total files, modules, AGENTS.md count)

Update `state.json`:
- `"phase": "review"`
- `"batches_total": {N}`
- `"phase_times": {"discovery": "{elapsed}"}`

Print (collapsed Discovery line — green ✓):
```
  ✓ Discovery · {N} files across {M} modules · {time}
```

### Step 5a: Gather recent changes context

If this is a git repository, capture recent intentional changes:

```bash
git log --oneline -10 2>/dev/null > .deep-review/recent-changes.txt || echo "Not a git repo" > .deep-review/recent-changes.txt
```

This file will be passed to review agents so they don't flag intentional recent changes as bugs.

---

## Phase 2: Review + Architecture

### Step 5b: Prepare architecture agent prompts

The architect and researcher agents will launch alongside the first review batch. Prepare their prompts now.

**Prompt for the architect agent:**
> You are the architect agent for a deep architecture assessment. Your job is to analyze the system's architecture: components, interactions, tooling choices, quality attributes, and design gaps.
>
> First read `.deep-review/discovery.md` for the full repository map and module inventory.
>
> Then follow the instructions in your agent definition file (`agents/architect.md`) exactly. Complete all 6 steps:
> 1. Infer the System Goal
> 2. Decompose into Components
> 3. Analyze Component Interactions
> 4. Evaluate Tooling Choices
> 5. Quality Attribute Tradeoff Analysis
> 6. Identify Design Gaps
>
> Write your output to `.deep-review/architecture-analysis.md`.
>
> Run your self-review checklist before completing.

**Prompt for the researcher agent:**
> You are the researcher agent for a deep architecture assessment. Your job is to research official documentation, best practices, and recommended patterns for every major technology in the stack.
>
> First read `.deep-review/discovery.md` to identify the technologies and architectural patterns in use.
>
> Then follow the instructions in your agent definition file (`agents/researcher.md`) exactly. Complete all 5 steps:
> 1. Identify Research Targets
> 2. Research Each Technology
> 3. Research the Architectural Pattern
> 4. Research Alternative Tools
> 5. Research Interaction Patterns
>
> Write your output to `.deep-review/best-practices-research.md`.
>
> Run your self-review checklist before completing.

Update `state.json`: set `"phase_2a_status": "running"`.

### Step 6: Process batches sequentially

For each batch in the batch plan (highest risk first).

Print (State 2 — Review + Architecture active):
```
  ✓ Discovery · {N} files across {M} modules · {time}
  ──────────────────────────────────────────────────
  ⠹ Review · batch {N}/{total} · {files} files, ~{LOC} LOC
  ⠹ Architecture · analyzing...

  ◇ Next: synthesize → REVIEW_REPORT.md + ARCHITECTURE_REPORT.md
```

For each batch:

#### 6a. Create batch directory

```bash
mkdir -p .deep-review/batch-{N}
```

#### 6b. Write state before launching agents

Update `state.json`:
- `"current_batch": {N}`
- Save all current progress

Write `.deep-review/progress.md` (v2 dense format):
```
<!-- progress v2 -->
phase: review
batch: {N}/{total}
completed: {comma-separated list of completed batch numbers}
critical: {N} | high: {N} | medium: {N} | low: {N}
failures: {comma-separated list or "none"}
arch: {pending|running|complete|failed}
```

**This state write is critical — it enables recovery after compaction.**

#### 6c. Launch review agents (and architecture agents for first batch) in PARALLEL (foreground)

Launch review agents as separate foreground Task calls in a **single response**. Claude Code executes them concurrently when multiple Task calls are made in the same message. Each agent writes its own output file using the Write tool (which is fully available in foreground agents).

**For the FIRST batch only:** Launch 7 agents in a single message — the 5 review agents PLUS the architect and researcher agents (using the prompts from Step 5b). All 7 run concurrently as parallel foreground calls.

**For subsequent batches (batch 2+):** Launch only the 5 review agents as normal.

For each of the 5 review agents (bug-hunter, security-auditor, error-inspector, performance-detector, stack-reviewer):

**Prompt template for each review agent:**
> You are the {agent-name} agent for a deep code review.
>
> **Your assignment — Batch {N}: {batch descriptive name}**
>
> Review these directories:
> {list of directories from batch plan}
>
> **Instructions:**
> 1. Read the AGENTS.md file in each assigned directory FIRST for module context
> 2. Read the FULL contents of EVERY file in your assigned directories — no skimming
> 3. Check every item in your agent checklist (from your agent definition)
> 4. Write your findings to `.deep-review/batch-{N}/{agent-output-file}.md`
>
> **Output file:** `.deep-review/batch-{N}/{output-name}.md`
>
> **Finding format (STRICT):**
> ```
> ## {SEVERITY_EMOJI} {SEVERITY}: {short description}
> **File:** `{exact/file/path.ext}`
> **Line(s):** {line number or range}
> **Category:** {category}
> **Confidence:** {High|Medium|Low}
> **Description:** {what's wrong}
> **Impact:** {what breaks}
> **Fix:** {specific fix}
> ```
>
> **Severity scale:**
> - 🔴 CRITICAL: Crashes, data loss, security breaches in production
> - 🟠 HIGH: Bugs under common conditions
> - 🟡 MEDIUM: Edge case issues
> - 🔵 LOW: Could lead to future bugs
>
> **Confidence rules:**
> - High: Exact code path shown
> - Medium: Pattern suggests issue, can't fully trace
> - Low: Heuristic/inferred
>
> **Mandatory rules:**
> - Read FULL contents of every file
> - Do NOT say "looks fine" or skip files
> - Every finding needs exact file path and line number
> - Re-read 100+ line files with zero findings
> - Report undocumented gotchas as "Documentation Gap" findings
> - Focus on YOUR domain expertise. If an issue clearly belongs to another agent's domain (e.g., security for bug-hunter), note it briefly as a one-liner but don't write a full finding
> - Read `.deep-review/recent-changes.txt` for context on recent intentional changes. Do not flag recently committed changes as missing features unless there's an actual bug.
>
> Start your output with a header:
> ```
> # {Agent Name} — Batch {N} Findings
> **Files reviewed:** {count}
> **Total findings:** {count}
> **By severity:** 🔴 {n} | 🟠 {n} | 🟡 {n} | 🔵 {n}
> ```
>
> Run your self-review checklist before completing.

**Agent-to-output mapping:**
| Agent | Output File |
|-------|------------|
| bug-hunter | `bugs.md` |
| security-auditor | `security.md` |
| error-inspector | `errors.md` |
| performance-detector | `performance.md` |
| stack-reviewer | `stack.md` |

#### 6d. Verify agent output

After all foreground agents return, verify their output files exist:

```bash
ls .deep-review/batch-{N}/bugs.md .deep-review/batch-{N}/security.md .deep-review/batch-{N}/errors.md .deep-review/batch-{N}/performance.md .deep-review/batch-{N}/stack.md 2>/dev/null | wc -l
```

For any missing file, the agent failed to write its output. Log to `state.json` under `"agent_failures"` and to `progress.md`. Optionally retry ONCE with a refined prompt (add: "Previous attempt failed. Ensure you write output to the specified file path using the Write tool.")

**Do NOT block the entire review on a single agent failure.**

#### 6d-arch. Verify architecture output (first batch only)

After the first batch completes, also verify the architecture agent outputs:

```bash
ls .deep-review/architecture-analysis.md .deep-review/best-practices-research.md 2>/dev/null | wc -l
```

- If both files exist: update `state.json` with `"phase_2a_status": "complete"`, `"architecture_analysis_exists": true`, `"best_practices_research_exists": true`.
- If one or both are missing: log to `"agent_failures"`, retry the missing agent(s) ONCE. If still missing after retry, set `"phase_2a_status": "failed"` and update the boolean flags accordingly. **Continue with code review — architecture failure never blocks the review pipeline.**

Print architecture status after first batch:
- Success: `✓ Architecture · {time}`
- Partial: `✓ Architecture · partial ({which} completed)`
- Failed: `✗ Architecture · failed (partial report available)`

#### 6e. Count findings and update progress

For each completed output file, count findings by severity:

```bash
grep -c '🔴 CRITICAL' .deep-review/batch-{N}/bugs.md 2>/dev/null || echo 0
grep -c '🟠 HIGH' .deep-review/batch-{N}/bugs.md 2>/dev/null || echo 0
# ... for each file and severity
```

Update `state.json` and `progress.md` with running totals.

Print agent completion status (one line, ✓ for success, ✗ in red for failure):
```
  ✓ Bug Hunter · ✓ Security · ✓ Error Handling · ✓ Performance · ✗ Stack Review
```

Then update the batch counter in the review status line.

#### 6f. Batch sizing strategy

The discovery agent creates the batch plan, but the orchestrator adapts its processing strategy based on repo size:

| Repo Size | Files | Parallel Batches | Compaction Strategy |
|-----------|-------|-----------------|---------------------|
| Small | < 50 | All at once | None needed |
| Medium | 50-200 | 1 batch at a time | Auto-compaction handles it |
| Large | 200-500 | 1 batch at a time | Write state before each batch |
| Huge | 500+ | 1 batch at a time | Write state before each batch; expect auto-compaction between batches |

### Step 7: Phase 2 summary

After all batches complete (and architecture agents have finished or failed):

Update `state.json`:
- `"phase": "synthesis"`
- `"phase_times": {..., "review": "{elapsed}"}`

Print (collapsed Review line — green ✓):
```
  ✓ Review · 5 agents · {time}
```

Print architecture status line (already printed after first batch in Step 6d-arch, but confirm here):
- If `phase_2a_status` is "complete": `✓ Architecture · {time}`
- If `phase_2a_status` is "failed": `✗ Architecture · failed`

---

## Phase 3: Synthesis

### Step 8: Launch both synthesizers in PARALLEL

Print (State 3 — Synthesis active):
```
  ✓ Discovery · {N} files across {M} modules · {time}
  ✓ Review · 5 agents · {time}
  ✓ Architecture · {time}
  ──────────────────────────────────────────────────
  ⠹ Synthesis · deduplicating + architecture report
```

If `phase_2a_status` is "failed", replace the Architecture line with `✗ Architecture · failed`.

Update `state.json`: set `"phase_3a_status": "running"`.

Launch BOTH synthesizers as separate foreground Task calls in a **single response** (2 parallel tasks):

**Prompt for the code review synthesizer (unchanged):**
> You are the synthesizer for a deep code review. Your job is to read all findings, deduplicate them, spot-check top issues, perform cross-cutting analysis, and produce the final report.
>
> Follow the instructions in your agent definition file (`agents/synthesizer.md`) exactly. Complete all 5 steps:
> 1. Read ALL findings from ALL `.deep-review/batch-*/` directories
> 2. Apply three-level deduplication (exact match, semantic overlap, systemic patterns)
> 3. Spot-check top 10 Critical/High findings by re-reading actual code
> 4. Perform all 4 cross-cutting analyses (architecture, dependencies, testing gaps, secrets)
> 5. Update root AGENTS.md and write REVIEW_REPORT.md to the repo root
>
> **Important:**
> - Elevate confidence for multi-agent consensus — do NOT change severity
> - Include ALL findings (no cap per severity level)
> - Update root AGENTS.md only (not subdirectory files)
>
> Run your self-review checklist before completing.

**Prompt for the architecture-synthesizer:**
> You are the architecture-synthesizer for a deep architecture assessment. Your job is to cross-reference all architecture analysis inputs and produce the final ARCHITECTURE_REPORT.md.
>
> Follow the instructions in your agent definition file (`agents/architecture-synthesizer.md`) exactly. Complete all 4 steps:
> 1. Read all inputs from `.deep-review/` (architecture-analysis.md, best-practices-research.md, discovery.md, batch-*/*.md for code evidence)
> 2. Cross-reference design gaps vs research, tooling vs research, code findings as evidence
> 3. Generate design recommendations with priority, effort, impact, and evidence
> 4. Write ARCHITECTURE_REPORT.md to the repo root
>
> **Important:**
> - If any input file is missing, follow your missing-input handling logic to produce a partial report with placeholders
> - Include evidence from code review findings where they support architecture recommendations
> - Every recommendation must cite specific evidence (analysis finding, research source, or code example)
>
> Run your self-review checklist before completing.

If `phase_2a_status` is "failed", prepend this to the architecture-synthesizer prompt:
> **Note:** Some architecture analysis inputs may be missing (architecture agents failed during Phase 2A). Follow your missing-input handling logic to produce a partial report noting which inputs were unavailable.

### Step 9: Verify synthesis output

After both synthesizers complete:

**9a. Verify code review synthesis:**
1. Verify `REVIEW_REPORT.md` exists in the repo root
2. If missing, retry code review synthesizer once
3. If `REVIEW_REPORT.md` still does not exist after retry, create a minimal fallback report:
   - List total files reviewed (from discovery.md)
   - List paths to raw findings: `ls .deep-review/batch-*/*.md`
   - Note: "Synthesis failed. Raw findings are available in the .deep-review/ directory."
   - This preserves the review work instead of losing it entirely.
4. Read the statistics section of REVIEW_REPORT.md for final counts

**9b. Verify architecture synthesis:**
1. Verify `ARCHITECTURE_REPORT.md` exists in the repo root
2. If missing, retry architecture-synthesizer once
3. If `ARCHITECTURE_REPORT.md` still does not exist after retry, create a minimal fallback:
   ```markdown
   # Architecture Report

   Architecture synthesis was unable to complete.

   ## Partial Results
   - architecture-analysis.md: {exists/missing}
   - best-practices-research.md: {exists/missing}

   ## Raw Data
   Available inputs are in `.deep-review/`. Run a full review again to regenerate.
   ```
4. Update `state.json`: `"phase_3a_status": "complete"` or `"phase_3a_status": "failed"`, `"architecture_report_exists": true/false`

Update `state.json`:
- `"phase": "complete"`
- `"phase_times": {..., "synthesis": "{elapsed}"}`

Print (collapsed Synthesis line — green ✓):
```
  ✓ Synthesis · {time}
```

---

## Final Summary

### Step 10: Print completion summary

Read final statistics from `state.json` and REVIEW_REPORT.md.

Read the Critical Issues section from `REVIEW_REPORT.md`. Extract each critical issue's title, one-line impact summary, and file path(s). If zero critical issues exist, extract the top 3 high-severity issues instead.

Print (State 4 — Complete):
```
  ✓ Discovery · {N} files · {time}
  ✓ Review · 5 agents · {time}
  ✓ Architecture · {time}
  ✓ Synthesis · {time}
  ──────────────────────────────────────────────────

  Deep Review Complete — {total time} total

  {N} critical  ·  {N} high  ·  {N} medium  ·  {N} low     {total} issues
```

If `phase_2a_status` is "failed", replace the Architecture line with `✗ Architecture · failed`.

Print severity counts with named colors: critical count in red, high count in orange, medium count in yellow, low count in dim.

Then print critical issues section. If critical issues exist:
```
  ── Critical ──────────────────────────────────────

  1  {Issue title}
     {Impact summary — one line}
     → {file path(s)}

  2  {Issue title}
     {Impact summary — one line}
     → {file path(s)}
```

If zero critical issues, show top 3 high-severity instead:
```
  ── High ──────────────────────────────────────────

  1  {Issue title}
     {Impact summary — one line}
     → {file path(s)}
```

If zero critical AND zero high issues, print: `No critical or high issues found.`

Then print next steps (numbered, imperative verbs — no "Consider" or "You should"):

If architecture succeeded:
```
  ── Next ──────────────────────────────────────────

  1. Fix critical issues first
  2. Read REVIEW_REPORT.md for code-level details
  3. Read ARCHITECTURE_REPORT.md for design-level assessment
  4. Run /deep-review:maintain-docs after changes
```

If architecture failed:
```
  ── Next ──────────────────────────────────────────

  1. Fix critical issues first
  2. Read REVIEW_REPORT.md for code-level details
  3. Re-run full review to regenerate architecture assessment
  4. Run /deep-review:maintain-docs after changes
```

Then print reports section:

If architecture succeeded:
```
  ── Reports ───────────────────────────────────────

  → REVIEW_REPORT.md         code review findings
  → ARCHITECTURE_REPORT.md   design assessment
  → AGENTS.md (×{N})         module documentation
  → .deep-review/            raw findings + state
```

If architecture failed (omit ARCHITECTURE_REPORT.md line):
```
  ── Reports ───────────────────────────────────────

  → REVIEW_REPORT.md         code review findings
  → AGENTS.md (×{N})         module documentation
  → .deep-review/            raw findings + state
```

Do NOT follow the summary with any prose paragraph.

---

## Compaction Recovery

If context compaction occurs at any point during the review:

1. **Read state:** `cat .deep-review/state.json`
2. **Read progress:** `cat .deep-review/progress.md`
3. **Parse progress.md (dual-format support):**
   - If first line contains `<!-- progress v2 -->`: parse as key-value pairs (phase, batch, completed, severity counts, failures, arch)
   - Otherwise: parse as v1 Markdown format (heading-based sections with emoji severity markers)
4. **Read batch plan:** `cat .deep-review/batch-plan.md`
5. **Recover architecture state:**
   - If `phase_2a_status` is "running": check for `.deep-review/architecture-analysis.md` and `.deep-review/best-practices-research.md`. If both exist, set `phase_2a_status` to "complete". If missing, set to "failed". **Architecture recovery never blocks code review recovery.**
   - If `phase_3a_status` is "running": check for `ARCHITECTURE_REPORT.md`. If exists, set to "complete". If missing and Phase 2 code review is complete, retry architecture-synthesizer once. If still missing, set to "failed".
6. **Resume from where you left off:**
   - If `phase` is "discovery": Discovery agent is running or failed — check for output files and proceed to Phase 2 if they exist
   - If `phase` is "review": Check `batches_completed` and `current_batch` — skip completed batches, resume from current. Also check architecture state per Step 5 above.
   - If `phase` is "synthesis": Check for REVIEW_REPORT.md and ARCHITECTURE_REPORT.md — retry missing synthesizers
   - If `phase` is "complete": Print the final summary

**This recovery logic is the reason we write state to disk before every batch.** Auto-compaction can happen at any time — the review must be resilient to it.

---

## Error Handling Summary

| Error | Action |
|-------|--------|
| Discovery agent fails | Abort review — cannot proceed without batch plan |
| Review agent output missing | Log failure, retry once, continue if still missing |
| All 5 agents fail for a batch | Log as failed batch, continue to next batch |
| Synthesizer fails | Retry once. If still fails, point user to raw findings in `.deep-review/batch-*/` |
| REVIEW_REPORT.md missing | Retry synthesizer. If still missing, create a minimal report listing raw finding file paths |
| Architect agent output missing | Log failure, retry once. If still missing, mark phase_2a as failed, continue code review |
| Researcher agent output missing | Log failure, retry once. If still missing, mark phase_2a as failed, continue code review |
| Architecture-synthesizer fails | Retry once. If still fails, create minimal fallback ARCHITECTURE_REPORT.md |
| ARCHITECTURE_REPORT.md missing | Retry architecture-synthesizer. If still missing, create fallback report |
| Architecture compaction recovery | Check output files exist, mark phase complete or failed accordingly |
| Context compaction | Re-read state.json + progress.md + batch-plan.md, recover architecture state, resume |
