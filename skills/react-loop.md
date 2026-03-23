---
name: react-loop
description: >
  ReAct reasoning pattern — Thought/Action/Observation cycle for iterative
  problem solving. Use when a task is ambiguous, uncertain, or requires
  validating an assumption before acting. Use for medium-complexity or above
  tasks that touch multiple files or systems. Also defines the hard situations
  recovery format — Situation/Thought/Action/Observation — used across all
  agents when the normal path is blocked.
---

# Skill: react-loop

ReAct (Reasoning + Acting) is the reasoning pattern used by every agent in this team when facing ambiguity, uncertainty, or a decision point. It prevents acting on unvalidated assumptions.

**Source:** Yao et al. 2022 — ReAct: Synergizing Reasoning and Acting in Language Models.

---

## The cycle

```
Thought  → What do I know? What am I assuming? What is the highest-risk unknown right now?
Action   → The smallest action that resolves the highest-risk unknown.
           Read a file. Run a command. Ask one targeted question. Request one artifact.
Observation → What did I actually find? Does it confirm or contradict my assumption?
              Update the model. Never carry a contradicted assumption forward.
```

Repeat until the task is complete or the unknown is resolved.

---

## Rules

- **One unknown at a time.** Identify the highest-risk assumption. Resolve it before moving to the next.
- **Actions must be minimal.** The action should be the smallest thing that resolves the current unknown — not a sweeping investigation.
- **Observations update the model.** If the observation contradicts the assumption, update before proceeding. Never continue as if the contradiction didn't happen.
- **Do not loop indefinitely.** If 3 full cycles have not resolved the unknown, name the block explicitly and either ask a targeted human question or escalate to the orchestrator.

---

## When to start a ReAct loop

- The task requirements are ambiguous.
- You are about to write code or make a change based on an assumption you haven't verified.
- A prior action produced an unexpected result.
- Two possible interpretations of a requirement exist and neither is clearly correct.
- You are in a hard situation (see below).
- The task is medium complexity or above — touches multiple files, spans multiple systems, or requires more than one specialist domain to complete correctly.

---

## Hard situations recovery format

When an agent encounters a hard situation — a scenario where the normal path is blocked — use this format:

```
**Situation:** [What went wrong or is blocked — be specific]
→ Thought:      [Why is this a problem? What is the risk of proceeding incorrectly?]
→ Action:       [The smallest action that resolves or works around the block]
→ Observation:  [What the action reveals — does it unblock or escalate?]
```

**Recovery path options:**
- **Unblocked:** continue the task with the new information.
- **Partially unblocked:** continue with a documented assumption; flag it in the ARTIFACT.
- **Still blocked:** escalate to orchestrator with the specific blocker stated. Do not guess past a hard block.

---

## Common hard situations by type

**Ambiguous requirements**
→ Action: Ask one targeted clarifying question. If from orchestrator: request a revised HANDOFF with the ambiguity resolved.

**Missing context (file doesn't exist, system unknown)**
→ Action: State what is missing and what you assumed in its absence. Proceed with the assumption documented.

**Conflicting signals (two files say different things)**
→ Action: Check git history for recency. Use the more recently modified version. Flag the conflict in the ARTIFACT.

**Technically impossible requirement**
→ Action: Stop. Do not attempt a workaround. Return an ARTIFACT with `Status: BLOCKED` and the specific constraint named.

**Pre-existing errors in unrelated code**
→ Action: Scope-check first. Did I introduce this? If no: note it as debt, do not fix it in this task.
