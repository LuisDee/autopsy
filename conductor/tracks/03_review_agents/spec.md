<!-- ARCHITECT CONTEXT | Track: 03_review_agents | Wave: 2 | CC: v1 -->

## Cross-Cutting Constraints
- Agent Output Format: severity, confidence, exact file path, line number, category, description, impact, fix
- Self-Review Step: every agent verifies its own output completeness before finishing
- Few-Shot Examples: 2-3 concrete finding examples per agent in the exact output format
- Thoroughness Rules: read full file contents, no skimming, no "looks fine", exact paths + lines, re-read 100+ line files with zero findings

## Interfaces

### Owns
- `.deep-review/batch-{N}/bugs.md` (bug-hunter findings)
- `.deep-review/batch-{N}/security.md` (security-auditor findings)
- `.deep-review/batch-{N}/errors.md` (error-inspector findings)
- `.deep-review/batch-{N}/performance.md` (performance-detector findings)
- `.deep-review/batch-{N}/stack.md` (stack-reviewer findings)

### Consumes
- `{dir}/AGENTS.md` (from Track 02 — read for module context before reviewing)
- `.deep-review/discovery.md` (from Track 02 — repo profile for stack context)

## Dependencies
- Track 01_plugin_scaffold: `agents/` directory must exist

<!-- END ARCHITECT CONTEXT -->

# Specification: Review Agents

## Overview

Define 5 specialized Phase 2 review agent definitions — the core value of the plugin. Each agent runs in parallel as a background task with its own context window, focuses on a single review domain, reads every file in its assigned batch, and produces severity-graded findings. The agents are: bug-hunter, security-auditor, error-inspector, performance-detector, and stack-reviewer.

## Design Decisions Applied

- **DD1 Model:** Inherit from parent (no model specified in frontmatter — user controls via launch config)
- **DD2 Stack checks:** Dynamic — stack-reviewer reads AGENTS.md/discovery.md to determine which checks apply
- **DD3 Few-shots:** Generic examples showing exact output format, universally applicable
- **DD4 Tools:** Read-only — Read, Grep, Glob only (no Bash)
- **DD5 Doc updates:** Read-only. Undocumented gotchas are reported as findings with category "Documentation Gap" rather than modifying AGENTS.md directly
- **DD6 Confidence:** Explicit rules — High (exact code path shown), Medium (pattern match with context), Low (heuristic/inferred)

## Shared Requirements (All 5 Agents)

### YAML Frontmatter

Each agent must have:
```yaml
---
name: {agent-name}
description: "{one-line description}"
tools:
  - Read
  - Grep
  - Glob
---
```

### Finding Output Format (STRICT)

Every finding must follow this exact format:

```markdown
## {SEVERITY_EMOJI} {SEVERITY}: {short description}
**File:** `{exact/file/path.ext}`
**Line(s):** {line number or range}
**Category:** {category name}
**Confidence:** {High|Medium|Low}
**Description:** {what's wrong and why}
**Impact:** {what breaks, data loss, crash, incorrect results}
**Fix:** {specific code change or approach}

---
```

### Severity Scale

- `🔴 CRITICAL`: Will cause crashes, data loss, security breaches, or incorrect results in production
- `🟠 HIGH`: Will cause bugs under common conditions, incorrect behavior users will notice
- `🟡 MEDIUM`: Edge case bugs, incorrect behavior under uncommon conditions
- `🔵 LOW`: Code quality issues that could lead to bugs in the future

### Confidence Score Rules (Explicit)

- **High:** You can show the exact code path that triggers the issue. You have concrete evidence.
- **Medium:** The pattern strongly suggests an issue but you cannot fully trace the execution path (e.g., depends on runtime config or external input).
- **Low:** Heuristic-based or inferred. The code looks suspicious but the issue depends on context you can't verify (e.g., how the function is called, what the data looks like at runtime).

### Mandatory Rules (All Agents)

1. Read the AGENTS.md in every assigned directory FIRST for module context
2. Read the FULL contents of EVERY assigned file — no skimming
3. Do NOT say "the rest of the code looks fine" or "similar issues in other files"
4. Every finding must have exact file path and line number
5. If you find 0 issues in a file with 100+ lines, re-read it — you are not looking hard enough
6. Focus on your domain — do not duplicate other agents' work
7. Report undocumented gotchas as findings with category "Documentation Gap" (do NOT modify AGENTS.md)

### Few-Shot Examples (2-3 per Agent)

Each agent must include 2-3 generic finding examples in the exact output format. These serve as calibration for the model. Examples must be universally applicable (not stack-specific) and demonstrate the expected level of specificity.

### Self-Review Step

Every agent must verify before completing:
- [ ] I read every assigned file completely
- [ ] Every finding has exact file path and line number
- [ ] Every finding follows the output format exactly
- [ ] I used the correct severity and confidence per the rules
- [ ] I did not say "looks fine" or skip any file
- [ ] Files with 100+ lines and zero findings were re-read

### Output File Header

Each agent's output file must start with:
```markdown
# {Agent Name} — Batch {N} Findings

**Files reviewed:** {count}
**Total findings:** {count}
**By severity:** 🔴 {n} | 🟠 {n} | 🟡 {n} | 🔵 {n}

---
```

## FR-1: Bug Hunter Agent (`agents/bug-hunter.md`)

**Checklist — what to check in EVERY file:**
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

## FR-2: Security Auditor Agent (`agents/security-auditor.md`)

**Checklist — what to check in EVERY file:**

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
- Sensitive data in URL parameters
- Missing data encryption at rest or in transit
- Overly permissive CORS configuration
- Missing security headers (CSP, X-Frame-Options, X-Content-Type-Options, HSTS)

**Input Handling:**
- Path traversal: user input in file paths without sanitization
- SSRF: user-controlled URLs in server-side HTTP requests
- File upload without type/size validation
- Deserialization of untrusted data (pickle, yaml.load, JSON.parse of user input into eval)
- XML external entity (XXE) processing

**Secrets & Configuration:**
- Hardcoded secrets (grep for: password=, secret=, token=, api_key=, private_key=)
- .env files committed to git
- Default credentials left in place
- Debug mode enabled in production config
- Insecure defaults (HTTP instead of HTTPS, weak crypto algorithms)

**Dependencies:**
- Known vulnerable packages (note versions for later audit)
- Dependency confusion risks (private package names that could be squatted)

**Additional rule:** For CRITICAL security findings, note whether exploitable remotely vs locally, and whether it requires authentication.

## FR-3: Error Inspector Agent (`agents/error-inspector.md`)

**Checklist — what to check in EVERY file:**

**Missing Error Handling:**
- I/O operations without try/catch or error checks
- External API calls without timeout configuration
- Missing retry logic for transient failures
- Unhandled promise rejections (.then() without .catch(), async without try/catch)
- Missing null/error checks on function return values
- Missing validation before operations that can fail

**Broken Error Handling:**
- Empty catch blocks (catch and ignore)
- Catch-all that swallows specific exceptions: `except Exception`, `catch (e) {}`
- Error logged but not re-thrown or propagated when it should be
- Error message hiding root cause (generic "something went wrong")
- Wrong exception type caught (too broad or too narrow)
- Error handling that introduces new errors

**Resilience Issues:**
- Missing timeouts on HTTP clients, DB connections, external calls
- Missing circuit breakers for external service dependencies
- Missing graceful shutdown handling (SIGTERM, SIGINT)
- Missing health checks
- Crash-inducing errors in main loops, workers, or background tasks
- Missing dead letter queues or failure queues for async processing
- Missing idempotency on retryable operations

**Resource Cleanup:**
- Missing finally blocks for resource cleanup
- Missing context managers (Python: `with`, JS: try/finally, Go: defer)
- Connections opened but never closed on error paths
- Transactions started but not committed or rolled back on error

**Logging & Observability:**
- Errors happening silently with no logging
- Missing correlation IDs or request context in error logs
- Inconsistent error response formats across the codebase
- Missing metrics/alerts for error conditions

## FR-4: Performance Detector Agent (`agents/performance-detector.md`)

**Checklist — what to check in EVERY file:**

**Database & Queries:**
- N+1 query patterns: loop that makes a DB query per iteration
- Missing query batching: multiple sequential queries that could be one
- Unbounded queries: SELECT without LIMIT, fetching all rows into memory
- Missing indexes: queries filtering/sorting on columns likely not indexed
- Full table scans: queries without partition filters (especially BigQuery)
- SELECT *: fetching all columns when only a few are needed
- Missing connection pooling
- Connection leaks: connections not returned to pool on all code paths

**Memory:**
- Unbounded growth: arrays/lists/maps that grow without eviction
- Loading entire large files/datasets into memory
- Event listener leaks: addEventListener without removeEventListener
- Closure leaks: closures capturing large scopes unnecessarily
- Buffer accumulation: streams or buffers not properly drained

**CPU & I/O:**
- Synchronous blocking in async contexts
- O(n²) or worse algorithms where O(n) or O(n log n) is possible
- Unnecessary loops: iterating when a hash lookup would work
- Repeated computation: same expensive calculation multiple times without caching
- Missing lazy loading: eagerly loading data that may not be needed
- Large object serialization in hot paths

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

**Performance-specific severity override:**
- 🔴 CRITICAL: Will cause OOM, crash under load, or extreme cost (e.g., BigQuery full table scan on petabyte table)
- 🟠 HIGH: Will cause noticeable slowness or resource pressure under normal load
- 🟡 MEDIUM: Performance issue under high load or with large data
- 🔵 LOW: Suboptimal but not causing problems at current scale

## FR-5: Stack Reviewer Agent (`agents/stack-reviewer.md`)

**Dynamic approach:** Read AGENTS.md and `.deep-review/discovery.md` to determine the tech stack, then apply ONLY the relevant checks below.

**Python checks:**
- Type hints present and correct
- Docstrings on public functions/classes
- No mutable default arguments (`def foo(bar=[])`)
- Using pathlib over os.path for file operations
- Proper use of dataclasses/Pydantic for data structures
- No bare except clauses
- Proper virtual environment / dependency management

**Airflow checks:**
- DAGs have retries, retry_delay, and execution_timeout configured
- No top-level code execution in DAG files (import-time side effects)
- Tasks are idempotent
- Proper use of Airflow Variables/Connections vs hardcoded values
- XCom usage is minimal and data passed is small
- Sensors have timeout and poke_interval set
- No circular DAG dependencies
- Proper task dependency chains
- Using TaskFlow API or proper operator patterns
- DAG schedule intervals are reasonable

**BigQuery/SQL checks:**
- ALL queries use partition filters (no full table scans)
- MERGE statements have proper matching conditions
- No SELECT * in production queries
- Cost-aware patterns (avoid repeated scans, use CTEs, materialize intermediate results)
- Proper use of ARRAY/STRUCT types
- Proper use of clustering and partitioning
- Parameterized queries (no string concatenation for SQL)
- Appropriate use of DML vs DDL
- Query complexity is reasonable

**FastAPI/Flask checks:**
- All endpoints have proper auth middleware/dependencies
- Request/response models use Pydantic with validation
- Proper HTTP status codes
- Async endpoints don't call sync blocking functions
- Dependency injection used correctly
- CORS configured appropriately
- Proper error response format

**Docker/Kubernetes checks:**
- No secrets in Dockerfiles or K8s manifests
- Resource limits (CPU/memory) set on containers
- Health/readiness/liveness probes configured
- Non-root user in Dockerfiles
- Multi-stage builds for production
- .dockerignore exists and is comprehensive
- No latest tags in production images

**JavaScript/TypeScript checks:**
- No `any` types in TypeScript (or justified)
- Proper async/await error handling
- No prototype pollution risks
- Dependencies pinned to exact versions
- No eval() or Function() with dynamic input
- Proper module boundaries and exports

**React checks:**
- No unnecessary re-renders (missing useMemo, useCallback)
- Proper key props in lists
- Effects have correct dependency arrays
- No direct DOM manipulation
- Proper error boundaries

**Go checks:**
- Errors checked (no `_ =` for error returns)
- Context passed through call chains
- Proper goroutine lifecycle management
- No goroutine leaks

**Rust checks:**
- Proper error handling with Result types
- No unwrap() in production paths
- Proper lifetime management

**Catch-all:** If the stack includes something not listed above, identify the framework/library and apply reasonable best practices.

## Non-Functional Requirements

- Each agent definition must be a single Markdown file with YAML frontmatter
- All checklists must be EXHAUSTIVE — they are the core value of the plugin. Do not abbreviate.
- Few-shot examples must demonstrate the exact output format with realistic content
- Agent definitions must work for any codebase regardless of language
- No external dependencies — everything is embedded in the agent definition

## Acceptance Criteria

- [ ] All 5 agent files exist in `deep-review/agents/` with valid YAML frontmatter
- [ ] Each agent has the shared output format documented
- [ ] Each agent has the complete checklist from the build plan (not abbreviated)
- [ ] Each agent has 2-3 generic few-shot examples
- [ ] Each agent has the self-review verification step
- [ ] Each agent has the output file header template
- [ ] Confidence score rules are documented in each agent
- [ ] Mandatory rules are documented in each agent
- [ ] Stack-reviewer includes all 10 stack sections with dynamic detection logic
- [ ] No agent specifies a model in frontmatter (inherits from parent)

## Out of Scope

- Orchestrator logic for launching agents (Track 05)
- Synthesizer logic for parsing output (Track 04)
- AGENTS.md modification (agents are read-only)
