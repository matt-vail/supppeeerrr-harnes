# /research — Research Team

Assemble a specialist team to investigate a complex technical question. Fans out to relevant domain experts in parallel, synthesises findings, and produces a confidence-rated research report with a clear recommendation.

## When to use

- Evaluating an unfamiliar technology, library, or architecture pattern.
- Investigating a performance, security, or scalability problem without a clear root cause.
- Answering a question that spans multiple domains (e.g. security + performance + architecture).
- Supporting a `/roadmap` when Phase 4 surfaces a blocking unknown.
- Due diligence before adopting a new dependency or third-party service.
- For a quick codebase orientation (not a research question), use `/map` instead.

## Process

### Step 1 — Research brief (orchestrator)

1. Parse the research question. Identify:
   - **Core question** — the single most important thing to answer.
   - **Domain areas** — which specialisms are relevant?
   - **Scope constraints** — what is explicitly out of scope?
   - **Required output** — recommendation, comparison, feasibility verdict, or open-ended findings?
2. Decompose into parallel investigation threads — one per domain area.
3. Write an intent anchor to the supervised scratchpad before invoking any specialist.

### Step 2 — Parallel investigation (specialist fan-out)

Fan out to relevant specialists. Each specialist investigates their thread and returns an ARTIFACT with:
- **Findings** — what they discovered, with evidence (file references, reasoning).
- **Confidence** — HIGH / MEDIUM / LOW — how certain are these findings?
- **Unknowns** — what they could not resolve.
- **Recommendation** — their domain-specific recommendation (if applicable).

**Routing guide:**

| Research domain | Specialist |
|---|---|
| Existing codebase structure, patterns, debt | `code-archaeologist` |
| Architecture, system design, technology selection | `architect` |
| Security, vulnerabilities, threat model | `security-engineer` |
| Performance, scalability, profiling | `performance-engineer` |
| Backend tech, API design, language choice | `backend-engineer` |
| Frontend tech, UI frameworks, bundle impact | `frontend-engineer` |
| Database design, query performance | `database-engineer` |
| Infrastructure, deployment, operational complexity | `devops-engineer` |
| Dependency health, CVEs, licence risk | `dependency-engineer` |

Use only specialists whose domain is relevant — do not fan out to all specialists on every task.

### Step 3 — Synthesis (orchestrator)

1. Read all specialist ARTIFACTs.
2. Identify agreements, disagreements, and gaps across threads.
3. Where specialists disagree: convene a HUDDLE to surface the conflict explicitly before resolving it.
4. Produce the unified research report.

### Step 4 — Research report

```
Research: [question]
Date: [YYYY-MM-DD]
Overall confidence: HIGH | MEDIUM | LOW

## Summary
[2–4 sentences: the key finding and the recommendation]

## Findings by domain

### [Domain 1]
Confidence: HIGH | MEDIUM | LOW
[findings with evidence]

### [Domain 2]
...

## Recommendation
[clear recommendation — what to do, why, and with what caveats]

## Alternatives considered
[what was evaluated but not recommended, and why]

## Open questions
[what was not resolved — and what would close it]

## Confidence rationale
[why the overall confidence is what it is — what is certain vs inferred]
```

**HitL gate:** Present the research report before it is used to drive any implementation or roadmap decision. The human must explicitly accept or challenge the recommendation before it is treated as resolved.

## Flags

| Flag | Effect |
|------|--------|
| `--quick` | Single specialist, no synthesis — fast answer for a focused single-domain question |
| `--deep` | Extended investigation — specialists explore more broadly, read more files |
| `--compare <A> vs <B>` | Explicit comparison — report structures findings as a head-to-head |

## Usage

```
/research <question or topic>
/research --quick <question>
/research --deep <question>
/research --compare <A> vs <B>
```

### Examples

```
/research is our current Redis caching strategy causing the latency spikes at peak load?
/research --compare GraphQL vs REST for the new public API
/research --quick what test framework is in use and does it support parallel execution?
/research --deep what would it take to support 10x our current database write throughput?
/research --compare Kafka vs SQS for the event pipeline migration
```

## Notes

- Research findings are saved to `agent-memory/scratchpad/supervised/RESEARCH-{YYYYMMDD-HHMM}.md` and an episodic entry is written on completion.
- `/roadmap` calls `/research` automatically when Phase 4 identifies blocking unknowns — the output feeds back as resolved or partially resolved open items.
- Research reports become graph nodes of type `spike-result`. Reference them in subsequent HANDOFF `<related-nodes>` fields.
