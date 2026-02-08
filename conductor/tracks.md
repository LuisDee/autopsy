# Tracks

| ID | Name | Wave | Complexity | Status | Dependencies |
|----|------|------|------------|--------|-------------|
| 01_plugin_scaffold | Plugin Scaffold | 1 | S | complete | — |
| 02_discovery_agent | Discovery Agent | 2 | M | complete | 01_plugin_scaffold |
| 03_review_agents | Review Agents | 2 | L | not started | 01_plugin_scaffold |
| 04_synthesizer_agent | Synthesizer Agent | 2 | M | not started | 01_plugin_scaffold |
| 05_orchestrator_command | Orchestrator Command | 3 | XL | not started | 02_discovery_agent, 03_review_agents, 04_synthesizer_agent |
| 06_secondary_commands | Secondary Commands | 3 | M | not started | 01_plugin_scaffold, 02_discovery_agent |
