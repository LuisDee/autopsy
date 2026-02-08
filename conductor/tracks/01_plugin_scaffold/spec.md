<!-- ARCHITECT CONTEXT | Track: 01_plugin_scaffold | Wave: 1 | CC: v1 -->

## Cross-Cutting Constraints
- None scoped to this track specifically — it establishes the structure others build on

## Interfaces

### Owns
- Plugin directory structure (`deep-review/` with `commands/`, `agents/`, `skills/`, `.claude-plugin/`)
- `plugin.json` manifest
- `README.md` documentation

### Consumes
- None (no dependencies)

## Dependencies
- None (Wave 1 foundation track)

<!-- END ARCHITECT CONTEXT -->

# Specification: Plugin Scaffold

## Overview

Create the complete directory structure, plugin manifest, placeholder files, and comprehensive README for the `deep-review` Claude Code plugin. This track establishes the skeleton that all subsequent tracks populate. The plugin name is `deep-review`, producing the `/deep-review:full-review` and `/deep-review:maintain-docs` command namespaces.

## Functional Requirements

### FR-1: Plugin Manifest
- Create `.claude-plugin/plugin.json` with:
  - `name`: `"deep-review"`
  - `version`: `"1.0.0"`
  - `description`: Brief description mentioning `/deep-review:full-review` for multi-agent code review and `/deep-review:maintain-docs` for documentation maintenance
  - Component paths listing all commands, agents, and skills

### FR-2: Directory Structure
Create the following directory tree inside `deep-review/`:
```
deep-review/
├── .claude-plugin/
│   └── plugin.json
├── commands/
│   ├── full-review.md          (placeholder)
│   └── maintain-docs.md        (placeholder)
├── agents/
│   ├── discovery.md            (placeholder)
│   ├── bug-hunter.md           (placeholder)
│   ├── security-auditor.md     (placeholder)
│   ├── error-inspector.md      (placeholder)
│   ├── performance-detector.md (placeholder)
│   ├── stack-reviewer.md       (placeholder)
│   └── synthesizer.md          (placeholder)
├── skills/
│   └── codebase-documentation/
│       └── SKILL.md            (placeholder)
└── README.md
```

### FR-3: Placeholder Files
Each placeholder `.md` file must contain:
- Valid YAML frontmatter with `name` and `description` fields
- A markdown body with a single TODO comment indicating which track will implement it
- Example for `agents/bug-hunter.md`:
  ```markdown
  ---
  name: bug-hunter
  description: Finds logic errors, bugs, and correctness issues in assigned files.
  ---
  <!-- TODO: Full agent definition will be implemented in Track 03 (Review Agents) -->
  ```

### FR-4: Comprehensive README
`README.md` must include:
1. **What the plugin does** — one paragraph summary
2. **Installation** — `claude plugin install` instructions
3. **Commands** — `/deep-review:full-review` and `/deep-review:maintain-docs` with descriptions
4. **What gets generated** — `REVIEW_REPORT.md`, `AGENTS.md` files, companion `CLAUDE.md` files
5. **Architecture overview** — sub-agent diagram showing the 3-phase flow (Discovery → Review → Synthesis)
6. **Recommended configuration** — `MAX_THINKING_TOKENS=63999`, `/effort max`
7. **Cost expectations table** — repo size (small/medium/large/huge) → estimated sessions/time/cost
8. **Companion plugin recommendations** — optional tools (ast-grep, etc.) with note that they're not required
9. **How documentation compounds** — explanation of AGENTS.md long-term value

## Non-Functional Requirements

- `plugin.json` must be valid JSON (parseable by `python3 -c "import json; json.load(open(...))"`)
- All placeholder files must have valid YAML frontmatter
- README must be standard GitHub-flavored Markdown
- No files outside the `deep-review/` directory

## Acceptance Criteria

- [ ] `find deep-review/ -type f | sort` shows all 12 files
- [ ] `plugin.json` passes JSON validation
- [ ] All placeholder files have valid YAML frontmatter with name and description
- [ ] README contains all 9 required sections
- [ ] Plugin loads without errors: `claude --plugin-dir ./deep-review`

## Out of Scope

- Agent definition content (Track 02, 03, 04)
- Command orchestration logic (Track 05)
- Maintain-docs command logic (Track 06)
- Passive skill logic (Track 06)
