# Tech Stack

## Platform
- **Claude Code Plugin System** — all files are Markdown with YAML frontmatter
- **Plugin Manifest:** `.claude-plugin/plugin.json` (only `name` field required)

## Plugin Components
| Component | Location | Format |
|-----------|----------|--------|
| Commands | `commands/*.md` | Markdown with YAML frontmatter |
| Agents | `agents/*.md` | Markdown with YAML frontmatter |
| Skills | `skills/*/SKILL.md` | Markdown with YAML frontmatter |
| Manifest | `.claude-plugin/plugin.json` | JSON |

## Agent Architecture
- **Orchestration pattern:** Orchestrator-Worker with sequential pipeline (Phase 1 → Phase 2 fan-out → Phase 3)
- **Sub-agent execution:** Task tool with `run_in_background: true`
- **Result retrieval:** File-based coordination via `.deep-review/` directory (NOT TaskOutput)
- **Max parallelism:** 5 agents per batch
- **Target per-agent context:** 30-40K tokens of code per invocation

## Coordination Layer
- **Working directory:** `.deep-review/` in the target repo root
- **Progress tracking:** `.deep-review/progress.md` (human-readable) + `.deep-review/state.json` (machine-readable)
- **Agent outputs:** `.deep-review/batch-{N}/{agent-name}.md`
- **Discovery output:** `.deep-review/discovery.md` + `.deep-review/batch-plan.md`
- **Final report:** `REVIEW_REPORT.md` in repo root

## Output Files
| File | Purpose | Persistence |
|------|---------|-------------|
| `AGENTS.md` | Cross-LLM documentation (primary) | Permanent — committed to repo |
| `CLAUDE.md` | Claude Code companion (`@AGENTS.md` reference) | Permanent — committed to repo |
| `REVIEW_REPORT.md` | Final severity-graded review report | Permanent — committed to repo |
| `.deep-review/*` | Working files, batch outputs, progress | Temporary — added to `.gitignore` |

## Sub-Agent Definitions
| Agent | Role | Phase |
|-------|------|-------|
| discovery | Map repo, create AGENTS.md files, produce batch plan | 1 |
| bug-hunter | Logic errors, null access, race conditions | 2 |
| security-auditor | Injection, secrets, auth, XSS | 2 |
| error-inspector | Missing/broken error handling, resilience | 2 |
| performance-detector | N+1 queries, memory leaks, unbounded ops | 2 |
| stack-reviewer | Framework-specific best practices | 2 |
| synthesizer | Merge findings, cross-cutting analysis, final report | 3 |

## Key Constraints
- Sub-agents cannot spawn other sub-agents — orchestrator chains all agents
- MCP tools not available in background sub-agents
- TaskOutput returns full JSONL transcripts — use file-based coordination instead
- Background task notifications may not fire reliably — poll for output files
- Auto-compaction at ~95% context — write state to disk before compaction risk points
