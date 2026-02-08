<!-- ARCHITECT CONTEXT | Track: 01_plugin_scaffold | Wave: 1 | CC: v1 -->

## Cross-Cutting Constraints
- None scoped to this track specifically — it establishes the structure others build on

## Interfaces

### Owns
- Plugin directory structure (`autopsy/` with `commands/`, `agents/`, `skills/`, `.claude-plugin/`)
- `plugin.json` manifest
- `README.md` documentation

### Consumes
- None (no dependencies)

## Dependencies
- None (Wave 1 foundation track)

<!-- END ARCHITECT CONTEXT -->

# Track 01: Plugin Scaffold

## What This Track Delivers

The foundational directory structure, plugin manifest (`plugin.json`), and README for the autopsy Claude Code plugin. This is the skeleton that all subsequent tracks populate with agent definitions, commands, and skills. It also establishes the plugin's public documentation and installation instructions.

## Scope

### IN
- `.claude-plugin/plugin.json` manifest with all component paths
- Directory structure: `commands/`, `agents/`, `skills/codebase-documentation/`
- `README.md` with installation, usage, architecture overview, cost expectations, and companion plugin recommendations
- Placeholder files or empty directories where agents/commands will be added

### OUT
- Agent definition content (Track 02, 03, 04)
- Command content (Track 05, 06)
- Skill content (Track 06)

## Key Design Decisions

These should be resolved with the developer during spec generation:

1. **Plugin name in manifest:** `autopsy` vs `repo-reviewer` vs something else?
   Trade-off: `autopsy` matches the build plan; `repo-reviewer` matches the repo directory name
2. **README detail level:** Comprehensive README with cost tables and architecture diagrams vs minimal README with just installation + usage?
   Trade-off: Comprehensive helps adoption but takes more time; minimal ships faster
3. **Placeholder strategy:** Empty placeholder files in `agents/` and `commands/` vs just empty directories?
   Trade-off: Placeholders make the structure visible immediately; empty dirs are cleaner for subsequent tracks

## Architectural Notes

- The `plugin.json` only requires the `name` field — everything else is optional but recommended for discoverability
- Custom paths in the manifest SUPPLEMENT default directories, they don't replace them
- `${CLAUDE_PLUGIN_ROOT}` resolves to the absolute plugin path at runtime — useful if any scripts are added later
- The plugin will be tested via `claude --plugin-dir ./autopsy` during development
- README should document `MAX_THINKING_TOKENS=63999` and `/effort max` as recommended launch configuration

## Complexity: S
## Estimated Phases: ~2
