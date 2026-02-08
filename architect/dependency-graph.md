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
Wave 3:  [05_orchestrator_command]                   │
                                                     │
Wave 4:                              [06_secondary_commands]
```

---

## Edge Change Log

| Date | Source | Edge Added | Reason |
|------|--------|------------|--------|
| 2026-02-07 | Initial decomposition | All edges | Initial dependency graph |
