# Plan: Plugin Scaffold (Track 01)

## Phase 1: Plugin Structure, Manifest & Placeholders

### Tasks

- [x] **1.1** Create directory structure: `autopsy/.claude-plugin/`, `autopsy/commands/`, `autopsy/agents/`, `autopsy/skills/codebase-documentation/` `ba280d1`
- [x] **1.2** Create `autopsy/.claude-plugin/plugin.json` with name, version, description, and component paths `ba280d1`
- [x] **1.3** Validate `plugin.json` is valid JSON: `python3 -c "import json; json.load(open('autopsy/.claude-plugin/plugin.json'))"` `ba280d1`
- [x] **1.4** Create placeholder command files: `commands/full-review.md`, `commands/maintain-docs.md` — each with YAML frontmatter (name, description) and TODO comment `ba280d1`
- [x] **1.5** Create placeholder agent files: `agents/discovery.md`, `agents/bug-hunter.md`, `agents/security-auditor.md`, `agents/error-inspector.md`, `agents/performance-detector.md`, `agents/stack-reviewer.md`, `agents/synthesizer.md` — each with YAML frontmatter and TODO comment `ba280d1`
- [x] **1.6** Create placeholder skill file: `skills/codebase-documentation/SKILL.md` with YAML frontmatter and TODO comment `ba280d1`
- [x] **1.7** Validate all 10 placeholder files have valid YAML frontmatter: `python3 -c` script checking each file `ba280d1`
- [x] **1.8** Verify file count: `find autopsy/ -type f | sort` shows 12 files (plugin.json + 10 placeholders + README placeholder) `ba280d1`

> **Note:** README is created as a minimal placeholder in this phase; full content is Phase 2. This ensures the 12-file structure is complete before README authoring begins.

## Phase 2: Comprehensive README & Final Validation

### Tasks

- [x] **2.1** Write `autopsy/README.md` with all 9 required sections: (1) What the plugin does, (2) Installation, (3) Commands, (4) What gets generated, (5) Architecture overview with 3-phase diagram, (6) Recommended configuration, (7) Cost expectations table, (8) Companion plugin recommendations, (9) How documentation compounds
- [x] **2.2** Verify README contains all 9 section headings
- [x] **2.3** Run full acceptance criteria suite: 12 files exist, plugin.json valid JSON, all YAML frontmatter valid, README has all sections
