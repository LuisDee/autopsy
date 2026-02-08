<!-- ARCHITECT CONTEXT | Track: 07_output_rendering | Wave: 4 | CC: v1, v2 -->

## Cross-Cutting Constraints
- Agent Output Format (v1): Finding format uses emoji severity markers and specific header format. This track proposes replacing emoji markers with text-only severity labels in terminal output while preserving structured data for file output.
- Agent Output Format (v2 -- introduced by this track): Terminal-facing output uses text-only severity with color codes. File-persisted findings (batch-*/\*.md, REVIEW_REPORT.md) may retain or remove emoji per design decision DD-3.
- Compaction-Safe State Management (v1): progress.md format changes must remain parseable by the orchestrator's compaction recovery logic.
- Self-Review Step (v1): Agents must still verify output completeness. Output rendering changes must not break self-review file checks.

## Interfaces

### Owns
- Output Rendering Guide (new reference document: `autopsy/references/output-rendering.md`)
- Terminal output format for all 4 states (Discovery, Review, Synthesis, Complete)
- Final summary format (replaces current double-line box format in full-review.md)
- Progress bar format for review agents
- Severity line format ("N critical . N high . N medium . N low")

### Consumes
- `commands/full-review.md` (Track 05 -- modifies terminal output sections)
- `commands/maintain-docs.md` (Track 06 -- modifies summary output section)
- `.autopsy/state.json` (Track 05 -- reads finding counts for severity line)
- `.autopsy/progress.md` (Track 05 -- modifies format for dense display)
- Finding format shared schema (interfaces.md -- severity emoji removal in terminal)
- `REVIEW_REPORT.md` (Track 04 -- critical-only terminal display, full report to file)

## Dependencies
- Track 05_orchestrator_command: COMPLETE. Contains the primary output to be overhauled (terminal progress, batch summaries, final summary). Must be modified.
- Track 06_secondary_commands: COMPLETE. Contains maintain-docs summary output. Must be modified.
- Track 03_review_agents: COMPLETE. Agent finding format uses emoji severity markers. Terminal display of findings changes; file format may change per DD-3.
- Track 04_synthesizer_agent: COMPLETE. REVIEW_REPORT.md format and terminal summary. Critical-only display in terminal.

<!-- END ARCHITECT CONTEXT -->

# Track 07: Output Rendering Overhaul

## What This Track Delivers

A complete replacement of the autopsy plugin's terminal output with a clean, dense CLI dashboard following Claude Code's native design language. Instead of verbose narration with emoji-heavy decorations, the plugin will show collapsed completed phases, an active spinner for in-progress work, progress bars for parallel review agents, color-coded severity counts on a single line, and a concise final summary showing only critical issues. All high/medium/low findings go exclusively to REVIEW_REPORT.md -- the terminal stays focused on what demands immediate attention.

## Scope

### IN
- Output Rendering Guide reference document (`autopsy/references/output-rendering.md`) capturing the full specification for all 4 output states
- Orchestrator command (`commands/full-review.md`) terminal output overhaul:
  - Phase collapsing: completed phases collapse to a single summary line
  - Braille dot spinner specification for active phases
  - Progress bars for review agents within each batch (format: `name bar% count`)
  - Severity line format: "N critical . N high . N medium . N low" (no emoji, color-coded)
  - Critical-only terminal display: only critical issues shown inline (3 lines max each)
  - Thin dividers (two-dash lines) replacing double-line box characters
  - 4 distinct output states: Discovery, Review, Synthesis, Complete
- Final summary format: collapsed phase lines, severity line, critical issues (3 lines each max), imperative next steps, reports section
- Maintain-docs command (`commands/maintain-docs.md`) summary output: same design language (thin dividers, no emoji, dense format)
- progress.md format update: dense, parseable, no emoji
- Cross-cutting v2: text-only severity labels for terminal output

### OUT
- Agent definition files (agents/*.md) internal logic -- agents still review code the same way
- Synthesizer deduplication/analysis logic -- only its output format is affected
- REVIEW_REPORT.md content structure (this track only changes what appears in terminal vs what goes to the file)
- Batch plan format (discovery output is unchanged)
- state.json schema (machine-readable format stays the same, only human-facing output changes)
- Compaction recovery logic (only the format of progress.md changes, recovery mechanism is untouched)
- Spinner/progress bar runtime implementation (Claude Code renders Markdown instructions -- this track defines the output text format, not a UI framework)

## Key Design Decisions

These should be resolved with the developer during spec generation:

1. **Spinner representation in command Markdown:** Claude Code commands are Markdown instructions, not executable code. How should the braille spinner (80ms rotation) be specified?
   Trade-off: Literal instruction ("print each frame at 80ms interval") vs acknowledging that Claude controls output timing and specifying the spinner character set as a preference the LLM should follow when printing status

2. **Progress bar granularity:** Per-agent progress bars require knowing how many files each agent has processed. How does the orchestrator track per-agent file progress when agents run as parallel foreground tasks?
   Trade-off: Agents could write incremental progress to a file (adds I/O but enables real bars) vs the orchestrator estimates progress by elapsed time (simpler but inaccurate) vs progress bars only show batch-level not agent-level progress

3. **Emoji in persisted files:** The finding format in agent output files (batch-*/\*.md) and REVIEW_REPORT.md currently uses emoji severity markers. Should the emoji-free design extend to persisted files or only to terminal output?
   Trade-off: Removing from files too gives full consistency and cleaner grep/parsing; keeping emoji in files preserves visual scanning in Markdown readers (GitHub, VS Code preview)

4. **Critical-only terminal threshold:** The spec says only critical issues display in terminal. Should there be a fallback if zero criticals exist (e.g., show top 3 high-severity instead)?
   Trade-off: Strict critical-only is cleanest but may result in an empty findings section for well-maintained codebases; fallback keeps the summary informative but adds conditional logic

5. **Color specification method:** Claude Code terminal supports ANSI color codes. How should colors be specified in the command Markdown?
   Trade-off: Explicit ANSI escape sequences in the template output (guaranteed colors but ugly in the Markdown) vs named color instructions ("print in red") letting Claude choose the implementation vs no color at all (simplest, works everywhere)

6. **Backward compatibility of progress.md:** The compaction recovery logic re-reads progress.md. Changing its format could break recovery for in-flight reviews started before this track's changes are applied.
   Trade-off: Add a version marker to progress.md and handle both formats vs only apply new format (breaking change, acceptable since reviews are short-lived) vs keep progress.md format unchanged and only change terminal rendering of it

7. **Maintain-docs output parity:** Should maintain-docs adopt the exact same design language (thin dividers, no emoji, severity counts) or a simpler variant since it has less information to display?
   Trade-off: Full parity gives a unified experience; simpler variant is faster to implement and maintain-docs has different information needs (file counts, not severity counts)

## Architectural Notes

- This track modifies files owned by completed tracks (05 and 06). The implementation must preserve all non-output logic: compaction recovery, error handling, agent launching, batch processing. Only the text printed to terminal and the format of progress.md change.
- The finding format shared schema in interfaces.md is consumed by both review agents and the synthesizer. If DD-3 decides to remove emoji from persisted files, this is an interface change that requires updating the schema in interfaces.md and modifying all 5 review agent definitions plus the synthesizer. This is the largest scope risk in the track.
- Claude Code commands are Markdown instructions interpreted by the LLM. There is no "runtime" for spinners or progress bars -- the output format is a strong suggestion to the LLM about what to print. The design should specify exact output strings that the LLM copies verbatim.
- The critical-only terminal display means the synthesizer (or orchestrator) must filter findings by severity before printing. Currently the orchestrator counts findings via grep but does not read finding content. Showing critical issue summaries in the terminal may require the orchestrator to read a small amount of finding data -- this slightly increases context pressure.
- The output rendering guide should be a reference document that both full-review.md and maintain-docs.md can reference, avoiding duplication of output format specifications across command files.

## Complexity: L
## Estimated Phases: ~4
