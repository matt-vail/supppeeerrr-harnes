---
name: human-in-the-loop
description: >
  Hard-stop gate protocol for human approval. Use before any irreversible
  action, production environment change, file or data deletion, breaking API
  change, CRITICAL security finding, architecture change, unresolved specialist
  conflict, scope expansion, or data migration. Defines the PAUSE gate format,
  the clarification gate format, how to document the human's decision, and what
  does NOT constitute a gate trigger. Required reading for every agent on every
  task — gates must fire deterministically, not based on keyword matching.
---

# Skill: human-in-the-loop

A Human-in-the-Loop gate is a hard stop — not a suggestion. When a trigger condition is met, pause and present to the human before taking any further action. Do not proceed until the human explicitly confirms.

---

## Universal trigger conditions

Pause before any action that meets one or more of these conditions:

| Trigger | Why |
|---------|-----|
| Action touches more than 5 files | Blast radius requires human judgement |
| Any file or data deletion | Irreversible — always confirm |
| Breaking API or interface change | Downstream consumers affected |
| CRITICAL security finding | Risk too high to proceed without sign-off |
| Architecture change | Long-term structural consequences |
| Any production environment action | Live system — mistakes are immediate |
| Unresolved specialist conflict after one round | Human is the tiebreaker |
| Task scope expands mid-execution | Re-confirm the new scope before continuing |
| Migration modifying or deleting existing data | Data loss risk |
| Deployment without staging validation | Bypasses the safety gate |

Individual agents may define additional domain-specific triggers. Those are additive — the universal triggers above always apply.

---

## Gate format

When a trigger is met, present this to the human before doing anything else:

```
PAUSE — Human review required
Trigger:  [which condition above was met]
Context:  [what the agent was about to do]
Risk:     [what could go wrong if this proceeds incorrectly]
Options:  [what the human can choose — e.g. proceed / adjust / cancel]
Question: [the specific decision needed to continue]
```

Do not include partial work, code, or recommendations in the gate — the gate is purely a decision request. Findings and artifacts go in the ARTIFACT once the human responds.

---

## After human responds

**If the human confirms:** proceed exactly as described in the gate. Do not expand scope.

**If the human adjusts:** re-read the adjusted scope before proceeding. If the adjustment changes what specialists need to do, issue new HANDOFFs.

**If the human cancels:** return an ARTIFACT with `Status: BLOCKED`, note the human cancelled the action, and record the reason in the episodic log.

---

## Documentation requirement

Every gate that fires must be documented. After the human responds (regardless of outcome), write to `~/.supppeeerrr-harnes/agent-memory/episodic.md`:

- What the trigger was
- What was proposed
- What the human decided
- Any scope change that resulted

Undocumented gate decisions cannot be audited or re-evaluated. The gate without the record is worthless.

---

## Clarity and feedback gates

Not all human gates are about risk. Pause and request human input when:

| Trigger | Format |
|---------|--------|
| A requirement is genuinely unclear and proceeding on an assumption could waste significant effort | State the assumption, ask the one question that resolves it |
| The user has asked for feedback or confirmation before continuing | Present what you have so far and ask specifically what to confirm |
| Two valid interpretations exist and the choice meaningfully changes the output | Present both options with trade-offs, ask the human to choose |
| Mid-task the scope feels larger than what was originally requested | Restate what was asked vs what you are finding, ask whether to proceed |

**Format for clarity gates:**
```
PAUSE — Clarification needed
Question: [the specific thing that is unclear]
Context:  [why this matters — what changes depending on the answer]
Options:  [if there are discrete choices — list them]
```

---

## What is NOT a gate trigger

- Routine file reads and writes during normal task execution.
- Creating scratchpad files.
- Writing episodic memory entries.
- Minor ambiguity that can be resolved with a reasonable default assumption — state the assumption and proceed.
- Escalating a blocked task to the orchestrator (that is internal routing, not a human gate).
