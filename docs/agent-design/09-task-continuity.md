# Task Continuity — Using Memory for Long-Context Direction

Reference document for anyone modifying or extending this plugin. Covers how the memory system maintains task direction across long-running sessions, context window pressure, and the boundaries between specialist agent invocations.

**Source material:** Agent memory system design (`agent-memory/README.md`), Claude limits research (2026-03-22), plugin peer review (2026-03-22).

---

## 1. The Core Problem

Long agentic tasks have three failure modes that all stem from the same root cause: the orchestrator loses track of where it is.

**Goal drift.** The orchestrator starts with a clear objective, fans out to four specialists, receives ARTIFACTs, synthesizes findings, and by the time it is writing the final output it is optimizing for completeness of the synthesis document rather than the user's original goal. The original goal is still in context — but diluted by thousands of tokens of intermediate work. The output is thorough and answers the wrong question.

**Decision amnesia.** The orchestrator makes a scope decision early in a task ("treat this as a backend-only change, do not touch the frontend") and two specialist fan-outs later, that decision is no longer salient. A specialist returns an ARTIFACT recommending frontend changes. The orchestrator, no longer holding the decision prominently, incorporates the recommendation. The original scope decision is violated without anyone noticing.

**Repeated work.** In multi-phase tasks, the orchestrator re-derives information it already established earlier in the session. It re-reads files it already processed, re-routes to specialists it already consulted, re-makes decisions it already recorded. The session is wasteful and the final output may contradict the earlier phase.

These failures share a common cause: important state — the original goal, key decisions, completed phases — is stored only in the context window. Context window state is subject to context rot (quality degradation as context depth increases), is lost when context pressure triggers compaction, and is not available to new context windows when a task must resume across sessions.

The solution is to treat the filesystem as the persistent brain and the context window as working memory. State that must survive context pressure goes to disk. This is the Task Continuity Protocol.

---

## 2. The Filesystem as Persistent Brain

The fundamental design principle of the memory system is:

- **Filesystem = persistent memory.** Anything written to disk survives context pressure, server-side compaction, and session boundaries. It is readable by any agent, in any context window, at any point in the future.
- **Context window = working memory.** Anything held only in context is ephemeral. It is subject to context rot, may be compacted away, and is gone when the session ends.

State that must survive goes to disk. The orchestrator does not hold the task goal in its head — it writes it to disk at task start and re-reads it when needed. It does not remember decisions from three phases ago — it logs them to disk immediately when made. It does not carry the full ARTIFACT from every specialist in working context — it externalizes to disk and keeps a compressed summary.

This principle is not aspirational — it is operational. Any agent behavior that depends on state surviving beyond the current context window must use the filesystem. Any state that is only in context should be treated as temporary.

---

## 3. The Task Continuity Hierarchy

The plugin uses five layers of persistent state, each serving a different continuity function.

### Layer 1: CLAUDE.md — Project North Star

**Purpose:** Project-level constants that never change within a session. The team structure, the orchestration pattern, the non-negotiables.

**Who reads it:** All agents at session start, automatically injected.

**What it contains:** Agent roster, agentic patterns in use, key directories, memory system overview, non-negotiable behaviors (Human-in-the-Loop gates, episodic entry immutability, supervised scratchpad surfacing).

**Continuity role:** Provides the fixed background against which all task-specific decisions are made. The orchestrator's routing logic, the specialist agents' identity, and the memory protocol are all grounded here. CLAUDE.md cannot drift within a session because it is always injected — it is not something an agent reads once and might forget.

### Layer 2: Episodic Memory — Cross-Session Continuity

**File:** `agent-memory/episodic.md`

**Purpose:** A record of what was done in prior sessions. Prevents repeating decisions that were already made, avoids mistakes that were already encountered, and provides the orchestrator with prior context before routing.

**Who reads it:** All agents at task start. Only the Index table — not full entries — unless the index indicates a directly relevant prior task.

**What it contains:** One index row per completed task (date, agent, ticket, category, one-line summary, outcome). Full entry detail below the separator, accessed only on demand.

**Continuity role:** Cross-session continuity. The episodic index answers: "Has this type of task been done before? Were there findings that should inform this run?"

**Size management:** When the index exceeds 50 rows, read only the 20 most recent rows plus any rows matching the current ticket ID or domain. Do not read the full index. The index grows with every task; without a size rule it becomes a context sink (at 200 entries: ~10,000 tokens consumed on every task start).

### Layer 3: Graph Memory — Relationships and Dependencies

**File:** `agent-memory/graph.md`

**Purpose:** Records nodes (tasks, findings, decisions, tickets) and edges (how they relate). Provides structured context for the orchestrator when assembling a complex task: which prior findings are relevant, which decisions constrain which areas, which tickets are related.

**Who reads it:** Orchestrator when assembling complex tasks or when graph links appear in episodic index rows. Specialists when their HANDOFF includes `Related nodes` fields.

**Continuity role:** Relationship continuity. The graph answers: "What decisions, findings, or prior tasks are structurally connected to the current work?"

### Layer 4: Supervised Scratchpad — Current Task State (the live layer)

**File:** `agent-memory/scratchpad/supervised/{TASK-ID}.md`

**Purpose:** The orchestrator's live working document for a complex task. Contains the intent anchor, progress checkpoints, decision log, and recovery snapshot. This is the primary continuity mechanism for any single long-running task.

**Who reads it:** Orchestrator reads it throughout the task. Specialist agents may be directed to read specific sections (e.g., the intent anchor) via their HANDOFF.

**Continuity role:** Within-task continuity. The supervised scratchpad answers: "What is this task trying to accomplish, what has been decided, and what has been completed?" Every write to this directory is echoed to the terminal by the PostToolUse hook — the orchestrator is always aware of its current state.

**Created and closed by:** Orchestrator only. Status transitions from ACTIVE to COMPLETE at task close; summary archived to episodic.md.

### Layer 5: Individual Scratchpad — Specialist Working Notes

**File:** `agent-memory/scratchpad/individual/{agent-name}-{YYYYMMDD-HHMM}.md`

**Purpose:** Private working notes for a single agent instance on a single task. Hypothesis generation, draft findings, intermediate calculations.

**Who reads it:** Only the agent that created it, during the same task.

**Continuity role:** Within-invocation working memory. Not a continuity mechanism for the task overall — individual scratchpads are cleaned up or archived at task complete. Their value is in giving a specialist a place to reason without loading everything into context simultaneously.

---

## 4. Intent Anchoring

The intent anchor is the first thing the orchestrator writes to the supervised scratchpad. It is written once, before any specialist is invoked, and is read-only after that point.

### Format

```
## INTENT ANCHOR — [TASK-ID]
Goal: [the original goal in 1-3 sentences — the north star]
Key constraints: [non-negotiables, scope limits, what's explicitly out of scope]
Success criteria: [what done looks like — specific and measurable]
Written: [YYYY-MM-DD HH:MM]
```

### When it is written

Immediately after the orchestrator has understood the task and before any orientation reads or specialist routing. The anchor represents the uncontaminated original goal — before any investigation has begun, before any specialist has offered scope-expanding recommendations, before any complexity has been discovered.

### When it is read

- By the orchestrator at the start of each major phase, before routing the next specialist
- By the orchestrator before writing the synthesis output, to verify alignment
- By specialist agents when the HANDOFF includes an explicit reference ("Align your ARTIFACT to the intent anchor in supervised/{TASK-ID}.md §INTENT ANCHOR")
- Whenever the orchestrator suspects goal drift — an anchor re-read is the first recovery action

### The anchor is immutable

Once written, the intent anchor is not edited. If the task scope legitimately changes (due to a Human-in-the-Loop decision or a material discovery), the change is recorded as a Decision Log entry, and the next Progress Checkpoint notes the scope change. The original anchor remains in place as a record of the original intent. This immutability prevents scope creep from accumulating invisibly through repeated anchor edits.

---

## 5. Progress Checkpointing

The orchestrator writes a progress checkpoint after each major phase of a complex task. A major phase is any milestone where: a specialist fan-out has completed, a Human-in-the-Loop gate has been passed, or a significant decision has been made that changes what comes next.

### Format

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

### Checkpoint supersession

Each checkpoint supersedes the prior one. Append new checkpoints below previous ones; mark superseded headers with `[SUPERSEDED by Phase N+1]`. The most recent checkpoint is always the authoritative state.

When the orchestrator re-reads the supervised scratchpad to orient itself, it reads the most recent checkpoint header, skips superseded blocks, and continues. The checkpoint history is preserved for audit purposes but does not require re-reading in full.

### What qualifies for the Completed field

A task is "completed" in a checkpoint only when its output has been evaluated and accepted by the evaluation gate. A specialist that returned an ARTIFACT is not completed; a specialist whose ARTIFACT passed evaluation and whose output has been incorporated into synthesis state is completed.

### Why checkpoints matter at the 400k threshold

When token usage crosses 400k, context rot begins to produce measurable quality degradation. At this depth, the orchestrator's recall of decisions made in Phase 1 is less reliable than it was at 50k. The checkpoint is the external record that compensates for this degradation. The orchestrator re-reads its most recent checkpoint before proceeding — this is a 200-token read that restores directional clarity regardless of context depth.

---

## 6. Decision Logging

Key decisions are logged to the supervised scratchpad immediately when made — not deferred to the next progress checkpoint.

### Format

```
## DECISION — [YYYY-MM-DD HH:MM]
Decision: [what was decided]
Rationale: [why]
Impact: [what this affects going forward]
```

### What qualifies as a key decision

**Log immediately:**
- Scope decisions (what is in scope, what is explicitly out of scope)
- Direction decisions (which approach to take when alternatives exist)
- Architectural decisions (a design choice that affects multiple specialists or phases)
- Constraint decisions (accepting a limitation that shapes subsequent work)
- Human-in-the-Loop gate outcomes (what the human confirmed or redirected)

**Do not log:**
- Implementation details decided within a single specialist's domain
- Tool and library choices that are reversible without cross-cutting impact
- Formatting and naming decisions
- Anything that only a single specialist needs to know and that goes in their HANDOFF

### Why immediate logging matters

The window between making a decision and logging it is the window during which it can be lost. An orchestrator that thinks "I'll record this in the next checkpoint" may fan out to two more specialists before the checkpoint is written. If context pressure compacts those intermediate turns, the decision is gone. Immediate logging closes this window.

---

## 7. Direction Verification

Agents should verify alignment with the original intent before returning final ARTIFACTs. This is not a full re-read of the scratchpad — it is a targeted check.

### How specialists verify direction

When a specialist's HANDOFF includes a reference to the intent anchor, the specialist's final self-check before returning the ARTIFACT includes:

**Direction check:**
- Re-read the `Goal` and `Success criteria` fields from the intent anchor.
- Confirm the ARTIFACT addresses them.
- Record the check result in the ARTIFACT's Summary field: `Direction: ALIGNED` / `Direction: PARTIAL` / `Direction: DRIFT`.

A `DRIFT` finding triggers immediate escalation — the specialist returns the ARTIFACT with the drift identified, rather than delivering output the orchestrator then has to route back.

### How the orchestrator verifies direction

Before writing the synthesis output, the orchestrator:

1. Re-reads the intent anchor.
2. Confirms the synthesis output addresses the `Goal` and meets the `Success criteria`.
3. If specialists have returned `Direction: PARTIAL` or `Direction: DRIFT`, addresses the gaps in synthesis framing before output.

This verification is not optional — it is the last gate before final output. A synthesis that is technically complete but directionally misaligned is a task failure.

---

## 8. Context Recovery Protocol

When token usage approaches 800k, the task is approaching the boundary where continued work risks either hitting the hard context limit (session failure) or producing output so degraded by context rot that it has no value. The recovery protocol prevents both.

### Trigger

The orchestrator detects token usage exceeding 800k via the `Token usage: N/1,000,000; N remaining` signal injected by the API after each tool call.

### Recovery snapshot format

```
## RECOVERY SNAPSHOT — [TASK-ID]
Context pressure: [token count at time of snapshot]
State at snapshot: [condensed summary of everything relevant — goal, decisions, what is done, what remains]
Handoff instructions: [exactly how a new context window would resume this task]
Critical context: [the 3-5 facts that must not be lost]
Written: [YYYY-MM-DD HH:MM]
```

### What the recovery snapshot contains

The snapshot is a self-contained document. A new context window that reads only the recovery snapshot — without reading anything else in the supervised scratchpad — must be able to resume the task correctly. This means:

- The original goal verbatim from the intent anchor
- All key decisions made, with rationale summaries
- Completed phases and their outcomes (not full ARTIFACTs — references to disk locations)
- The current in-progress phase and its state
- Blocked items and why
- The specific next action a resumed context should take
- Any critical facts that would not be obvious from reading the task description alone

### Human-in-the-Loop gate at 800k

Writing the recovery snapshot is followed immediately by a Human-in-the-Loop gate. The gate presents:

- What has been completed
- What remains
- The context constraint
- The recovery snapshot location
- A recommendation: continue in a new session reading from the recovery snapshot

The human confirms continuation or redirects. The gate is non-negotiable — the orchestrator does not continue past 800k without explicit human confirmation. Continuing past this threshold risks session failure at an unpredictable point, which would leave the task in an unrecoverable state without the snapshot.

### Resuming from a recovery snapshot

A new context window resuming from a snapshot:

1. Reads `agent-memory/config.md` — confirms memory enabled.
2. Reads the recovery snapshot from the supervised scratchpad.
3. Reads the episodic index (most recent rows only).
4. Does not re-read the full scratchpad history — the snapshot is the authoritative state summary.
5. Continues from the `Handoff instructions` field.

---

## 9. Memory Index Management

The episodic index is the plugin's primary unbounded growth vector. Without active management it becomes a context sink that taxes every task start across all agents.

### The 50-row threshold rule

When the Index table contains more than 50 rows:
- Read only the 20 most recent rows.
- Plus any rows matching the current ticket ID or domain.
- Do not read the full index when it exceeds this threshold.

This rule is encoded in `agent-memory/README.md` and must be applied by every agent. An agent that reads the full index at 200+ entries is consuming ~10,000 tokens before any actual work begins — a significant tax on every invocation.

### Category-filtered reads

When reading the index to find relevant prior work, agents should first read the most recent rows to establish whether the task is familiar, then scan only rows matching relevant categories. The `Category` field exists for this filtering. An orchestrator routing a `security-review` task does not need to scan `migration` or `spike` rows.

### Reading the Index table only

The episodic.md file has two sections: the Index table (above the `---` separator) and the Full Entries section (below). Agents read only the Index table section at task start. Full entries are read only when the index row indicates a directly relevant prior task (matching ticket ID, same domain, `⚠️` or `❌` outcome that requires investigation).

The comment at the top of `agent-memory/episodic.md` makes this explicit: "Read the Index table only. Scan for your ticket number or related domain. Read a full entry only if the index shows directly relevant prior work."

This distinction matters as the file grows. At 100 full entries, the file below the separator may be 20,000–30,000 tokens. Reading the full file at every task start would consume more context than the episodic system saves. The Index table, by contrast, stays compact — each row is approximately 25 words.

---

*Written: 2026-03-22. Applies to: all agents in this plugin. Review trigger: changes to the memory format spec, significant growth in episodic index (review the 50-row threshold when index reaches 100 entries), or observed goal drift in completed tasks.*
