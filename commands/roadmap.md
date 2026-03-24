# /roadmap — Product Roadmap

Turn an idea or strategic goal into a confirmed, engineering-ready product roadmap. Brings product, architecture, and engineering specialists together to assess feasibility, surface unknowns, and produce a phased plan with milestones and risk-weighted sequencing.

## When to use

- Starting a new product, feature set, or platform initiative.
- When stakeholders have a vision but no concrete execution plan.
- When an engineering team needs to align on priorities before sprint planning.
- Use `/sprint` after `/roadmap` to convert confirmed phases into sprint-level plans.
- For a single feature, use `/breakdown` instead — `/roadmap` is for multi-phase, multi-sprint work.

## Process

### Phase 1 — Idea capture (product-manager)

1. Read the idea, brief, or goal provided by the user.
2. Identify what is known and what is unknown.
3. Produce an initial framing:
   - **Goal** — what outcome does this achieve?
   - **Users** — who is affected and how?
   - **Success criteria** — what does "done" look like at the product level?
   - **Known constraints** — timeline, budget, technical, regulatory.
   - **Open questions** — things that must be resolved before the roadmap can be confirmed.

**HitL gate:** Present the framing to the user. Do not proceed until confirmed. If the user adds constraints or corrections, update and re-confirm.

### Phase 2 — Feasibility assessment (architect)

1. `architect` receives the confirmed framing.
2. Assesses technical feasibility:
   - What systems are involved or impacted?
   - What are the architectural implications?
   - Are there technology decisions that must be made before work can start?
   - What are the build-vs-buy decision points?
3. Flags questions requiring engineering input (route to Phase 3) or external investigation (route to Phase 4).

### Phase 3 — Engineering consult (specialist fan-out)

Fan out to relevant specialists. Each specialist answers:
- What is the complexity of their domain slice — Low / Medium / High?
- What are the key risks or unknowns in their area?
- Are there dependencies between slices that affect sequencing?

Route based on scope:

| If the roadmap involves... | Consult... |
|---|---|
| APIs, backend services | `backend-engineer` |
| Database schema, migrations | `database-engineer` |
| UI, components, frontend | `frontend-engineer` |
| Auth, compliance, data handling | `security-engineer` |
| Infrastructure, CI/CD, deployment | `devops-engineer` |
| Monitoring, alerting, SLOs | `observability-engineer` |

Synthesise specialist inputs into a **complexity and risk matrix**:

```
| Area       | Complexity | Key risk              | Dependencies |
|------------|------------|-----------------------|--------------|
| [area]     | Low/Med/Hi | [specific risk]       | [list]       |
```

### Phase 4 — Research gaps (conditional)

If Phases 2 or 3 surface unknowns that cannot be resolved without investigation — unfamiliar technology, unclear feasibility, no prior art in this codebase — kick off a `/research` task:

```
Research needed: [question]
Assigned to: /research [research prompt]
Blocking: [phase or milestone]
```

Do not proceed to Phase 5 until all **blocking** research is complete. Non-blocking items are noted in the roadmap as open items.

Use `--skip-research` to proceed without waiting for research, treating all unknowns as accepted risk.

### Phase 5 — Roadmap solidification (product-manager + architect)

Produce the confirmed roadmap:

```
Roadmap: [project name]
Goal: [one sentence]
Success criteria: [measurable outcomes]

Phase 1 — [name] ([target timeframe])
  Scope: [what is built]
  Dependencies: [what must be true before this phase starts]
  Risks: [specific risks and mitigations]
  Exit criteria: [what must be true before Phase 2 starts]

Phase 2 — [name] ([target timeframe])
  ...

Open items:
  [unresolved research, deferred decisions, external dependencies]

Assumptions:
  [what this roadmap assumes — flag for review if assumptions change]
```

**HitL gate:** Present the complete roadmap. Do not declare it confirmed until the user explicitly approves.

The roadmap is saved to `~/.supppeeerrr-harnes/agent-memory/scratchpad/supervised/ROADMAP-{YYYYMMDD-HHMM}.md`.

## Usage

```
/roadmap <idea or goal>
/roadmap --skip-research <idea or goal>
```

### Examples

```
/roadmap multi-tenant support for the SaaS platform
/roadmap replace the monolith auth system with an OAuth2 service
/roadmap add real-time collaboration to the document editor
/roadmap --skip-research migrate the frontend from CRA to Vite
```

## Integration

- **Output feeds `/sprint`** — each roadmap phase becomes a sprint planning input.
- **Calls `/research`** — when Phase 4 identifies blocking unknowns.
- **Calls `/breakdown`** — when a phase needs to be decomposed into user stories before sprint planning.
