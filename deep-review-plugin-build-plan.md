# BUILD PLAN: deep-review — Claude Code Plugin

## What You Are Building

A Claude Code plugin called `deep-review` that provides a single slash command `/deep-review:full-review` which autonomously performs an exhaustive, multi-agent code review of an entire repository and generates living documentation (CLAUDE.md files) throughout the codebase.

It also provides:
- `/deep-review:maintain-docs` — a command to keep documentation current after code changes
- A passive skill (`codebase-documentation`) that auto-triggers during normal work to remind Claude to read/update CLAUDE.md files
- 7 specialized agents that the commands orchestrate

The user runs ONE command and walks away. When it's done they have:
1. `REVIEW_REPORT.md` in the repo root — a prioritized, severity-graded report of every issue found
2. `CLAUDE.md` files in every significant directory — persistent documentation that makes all future Claude Code sessions faster
3. Updated root `CLAUDE.md` — comprehensive project context

---

## Architecture Overview

```
/deep-review:full-review (command — the orchestrator)
  │
  │  PHASE 1: DISCOVERY
  ├──► discovery agent (sub-agent, fresh context)
  │      Reads: every file in repo (via find + representative samples)
  │      Writes: .deep-review/discovery.md (repo profile)
  │      Writes: .deep-review/batch-plan.md (grouped file batches for review)
  │      Writes: CLAUDE.md files in every significant directory
  │      Writes: root CLAUDE.md (creates or updates)
  │
  │  PHASE 2: REVIEW (per batch, agents launched in parallel)
  ├──► bug-hunter agent ────────► .deep-review/batch-{N}/bugs.md
  ├──► security-auditor agent ──► .deep-review/batch-{N}/security.md
  ├──► error-inspector agent ───► .deep-review/batch-{N}/errors.md
  ├──► performance-detector ────► .deep-review/batch-{N}/performance.md
  ├──► stack-reviewer agent ────► .deep-review/batch-{N}/stack.md
  │      (all 5 run in parallel per batch, each is a fresh context)
  │      (each reads the local CLAUDE.md first for module context)
  │      (each reads EVERY file in its assigned batch completely)
  │      (each writes findings with exact file paths + line numbers)
  │      (each updates module CLAUDE.md if it finds undocumented gotchas)
  │
  │  PHASE 3: SYNTHESIS
  └──► synthesizer agent (sub-agent, fresh context)
         Reads: all .deep-review/batch-*//*.md files
         Performs: deduplication, cross-cutting analysis, dependency audit, testing gap analysis
         Writes: REVIEW_REPORT.md (final report in repo root)
         Updates: root CLAUDE.md with new insights
```

The orchestrator's context stays LIGHT. It never reads file contents directly. It:
1. Launches the discovery agent, waits, reads the batch plan
2. For each batch: launches 5 review agents in parallel, waits, confirms output files exist
3. Launches the synthesizer, waits, confirms REVIEW_REPORT.md exists
4. Prints a summary to the user

All heavy lifting happens in sub-agents with fresh context windows.
Disk (.deep-review/ directory) is the coordination layer between phases.

---

## Plugin Directory Structure

Build this exact structure:

```
deep-review/
├── .claude-plugin/
│   └── plugin.json                          ← Plugin manifest
├── commands/
│   ├── full-review.md                       ← Main orchestrator command
│   └── maintain-docs.md                     ← Documentation maintenance command
├── agents/
│   ├── discovery.md                         ← Phase 1: repo mapping + documentation
│   ├── bug-hunter.md                        ← Phase 2: logic errors, null access, race conditions
│   ├── security-auditor.md                  ← Phase 2: injection, secrets, auth, XSS
│   ├── error-inspector.md                   ← Phase 2: exception handling, resilience
│   ├── performance-detector.md              ← Phase 2: N+1, leaks, unbounded ops
│   ├── stack-reviewer.md                    ← Phase 2: stack-specific best practices
│   └── synthesizer.md                       ← Phase 3: merge, cross-cutting, final report
├── skills/
│   └── codebase-documentation/
│       └── SKILL.md                         ← Passive skill: read/update CLAUDE.md during normal work
└── README.md                                ← Plugin documentation
```

---

## FILE SPECIFICATIONS

### 1. `.claude-plugin/plugin.json`

Standard Claude Code plugin manifest. Must include:
- name: "deep-review"
- version: "1.0.0"
- description: Mention that /deep-review:full-review runs an exhaustive multi-agent codebase review and generates living documentation. Mention /deep-review:maintain-docs keeps docs current.
- List all commands, agents, and skills by their file paths

---

### 2. `commands/full-review.md` — THE MAIN ORCHESTRATOR

This is the most important file. It's the brains of the operation. When the user runs `/deep-review:full-review`, this command executes.

The command must instruct Claude to:

#### Setup
- Run `/effort max`
- Create `.deep-review/` working directory in the repo root
- Add `.deep-review/` to `.gitignore` if not already there (these are temp working files, not permanent)
- Note start time for duration tracking

#### Phase 1: Discovery
- Launch the **discovery** agent as a sub-agent
- The discovery agent prompt must include:
  - "You are the discovery agent for a deep code review. Your job is to map the entire repository, understand every module, create CLAUDE.md documentation files, and produce a batch plan for the review phase."
  - "Follow the instructions in your agent definition exactly."
  - "Write your repo profile to `.deep-review/discovery.md`"
  - "Write the batch plan to `.deep-review/batch-plan.md`"
  - "Create CLAUDE.md files in every significant directory"
  - "Create or update the root CLAUDE.md"
- After the discovery agent completes, the orchestrator reads `.deep-review/batch-plan.md` to learn how many batches there are and what directories are in each batch
- Print: "Phase 1 complete. Discovered {N} files across {M} modules. Created {X} CLAUDE.md files. Generated {B} review batches."

#### Phase 2: Review
- Read `.deep-review/batch-plan.md` to get the batch list
- For each batch:
  - Create `.deep-review/batch-{N}/` directory
  - Launch 5 review agents IN PARALLEL as sub-agents:
    - bug-hunter
    - security-auditor
    - error-inspector
    - performance-detector
    - stack-reviewer
  - Each agent's prompt must include:
    - Which batch number and which directories/files to review (from the batch plan)
    - "Read the CLAUDE.md file in each directory you review FIRST for module context"
    - "Read the FULL contents of EVERY file listed. Do not skim or skip."
    - "Write your findings to `.deep-review/batch-{N}/{your-name}.md`"
    - "If you discover undocumented gotchas, fragile areas, or missing context, update the module's CLAUDE.md file"
    - "Follow your agent definition instructions exactly"
  - After all 5 agents complete for this batch, confirm all output files exist
  - Write a progress update to `.deep-review/progress.md`:
    - Which batches are done
    - Running count of findings by severity
  - Print: "Batch {N}/{total} complete. Found {X} issues so far."
- After ALL batches complete, print: "Phase 2 complete. {total findings} issues found across {B} batches."

**Batch sizing strategy (the orchestrator must decide based on discovery output):**
- Small repo (< 50 files): 1-2 batches, launch everything in parallel
- Medium repo (50-200 files): 3-6 batches, process 2 batches in parallel
- Large repo (200-1000 files): 6-15 batches, process 1-2 batches at a time
- Huge repo (1000+ files): 15+ batches, process 1 batch at a time, use /compact between batches

**Context pressure management:**
- The orchestrator should NEVER read file contents itself — only batch plans and progress files
- If context feels heavy, run /compact and re-read `.deep-review/progress.md` to resume
- Write phase/batch progress to disk before each compaction risk point
- After any compaction, read `.deep-review/progress.md` and `.deep-review/batch-plan.md` to know where to resume

#### Phase 3: Synthesis
- Launch the **synthesizer** agent as a sub-agent
- The synthesizer prompt must include:
  - "You are the synthesizer for a deep code review. Read all findings from `.deep-review/batch-*/` directories."
  - "Follow your agent definition instructions exactly."
  - "Write the final report to `REVIEW_REPORT.md` in the repo root."
  - "Update the root CLAUDE.md with architectural insights and known issues."
- After the synthesizer completes, confirm REVIEW_REPORT.md exists
- Print: "Phase 3 complete."

#### Final Summary
Print to terminal:
```
═══════════════════════════════════════════
  DEEP REVIEW COMPLETE
═══════════════════════════════════════════

  Duration:     {time}
  Files reviewed: {count}
  Modules documented: {count} CLAUDE.md files

  Findings:
    🔴 Critical:  {N}
    🟠 High:      {N}
    🟡 Medium:    {N}
    🔵 Low:       {N}

  Reports:
    → REVIEW_REPORT.md    (full findings)
    → CLAUDE.md files     (persistent documentation)

  Next steps:
    1. Read REVIEW_REPORT.md for prioritized findings
    2. Fix 🔴 Critical issues first
    3. Run /deep-review:maintain-docs after making changes
═══════════════════════════════════════════
```

---

### 3. `commands/maintain-docs.md` — DOCUMENTATION MAINTENANCE

When the user runs `/deep-review:maintain-docs`, this command:

1. Detects what changed via git:
   ```bash
   git diff --name-only HEAD~10
   git log --since="7 days ago" --name-only --pretty=format: | sort -u | grep -v '^$'
   ```
2. Identifies which directories were affected
3. For each affected directory that has a CLAUDE.md:
   - Reads the CLAUDE.md
   - Reads the changed files
   - Updates the CLAUDE.md sections: Key Files, Data Flow, Dependencies, Configuration, Gotchas, Recent Changes
   - Adds dated entries to the "Recent Changes" section
4. For new directories with 3+ code files and no CLAUDE.md:
   - Creates a CLAUDE.md using the standard template
5. Updates root CLAUDE.md:
   - Directory Map (add/remove entries)
   - Tech Stack (new deps or version changes)
   - Conventions (new patterns established)
   - Key Commands (any changes)
6. Runs validation:
   ```bash
   # Find directories with 3+ code files but no CLAUDE.md
   for dir in $(find . -type d | grep -v -E '(node_modules|\.git|__pycache__|venv|dist|build|\.cache|\.idea|\.vscode|\.deep-review)'); do
     code_files=$(find "$dir" -maxdepth 1 -type f \( -name '*.py' -o -name '*.js' -o -name '*.ts' -o -name '*.go' -o -name '*.rs' -o -name '*.java' \) | wc -l)
     has_claude=$(test -f "$dir/CLAUDE.md" && echo "yes" || echo "no")
     if [ "$code_files" -ge 3 ] && [ "$has_claude" = "no" ]; then
       echo "MISSING: $dir ($code_files files)"
     fi
   done
   ```
7. Prints summary of what was updated/created

**Key rules for this command:**
- Only update what actually changed — don't rewrite everything
- Be specific: "Added retry logic to BigQuery client" not "Updated module"
- Don't remove information unless it's provably wrong — append, don't replace
- Mark uncertainties with "TODO: verify" rather than guessing
- Keep Recent Changes entries to one line per change

---

### 4. `agents/discovery.md` — DISCOVERY AGENT

This agent is launched by the orchestrator in Phase 1. It runs in its own fresh context.

**Role:** Map the entire repository, understand every module, create documentation, produce a review batch plan.

**Instructions the agent must follow:**

#### Step 1: Map the repo
Run these commands:
```bash
find . -type f | grep -v -E '(node_modules|\.git/|__pycache__|\.pyc|venv|\.venv|\.env$|dist/|build/|\.next|\.cache|coverage|\.idea|\.vscode|\.mypy_cache|\.pytest_cache|\.ruff_cache|egg-info|\.tox|target/|bin/|obj/|\.deep-review)' | sort > .deep-review/file-list.txt

wc -l .deep-review/file-list.txt

cat .deep-review/file-list.txt | sed 's/.*\.//' | sort | uniq -c | sort -rn | head -30

cat .deep-review/file-list.txt | cut -d/ -f2 | sort | uniq -c | sort -rn

find . -type f \( -name '*.py' -o -name '*.js' -o -name '*.ts' -o -name '*.tsx' -o -name '*.jsx' -o -name '*.go' -o -name '*.rs' -o -name '*.java' -o -name '*.sql' \) | grep -v -E '(node_modules|\.git/|__pycache__|venv|dist|build)' | xargs wc -l 2>/dev/null | tail -1

ls -la
cat README.md 2>/dev/null || echo "No README"
cat pyproject.toml 2>/dev/null || cat package.json 2>/dev/null || cat Cargo.toml 2>/dev/null || cat go.mod 2>/dev/null || echo "No manifest"
cat requirements.txt 2>/dev/null || cat Pipfile 2>/dev/null || echo "No requirements"
cat Dockerfile 2>/dev/null || echo "No Dockerfile"
cat docker-compose.yml 2>/dev/null || cat docker-compose.yaml 2>/dev/null || echo "No docker-compose"
ls .github/workflows/ 2>/dev/null || echo "No CI"
```

#### Step 2: Understand every module
For each top-level directory, read 3-5 representative files to determine:
- Purpose (what does it do?)
- Key files (entry points, core logic)
- Internal dependencies (what other modules it imports)
- External dependencies (libraries, APIs, databases)
- Data flow (input → processing → output)
- Patterns and conventions used
- Configuration (env vars, config files)
- Gotchas (non-obvious behavior, complexity)
- Testing (where tests live, framework, gaps)

#### Step 3: Create CLAUDE.md files
For every directory with 3+ code files or complex logic, create a CLAUDE.md:

```markdown
# {Module Name}

## Purpose
{2-3 sentences}

## Key Files
| File | Purpose | Key Exports |
|------|---------|-------------|
| {f}  | {what}  | {functions/classes} |

## Data Flow
{input → processing → output}

## Dependencies
- Internal: {other modules}
- External: {libraries, APIs, databases}

## Patterns & Conventions
{module-specific patterns}

## Configuration
{env vars, config files}

## Gotchas
- {non-obvious behavior}
- {fragile areas}

## Testing
- Tests: {path}
- Run: {command}
- Gaps: {what's not tested}

## Recent Changes
{empty — will be populated by maintenance}
```

Also create or update the root CLAUDE.md with the comprehensive template (project overview, architecture diagram, tech stack, directory map, conventions, key commands, important context, documentation maintenance rules).

#### Step 4: Write the discovery profile
Write `.deep-review/discovery.md`:
```markdown
# Repository Profile
- Name: {repo name}
- Stack: {languages, frameworks, databases}
- Architecture: {monolith/microservices/pipeline/library/etc}
- Total files: {count}
- Total LOC: {count}
- Modules: {count}
- CLAUDE.md files created: {count}

## Architecture Diagram
{text-based diagram}

## Module Summary
| Module | Purpose | Files | LOC | Risk Level |
|--------|---------|-------|-----|------------|
| {dir}  | {what}  | {n}   | {n} | {high/med/low} |

## Risk Areas
1. {highest risk area and why}
2. {second highest}
3. {third highest}
```

#### Step 5: Generate batch plan
Write `.deep-review/batch-plan.md`:
```markdown
# Review Batch Plan

Total batches: {N}
Batch sizing: {strategy chosen based on repo size}

## Batch 1: {descriptive name}
Risk level: {high/medium/low}
Directories:
- {path/to/dir1}
- {path/to/dir2}
Files: {count}
LOC: {count}
Focus areas: {what to look for specifically in this batch}
Stack-specific checks:
- {check 1 relevant to this batch}
- {check 2}

## Batch 2: {descriptive name}
...
```

**Batch grouping rules:**
- Under 50 files or 5000 LOC per batch (whichever is smaller)
- Group related modules together (API routes + their models + their services)
- Order by risk: highest-risk directories first
- Include stack-specific focus areas per batch based on what you learned about the code

**What makes each CLAUDE.md valuable (quality bar):**
- It must contain information that saves a developer (human or AI) 30+ minutes of code reading
- It must be specific to THIS code, not generic boilerplate
- File tables must list actual files with actual descriptions
- Data flow must describe actual data transformations
- Gotchas must describe actual non-obvious things, not generic advice
- If you can't determine something, say "TODO: investigate" rather than making it up

---

### 5. `agents/bug-hunter.md` — BUG HUNTER AGENT

**Role:** Find logic errors, bugs, and correctness issues in assigned files.

**What to check in EVERY file:**
- Logic errors: incorrect comparisons, wrong operators, inverted conditions
- Off-by-one errors: loop bounds, array indexing, string slicing
- Null/undefined/None access: missing null checks, optional chaining gaps, unguarded attribute access
- Wrong variable used: copy-paste errors, shadowed variables, incorrect scope
- Incorrect state management: stale closures, race conditions between state updates, missing state resets
- TOCTOU (time-of-check-time-of-use): file existence checks followed by file operations, check-then-act patterns
- Integer overflow / floating point issues: arithmetic that could overflow, float equality comparisons
- Infinite loops: missing break conditions, incorrect loop termination
- Incorrect function calls: wrong argument order, missing required args, wrong function entirely
- Dead branches: unreachable code after returns/throws, impossible conditions
- Type confusion: operations on wrong types, missing type narrowing, unsafe casts
- Boolean logic: De Morgan violations, double negation confusion, short-circuit side effects
- String handling: encoding issues, locale-dependent operations, regex edge cases
- Date/time: timezone mishandling, DST edge cases, epoch comparison errors
- Collection operations: modifying while iterating, incorrect sort comparators, empty collection edge cases

**Output format (STRICT — every finding must follow this):**
```markdown
## 🔴 CRITICAL: {short description}
**File:** `{exact/file/path.ext}`
**Line(s):** {line number or range}
**Category:** {Logic Error | Null Access | Race Condition | Type Error | ...}
**Description:** {what's wrong and why it's a problem}
**Impact:** {what breaks, data loss, crash, incorrect results, etc.}
**Fix:** {specific code change or approach to fix it}

---
```

**Severity guide:**
- 🔴 CRITICAL: Will cause crashes, data loss, security breaches, or incorrect results in production
- 🟠 HIGH: Will cause bugs under common conditions, incorrect behavior users will notice
- 🟡 MEDIUM: Edge case bugs, incorrect behavior under uncommon conditions
- 🔵 LOW: Code quality issues that could lead to bugs in the future

**Mandatory rules:**
- Read the FULL contents of every assigned file
- Do NOT say "the rest of the code looks fine" or "similar issues in other files"
- Every finding must have exact file path and line number
- If you find 0 issues in a file with 100+ lines, you are not looking hard enough — re-read it
- Focus on actual bugs, not style issues

---

### 6. `agents/security-auditor.md` — SECURITY AUDITOR AGENT

**Role:** Find security vulnerabilities in assigned files.

**What to check in EVERY file:**

**Injection:**
- SQL injection: string concatenation or f-strings in SQL queries, unsanitized input in raw queries
- Command injection: user input reaching subprocess, os.system, exec, eval, child_process
- XSS: unescaped output in HTML templates, innerHTML, dangerouslySetInnerHTML, document.write
- Template injection: user input in template strings without escaping
- LDAP injection, XPath injection, NoSQL injection

**Authentication & Authorization:**
- Missing auth checks on endpoints/routes
- Broken access control: horizontal privilege escalation, IDOR
- Hardcoded credentials, API keys, tokens, passwords anywhere in code
- Weak password requirements, missing brute-force protection
- JWT issues: none algorithm accepted, missing expiry validation, weak secret, token not invalidated on logout
- Session management: missing session expiry, predictable session IDs, session fixation
- CSRF: missing token validation on state-changing requests

**Data Exposure:**
- Sensitive data in logs (passwords, tokens, PII)
- Verbose error messages exposing stack traces, DB schemas, file paths in production
- Sensitive data in URL parameters (appears in server logs, browser history)
- Missing data encryption at rest or in transit
- Overly permissive CORS configuration
- Missing security headers (CSP, X-Frame-Options, X-Content-Type-Options, HSTS)

**Input Handling:**
- Path traversal: user input in file paths without sanitization (../../etc/passwd)
- SSRF: user-controlled URLs in server-side HTTP requests
- File upload without type/size validation
- Deserialization of untrusted data (pickle, yaml.load, JSON.parse of user input into eval)
- XML external entity (XXE) processing

**Secrets & Configuration:**
- Hardcoded secrets (grep for patterns: password=, secret=, token=, api_key=, private_key=)
- .env files committed to git
- Default credentials left in place
- Debug mode enabled in production config
- Insecure defaults (HTTP instead of HTTPS, weak crypto algorithms)

**Dependencies:**
- Known vulnerable packages (note versions for later audit)
- Dependency confusion risks (private package names that could be squatted)

**Same output format as bug-hunter. Same severity scale. Same mandatory rules.**

**Additional security-specific rule:** If you find a CRITICAL security vulnerability, also note whether it's exploitable remotely (RCE, SQLi, etc.) vs locally, and whether it requires authentication.

---

### 7. `agents/error-inspector.md` — ERROR INSPECTOR AGENT

**Role:** Find missing, broken, or inadequate error handling in assigned files.

**What to check in EVERY file:**

**Missing Error Handling:**
- I/O operations (file read/write, network calls, DB queries) without try/catch or error checks
- External API calls without timeout configuration
- Missing retry logic for transient failures (network timeouts, DB connection drops)
- Unhandled promise rejections (.then() without .catch(), async functions without try/catch)
- Missing null/error checks on function return values
- Missing validation before operations that can fail

**Broken Error Handling:**
- Empty catch blocks (catch and ignore)
- Catch-all that swallows specific exceptions: `except Exception`, `catch (e) {}`
- Error logged but not re-thrown or propagated when it should be
- Error message that hides the root cause (generic "something went wrong")
- Wrong exception type caught (too broad or too narrow)
- Error handling that introduces new errors (e.g., logging inside catch that can itself fail)

**Resilience Issues:**
- Missing timeouts on HTTP clients, DB connections, external calls
- Missing circuit breakers for external service dependencies
- Missing graceful shutdown handling (SIGTERM, SIGINT)
- Missing health checks
- Crash-inducing errors in main loops, workers, or background tasks
- Missing dead letter queues or failure queues for async processing
- Missing idempotency on retryable operations

**Resource Cleanup:**
- Missing finally blocks for resource cleanup (file handles, DB connections, locks)
- Missing context managers (Python: `with`, JS: try/finally, Go: defer)
- Connections opened but never closed on error paths
- Transactions started but not committed or rolled back on error

**Logging & Observability:**
- Errors that happen silently with no logging
- Missing correlation IDs or request context in error logs
- Inconsistent error response formats across the codebase
- Missing metrics/alerts for error conditions

**Same output format, severity scale, and mandatory rules as bug-hunter.**

---

### 8. `agents/performance-detector.md` — PERFORMANCE DETECTOR AGENT

**Role:** Find performance issues and resource leaks in assigned files.

**What to check in EVERY file:**

**Database & Queries:**
- N+1 query patterns: loop that makes a DB query per iteration
- Missing query batching: multiple sequential queries that could be one
- Unbounded queries: SELECT without LIMIT, fetching all rows into memory
- Missing indexes: queries filtering/sorting on columns likely not indexed (note for later verification)
- Full table scans: queries without partition filters (especially BigQuery)
- SELECT *: fetching all columns when only a few are needed
- Missing connection pooling
- Connection leaks: connections opened but not returned to pool on all code paths

**Memory:**
- Unbounded growth: arrays/lists/maps that grow without eviction (caches, buffers, history)
- Loading entire large files/datasets into memory
- Event listener leaks: addEventListener without removeEventListener
- Closure leaks: closures capturing large scopes unnecessarily
- Buffer accumulation: streams or buffers not properly drained

**CPU & I/O:**
- Synchronous blocking in async contexts (sync file I/O in async handler, blocking DB call in event loop)
- O(n²) or worse algorithms where O(n) or O(n log n) is possible
- Unnecessary loops: iterating when a hash lookup would work
- Repeated computation: same expensive calculation done multiple times without caching
- Missing lazy loading: eagerly loading data that may not be needed
- Large object serialization in hot paths (JSON.stringify on large objects per request)

**Concurrency:**
- Missing connection/thread pool limits
- Unbounded parallelism: launching unlimited concurrent tasks
- Lock contention: overly broad locks, locks held during I/O
- Missing rate limiting on outbound calls

**Resources:**
- Unclosed file handles, streams, cursors, sockets
- Missing cleanup on process exit
- Temp files created but never deleted
- Log files that grow unbounded without rotation

**Network:**
- Missing pagination for list endpoints
- Missing compression for large responses
- Chatty APIs: many small requests where one batch request would work
- Missing caching for repeated identical requests

**Same output format, severity scale, and mandatory rules as bug-hunter.**

**Performance-specific severity:**
- 🔴 CRITICAL: Will cause OOM, crash under load, or extreme cost (e.g., BigQuery full table scan on a petabyte table)
- 🟠 HIGH: Will cause noticeable slowness or resource pressure under normal load
- 🟡 MEDIUM: Performance issue under high load or with large data
- 🔵 LOW: Suboptimal but not causing problems at current scale

---

### 9. `agents/stack-reviewer.md` — STACK-SPECIFIC REVIEWER AGENT

**Role:** Check code against best practices specific to the tech stack used.

**This agent is unique because its checks depend on what the discovery agent found.**

**Instructions:**
1. Read the root CLAUDE.md to identify the tech stack
2. Read each module's CLAUDE.md to understand what each directory does
3. Apply the relevant checks below based on the stack

**Python checks:**
- Type hints present and correct
- Docstrings on public functions/classes
- No mutable default arguments (def foo(bar=[]))
- Using pathlib over os.path for file operations
- Proper use of dataclasses/Pydantic for data structures
- No bare except clauses
- Proper virtual environment / dependency management

**Airflow checks:**
- DAGs have retries, retry_delay, and execution_timeout configured
- No top-level code execution in DAG files (import-time side effects)
- Tasks are idempotent (re-running produces same result)
- Proper use of Airflow Variables/Connections vs hardcoded values
- XCom usage is minimal and data passed is small (not DataFrames)
- Sensors have timeout and poke_interval set
- No circular DAG dependencies
- Proper task dependency chains (no implicit ordering reliance)
- Using TaskFlow API or proper operator patterns
- DAG schedule intervals are reasonable

**BigQuery/SQL checks:**
- ALL queries use partition filters (no full table scans — this is a cost issue)
- MERGE statements have proper matching conditions
- No SELECT * in production queries
- Cost-aware patterns (avoid repeated scans, use CTEs, materialize intermediate results)
- Proper use of ARRAY/STRUCT types
- Proper use of clustering and partitioning on tables
- Parameterized queries (no string concatenation/f-strings for SQL)
- Appropriate use of DML vs DDL
- Query complexity is reasonable (not 500-line single queries)

**FastAPI/Flask checks:**
- All endpoints have proper auth middleware/dependencies
- Request/response models use Pydantic with validation
- Proper HTTP status codes
- Async endpoints don't call sync blocking functions
- Dependency injection used correctly
- CORS configured appropriately
- Proper error response format (consistent across endpoints)

**Docker/Kubernetes checks:**
- No secrets in Dockerfiles or K8s manifests
- Resource limits (CPU/memory) set on containers
- Health/readiness/liveness probes configured
- Non-root user in Dockerfiles
- Multi-stage builds for production
- .dockerignore exists and is comprehensive
- No latest tags in production images

**JavaScript/TypeScript checks:**
- No `any` types in TypeScript (or justified with comments)
- Proper async/await error handling
- No prototype pollution risks
- Dependencies pinned to exact versions (or at least locked)
- No eval() or Function() with dynamic input
- Proper module boundaries and exports

**React checks:**
- No unnecessary re-renders (missing useMemo, useCallback where needed)
- Proper key props in lists
- Effects have correct dependency arrays
- No direct DOM manipulation
- Proper error boundaries

**Go checks:**
- Errors checked (no _ = for error returns)
- Context passed through call chains
- Proper goroutine lifecycle management
- No goroutine leaks

**Rust checks:**
- Proper error handling with Result types
- No unwrap() in production paths
- Proper lifetime management

**Same output format and mandatory rules as other agents.**

**If the stack includes something not listed above**, the agent should identify the framework/library and apply reasonable best practices based on its training knowledge + any CLAUDE.md context about conventions.

---

### 10. `agents/synthesizer.md` — SYNTHESIZER AGENT

**Role:** Combine all findings, run cross-cutting analysis, produce the final report.

**Instructions:**

#### Step 1: Read all findings
Read every `.md` file in every `.deep-review/batch-*/` directory.
Compile into a single list. Deduplicate: if the same issue (same file, same line, same problem) was found by multiple agents, keep the most detailed report and note it was flagged by multiple agents.

#### Step 2: Cross-cutting analysis (perform each as investigation)

**Architecture:**
- Map module dependencies by grepping imports across the codebase
- Identify circular dependencies
- Identify inconsistent patterns (e.g., some modules use one error handling pattern, others use a different one)
- Identify god modules (too many responsibilities)
- Identify tight coupling between components that should be independent

**Dependencies:**
- Run the appropriate audit command for the stack:
  - Python: `pip audit 2>/dev/null || safety check 2>/dev/null || echo "No audit tool"`
  - Node: `npm audit 2>/dev/null || echo "No npm"`
  - Go: `govulncheck ./... 2>/dev/null || echo "No govulncheck"`
  - Rust: `cargo audit 2>/dev/null || echo "No cargo-audit"`
- Check for unused dependencies
- Check for version conflicts

**Testing gaps:**
- Compare the list of modules/files reviewed against available tests
- Identify critical paths (auth, data mutations, payment, etc.) with no tests
- Note test quality issues found by review agents

**Secrets scan:**
```bash
grep -rn -E '(password|secret|token|api_key|apikey|private_key|credential|passwd)\s*=' \
  --include='*.py' --include='*.js' --include='*.ts' --include='*.yaml' \
  --include='*.yml' --include='*.json' --include='*.toml' --include='*.cfg' \
  --include='*.ini' . 2>/dev/null | grep -v node_modules | grep -v venv | grep -v '.git/' | head -50
```

#### Step 3: Update root CLAUDE.md
Read the current root CLAUDE.md and add/update:
- "Known Issues" section with critical/high findings summary
- "Technical Debt" section with architectural issues found
- "Important Context" additions from the review
- Any new conventions or patterns discovered

#### Step 4: Write REVIEW_REPORT.md
Write to the repo root with this structure:

```markdown
# Code Review Report

**Generated:** {date}
**Repository:** {name}
**Review method:** Automated multi-agent deep review (deep-review plugin)

---

## Executive Summary
{2-3 sentences: overall health assessment, biggest risk, top recommendation}

## Statistics
| Metric | Count |
|--------|-------|
| Files reviewed | {N} |
| Total issues | {N} |
| 🔴 Critical | {N} |
| 🟠 High | {N} |
| 🟡 Medium | {N} |
| 🔵 Low | {N} |
| Modules documented | {N} CLAUDE.md files |

---

## 🔴 Critical Issues — Fix Immediately

### {Issue title}
**File:** `{path}` **Line(s):** {N}
**Category:** {Security | Bug | Error Handling | Performance}
**Found by:** {agent name(s)}
**Description:** {what's wrong}
**Impact:** {what could happen}
**Fix:** {how to fix}

---

## 🟠 High Priority Issues

### {Category}: {Subcategory}
{grouped findings}

---

## 🟡 Medium Priority Issues

### {Category}: {Subcategory}
{grouped findings}

---

## 🔵 Low Priority Issues

### {Category}: {Subcategory}
{grouped findings}

---

## Cross-Cutting Analysis

### Architecture
{findings}

### Dependencies
{audit results, vulnerabilities, unused deps}

### Testing Gaps
{untested critical paths}

### Configuration & Secrets
{any issues found}

---

## Recommendations
1. **{Highest priority}:** {action}
2. **{Second priority}:** {action}
3. **{Third priority}:** {action}
...

---

## Documentation Generated
This review created/updated the following CLAUDE.md files:
{list all CLAUDE.md file paths}

These files provide persistent context for future Claude Code sessions.
Run `/deep-review:maintain-docs` after making changes to keep them current.

---

## Appendix: Files Reviewed
{complete list of every file reviewed, grouped by batch}
```

---

### 11. `skills/codebase-documentation/SKILL.md` — PASSIVE DOCUMENTATION SKILL

This skill auto-triggers during normal Claude Code usage (NOT during reviews — it's for everyday development).

```yaml
---
name: codebase-documentation
description: >
  Maintains living documentation in CLAUDE.md files throughout the codebase.
  Auto-triggers when Claude is working on code, modifying files, or exploring
  the codebase. Ensures Claude reads local CLAUDE.md before making changes
  and updates them after significant modifications.
---
```

**Skill instructions:**

When working on code in this repository:

1. **Before modifying any file**, check if the directory has a CLAUDE.md. If it does, read it first to understand the module context, conventions, and gotchas.

2. **After making significant changes** (adding files, changing function signatures, adding dependencies, modifying data flow, adding configuration), update the relevant CLAUDE.md:
   - Add a dated entry to "Recent Changes"
   - Update any sections that are now outdated
   - Add new gotchas if you discovered non-obvious behavior

3. **If you create a new directory** with 3+ code files, create a CLAUDE.md for it following the standard template.

4. **If you notice a CLAUDE.md is outdated or inaccurate**, fix it as you go.

5. **Never delete information from CLAUDE.md** unless it's provably wrong. Append instead.

This skill should NOT trigger during deep-review operations (when .deep-review/ directory work is happening).

---

### 12. `README.md` — PLUGIN DOCUMENTATION

Standard README with:
- What the plugin does (1 paragraph)
- Installation instructions
- Commands: `/deep-review:full-review` and `/deep-review:maintain-docs`
- What gets generated (REVIEW_REPORT.md, CLAUDE.md files)
- Architecture overview (the sub-agent diagram from above)
- Recommended companion plugins (GrepAI or claude-context-local, ast-grep, pr-review-toolkit)
- Configuration: recommended CLAUDE.md compaction rules, recommended MCP settings, MAX_THINKING_TOKENS tip
- Cost expectations table (repo size → estimated sessions/time/cost)
- How the documentation compounds over time

---

## IMPORTANT BUILD INSTRUCTIONS

1. **Build every file.** Don't skip any of the 12 files specified above. Each has a specific role.

2. **Agent definitions must be exhaustive.** The checklists in each agent are the core value — they tell Claude exactly what to look for. Don't abbreviate them. The agents need to be extremely thorough.

3. **The orchestrator is the hardest file.** It must correctly coordinate phases, launch sub-agents with the right prompts (including batch-specific context), handle context pressure, and track progress via disk. Spend extra time getting this right.

4. **Test the flow mentally:**
   - User runs `/deep-review:full-review`
   - Orchestrator creates .deep-review/, launches discovery agent
   - Discovery agent maps repo, creates CLAUDE.md files, writes batch plan
   - Orchestrator reads batch plan, launches 5 agents per batch in parallel
   - Each agent reads files, writes findings to disk, updates CLAUDE.md files
   - Orchestrator tracks progress, launches synthesizer
   - Synthesizer reads all findings, runs cross-cutting analysis, writes REVIEW_REPORT.md
   - Orchestrator prints summary
   - User reads REVIEW_REPORT.md

5. **The CLAUDE.md files are the long-term value.** The review report is useful once. The CLAUDE.md files are useful forever. Make sure the discovery agent and review agents create thorough, specific documentation — not generic templates.

6. **Every agent must enforce thoroughness.** Include these rules in EVERY agent:
   - "Read the FULL contents of every assigned file — do not skim"
   - "Do NOT say 'the rest looks fine' or 'similar issues in other files'"
   - "Every finding must have exact file path, line number, severity, description, and fix"
   - "If you find 0 issues in a file with 100+ lines, re-read it more carefully"

7. **Disk is the coordination layer.** Agents communicate through files in `.deep-review/`, not through the orchestrator's context. The orchestrator should never hold file contents — only batch plans and progress summaries.

8. **The skill is passive, not active.** It triggers during normal development, not during reviews. It's a gentle reminder to read/update CLAUDE.md files. Keep it lightweight.

9. **After building**, verify the directory structure matches the specification exactly. Run `find deep-review/ -type f | sort` and confirm all 12 files exist.
