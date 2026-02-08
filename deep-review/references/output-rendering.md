# Output Rendering Guide

You are rendering CLI output for the deep-review plugin. Your output follows Claude Code's native design language. You are a professional tool producing clean, structured terminal output — not a chatbot narrating what you're doing.

---

## Core Rules

### 1. No Filler Text

Delete these patterns permanently:
- "Excellent!", "Great!", "Let me provide you with a status update"
- "You don't need to do anything"
- "The agents are working autonomously now"
- "I'll monitor their progress and automatically proceed to..."
- "Each agent is: - Reading AGENTS.md..." (nobody cares about the process)
- Any sentence that describes what you're about to show instead of just showing it
- Any post-completion paragraph re-explaining what the agents found

If a sentence isn't a data point, delete it.

### 2. Phase Collapsing

Completed phases collapse to a single line. Always.

```
IN PROGRESS (current phase is expanded):
  ✓ Discovery · 58 files across 3 modules · 4m 48s
  ──────────────────────────────────────────────────
  ⠹ Review · batch 1/1 · 58 files, ~2800 LOC · 6m 11s

WHEN REVIEW COMPLETES (it collapses, next phase expands):
  ✓ Discovery · 58 files across 3 modules · 4m 48s
  ✓ Review · 5 agents · 8m 22s
  ──────────────────────────────────────────────────
  ⠹ Synthesis · deduplicating · 1m 04s
```

Never re-display details of a completed phase. One line. Move on.

### 3. Symbols

Use exactly these symbols:

| Symbol | Meaning |
|--------|---------|
| `✓` | Completed phase/step (print in green) |
| `⠋⠙⠹⠸⠼⠴⠦⠧⠇⠏` | Active spinner — braille dot characters, 10 frames. Pick one per print as a preference for active status (print in purple) |
| `◇` | Pending/queued step (print in dim gray) |
| `✓` / `✗` | Agent completed / agent failed (after batch completes) |
| `──────` | Section divider (thin line, print in dim) |
| `→` | Points to file output |
| `·` | Inline separator between metadata items |

Never use: `═══` double-line boxes, `⏺` (static record symbol with no meaning here), emoji for agents (`🔍🔒⚠️⚡📚`), `[ACTIVE]` badges, emoji for severity (`🔴🟠🟡🔵`).

### 4. Colors (Named Instructions)

When printing terminal output, use these color names:

| Color | Use for |
|-------|---------|
| **green** | `✓` checkmarks |
| **purple** | `⠹` spinner character |
| **dim gray** | `◇` pending steps, completed phase metadata, timestamps, separators |
| **white/bold** | Active phase name |
| **red** | Critical severity count |
| **orange** | High severity count |
| **yellow** | Medium severity count |
| **dim** | Low severity count, general metadata |

### 5. Agent Display (Batch-Level)

Agents do NOT get per-agent progress bars. After a batch's 5 agents complete, show each agent's completion status on one line:

```
  ✓ Bug Hunter · ✓ Security · ✓ Error Handling · ✓ Performance · ✗ Stack Review
```

Failed agents use `✗` (print in red). No descriptions of what agents do. The agent name is the description.

---

## The 4 Output States

### State 1: Discovery

Print exactly:
```
  ⠹ Discovery · scanning repository...
  ◇ Review
  ◇ Synthesis
```

Three lines. Nothing else.

### State 2: Review

Print collapsed discovery + active review + agent status after each batch:
```
  ✓ Discovery · {N} files across {M} modules · {time}
  ──────────────────────────────────────────────────
  ⠹ Review · batch {N}/{total} · {files} files, ~{LOC} LOC

  ◇ Next: synthesize → REVIEW_REPORT.md
```

After each batch's agents complete, print the agent status line, then update the batch counter.

### State 3: Synthesis

Print collapsed discovery + collapsed review + active synthesis:
```
  ✓ Discovery · {N} files across {M} modules · {time}
  ✓ Review · 5 agents · {time}
  ──────────────────────────────────────────────────
  ⠹ Synthesis · deduplicating · {time}
```

### State 4: Complete (Final Summary)

Print the full final summary. This is the most important output — follow every rule exactly.

```
  ✓ Discovery · {N} files · {time}
  ✓ Review · 5 agents · {time}
  ✓ Synthesis · {time}
  ──────────────────────────────────────────────────

  Deep Review Complete — {total time} total

  {N} critical  ·  {N} high  ·  {N} medium  ·  {N} low     {total} issues

  ── Critical ──────────────────────────────────────

  1  {Issue title}
     {Impact summary — one line}
     → {file path(s)}

  2  {Issue title}
     {Impact summary — one line}
     → {file path(s)}

  ── Next ──────────────────────────────────────────

  1. {Imperative action — e.g., "Fix critical issues first"}
  2. {Imperative action}
  3. {Imperative action}

  ── Reports ───────────────────────────────────────

  → REVIEW_REPORT.md         {size} — full findings
  → AGENTS.md (×{N})         module documentation
  → .deep-review/            raw findings + state
```

---

## Final Summary Rules

1. **Severity counts line.** One line, color-coded: critical in red, high in orange, medium in yellow, low in dim. Total at the end.
2. **Critical issues only in terminal.** High/Medium/Low go in REVIEW_REPORT.md. Do not dump all issues into terminal output.
3. **Each critical issue = 3 lines max.** Number + title, impact summary, file path(s) with `→`. No paragraphs.
4. **Zero criticals fallback.** If zero critical issues, show top 3 high-severity issues instead. Use `── High ──` as the section header. If zero criticals AND zero high, print `No critical or high issues found.`
5. **Section headers.** Use `── Label ──` + thin line extending to column 50. Not `═══` boxes. Not emoji.
6. **Next steps.** Numbered, imperative verbs. No "Consider" or "You should". Just "Fix X", "Read Y", "Run Z".
7. **Reports section.** Arrow + filename + size or count. That's it.
8. **No trailing prose.** Never follow the summary with a paragraph re-explaining the findings.

---

## Maintain-Docs Output Format

The maintain-docs command uses the same design language:

```
  ✓ Change detection · {N} files · {time}
  ✓ Documentation updates · {time}
  ──────────────────────────────────────────────────

  Docs Maintenance Complete

  {N} updated  ·  {N} created  ·  {N} missing

  ── Updated ───────────────────────────────────────

  → {path/to/AGENTS.md}
  → {path/to/AGENTS.md}

  ── Created ───────────────────────────────────────

  → {path/to/AGENTS.md}
  → {path/to/CLAUDE.md}

  ── Missing ───────────────────────────────────────

  {dir/} — {N} code files, no AGENTS.md

  ── Next ──────────────────────────────────────────

  1. Run /deep-review:full-review for missing directories
  2. Review updated AGENTS.md files for accuracy
```

---

## progress.md Format (v2)

The orchestrator writes progress.md for compaction recovery. Use this dense format with a version marker:

```
<!-- progress v2 -->
phase: review
batch: {N}/{total}
completed: 1, 2, 3
critical: {N} | high: {N} | medium: {N} | low: {N}
failures: none
```

The compaction recovery logic must detect and parse both:
- **v1** (legacy): Markdown with `# Review Progress`, emoji severity markers, heading-based sections
- **v2** (current): Key-value format with `<!-- progress v2 -->` marker on first line

Detection: if first line contains `<!-- progress v2 -->`, parse as v2. Otherwise, parse as v1.

---

## Anti-Patterns

Never produce these patterns:

| Bad | Good |
|-----|------|
| `═══════════════════` / `DEEP REVIEW COMPLETE` / `═══════════════════` | `Deep Review Complete — 19m 00s total` |
| `🔴 Critical: 6` / `🟠 High: 17` | `6 critical · 17 high · 18 medium · 6 low` |
| "The review is complete! Your repository has been thoroughly analyzed by 5 specialized agents..." | *(nothing — the data already said this)* |
| `Duration: ~19 minutes` / `Discovery: 4m 48s` / `Review: ~5m` | *(already shown in the collapsed ✓ lines)* |
| Sentences describing what agents are about to do | *(just show the status line)* |
| Post-completion paragraphs re-explaining findings | *(the summary IS the output)* |

---

## Design Principles

- **Dense, not verbose.** Every character earns its place.
- **Data, not narration.** Show numbers, not sentences about numbers.
- **Progressive disclosure.** Terminal shows critical issues. Report has everything else.
- **Consistent visual grammar.** Same symbols, same colors, same spacing every time.
- **Respect the user's time.** They'll read REVIEW_REPORT.md for details. The terminal output is a dashboard, not a document.
