---
description: "Orchestrates a multi-agent, exhaustive code review of the entire repository across three phases: Discovery, Review, and Synthesis."
---

# Deep Review: Full Review

You are the orchestrator for a deep, multi-agent code review. You coordinate three phases:
1. **Discovery** — Map the repo, generate documentation, create a batch plan
2. **Review** — Launch 5 specialized agents in parallel per batch
3. **Synthesis** — Deduplicate findings, cross-cut analysis, produce final report

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
  "false_positives_removed": 0
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
  ◇ Review
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

## Phase 2: Review

### Step 6: Process batches sequentially

For each batch in the batch plan (highest risk first).

Print (State 2 — Review active):
```
  ✓ Discovery · {N} files across {M} modules · {time}
  ──────────────────────────────────────────────────
  ⠹ Review · batch {N}/{total} · {files} files, ~{LOC} LOC

  ◇ Next: synthesize → REVIEW_REPORT.md
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
```

**This state write is critical — it enables recovery after compaction.**

#### 6c. Launch 5 review agents in PARALLEL (foreground)

Launch all 5 review agents as separate foreground Task calls in a **single response**. Claude Code executes them concurrently when multiple Task calls are made in the same message. Each agent writes its own output file using the Write tool (which is fully available in foreground agents).

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

After all 5 foreground agents return, verify their output files exist:

```bash
ls .deep-review/batch-{N}/bugs.md .deep-review/batch-{N}/security.md .deep-review/batch-{N}/errors.md .deep-review/batch-{N}/performance.md .deep-review/batch-{N}/stack.md 2>/dev/null | wc -l
```

For any missing file, the agent failed to write its output. Log to `state.json` under `"agent_failures"` and to `progress.md`. Optionally retry ONCE with a refined prompt (add: "Previous attempt failed. Ensure you write output to the specified file path using the Write tool.")

**Do NOT block the entire review on a single agent failure.**

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

After all batches complete:

Update `state.json`:
- `"phase": "synthesis"`
- `"phase_times": {..., "review": "{elapsed}"}`

Print (collapsed Review line — green ✓):
```
  ✓ Review · 5 agents · {time}
```

---

## Phase 3: Synthesis

### Step 8: Launch synthesizer agent

Print (State 3 — Synthesis active):
```
  ✓ Discovery · {N} files across {M} modules · {time}
  ✓ Review · 5 agents · {time}
  ──────────────────────────────────────────────────
  ⠹ Synthesis · deduplicating
```

Launch the synthesizer agent using the **Task tool** in FOREGROUND (blocking):

**Prompt for the synthesizer agent:**
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

### Step 9: Verify synthesis output

After the synthesizer completes:

1. Verify `REVIEW_REPORT.md` exists in the repo root
2. If missing, retry synthesizer once
3. If `REVIEW_REPORT.md` still does not exist after retry, create a minimal fallback report:
   - List total files reviewed (from discovery.md)
   - List paths to raw findings: `ls .deep-review/batch-*/*.md`
   - Note: "Synthesis failed. Raw findings are available in the .deep-review/ directory."
   - This preserves the review work instead of losing it entirely.
4. Read the statistics section of REVIEW_REPORT.md for final counts

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
  ✓ Synthesis · {time}
  ──────────────────────────────────────────────────

  Deep Review Complete — {total time} total

  {N} critical  ·  {N} high  ·  {N} medium  ·  {N} low     {total} issues
```

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
```
  ── Next ──────────────────────────────────────────

  1. Fix critical issues first
  2. Read REVIEW_REPORT.md for high/medium details
  3. Run /deep-review:maintain-docs after changes
```

Then print reports section:
```
  ── Reports ───────────────────────────────────────

  → REVIEW_REPORT.md         full findings
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
   - If first line contains `<!-- progress v2 -->`: parse as key-value pairs (phase, batch, completed, severity counts, failures)
   - Otherwise: parse as v1 Markdown format (heading-based sections with emoji severity markers)
4. **Read batch plan:** `cat .deep-review/batch-plan.md`
5. **Resume from where you left off:**
   - If `phase` is "discovery": Discovery agent is running or failed — check for output files and proceed to Phase 2 if they exist
   - If `phase` is "review": Check `batches_completed` and `current_batch` — skip completed batches, resume from current
   - If `phase` is "synthesis": Synthesizer is running or failed — check for REVIEW_REPORT.md
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
| Context compaction | Re-read state.json + progress.md + batch-plan.md, resume |
