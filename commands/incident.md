# /incident — Production Incident Response

Activate structured incident response. The `observability-engineer` acts as incident commander. Depending on the trigger, `security-engineer` and `performance-engineer` are brought in for domain-specific containment.

## When to use

- Something is broken in production right now.
- An alert fired and the team needs to understand what it means.
- Behaviour has degraded below an SLO threshold.
- A security event is suspected.
- After an incident is resolved, to run a blameless post-mortem.

## Modes

```
/incident <description>                # Active incident — triage and mitigate
/incident post-mortem <description>    # Post-resolution — structured post-mortem
```

---

## Active incident response

### Phase 1 — Declare and orient (< 5 minutes)

State clearly:
- **What is broken**: the observed symptom, not a hypothesis.
- **Who is affected**: users, services, data, region.
- **When it started**: timestamp or "unknown — approximate from alerts".
- **What is currently known**: any signals already in hand (alert, log excerpt, error rate).

Do not begin investigation until the symptom is stated precisely. Vague symptoms produce vague mitigations.

Open a scratchpad at `agent-memory/scratchpad/supervised/INCIDENT-{YYYYMMDD-HHMM}.md` using the incident template:

```
# Incident: [short description]
Started: [timestamp]
Commander: observability-engineer
Status: ACTIVE

## Symptom
[exact symptom statement]

## Timeline
[HH:MM] — [event]

## Hypotheses
[ordered by probability]

## Actions taken
[what was done, when, result]

## Current status
[INVESTIGATING | MITIGATING | RESOLVED]
```

### Phase 2 — Triage (< 10 minutes)

The `observability-engineer` leads. Based on the symptom:

| Symptom type | Bring in |
|-------------|---------|
| Error rate spike, 5xx flood, exception storm | `observability-engineer` only initially |
| Latency degradation, timeout cascade, N+1 explosion | `performance-engineer` |
| Auth failures, credential leak, unexpected data access | `security-engineer` immediately |
| Multi-region or infrastructure failure | `devops-engineer` |
| Data corruption or integrity violation | `database-engineer` |

Specialists are brought in only if their domain is implicated. A latency spike does not need a security review. Do not fan out to all specialists by default — incident response requires focus, not noise.

### Phase 3 — Hypotheses and investigation

Generate 3 candidate root causes ranked by probability. For each:
- State as a falsifiable claim: "Latency spiked because query X is performing a full table scan."
- State the evidence that would confirm or deny it.
- State the fastest way to check.

Work through hypotheses in order. Do not pursue all simultaneously.

**Evidence sources to check first:**
1. Error logs for the affected service in the last 30 minutes.
2. Latency metrics at p50/p95/p99 for the last hour.
3. Recent deployments — what changed in the last 4 hours?
4. Dependency health — is a downstream service degraded?
5. Database: slow query log, connection pool utilisation.

### Phase 4 — Mitigate

Once a root cause is confirmed or a working hypothesis has high confidence:

**Prefer reversible mitigations first:**
- Roll back a recent deploy.
- Feature flag off a new code path.
- Increase timeout / reduce retry aggression.
- Scale up a saturated service.

**Pause before irreversible actions:**
- Data deletions.
- Credential rotations that will cause session invalidation.
- Schema changes under load.

For any action that touches more than 5 files or has downstream blast radius: Human-in-the-Loop gate before executing.

```
PAUSE — Incident action requires confirmation
Action: [what is about to be done]
Expected effect: [what this should fix]
Risk: [what could go wrong]
Reversible: YES / NO
Proceed?
```

### Phase 5 — Resolution and hand-off

When the incident is resolved:
1. Update scratchpad status to `RESOLVED`.
2. Record the resolution time.
3. State the confirmed root cause in one sentence.
4. List all actions taken.
5. Flag immediately for post-mortem: "Post-mortem should be run within 24–48 hours."

---

## Post-mortem workflow

Run after resolution, ideally within 24–48 hours while context is fresh.

```
/incident post-mortem <incident description or scratchpad ID>
```

### Post-mortem structure

```
# Post-mortem: [incident name]
Date: [YYYY-MM-DD]
Duration: [start] → [end] ([total duration])
Severity: [SEV1 | SEV2 | SEV3]
Confirmed root cause: [one sentence]

## Impact
- Users affected: [N or "unknown"]
- Services affected: [list]
- Data affected: [yes/no + detail]
- SLO impact: [which SLO breached, by how much, for how long]

## Timeline
[chronological — what happened, when, and who took what action]

## Root cause analysis
[5 Whys or similar — trace from symptom to root]

## What went well
[things that helped — fast detection, effective rollback, good alerting]

## What went wrong
[things that slowed response — missing alerts, unclear runbooks, slow escalation]

## Action items
| Action | Owner | Due date | Priority |
|--------|-------|----------|----------|
| [fix] | [agent or role] | [date] | HIGH/MED/LOW |

## Blameless principle
This post-mortem identifies systemic and process failures. No individual is blamed.
People make reasonable decisions based on information available at the time.
The goal is to make the system more resilient, not to assign fault.
```

### Post-mortem routing

After the post-mortem is written, route action items to the appropriate specialists:
- Alert coverage gaps → `observability-engineer`
- Security hardening → `security-engineer`
- Performance optimisation → `performance-engineer`
- Runbook or documentation gaps → `tech-writer`
- Infrastructure resilience → `devops-engineer`
- Query or schema issues → `database-engineer`

---

## Usage examples

```
/incident API returning 503 for all authenticated requests since 14:32
/incident p99 latency on /checkout jumped from 120ms to 4200ms at 09:15
/incident security — unusual spike in failed auth attempts, possible credential stuffing
/incident post-mortem database outage 2026-03-21
```

## Gate cycle

Incident response uses a **compressed gate cycle** — decisions are made on the fastest defensible evidence, not perfect information. The goal is containment and restoration of service, not a complete root cause before any action.

Human-in-the-Loop gates are **not bypassed** but are kept short:
- State the action, the expected outcome, and the risk.
- Request a yes/no confirmation.
- Do not produce long justification documents during an active incident.

Full documentation is written in the post-mortem after the incident is resolved.
