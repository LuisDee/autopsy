# Tracks

| ID | Name | Wave | Complexity | Status | Dependencies |
|----|------|------|------------|--------|-------------|
| 01_plugin_scaffold | Plugin Scaffold | 1 | S | complete | -- |
| 02_discovery_agent | Discovery Agent | 2 | M | complete | 01_plugin_scaffold |
| 03_review_agents | Review Agents | 2 | L | complete | 01_plugin_scaffold |
| 04_synthesizer_agent | Synthesizer Agent | 2 | M | complete | 01_plugin_scaffold |
| 05_orchestrator_command | Orchestrator Command | 3 | XL | complete | 02_discovery_agent, 03_review_agents, 04_synthesizer_agent |
| 06_secondary_commands | Secondary Commands | 3 | M | complete | 01_plugin_scaffold, 02_discovery_agent |
| 07_output_rendering | Output Rendering Overhaul | 4 | L | complete | 05_orchestrator_command, 06_secondary_commands, 03_review_agents, 04_synthesizer_agent |
| 08_architecture_agents | Architecture Agents | 5 | L | complete | 01_plugin_scaffold |
| 09_architecture_orchestration | Architecture Orchestration | 5 | M | in_progress | 08_architecture_agents, 05_orchestrator_command, 07_output_rendering |
