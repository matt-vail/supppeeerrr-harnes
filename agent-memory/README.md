# Agent Memory — Format Specification

Authoritative format spec for all agent memory. Read this before writing to any memory file.

## ⚠️ Check config first — always

Before any memory operation (read or write), check `agent-memory/config.md`.

- If `MEMORY_ENABLED: true` — proceed normally.
- If `MEMORY_ENABLED: false` — skip all memory operations. Do not read episodic, do not write entries, do not create scratchpads, do not update the graph. Existing files are preserved but must not be accessed.

---

## Episodic memory (`episodic.md`)

### Index row format
One row per task, newest first:

| # | Date | Agent | Ticket | Category | Summary | Outcome |
|---|------|-------|--------|----------|---------|---------|
| 001 | YYYY-MM-DD | agent-name | TICKET-ID | category | One-line summary | ✅ |

**Categories:** `feature` | `bug-fix` | `security-review` | `refactor` | `architecture` | `migration` | `test` | `incident` | `spike` | `review` | `exploration`

**Outcome icons:** ✅ complete | ⚠️ complete with findings or warnings | ❌ blocked or failed

### Full entry format
Appended below the `---` separator, newest at bottom:

```
### Entry #NNN
**Agent:** {agent-name}
**Ticket:** {TICKET-ID or "none"}
**Date:** YYYY-MM-DD
**Category:** {category}
**What:** One sentence — the specific action taken.
**Why:** One sentence — the reason this was done.
**Outcome:** One sentence — what changed as a result. Include file paths.
**Artifacts:** Comma-separated list of files created or modified.
**Graph links:** Comma-separated node IDs if edges were written to graph.md.
```

### Rules
- Write the index row first, then the full entry.
- Entries are immutable once written. To correct: add a new entry with `**Corrects:** #NNN`.
- Read ONLY the Index table section (content before the `---` separator). Do not read full entries unless the index row indicates a directly relevant prior task.
- If the Index table contains more than 50 rows, read only the 20 most recent rows plus any rows matching the current ticket ID or domain. Do not read the full index when it exceeds this threshold.
- Entry numbers are sequential and never reused.

---

## Graph (`graph.md`)

### Node format

| ID | Type | Agent | Summary | Date |
|----|------|-------|---------|------|
| TASK-001 | task | backend-engineer | One-line description | YYYY-MM-DD |

**Node types:** `task` | `finding` | `decision` | `ticket` | `incident` | `spike-result`

**ID format:** `{TYPE}-{NNN}` — e.g. `TASK-001`, `FINDING-003`, `ADR-005`

### Edge format

| From | Relation | To | Note |
|------|----------|----|------|
| TASK-001 | resolves | FINDING-003 | |

**Edge types:** `implements` | `resolves` | `surfaces` | `blocks` | `depends-on` | `contradicts` | `causes` | `related-to` | `corrects`

### Rules
- Add yourself as a node before adding edges that involve your work.
- One edge per row. Add a Note only when the relationship is non-obvious.
- Node IDs are permanent — never reuse or rename an ID.

---

## Scratchpads (`scratchpad/`)

### Individual — `scratchpad/individual/{agent-name}-{YYYYMMDD-HHMM}.md`
Private to one agent instance for one task. No format required — working notes only. Delete or archive on task complete.

### Team — `scratchpad/team/{agent-type}.md`
Created by orchestrator when fanning out multiple instances of the same agent type. All instances read and write. Orchestrator passive.

```
# Team Scratchpad — {agent-type} — {TASK-ID}
Created: {timestamp} | Agents: {list}

## Shared findings
{agents write here}

## Consensus decisions
{agents record agreed directions here}
```

### Supervised — `scratchpad/supervised/{TASK-ID}.md`
Created and actively monitored by orchestrator for complex tasks. **Every write to this directory is echoed to the terminal by the PostToolUse hook.** Orchestrator reviews after every write, corrects drift, and updates Current Direction.

```
# Supervised Scratchpad — {TASK-ID}
> ORCHESTRATOR SUPERVISED | Initialized: {timestamp} | Status: ACTIVE

## Mission (read-only for workers — set by orchestrator)
Objective: {exact task, one sentence}
Agents: {list}
Success criteria: {measurable done state}

## Current Direction (orchestrator-authoritative — workers align to this)
{orchestrator writes the live agreed direction here}

---

## Workspace

[{agent-name} | {HH:MM}]
{entry}

[ORCHESTRATOR | {HH:MM}] ⚠️ CORRECTION
Deleted: {what was removed and why}
Redirect: {new direction}. Current Direction updated above.
```

### Rules
- Individual scratchpads: created and cleaned up by the agent.
- Team and supervised scratchpads: created and closed by orchestrator only.
- Never write credentials, tokens, or PII to any scratchpad.
- Supervised scratchpad status changes to `COMPLETE` when orchestrator closes it — archive summary to episodic.md.

---

## Task Continuity Protocol formats

These memory types are written into supervised scratchpads by the orchestrator during long-running tasks. Full protocol rules are in `skills/memory-protocol.md`. The formats below are the authoritative structural specifications.

### Intent Anchor

Written once at the start of a complex task, before any specialist is invoked. Read-only after writing.

```
## INTENT ANCHOR — [TASK-ID]
Goal: [the original goal in 1-3 sentences — the north star]
Key constraints: [non-negotiables, scope limits, what's explicitly out of scope]
Success criteria: [what done looks like — specific and measurable]
Written: [YYYY-MM-DD HH:MM]
```

### Progress Checkpoint

Written by the orchestrator after each major phase. Supersedes the previous checkpoint — append only, mark superseded headers with `[SUPERSEDED by Phase N+1]`.

```
## CHECKPOINT — [TASK-ID] — Phase [N]
Completed:
  - [bullet list of what is done]
Decisions made: [key decisions that would cause drift if forgotten]
In progress: [what is currently being worked]
Blocked: [anything stuck and why — "none" if not applicable]
Next: [the immediate next step]
Updated: [YYYY-MM-DD HH:MM]
```

### Decision Log

Written immediately when a key architectural, scope, or direction decision is made — not deferred to the next checkpoint.

```
## DECISION — [YYYY-MM-DD HH:MM]
Decision: [what was decided]
Rationale: [why]
Impact: [what this affects going forward]
```

### Context Recovery Snapshot

Written when the orchestrator detects context approaching 600,000 tokens. When a new context window finds this, it reads only the snapshot to resume — not the full scratchpad.

```
## RECOVERY SNAPSHOT — [TASK-ID]
Context pressure: [token count at time of snapshot]
State at snapshot: [condensed summary of everything relevant — goal, decisions, what is done, what remains]
Handoff instructions: [exactly how a new context window would resume this task]
Critical context: [the 3-5 facts that must not be lost]
Written: [YYYY-MM-DD HH:MM]
```
