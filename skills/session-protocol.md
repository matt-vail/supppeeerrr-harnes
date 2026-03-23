---
name: session-protocol
description: >
  Session Resume pattern for long-running multi-session tasks. Defines how to
  write a lightweight end-of-session snapshot (150-250 tokens) to a supervised
  scratchpad so the next session can resume complex in-progress work without
  re-reading a full 5k-token scratchpad. Also defines stale scratchpad detection
  to prevent stale context from loading at session start. Used by the orchestrator
  on any task that spans multiple sessions.
---

# Skill: session-protocol

The Session Resume pattern bridges Claude Code sessions for long-running in-progress tasks. It produces a compact checkpoint — 150–250 tokens — that lets the next session orient and continue without re-reading a full scratchpad.

This skill applies to the orchestrator only. Specialists write to supervised scratchpads via the Task Continuity Protocol (see `memory-protocol`); the orchestrator owns the session boundary.

---

## When to write a session resume snapshot

Write a session resume snapshot to the supervised scratchpad at the end of ANY session where:

- A supervised scratchpad was active (multi-specialist task, `/ship`, `/audit`, or any complex multi-step work), AND
- The task is IN PROGRESS — not yet complete, AND
- There is a reasonable chance the session will be resumed in a future conversation.

Do NOT write a resume snapshot when:

- The task is complete — write a normal episodic entry instead.
- The task was a simple single-turn interaction with no scratchpad.
- All state is already captured in episodic memory and no resumption is needed.

---

## Session resume snapshot format

Append to the supervised scratchpad as the final entry:

```markdown
## SESSION-RESUME — [TASK-ID] — [YYYY-MM-DD HH:MM]
Command: [the /command that was running]
Profile: [--full | --fast | --hotfix | n/a]
Status: IN PROGRESS

Completed stages:
- [stage name]: [one-line outcome]
- [stage name]: [one-line outcome]

Current stage: [exact stage name]
Next action: [precise instruction — specific enough that a new session can act on it without reading the full scratchpad]

Key decisions made:
- [decision 1 — one sentence]
- [decision 2 — one sentence]

Open HANDOFFs: [none | list of pending specialist work with status]
Blockers: [none | what is blocking and why]

For full context: [path to supervised scratchpad file]
```

**Target size: 150–250 tokens.** The resume snapshot is a pointer and briefing, not a full record. The scratchpad is the authoritative record.

**Rules:**

- "Next action" must be specific enough to act on cold — no assumed context.
- List only decisions that would cause drift or rework if forgotten.
- Open HANDOFFs lists any specialist invocation that was dispatched but not yet returned an ARTIFACT, with the specialist name and what was requested.
- One SESSION-RESUME per session boundary. If a scratchpad already contains a SESSION-RESUME, supersede it by appending a new one — never edit the prior entry.

---

## How to resume from a session resume snapshot

When a new session starts and finds a supervised scratchpad that contains a SESSION-RESUME block:

1. Read the SESSION-RESUME block only — not the full scratchpad.
2. Determine if the task is still relevant: check whether it was superseded by other work (compare episodic index entries dated after the snapshot).
3. If resuming: read only the scratchpad sections referenced in "Next action" or "Open HANDOFFs" before proceeding.
4. Continue from "Next action" exactly as specified.

Do not re-read the full scratchpad unless the "Next action" instruction explicitly requires it. The snapshot is sufficient to resume in most cases.

---

## Stale scratchpad detection

A supervised scratchpad is stale when:

- Its most recent entry (SESSION-RESUME or CHECKPOINT) is **older than 30 days**, AND
- Its status is not explicitly marked ACTIVE by a SESSION-RESUME block written in the last 7 days.

**On detecting a stale scratchpad:**

- Do not load it into context.
- Note in the episodic memory scan: `Stale scratchpad found: [filename] — [date of last entry]. Skipping.`
- The `/agent-memory archive` command handles physical cleanup.

---

## Distinguishing active vs stale scratchpads

The file naming convention makes this unambiguous:

```
agent-memory/scratchpad/supervised/TASK-{NNN}.md               # active task scratchpad
agent-memory/scratchpad/supervised/HUDDLE-{NNN}.md             # active huddle scratchpad
agent-memory/scratchpad/supervised/INCIDENT-{YYYYMMDD-HHMM}.md # incident scratchpad
```

At session start, when scanning supervised scratchpads: only load a scratchpad if it contains a SESSION-RESUME block dated within the last 7 days OR if the orchestrator is actively working on that TASK-ID in the current session.

All other scratchpads are skipped. Do not scan their full contents — check only the date on the most recent SESSION-RESUME or CHECKPOINT header line.
