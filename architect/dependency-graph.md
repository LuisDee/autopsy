# Dependency Graph

> Directed Acyclic Graph (DAG) of track dependencies.
> An edge A → B means "A depends on B" (B must complete before A starts).

---

## Track Dependencies

| Track | Depends On | Interfaces Consumed |
|-------|-----------|---------------------|
| 01_plugin_scaffold | — | — |
| 02_discovery_agent | 01_plugin_scaffold | Plugin directory structure |
| 03_review_agents | 01_plugin_scaffold | Plugin directory structure |
| 04_synthesizer_agent | 01_plugin_scaffold | Plugin directory structure, Finding format (from 03) |
| 05_orchestrator_command | 02_discovery_agent, 03_review_agents, 04_synthesizer_agent | Batch plan format, finding format, report format |
| 06_secondary_commands | 01_plugin_scaffold, 02_discovery_agent | Plugin directory structure, AGENTS.md format |
| 07_output_rendering | 05_orchestrator_command, 06_secondary_commands, 03_review_agents, 04_synthesizer_agent | Orchestrator output, command output, agent formats |
| 08_architecture_agents | 01_plugin_scaffold | Plugin directory structure (agents/ directory) |
| 09_architecture_orchestration | 08_architecture_agents, 05_orchestrator_command, 07_output_rendering | Agent definitions, orchestrator command, output rendering conventions |

---

## DAG Visualization

```
Wave 1:  [01_plugin_scaffold]
              │
              ├─────────────────────────────────────┐
              │                  │                   │
Wave 2:  [02_discovery_agent]  [03_review_agents]   │
              │                  │                   │
              │                  ├───────────────────┤
              │                  │                   │
              │            [04_synthesizer_agent]    │
              │                  │                   │
              ├──────────────────┤                   │
              │                                      │
Wave 3:  [05_orchestrator_command]     [06_secondary_commands]
              │                              │
              ├──────────────────────────────┤
              │                              │
Wave 4:  [07_output_rendering]              │
              │                              │
              │    ┌─────────────────────────┘
              │    │
Wave 5:  [08_architecture_agents]────→[09_architecture_orchestration]
              (also depends on 01)      (also depends on 05, 07)
```

---

## Edge Change Log

| Date | Source | Edge Added | Reason |
|------|--------|------------|--------|
| 2026-02-07 | Initial decomposition | All edges (01→07) | Initial dependency graph |
| 2026-02-08 | /architect-feature | 08→01, 09→08, 09→05, 09→07 | Architecture assessment feature decomposition |
