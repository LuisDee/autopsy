# Plan: Review Agents (Track 03)

## Phase 1: Bug Hunter & Security Auditor

### Tasks

- [x] **1.1** Write `autopsy/agents/bug-hunter.md` — complete agent definition with YAML frontmatter, 15-item checklist, 3 few-shot examples, mandatory rules, self-review step
- [x] **1.2** Write `autopsy/agents/security-auditor.md` — complete agent definition with YAML frontmatter, 6-category checklist (injection, auth, data exposure, input handling, secrets, dependencies), 3 few-shot examples, Exploitability field for criticals

## Phase 2: Error Inspector & Performance Detector

### Tasks

- [x] **2.1** Write `autopsy/agents/error-inspector.md` — complete agent definition with YAML frontmatter, 5-category checklist (missing, broken, resilience, cleanup, logging), 3 few-shot examples
- [x] **2.2** Write `autopsy/agents/performance-detector.md` — complete agent definition with YAML frontmatter, 6-category checklist (database, memory, CPU/IO, concurrency, resources, network), performance-specific severity override, 3 few-shot examples

## Phase 3: Stack Reviewer

### Tasks

- [x] **3.1** Write `autopsy/agents/stack-reviewer.md` — complete agent definition with YAML frontmatter, dynamic stack detection logic, all 10 stack sections (Python, Airflow, BigQuery/SQL, FastAPI/Flask, Docker/K8s, JS/TS, React, Go, Rust, Catch-All), 3 few-shot examples

## Phase 4: Validation

### Tasks

- [x] **4.1** Validate all 5 agents have valid YAML frontmatter with name, description, tools (Read, Grep, Glob only — no Bash, no model)
- [x] **4.2** Verify each agent has the complete checklist (not abbreviated)
- [x] **4.3** Verify each agent has 2-3 few-shot examples in exact output format
- [x] **4.4** Verify each agent has self-review step and mandatory rules
- [x] **4.5** Verify output format matches interface contract (severity emoji, confidence, file, line, category, description, impact, fix)
