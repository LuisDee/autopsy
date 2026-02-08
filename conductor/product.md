# Initial Concept

A Claude Code plugin called `deep-review` that provides a single slash command `/deep-review:full-review` which autonomously performs an exhaustive, multi-agent code review of an entire repository and generates living documentation (CLAUDE.md files) throughout the codebase.

It also provides:
- `/deep-review:maintain-docs` — a command to keep documentation current after code changes
- A passive skill (`codebase-documentation`) that auto-triggers during normal work to remind Claude to read/update CLAUDE.md files
- 7 specialized agents that the commands orchestrate

The user runs ONE command and walks away. When it's done they have:
1. `REVIEW_REPORT.md` in the repo root — a prioritized, severity-graded report of every issue found
2. `CLAUDE.md` files in every significant directory — persistent documentation that makes all future Claude Code sessions faster
3. Updated root `CLAUDE.md` — comprehensive project context

# Product Guide

## Vision
A "one command and walk away" code review tool for solo developers using Claude Code. Deep-review turns Claude's multi-agent capabilities into an autonomous review team that finds bugs, security issues, performance problems, and error handling gaps — then leaves behind living documentation that compounds in value over time.

## Target Users
Solo developers who want the rigor of a multi-person code review without the overhead. Developers who maintain personal projects, freelance codebases, or side projects where there's no second pair of eyes.

## Core Goals
1. **Exhaustive automated review** — Find bugs, security vulnerabilities, error handling gaps, performance issues, and stack-specific anti-patterns across an entire repository
2. **Living documentation** — Generate CLAUDE.md files in every significant directory that persist between sessions and make all future Claude Code interactions faster and more context-aware
3. **Zero babysitting** — The user runs one command and walks away. The plugin handles discovery, batching, parallel agent coordination, progress tracking, and synthesis autonomously
4. **Actionable output** — Every finding includes exact file path, line number, severity, description, impact, and a specific fix recommendation

## Key Features
- `/deep-review:full-review` — 3-phase autonomous review: discovery, parallel multi-agent review, synthesis
- `/deep-review:maintain-docs` — Incremental documentation updates after code changes
- Passive `codebase-documentation` skill — Reminds Claude to read/update CLAUDE.md during normal development
- 7 specialized agents: discovery, bug-hunter, security-auditor, error-inspector, performance-detector, stack-reviewer, synthesizer
- Disk-based coordination (`.deep-review/` directory) keeps orchestrator context light
- Automatic batch sizing based on repo size (small to huge repos supported)
- Context pressure management with compaction-safe progress tracking

## Success Criteria
- Plugin installs and loads cleanly via `claude plugin install`
- `/deep-review:full-review` completes autonomously on repos of varying sizes
- `REVIEW_REPORT.md` contains deduplicated, severity-graded findings with exact file paths and line numbers
- CLAUDE.md files contain specific, useful documentation (not generic boilerplate)
- Future Claude Code sessions are measurably faster due to CLAUDE.md context
