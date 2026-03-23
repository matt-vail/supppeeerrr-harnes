# /sprint — Sprint and Release Planning

Take confirmed user stories from `/breakdown` and produce a sprint plan: story selection, dependency ordering, risk-weighted sequencing, and a release readiness checklist. This is a planning assist for a human making sprint decisions — it organises the work, it does not execute it.

## When to use

- After `/breakdown` has confirmed scope, user stories, and story-point estimates.
- When the team is deciding what goes into the next sprint from a backlog.
- Before a release to produce a release readiness checklist.
- When dependencies between stories need to be made explicit before scheduling.

## Process

### Step 1 — Ingest the backlog

The `product-manager` reads the output of `/breakdown` or accepts a list of user stories directly. Each story must have:

- **ID** — a stable identifier (e.g. `S-01`)
- **Title** — short, human-readable label
- **Estimate** — story points or T-shirt size (XS / S / M / L / XL)
- **Dependencies** — IDs of any stories that must complete first
- **Acceptance criteria** — at minimum one "given / when / then" statement

If any story is missing an estimate, flag it as a blocker before proceeding:

```
BLOCKED: Story [ID] — "[title]" has no estimate.
Resolve: add a story-point or T-shirt size estimate before planning continues.
```

Do not plan unestimated work. Return to `/breakdown` or estimate inline before running `/sprint` again.

### Step 2 — Dependency mapping

The `architect` identifies which stories must complete before others can start, across three dependency types:

- **Technical** — story B requires an API, schema, or interface delivered by story A
- **Data** — story B requires a migration or seed data delivered by story A
- **UX** — story B is blocked on a design decision confirmed in story A

Produce a dependency graph in text form:

```
Dependencies:
  S-01 blocks S-03  (technical: S-03 calls the endpoint built in S-01)
  S-02 blocks S-04  (data: S-04 reads rows created by S-02 migration)
  S-03 blocks S-05  (ux: S-05 reuses the component pattern established in S-03)
```

Stories with no dependencies are free-floating and can start immediately.

### Step 3 — Risk assessment

The `product-manager` rates each story by risk level:

| Risk | Conditions |
|------|------------|
| HIGH | New external dependency, security-sensitive code, schema migration, first implementation of a new pattern in this codebase |
| MEDIUM | Extension of an existing pattern, touches shared utilities, integration work between two existing systems |
| LOW | Isolated feature, well-understood pattern, no external dependencies, no shared surface area |

HIGH-risk stories are scheduled early in the sprint to allow time for discovery and iteration. Do not push them to the end of the sprint where there is no recovery window.

### Step 4 — Sprint selection

Given the velocity (if provided with `--velocity`), the `product-manager` selects stories for the sprint:

- Fill to **~80% of velocity** — leave buffer for unexpected work, review cycles, and incidents
- **Required stories first**: stories that are dependencies for later sprints, explicit commitments, or active blockers
- **HIGH-risk stories early**: schedule them at the start of the sprint
- **Quick wins last**: LOW risk, LOW effort stories to maintain momentum and close the sprint cleanly

If no velocity is provided, present the full prioritised set with estimates and flag that sprint boundary is unconfirmed.

Present the selected set as a table:

```
| ID   | Title                        | Est | Risk   | When  |
|------|------------------------------|-----|--------|-------|
| S-01 | Create user auth endpoint    | 5   | HIGH   | Start |
| S-02 | Add database migration       | 3   | HIGH   | Start |
| S-03 | Build login UI               | 5   | MEDIUM | Mid   |
| S-04 | Write integration tests      | 3   | LOW    | Mid   |
| S-05 | Update onboarding copy       | 1   | LOW    | End   |
```

Stories that do not fit the sprint go into the deferred table with a reason:

```
| ID   | Title                        | Est | Reason                          |
|------|------------------------------|-----|---------------------------------|
| S-06 | Export to CSV                | 8   | Exceeds velocity buffer         |
| S-07 | Admin audit log              | 5   | Depends on S-05, deferred scope |
```

### Step 5 — Release readiness checklist

If a release target is specified with `--release <tag>`, the `devops-engineer` produces the following checklist. This checklist is the same gate used by `/ship`'s Ship Summary for production readiness — completing it here means `/ship` has a pre-filled checklist to verify against, not a second checklist to create from scratch.

```
Release readiness: v{tag}
---
Implementation
[ ] All stories in scope: COMPLETE
[ ] All CRITICAL and HIGH review findings: RESOLVED
[ ] All acceptance criteria verified by reviewer

Testing
[ ] Test suite: GREEN
[ ] No regressions vs. last release baseline
[ ] Performance: p99 latency within target (from /ship output)

Security
[ ] Security review: COMPLETE (security-engineer sign-off)
[ ] Dependency audit: no CRITICAL CVEs (/dependencies clean)

Documentation
[ ] CHANGELOG entry: written
[ ] API docs: updated for any contract changes
[ ] ADR: written for architectural decisions

Operations
[ ] Deployment runbook: updated
[ ] Rollback procedure: tested in staging
[ ] Monitoring/alerting: covers new functionality
[ ] Feature flags: configured if applicable
```

If `--release` is not provided, omit the checklist entirely. Do not produce a partial or placeholder version.

### Step 6 — Sprint plan output

Deliver the complete plan in this format:

```
Sprint [N] Plan
Velocity: [N] / Committed: [N] ([N]% utilised)
Duration: [start] → [end]

Stories selected:
| ID | Title | Est | Risk | When |
|----|-------|-----|------|------|
| ...

Deferred to next sprint:
| ID | Title | Est | Reason |
|----|-------|-----|--------|
| ...

Dependencies:
  Story X blocks Story Y
  Story Y blocks Story Z

Risks to watch:
  [Story ID] — [why it is HIGH risk and what to monitor during the sprint]

Release checklist: [attached below if --release flag used, otherwise omitted]
```

If sprint number is provided with `--sprint N`, use it as the label. If it is not provided, label the sprint "Next Sprint" and note that the number is unset.

If duration dates are not provided, omit the Duration line rather than inventing dates.

## Usage

```
/sprint                          # plan from existing breakdown output
/sprint --sprint N               # target sprint number (cosmetic label)
/sprint --velocity N             # team velocity in story points per sprint
/sprint --release <tag>          # plan toward a specific release target
```

### Examples

```
/sprint
/sprint --sprint 14
/sprint --velocity 32
/sprint --sprint 14 --velocity 32 --release v2.4.0
/sprint --release v2.4.0
```

## Flags

```
/sprint --sprint N        # label the sprint plan with sprint number N
/sprint --velocity N      # set team velocity; limits story selection to ~80% of N points
/sprint --release <tag>   # attach a release readiness checklist for the given tag
```

## Integration

- **Input**: output from `/breakdown` (confirmed user stories with estimates, dependencies, and acceptance criteria)
- **Feeds `/ship`**: the selected stories list tells `/ship` which features to start with and in what order
- **Feeds `/code-review`**: the release scope table tells `/code-review` what is in-flight and what the release boundary is
- **Release checklist**: the checklist produced here is the same gate `/ship` uses for its Ship Summary production readiness block — not a duplicate, a shared contract

## Notes

- The most important output of `/sprint` is the dependency order and the deferred list — knowing what is not in the sprint is as valuable as knowing what is.
- If two stories have mutual dependencies, surface the cycle explicitly and ask the human to resolve it before continuing. Do not silently break the cycle.
- Story-point estimates from `/breakdown` use T-shirt sizes. Convert to points for velocity math using the team's agreed mapping; if no mapping exists, ask before converting.
- `/sprint` does not commit anyone to anything. It is a structured input to a human decision, not a substitute for one.
- If the backlog is large and spans multiple potential sprints, produce a table for the current sprint only, then list remaining stories in priority order as "future sprint candidates" — do not pre-fill sprint 2, 3, etc.
