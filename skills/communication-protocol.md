---
name: communication-protocol
description: >
  Defines the three mandatory structured formats for all agent-to-agent
  communication: HANDOFF (orchestrator to specialist), ARTIFACT (specialist to
  orchestrator), and HUDDLE (multi-agent discussion). Formats use XML tags for
  unambiguous field parsing. Use on every task — prose handoffs and freeform
  returns are not acceptable substitutes. Covers field definitions, the Detail
  field for summary vs full ARTIFACTs, HUDDLE venue and contribution rules, and
  the routing restrictions that keep all flow through the orchestrator.
  Required by all agents on every task.
---

# Skill: communication-protocol

All agent-to-agent communication in this team uses one of three structured formats. Prose handoffs and freeform returns are not acceptable — they cannot be evaluated, parsed, or routed reliably.

Formats use XML tags. XML tags eliminate field boundary ambiguity when content spans multiple lines or contains special characters.

---

## HANDOFF — orchestrator to specialist

Used when the orchestrator assigns a bounded subtask to one specialist.

```xml
<handoff>
  <task-id>TASK-{NNN}</task-id>
  <to>{agent-name}</to>
  <ticket>{TICKET-ID or "none"}</ticket>
  <goal>{one sentence — what to produce}</goal>
  <inputs>{comma-separated files, URLs, or prior artifact IDs to read}</inputs>
  <expected-output>{exact format — findings list, OpenAPI spec, test suite, migration file, etc.}</expected-output>
  <in-scope>{what this specialist should address}</in-scope>
  <out-of-scope>{what to explicitly ignore}</out-of-scope>
  <related-nodes>{comma-separated graph node IDs or "none"}</related-nodes>
  <detail>summary | full</detail>
  <file-budget>{number}</file-budget>
  <model-hint>haiku | sonnet | opus | omit to use default</model-hint>
</handoff>
```

**Detail field:** `summary` = specialist returns key findings only (status, CRITICAL/HIGH items, blockers). `full` = specialist returns complete verbose output with full evidence blocks.

**Default: all HANDOFFs use `<detail>summary</detail>`.** The orchestrator upgrades to `<detail>full</detail>` only when:
1. A CRITICAL finding requires full evidence to evaluate (e.g. a security vulnerability where the exploit path must be traced).
2. The task is BLOCKED and the root cause is unclear from summary information alone.
3. The user explicitly requests full output.

Summary artifacts are 60–70% smaller. Full artifacts are a deliberate escalation, not the default.

**Model-hint field (optional):** Guides model selection for this task. Omit to use the session default.
- `haiku` — orientation lookups, documentation tasks, single-file reviews, simple formatting
- `sonnet` — standard implementation, multi-file reviews, test writing, debugging
- `opus` — security threat modeling, architecture decisions, complex multi-domain analysis, TDD sessions on novel patterns

Routing hint defaults (orchestrator guidance):
| Task type | Suggested model |
|-----------|----------------|
| code-archaeologist orientation | haiku |
| tech-writer documentation | haiku |
| Single-file code review | haiku |
| Standard implementation | sonnet |
| Multi-file review | sonnet |
| TDD session | sonnet |
| Security threat modeling | opus |
| Architecture / ADR | opus |
| Complex multi-domain /ship | opus for design stage, sonnet for implementation |

**File-budget field:** Limits how many files the receiving specialist may read. Default budgets by model hint:
- `haiku` tasks: 5 files
- `sonnet` tasks: 15 files
- `opus` tasks: 25 files

Rules for specialists receiving a HANDOFF with a `<file-budget>`:
1. Before reading any files, declare the list of files you intend to read and why.
2. Stay within the budget. If the task genuinely requires more files, return `PARTIAL` with the list of additional files needed and why — do not silently exceed the budget.
3. Prioritise files by relevance: config/schema/entry-point files first; implementation files second; test files only if the task requires test coverage analysis.

**Rules:**
- Every HANDOFF has exactly one recipient.
- `<goal>` is one sentence. If it requires more, decompose further.
- `<expected-output>` must be specific enough that a pass/fail evaluation is unambiguous.
- Specialists do not receive each other's HANDOFFs — all flow goes through the orchestrator.

---

## ARTIFACT — specialist to orchestrator

Used when a specialist returns completed work to the orchestrator.

```xml
<artifact>
  <task-id>TASK-{NNN}</task-id>
  <from>{agent-name}</from>
  <status>COMPLETE | PARTIAL | BLOCKED</status>
  <summary>{one sentence — what was produced or what is blocking}</summary>
  <content>
{domain-specific content — findings, code, spec, migration, test suite, etc.}
  </content>
  <blockers>{if PARTIAL or BLOCKED — exactly what is needed to complete; "none" if COMPLETE}</blockers>
  <next-agent>{recommended next agent and why; "none" otherwise}</next-agent>
  <graph-nodes>{new node IDs added to ~/.supppeeerrr-harnes/agent-memory/graph.md; "none" if nothing added}</graph-nodes>
</artifact>
```

**Rules:**
- `<status>PARTIAL</status>` means work is incomplete but progress was made — specify what is done and what remains in `<blockers>`.
- `<status>BLOCKED</status>` means work cannot proceed — the blocker must be specific and actionable.
- `<next-agent>` is a recommendation only — the orchestrator decides routing.
- The content inside `<content>` is the actual deliverable. All other tags are metadata for the orchestrator.

---

## HUDDLE — orchestrator convenes multi-agent discussion

Used when a decision benefits from multiple specialist perspectives before a call is made. The orchestrator opens the question, named specialists each contribute a structured position, and the orchestrator closes with a decision.

**When to convene a huddle:**
- A technical decision has significant trade-offs across two or more domains (e.g. security vs. performance, API design vs. database schema).
- A specialist conflict cannot be resolved by the orchestrator alone.
- A novel architecture decision with no prior ADR.

**Huddle venue:** `~/.supppeeerrr-harnes/agent-memory/scratchpad/supervised/HUDDLE-{NNN}.md` — always monitored by the orchestrator.

```xml
<huddle>
  <huddle-id>HUDDLE-{NNN}</huddle-id>
  <question>{the specific decision or question — one sentence}</question>
  <participants>{agent-1}, {agent-2}, {agent-3}</participants>
  <context>{relevant files, prior artifacts, or graph nodes participants should read first}</context>

  <contribution agent="{agent-name}" time="{HH:MM}">
    <position>{clear stance — what you recommend}</position>
    <rationale>{why — evidence, standards, prior art}</rationale>
    <risk-if-wrong>{what breaks if this position is incorrect}</risk-if-wrong>
    <recommendation>{concrete next action}</recommendation>
  </contribution>

  <decision time="{HH:MM}">
    <synthesis>{points of agreement across positions}</synthesis>
    <conflicts>{points of disagreement and how they were resolved}</conflicts>
    <call>{the decision}</call>
    <rationale>{why this position over the alternatives}</rationale>
    <adr-needed>yes — route to architect + tech-writer | no</adr-needed>
  </decision>
</huddle>
```

**Rules:**
- The orchestrator writes the `<huddle>` header and opens the question.
- Each participant writes exactly one `<contribution>` block unless the orchestrator explicitly asks for a follow-up.
- Participants do not respond to each other directly — they respond to the question. Cross-cutting comments go to the orchestrator.
- The orchestrator closes every huddle with a `<decision>` block. An open huddle is a blocked task.
- If `<adr-needed>yes</adr-needed>`, route the decision to `architect` and `tech-writer` after closing.
