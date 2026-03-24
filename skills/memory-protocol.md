---
name: memory-protocol
description: >
  Standard protocol for reading and writing agent memory across all sessions.
  Use at the start of every task (to read the episodic index for prior context)
  and at the end of every task (to write an episodic entry and update the
  relationship graph). Also covers the Task Continuity Protocol — intent
  anchors, progress checkpoints, decision logs, and context recovery snapshots
  for complex multi-specialist tasks. Required by all agents on every task.
---

# Skill: memory-protocol

Before any memory operation, check `~/.supppeeerrr-harnes/agent-memory/config.md`. If `MEMORY_ENABLED: false`, skip everything in this skill.

---

## On task start

1. Read `~/.supppeeerrr-harnes/agent-memory/episodic.md` — scan the **Index** table only.
2. Check for rows matching the current ticket number or related domain.
3. If a relevant row exists, read that full entry before proceeding.
4. Do not read full entries speculatively — only when the index confirms relevance.

**Why:** Prior work on the same ticket or domain may change what you need to do, prevent duplicate work, or reveal known pitfalls.

---

## During complex tasks

Create an individual scratchpad at:
```
~/.supppeeerrr-harnes/agent-memory/scratchpad/individual/{your-agent-name}-{YYYYMMDD-HHMM}.md
```

Use it for working notes, hypothesis tracking, and intermediate state. No other agent reads this file. Delete or archive it when the task is complete.

**Complex task signals:** multiple files to read, a multi-step investigation, a ReAct loop with more than 2 iterations, or any task expected to take more than one response.

---

## On task complete

Write one entry to `~/.supppeeerrr-harnes/agent-memory/episodic.md`:

**Step 1 — Add an index row** at the top of the Index table (newest first):
```
| {NNN} | {YYYY-MM-DD} | {agent-name} | {TICKET-ID} | {category} | {one-line summary} | ✅/⚠️/❌ |
```

**Step 2 — Append a full entry** below the `---` separator:
```
### Entry #{NNN}
**Agent:** {agent-name}
**Ticket:** {TICKET-ID or "none"}
**Date:** {YYYY-MM-DD}
**Category:** {category}
**What:** {one sentence — the specific action taken}
**Why:** {one sentence — the reason}
**Outcome:** {one sentence — what changed. Include file paths.}
**Artifacts:** {comma-separated files created or modified}
**Graph links:** {node IDs added to graph.md, or "none"}
```

**Categories:** `feature` | `bug-fix` | `security-review` | `refactor` | `architecture` | `migration` | `test` | `incident` | `spike` | `review` | `exploration`

**Outcome icons:** ✅ complete | ⚠️ complete with findings or warnings | ❌ blocked or failed

**Rules:**
- Entries are immutable. To correct a prior entry, add a new one with `**Corrects:** #NNN`.
- Entry numbers are sequential and never reused.

**When outcome is BLOCKED or PARTIAL (⚠️ or ❌):** add one extra field to the entry:
```
**Failure mode:** {the specific cause — not the symptom. One sentence. e.g. "Missing auth middleware on new endpoint" not "security review failed"}
```

Leave the failure mode in the entry permanently. When a follow-up task resolves it, the resolving entry references the original: `**Resolves failure:** #NNN`.

---

## Failure context at task start

When the episodic index contains prior entries for the same domain or ticket with ⚠️ or ❌ outcomes — read those full entries before proceeding. The `**Failure mode:**` field tells you exactly what went wrong before. Include it in the HANDOFF `<inputs>` field:

```
Prior failure in this area: [failure mode from entry #NNN, date]
```

This step is only triggered when the index scan reveals a relevant prior failure. Do not read all BLOCKED entries speculatively.

---

## Graph writes

When your output **resolves**, **blocks**, **depends on**, **surfaces**, or **contradicts** an existing node — add the relationship to `~/.supppeeerrr-harnes/agent-memory/graph.md`.

**Step 1 — Add a node** (if your task is not already there):
```
| {TYPE}-{NNN} | {node-type} | {agent-name} | {one-line description} | {YYYY-MM-DD} |
```

**Step 2 — Add edges:**
```
| {FROM-ID} | {relation} | {TO-ID} | {optional note} |
```

**Node types:** `task` | `finding` | `decision` | `ticket` | `incident` | `spike-result`

**Edge types:** `implements` | `resolves` | `surfaces` | `blocks` | `depends-on` | `contradicts` | `causes` | `related-to` | `corrects`

**Rule:** Add the node before adding edges that reference it. Node IDs are permanent.

---

## Task Continuity Protocol

This protocol maintains direction during a single long-running task — preventing drift, decision amnesia, and repeated work within a context window.

Apply this protocol on any task where the orchestrator creates a supervised scratchpad (multi-specialist fan-out, `/ship`, `/audit`, or any complex multi-step task).

---

### Intent Anchor

Write an intent anchor to the supervised scratchpad immediately after task decomposition — before any specialist is invoked.

```
## INTENT ANCHOR — [TASK-ID]
Goal: [the original goal in 1-3 sentences — the north star]
Key constraints: [non-negotiables, scope limits, what's explicitly out of scope]
Success criteria: [what done looks like — specific and measurable]
Written: [YYYY-MM-DD HH:MM]
```

**Rule:** Every agent, before producing a final ARTIFACT on a task that has an intent anchor, reads the intent anchor and verifies their output serves it. If it does not, they note the drift in their ARTIFACT.

---

### Progress Checkpoint

The orchestrator writes a checkpoint after each major phase completes: after fan-out, after evaluation, after synthesis draft.

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

**Rule:** A checkpoint supersedes the previous one. Always append — never delete — but mark superseded checkpoints with `[SUPERSEDED by Phase N+1]` on the header line.

---

### Decision Log

Log key decisions immediately when made — not in the next checkpoint.

```
## DECISION — [YYYY-MM-DD HH:MM]
Decision: [what was decided]
Rationale: [why]
Impact: [what this affects going forward]
```

**Rule:** Only log decisions that would cause drift or rework if forgotten. Do not log implementation details. Architectural, scope, and direction decisions are always logged.

---

### Context Recovery Snapshot

When the orchestrator detects context approaching 600,000 tokens, write a recovery snapshot before continuing.

```
## RECOVERY SNAPSHOT — [TASK-ID]
Context pressure: [token count at time of snapshot]
State at snapshot: [condensed summary of everything relevant — goal, decisions, what is done, what remains]
Handoff instructions: [exactly how a new context window would resume this task]
Critical context: [the 3-5 facts that must not be lost]
Written: [YYYY-MM-DD HH:MM]
```

**Rule:** When a new context window starts and finds a recovery snapshot, read only the snapshot — not the full scratchpad — to resume efficiently.

---

### Direction Verification

Before any agent returns a final ARTIFACT on a task with an intent anchor, the agent reads the intent anchor and appends one line to their ARTIFACT:

```
Direction check: [ALIGNED | PARTIAL | DRIFT] — [one sentence explaining]
```

- **ALIGNED** — output fully serves the stated goal and constraints.
- **PARTIAL** — output serves the goal but with noted gaps or scope deviations.
- **DRIFT** — output does not serve the intent anchor. This triggers a Human-in-the-Loop gate before the ARTIFACT is accepted by the orchestrator.
