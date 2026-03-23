# Context Management — Design Decisions and Best Practices

Reference document for anyone modifying or extending this plugin. Covers the concrete model limits, how context accumulates in agentic sessions, and the techniques used to keep the orchestrator context lean enough to produce high-quality output across complex multi-specialist tasks.

**Source material:** Claude limits research (2026-03-22), plugin peer review (2026-03-22), Anthropic documentation (platform.claude.com, March 2026).

---

## 1. Claude Model Limits

Exact figures for the models this plugin runs on, as of March 2026:

| Model | Context window | Max output | Notes |
|-------|---------------|------------|-------|
| Claude Sonnet 4.6 | 1,000,000 tokens | 64,000 tokens | Current model running this plugin |
| Claude Opus 4.6 | 1,000,000 tokens | 128,000 tokens | Recommended for longest-horizon tasks |
| Claude Haiku 4.5 | 200,000 tokens | 64,000 tokens | Fastest; smaller context is the hard limit |

### What these numbers mean operationally

**The 1M context window is a ceiling, not a safe operating budget.** Anthropic explicitly documents "context rot": accuracy and recall degrade as context grows, even within the available window. The 1M limit tells you when the session will fail with a hard error; context rot tells you when the session will quietly produce worse output. For an orchestrator managing a multi-specialist fan-out, output quality begins degrading measurably well before any hard limit is reached.

**Sonnet 4.6 fails hard when exceeded.** Unlike older models that silently truncated input, Sonnet 4.6 returns a validation error when the context limit is reached. The session stops. There is no graceful degradation — the agent does not summarize what it managed to process. Planning for the limit is the only mitigation.

**The 64k output cap is meaningful for synthesis.** A full `/audit` merging security, performance, accessibility, and code-review findings across a large codebase can approach this cap. The orchestrator has no automatic chunking behavior — if the synthesis output exceeds 64k tokens, it truncates. Summary ARTIFACTs from specialists and selective synthesis coverage are the practical mitigations.

**Context awareness is available but must be directed.** After each tool call, Sonnet 4.6 receives: `Token usage: N/1,000,000; N remaining`. This runtime injection is provided by the API and is not an agent instruction. The model has the signal; without explicit guidance in the orchestrator instructions to act on it, the signal is noted but not acted upon.

---

## 2. How Context Fills in Agentic Sessions

Context accumulates linearly across turns. Every user message, assistant response, tool call, and tool result is preserved in full. Nothing is dropped unless server-side compaction is enabled. The only automatic exception is extended thinking tokens.

### The 13-step accumulation model

A full orchestrated task adds context in this sequence:

| Step | What accumulates | Approximate tokens (this plugin) |
|------|-----------------|----------------------------------|
| 1 | System prompt injection — CLAUDE.md + orchestrator.md | ~3,500 |
| 2 | Initial user message | ~100–500 |
| 3 | Memory read — episodic.md Index table | ~200 (grows over time) |
| 4 | Project orientation reads (package.json, README, etc.) | ~2,500 (5 files × ~500 avg) |
| 5 | HANDOFF blocks written to context (one per specialist) | ~150 per specialist |
| 6 | Subagent invocations — each produces tool call in parent context | overhead only; specialist context is isolated |
| 7 | ARTIFACT blocks returned from each specialist | ~1,000–4,000 per specialist |
| 8 | HUDDLE transcript if convened | ~1,000–3,000 |
| 9 | Evaluation pass — orchestrator reasoning over all artifacts | ~1,000–2,000 |
| 10 | Scratchpad write and read-back | ~500–1,500 |
| 11 | Synthesis output generation | ~2,000–8,000 |
| 12 | Episodic memory write | ~200 |
| 13 | Graph write | ~100 |

Steps 6–8 are the compounding surface. A four-specialist fan-out adds four HANDOFF blocks, four ARTIFACT blocks, and potentially a HUDDLE — all before synthesis begins.

### What persists vs. what is stripped

**Tool results persist.** When the orchestrator reads five project files in Step 4, all five file contents remain in context for the rest of the session. When a specialist returns an ARTIFACT via the Agent tool, that ARTIFACT persists in full. There is no automatic eviction of tool results.

**Extended thinking tokens are stripped automatically.** If the orchestrator uses adaptive thinking, thinking tokens from prior turns are stripped before the next turn. This is handled by the API and is the one exception to the additive accumulation rule. Extended thinking is counted as output tokens when generated but does not accumulate as input tokens in subsequent turns.

**Subagent internal reasoning does not appear in the parent context.** When the orchestrator spawns a specialist via the Agent tool, the specialist's internal reasoning, file reads, and intermediate steps all occur in the specialist's isolated context. Only the final ARTIFACT return reaches the orchestrator's context. This is the architecture's primary mechanism for preventing specialist work from overloading the orchestrator — but it means the ARTIFACT is the only content the orchestrator receives.

### Practical cost estimate for a full `/ship` run

| Component | Approx tokens |
|-----------|---------------|
| Orchestrator system prompt (orchestrator.md) | ~3,500 |
| Project orientation (5 files × ~500 tokens avg) | ~2,500 |
| Episodic memory index read (current size) | ~200 |
| HANDOFF blocks × 6 specialists | ~900 |
| Specialist ARTIFACTs × 6 (avg 3,000 tokens each) | ~18,000 |
| Intermediate reasoning and scratchpad | ~3,000 |
| Synthesis output | ~4,000 |
| **Total — moderate codebase** | **~32,000 tokens** |

This is comfortably within 1M but not trivial. On a large codebase where each specialist reads many source files before producing findings, ARTIFACT sizes grow proportionally. A 10,000-line codebase audit where each specialist reads relevant source files can push totals to 100,000–200,000 tokens — still within the window but entering the range where context rot produces measurable quality degradation.

---

## 3. The Compress-and-Externalize Pattern

The core technique for keeping orchestrator context lean is: after receiving each specialist ARTIFACT, extract a short summary to working context, write the full ARTIFACT to a scratchpad file on disk, and discard the verbose content from the orchestrator's active reasoning.

### How it works

After each specialist returns an ARTIFACT:

1. **Extract key findings.** In ~30 tokens: specialist name, status, number of findings, the two or three most critical items.
2. **Write the full ARTIFACT to disk.** Path: `agent-memory/scratchpad/supervised/{TASK-ID}.md` under the relevant specialist section.
3. **Discard the verbose content from working context.** The orchestrator proceeds with the 30-token summary. The full content is on disk and can be re-read if needed during synthesis.

### Why this matters: the fan-out token reduction

Without compress-and-externalize, a four-specialist fan-out looks like this in the orchestrator's context:

```
[security-engineer ARTIFACT — 3,200 tokens]
[performance-engineer ARTIFACT — 2,800 tokens]
[accessibility-pm ARTIFACT — 1,400 tokens]
[reviewer ARTIFACT — 3,100 tokens]
Total from ARTIFACTs alone: ~10,500 tokens
```

With compress-and-externalize:

```
security-engineer: COMPLETE, 4 findings (1 CRITICAL: SQL injection in /search endpoint)
performance-engineer: COMPLETE, 3 findings (N+1 query in user feed, est. 340ms overhead)
accessibility-pm: COMPLETE, 2 findings (missing form labels, colour contrast failure)
reviewer: COMPLETE, 6 findings (2 HIGH: missing error handling, no input validation)
Total in orchestrator working context: ~120 tokens
```

A reduction from ~10,500 to ~120 tokens in the orchestrator's working context. The full ARTIFACTs are on disk and available for synthesis. The orchestrator evaluates quality, identifies conflicts, and decides on synthesis coverage using the compressed summaries. It reads specific ARTIFACT sections from disk only when needed for the actual synthesis output.

### What a compressed ARTIFACT summary looks like

**Full ARTIFACT (as returned by specialist — 3,200 tokens):**

```
---
ARTIFACT
Task: TASK-042 — Security review of /api/v1/search endpoint
From: security-engineer
Status: COMPLETE
Summary: Four findings identified, one critical.

[Finding 1 — CRITICAL]
Level: CRITICAL
Asset at risk: User data, database integrity
Attack vector: Unsanitised query parameter `q` passed directly to SQL string interpolation
Finding: Line 47 of api/search.py — `f"SELECT * FROM items WHERE name LIKE '%{q}%'"` —
  is vulnerable to SQL injection. The `q` parameter accepts arbitrary string input
  with no sanitization or parameterisation. Confirmed exploitable with basic `' OR '1'='1`
  test input during review.
Fix: Replace with parameterised query: `cursor.execute("SELECT * FROM items WHERE
  name LIKE %s", (f"%{q}%",))`. Review all other SQL string interpolations in api/.
Verify: Run sqlmap against /api/v1/search with default payloads; confirm no data
  returned beyond normal results.

[Finding 2 — HIGH]
...
[Finding 3 — MEDIUM]
...
[Finding 4 — LOW]
...

Artifacts: none
Graph links: FINDING-007, FINDING-008, FINDING-009, FINDING-010
---
```

**Compressed summary in orchestrator working context (~30 tokens):**

```
security-engineer: COMPLETE — 4 findings (CRITICAL: SQL injection api/search.py:47;
HIGH: missing auth on /admin; MEDIUM: CORS misconfiguration; LOW: verbose errors).
Full ARTIFACT: supervised/TASK-042.md §security
```

The `§security` reference tells the orchestrator exactly where to read if it needs the full text during synthesis.

---

## 4. Context Budget Thresholds

The orchestrator receives token usage updates after each tool call. These updates should be acted upon. The following four-tier table defines the actions at each threshold:

| Usage | Tier | Action |
|-------|------|--------|
| Under 400,000 tokens | Green | Standard operation — full ARTIFACTs, no compression required |
| 400,000–600,000 tokens | Yellow | Request summary ARTIFACTs from pending specialists: "Return findings summary only — 10 key findings max, no full evidence blocks." Apply compress-and-externalize to ARTIFACTs already received. |
| 600,000–800,000 tokens | Orange | Do not start new specialist invocations. If a fan-out is already in progress, allow it to complete but request summary ARTIFACTs. Synthesize from what you have. Surface the context constraint explicitly in synthesis output: "Note: synthesis is based on summary ARTIFACTs due to context budget." |
| Over 800,000 tokens | Red — Human-in-the-Loop | Context is approaching limits. Write a recovery snapshot to disk. Surface to the human: what has been completed, what remains, the context constraint. Task may need to continue in a new session from the recovery snapshot. |

The threshold levels are calibrated for Sonnet 4.6's 1M window with headroom for synthesis output (64k cap) and margin against context rot. A task still running at 800k tokens of input has used 80% of the window before synthesis output begins.

---

## 5. The Skills Trigger Architecture Finding

This is the most consequential design finding from the March 2026 research. It explains why some plugin behaviors cannot be left to skill trigger activation.

### How the Anthropic Skills architecture actually works

The official architecture has three levels:

**Level 1 — Metadata (~100 tokens per skill, always loaded at startup).** The YAML frontmatter `name` and `description` fields are loaded into the system prompt at session start. This is automatic. The `description` field is the signal Claude uses to decide whether a skill is relevant to the current task.

**Level 2 — Instructions (under 5,000 tokens, loaded when triggered).** When Claude determines a skill is relevant based on the Level 1 description, it reads the SKILL.md file. Only then does the skill body enter context.

**Level 3 — Resources (loaded on demand via bash).** Bundled files — scripts, templates, reference docs — are never loaded into context until explicitly read.

### The `triggers` frontmatter key is not a documented Anthropic feature

This plugin's skill files use a custom `triggers` key with lists of keyword phrases. This field has no documented operational meaning in the Anthropic Skills architecture. The official activation mechanism is the `description` field — a single natural-language summary of what the skill does and when to use it. The `description` field is Level 1 metadata; trigger phrase lists are read only after Claude has already decided to read the skill file.

The consequence: there is no documented mechanism that performs keyword matching against trigger phrases before the skill file is loaded. Trigger phrases in the `triggers` key are body content, not pre-load filters.

### The difference between Level 1 (description) and trigger phrase lists

| | `description` field | `triggers` list |
|---|---|---|
| When loaded | At session startup — always | After Claude has already decided to load the skill |
| Function | Tells Claude whether this skill is relevant to the current task | Body content read after load decision is made |
| Reliability | Deterministic (always present as Level 1 metadata) | Probabilistic (depends on Claude's reasoning after load) |
| Correct use | "Use when X" — concise natural language | Not a documented activation mechanism |

### Why constitutional skills cannot rely on trigger activation

Three skills in this plugin define behaviors that must be active on every task:

- `communication-protocol.md` — defines HANDOFF/ARTIFACT format required for all inter-agent communication
- `memory-protocol.md` — defines required read/write operations at task start and task complete
- `human-in-the-loop.md` — defines risk gates that must fire without fail

These skills currently rely on trigger phrases for activation. The failure mode is silent omission: if the trigger phrases do not appear in the session context, the constitutional behaviors are not applied. An agent that does not trigger `communication-protocol.md` returns a prose response instead of a structured ARTIFACT, and the orchestrator's evaluation gate has no protocol for detecting this.

The correct architecture for constitutional skills is explicit, mandatory loading — either via `read` instructions in every agent file or via a shared base configuration that all agent definitions reference. Trigger-based activation is appropriate for on-demand domain skills (like `fastapi-standards.md`), not for behaviors that must be present on every invocation.

---

## 6. Lazy Orientation

**Do not front-load project files at the start of every task.** The orchestrator's Step 0 reads a set of orientation files before routing. On a simple, well-understood task in a familiar codebase, front-loading seven files that will not be needed adds thousands of tokens of context before any work begins — and those tokens remain in context for the entire session, contributing to context rot.

The correct approach is to read only what the current task requires:

- **Simple, single-domain task in a familiar codebase:** Read the routing-relevant file (e.g., `package.json` to confirm tech stack) and proceed. Skip files that are not relevant to this task.
- **Multi-domain or unfamiliar codebase:** Full orientation is justified. Delegate to `code-archaeologist` whose findings stay in the specialist's context, not the orchestrator's — and structure the HANDOFF to get a summary ARTIFACT rather than a full file dump.
- **Large codebase:** Delegate orientation to `code-archaeologist`. The archaeologist's internal reads (which may span dozens of files) do not appear in the orchestrator's context. Only the orientation summary ARTIFACT does.

The principle: the orchestrator's context is the most expensive real estate in the plugin. Load into it only what is needed for the current step.

---

## 7. Summary ARTIFACTs by Default

For routine tasks, the orchestrator should request compressed output from specialists by default rather than full verbose findings.

**Request a summary ARTIFACT when:**
- The task is a check or review where the orchestrator needs to know whether something passed, not the full evidence trail
- Token usage is above 400k
- The specialist's domain is well-understood and edge cases are not expected
- The output will be synthesized with several other specialists' outputs

**HANDOFF language for summary ARTIFACT:**
```
Expected output: Summary ARTIFACT — key findings only. Up to 10 findings maximum.
No full evidence blocks or extended code excerpts. Flag any CRITICAL or HIGH
findings clearly; group MEDIUM and LOW findings by category with counts.
```

**Request a full verbose ARTIFACT when:**
- The task is exploratory or the findings will be primary deliverables (e.g., a full security audit report going to a client)
- The specialist is being asked to produce implementation, not just findings
- A CRITICAL finding needs full evidence for the Human-in-the-Loop gate presentation
- Token usage is below 400k and the output will not be synthesized with many other specialists

The default for pipeline commands (`/ship`, `/audit`, `/review`) should be summary ARTIFACTs from each specialist, with the synthesis stage reading specific sections from disk for any finding that requires full evidence.

---

## 8. Evaluator-Optimizer Loop Cost

The Evaluator-Optimizer pattern runs when the orchestrator determines a specialist's ARTIFACT does not meet the HANDOFF's definition of done. Each refinement cycle is a HANDOFF back to the specialist and an ARTIFACT returned — another token cost in the orchestrator's context.

### How cycles compound

| Scenario | Tokens per cycle | 3-cycle total |
|----------|-----------------|---------------|
| reviewer on a 2,000-line codebase | ~3,000 (review ARTIFACT) + ~500 (HANDOFF) | ~10,500 |
| reviewer with full verbose output | ~5,000 + ~500 | ~16,500 |
| code-reviewer with Franky persona | ~4,000 + ~500 | ~13,500 |

For the reviewer agent specifically, each cycle also reloads the 370-line agent definition file (~5,500 tokens) into the specialist's fresh context. This does not appear in the orchestrator's context directly, but it delays the specialist's response as it reasons through instructions before addressing the actual findings.

### The cycle scope guard

On cycles 2 and 3, the orchestrator should not request a full re-review of the complete artifact. Instead, return only the specific unresolved findings from the prior cycle:

```
Refinement cycle 2/3 HANDOFF:
The following findings from your prior ARTIFACT were not resolved:
  - [Finding 1]: [describe what was expected and what was delivered]
  - [Finding 3]: [describe the gap]
Address only these specific findings. Do not re-review areas that were already
satisfactory. Return a partial ARTIFACT covering only the unresolved items.
```

Full re-review on cycles 2–3 doubles the ARTIFACT token cost without adding coverage of new issues. Focused revision maintains the quality bar while halving the per-cycle cost.

---

## 9. Anti-Patterns

These are the failure modes identified in the March 2026 research. Do not do these.

**Front-loading all project files.** Reading every orientation file unconditionally at task start before any routing decision. Adds thousands of tokens of context that may be irrelevant to the task. Apply lazy orientation instead.

**Keeping full ARTIFACTs in orchestrator working context.** After a specialist returns a 3,000-token ARTIFACT, retaining the full text in the orchestrator's working context for the duration of the session. Apply compress-and-externalize instead. Full content goes to disk; a 30-token summary stays in working context.

**Evaluating unstructured responses.** Incorporating a specialist's prose response into synthesis as if it were a structured ARTIFACT. An unstructured response should be returned to the specialist with an explicit format error before any evaluation occurs. The ARTIFACT format is load-bearing for the evaluation gate — evaluating prose degrades synthesis quality.

**Unlimited ReAct loops.** Running the orchestrator's ReAct loop without a cycle cap when ambiguity persists. Each cycle adds context. Without a cap, an ambiguous task can loop until context is exhausted. The cap is 3 cycles; on the third unresolved cycle, the ambiguity is surfaced to the Human-in-the-Loop gate.

**Relying on trigger phrases for constitutional skill activation.** Setting `communication-protocol.md`, `memory-protocol.md`, or `human-in-the-loop.md` as trigger-activated skills with the expectation that they will fire reliably. These three behaviors must be loaded deterministically. Trigger phrase lists are not pre-load filters. They are body content and are only read after the activation decision has already been made.

**Requesting full verbose ARTIFACTs when above 400k tokens.** Continuing to receive 3,000–5,000 token ARTIFACTs from multiple specialists after token usage has crossed the yellow threshold. Once above 400k, switch to summary ARTIFACTs from pending specialists. The quality of the full ARTIFACT at that context depth is already degraded by context rot — summary output is both cheaper and likely of equivalent quality.

**Allowing the episodic index to grow without a size management rule.** Reading the full `agent-memory/episodic.md` index at every task start as it grows toward hundreds of entries. At 50 entries the index consumes ~2,500 tokens on every task start across all agents; at 200 entries that rises to ~10,000 tokens — a meaningful tax on every invocation before any actual work begins. The index size cap rule (read only the 20 most recent rows plus rows matching the current ticket or domain when the index exceeds 50 rows) must be applied.

---

*Written: 2026-03-22. Applies to: Sonnet 4.6, Opus 4.6. Review trigger: significant model capability improvements, new compaction features, or observed context rot at thresholds lower than documented here.*
