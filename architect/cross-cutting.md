# Cross-Cutting Concerns

> This file is **append-only**. New versions are added below existing ones.
> Never modify a published version — add a new version section instead.
> Each version is tagged to the wave where it was introduced.

---

## v1 — Initial (Wave 1)

### Agent Output Format
Every review agent must produce findings in identical format: severity (Critical/High/Medium/Low), confidence (High/Medium/Low), exact file path, line number/range, category, one-line description, impact statement, specific fix recommendation with code. No exceptions.
- Applies to: ALL review agents (bug-hunter, security-auditor, error-inspector, performance-detector, stack-reviewer)
- Source: Product guidelines

### Self-Review Step
Every agent must verify its own output before completing: "Did I examine every assigned file? Does every finding have exact file + line + severity? Did I follow the output format exactly?"
- Applies to: ALL agents
- Source: Architecture research (accepted pattern)

### Few-Shot Examples
Every review agent definition must include 2-3 concrete finding examples in the exact output format. This dramatically improves review quality and reduces "looks fine" responses.
- Applies to: ALL review agents
- Source: Architecture research (accepted pattern)

### Thoroughness Rules
Every agent must: read the FULL contents of every assigned file (no skimming); never say "the rest looks fine" or "similar issues in other files"; provide exact file path and line number for every finding; re-read files with 100+ lines that produced zero findings.
- Applies to: ALL agents
- Source: Build plan (mandatory rules)

### AGENTS.md Quality Bar
Generated AGENTS.md files must: contain information that saves 30+ minutes of code reading; be specific to the actual code (not generic boilerplate); use "TODO: investigate" over guesses; stay under 80 lines (root) or 60 lines (subdirectory); not include file trees; not duplicate root-level info in subdirectories.
- Applies to: Discovery agent, maintain-docs command
- Source: Product guidelines

### Compaction-Safe State Management
The orchestrator must write all critical state to disk before every compaction risk point. After any compaction, it must re-read `state.json`, `batch-plan.md`, and `progress.md` to resume. Agents write results to specific file paths — never via TaskOutput into orchestrator context.
- Applies to: Orchestrator command
- Source: Architecture research (ADR-001)

### Dual Documentation Output
Generate `AGENTS.md` as primary documentation file. Generate companion `CLAUDE.md` containing only `@AGENTS.md` at each level. Preserve and augment existing documentation — never delete unless provably wrong.
- Applies to: Discovery agent, maintain-docs command, passive skill
- Source: Architecture research (ADR-002)

### Batch Sizing Constraints
Under 50 files or 5000 LOC per batch (whichever is smaller). Group related modules together. Order batches by risk (highest first). Include stack-specific focus areas per batch. Target 30-40K tokens of code per agent per batch.
- Applies to: Discovery agent (batch plan generation)
- Source: Tech stack + architecture research

### Error Recovery
If an agent fails (output file missing after timeout), log the failure to `progress.md` and `state.json`, optionally retry with a refined prompt (max 1 retry), then continue with remaining batches. Never block the entire review on a single agent failure.
- Applies to: Orchestrator command
- Source: Architecture research
