---
name: adr-format
description: >
  Architecture Decision Record template and numbering guidelines. Use when
  recording a technology choice, documenting why an alternative was rejected,
  capturing a constraint that shapes future decisions, or following up a
  Human-in-the-Loop gate on an architectural change. Produces an immutable ADR
  with Context, Decision, Rationale, Alternatives considered, Consequences, and
  Review trigger sections.
---

# Skill: adr-format

An Architecture Decision Record (ADR) is the permanent record of a significant technical decision — what was decided, why, and what was rejected. ADRs are immutable once accepted. To change a decision, supersede the ADR with a new one.

**Write an ADR when:**
- A technology, pattern, or architecture is chosen and it would be expensive to reverse.
- A significant option was considered and rejected — record why so it is not re-litigated.
- A constraint was accepted (legal, performance, compliance) that shapes future decisions.
- The Human-in-the-Loop gate fired on an architecture change.

**Do not write an ADR for:**
- Routine implementation choices (which library function to call, how to name a variable).
- Decisions that can be changed cheaply without cascading consequences.
- Decisions the team hasn't confirmed yet — ADRs record confirmed decisions, not proposals.

---

## ADR template

```markdown
# ADR-{NNN}: {Short title — what was decided}

**Date:** YYYY-MM-DD
**Status:** Proposed | Accepted | Superseded by ADR-{NNN}
**Deciders:** {names or roles who made this call}
**Supersedes:** ADR-{NNN} | N/A

---

## Context

{What is the situation or problem that requires a decision?
What constraints, requirements, or forces are in play?
Keep this factual — no opinions yet.}

## Decision

{What was decided? State it as a clear, active sentence.
"We will use X" not "X was considered."}

## Rationale

{Why this option over the alternatives?
What made this the right choice given the context?}

## Alternatives considered

| Option | Why rejected |
|--------|-------------|
| {Alt 1} | {Specific reason — not "it was worse"} |
| {Alt 2} | {Specific reason} |

## Consequences

**Positive:**
- {What this decision enables or improves}

**Negative / Trade-offs:**
- {What this decision costs, constrains, or makes harder}

**Risks:**
- {What could go wrong if the assumptions behind this decision are incorrect}

## Review trigger

{What condition would cause us to revisit this decision?
e.g. "If throughput exceeds 10k req/s", "If team grows beyond 10 engineers"}
```

---

## Numbering

ADRs are numbered sequentially: `ADR-001`, `ADR-002`, etc. Numbers are never reused. Store in `docs/adr/`.

## Status lifecycle

```
Proposed → Accepted → Superseded
```

- **Proposed:** drafted, not yet confirmed by decision-makers.
- **Accepted:** confirmed. The decision is in force.
- **Superseded:** replaced by a newer ADR. The old ADR is kept for history — never deleted.

## Graph entry

Every accepted ADR is a `decision` node in `agent-memory/graph.md`. Link it to the tasks it enables and any ADRs it supersedes.

```
| ADR-{NNN} | decision | architect | {title} | {date} |
```
