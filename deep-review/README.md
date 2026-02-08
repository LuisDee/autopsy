# deep-review

## What It Does

A Claude Code plugin that performs exhaustive, multi-agent code review and architecture assessment of your entire repository with a single command. You run `/deep-review:full-review` and walk away. When it finishes, you have two complementary reports: `REVIEW_REPORT.md` with every bug, security vulnerability, error handling gap, and performance issue found, and `ARCHITECTURE_REPORT.md` with a design-level assessment of your system's components, tooling choices, quality attributes, and improvement recommendations — plus `AGENTS.md` files in every significant directory that make all future Claude Code sessions faster and more context-aware.

## Installation

```bash
claude plugin install deep-review
```

Or install from a local directory during development:

```bash
claude --plugin-dir ./deep-review
```

## Commands

### `/deep-review:full-review`

Launches the full autonomous review pipeline. Discovers your repo structure, dispatches 5 specialized code review agents in parallel across file batches while simultaneously running architecture assessment agents (architect + researcher), then synthesizes all findings into two complementary reports.

**Output:** `REVIEW_REPORT.md` (code-level findings) and `ARCHITECTURE_REPORT.md` (design-level assessment) in your repo root, `AGENTS.md` files throughout the codebase, companion `CLAUDE.md` files with `@AGENTS.md` references.

**Usage:**
```
/deep-review:full-review
```

No arguments needed. The plugin auto-detects repo size and adjusts batch sizing accordingly.

### `/deep-review:maintain-docs`

Detects which files changed since the last review (via `git diff`) and incrementally updates the relevant `AGENTS.md` files. Use this after significant code changes to keep documentation current without running a full review.

**Usage:**
```
/deep-review:maintain-docs
```

## What Gets Generated

| File | Location | Purpose |
|------|----------|---------|
| `REVIEW_REPORT.md` | Repo root | Code-level findings: bugs, security issues, error handling gaps, performance problems |
| `ARCHITECTURE_REPORT.md` | Repo root | Design-level assessment: components, tooling, quality attributes, recommendations |
| `AGENTS.md` | Every significant directory | Persistent documentation: module purpose, key files, boundaries, gotchas |
| `CLAUDE.md` | Alongside each `AGENTS.md` | Companion file with `@AGENTS.md` reference for Claude Code compatibility |
| `.deep-review/` | Repo root | Working directory with discovery data, batch results, and progress state (can be gitignored) |

## Architecture Overview

The plugin uses a 3-phase orchestrator-worker architecture. The orchestrator command stays lightweight — it never reads file contents directly. All heavy analysis happens in sub-agents with fresh context windows, coordinated via disk (the `.deep-review/` directory).

```
/deep-review:full-review (orchestrator)
  │
  │  PHASE 1: DISCOVERY
  ├──► discovery agent
  │      Maps repo structure, generates AGENTS.md files,
  │      produces batch plan for review agents
  │      Writes: .deep-review/discovery.md, batch-plan.md
  │
  │  PHASE 2: REVIEW + ARCHITECTURE (parallel)
  │
  │  Phase 2B — Code Review (per batch, 5 agents in parallel)
  ├──► bug-hunter ──────────► .deep-review/batch-N/bugs.md
  ├──► security-auditor ────► .deep-review/batch-N/security.md
  ├──► error-inspector ─────► .deep-review/batch-N/errors.md
  ├──► performance-detector ► .deep-review/batch-N/performance.md
  ├──► stack-reviewer ──────► .deep-review/batch-N/stack.md
  │
  │  Phase 2A — Architecture (runs alongside first batch)
  ├──► architect ───────────► .deep-review/architecture-analysis.md
  ├──► researcher ──────────► .deep-review/best-practices-research.md
  │
  │  PHASE 3: SYNTHESIS (both in parallel)
  ├──► synthesizer ─────────► REVIEW_REPORT.md
  └──► arch-synthesizer ────► ARCHITECTURE_REPORT.md
```

**Key design principles:**
- Sub-agents cannot spawn other sub-agents — the orchestrator chains all agents
- Each review agent reads every file in its assigned batch completely (no sampling)
- Architecture agents (architect + researcher) run alongside the first code review batch for maximum parallelism
- Architecture assessment failure never blocks the code review pipeline
- Disk-based coordination keeps the orchestrator context light
- Progress is tracked in `.deep-review/state.json` and `.deep-review/progress.md` for compaction safety

## Two Complementary Reports

A full review produces two reports that cover different levels of analysis:

**REVIEW_REPORT.md** — Code-level findings. Bugs, security vulnerabilities, error handling gaps, performance issues. Each finding has an exact file path, line number, severity, and fix recommendation. This is what you fix first.

**ARCHITECTURE_REPORT.md** — Design-level assessment. Component decomposition, interaction analysis, tooling evaluations, quality attribute tradeoffs, and design gap identification. Based on ATAM (Architecture Tradeoff Analysis Method) adapted for automated analysis, cross-referenced with best-practices research from official documentation. This is what you use for strategic technical decisions.

The reports may cross-reference each other (e.g., architecture recommendations citing code-level evidence) but do not duplicate findings.

## Recommended Configuration

For best results, launch Claude Code with extended thinking enabled:

```bash
export MAX_THINKING_TOKENS=63999
claude
```

Then inside the session:

```
/effort max
```

This gives each sub-agent maximum reasoning depth, which significantly improves finding quality — especially for subtle bugs and cross-cutting security issues.

## Cost Expectations

Approximate costs based on repository size (Claude Sonnet-class pricing):

| Repo Size | Files | Batches | Agents Spawned | Estimated Time | Estimated Cost |
|-----------|-------|---------|----------------|----------------|----------------|
| **Small** | < 50 | 1 | 10 | 10-20 min | $3-7 |
| **Medium** | 50-200 | 2-3 | 15-20 | 30-60 min | $7-18 |
| **Large** | 200-500 | 4-8 | 25-45 | 1-3 hours | $18-45 |
| **Huge** | 500+ | 8+ | 45+ | 3+ hours | $45+ |

Costs scale linearly with file count. Each review batch spawns 5 parallel agents. Fixed overhead per run: discovery (1) + architect (1) + researcher (1) + code review synthesizer (1) + architecture synthesizer (1) = 5 agents.

## Companion Plugin Recommendations

deep-review works standalone with no external dependencies. The following tools are **optional** and may provide marginal benefit in specific scenarios:

- **ast-grep** — Structural code search. Limited value since review agents already read every file completely. May help for very large repos where you want to pre-filter.
- **GrepAI** — Semantic search via Ollama. Requires local Ollama setup. Sub-agents cannot access MCP tools when running in background, so this is only useful in the orchestrator context.

**Recommendation:** Start without companion plugins. The sub-agent approach is more context-cost-effective than tool-augmented search for thorough review.

## How Documentation Compounds

The `AGENTS.md` files generated by deep-review are not just review artifacts — they are persistent, cross-LLM documentation that compounds in value:

**First run:** deep-review maps your entire repo and writes `AGENTS.md` in every significant directory, documenting module purpose, key files, architectural boundaries, common patterns, and gotchas.

**Every subsequent Claude Code session:** Claude reads these files automatically (via companion `CLAUDE.md` with `@AGENTS.md`), gaining instant context about your codebase without re-reading source files. This means faster responses, fewer hallucinations, and better adherence to your project's patterns.

**After code changes:** Run `/deep-review:maintain-docs` to incrementally update documentation for changed modules. The cost is minimal compared to a full review.

**Cross-LLM compatibility:** `AGENTS.md` is an open standard (Linux Foundation) supported by Claude Code, Cursor, GitHub Copilot, Codex CLI, and Gemini CLI. Your documentation investment isn't locked to a single tool.

Over time, the documentation becomes the most valuable output of deep-review — a living, machine-readable map of your codebase that every AI coding assistant can leverage.
