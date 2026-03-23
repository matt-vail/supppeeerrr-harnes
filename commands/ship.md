# /ship — End-to-End Feature Delivery

Orchestrate the full team to take a feature from idea to production-ready code: design the contract, build it, test it, review it, and document it.

## When to use

Use `/ship` when you are building something net-new and want the full team involved from the start — not bolted on at the end. It is slower than `/generate` but produces significantly higher-quality output because each specialist reviews their domain before the feature is "done".

## Stages

### Stage 1 — Design (api-designer + security-engineer)

Before any code is written:

1. Define the interface: what inputs does this feature accept, what does it return, what can go wrong?
   - If it is a backend API: produce a minimal OpenAPI spec for the new endpoints.
   - If it is a UI component: define the props interface and visible states.
   - If it is a library function: define the function signature, return type, and error contract.
2. `security-engineer` reviews the interface for: auth requirements, input validation needs, trust boundaries.
3. Agree on the contract before any implementation. Do not proceed until the contract is signed off.

### Stage 2 — Implementation (code-generator + backend-engineer / frontend-engineer)

1. Detect the primary language/framework and route to the appropriate specialist.
2. Generate implementation matching project conventions exactly.
3. Mark all TODOs requiring developer decision.
4. Do not add unrequested features.

### Stage 3 — Tests (tdd-coach)

1. Write tests for the new code following red → green → refactor.
2. Cover: happy path, key edge cases, error paths.
3. Do not proceed to review until the suite is green.

### Stage 4 — Review (reviewer + specialists)

1. `reviewer` performs a full four-lens review of the implementation and tests.
2. Escalate findings to domain specialists as needed:
   - Security findings → `security-engineer`
   - Performance findings → `performance-engineer`
   - Accessibility findings → `accessibility-pm`
3. Apply all CRITICAL and HIGH findings before proceeding.

### Stage 5 — Documentation (tech-writer)

1. Update or create:
   - Inline comments for non-obvious logic.
   - API docs or component docs if the interface is consumer-facing.
   - CHANGELOG entry for user-visible changes.
   - ADR if a significant architectural decision was made.

### Stage 6 — Definition of Done + Ship summary

Before producing the Ship Summary, run the `skills/definition-of-done.md` checklist. The gates in scope depend on the pipeline profile:

| Gate | `--full` | `--fast` | `--hotfix` |
|------|---------|---------|-----------|
| Code quality | ✅ required | ✅ required | ✅ required |
| Tests | ✅ required | ✅ required | ✅ required |
| Documentation | ✅ required | N/A (skipped) | ⚠️ CHANGELOG only |
| Security | ✅ required | ✅ required | ✅ required |
| Operations | ✅ required | ✅ required | ✅ required |

A BLOCKED DoD result halts the Ship Summary. The specific failing check must be resolved before the summary is produced.

**Ship Summary** (after DoD PASS):
- What was built.
- The contract (API spec or function signature).
- Tests written and coverage.
- Findings from review and how they were resolved.
- Documentation produced.
- Definition of Done result.
- Any open TODOs that must be addressed before production.

## Pipeline profiles

### `--full` (default)

All 6 stages as currently defined. Use for net-new features, significant changes, or anything touching security-sensitive code.

### `--fast`

For small bug fixes and minor changes (< ~50 lines of new code, no new endpoints, no schema changes).

- Skips Stage 1 (Design) — no api-designer, no threat model
- Skips Stage 5 (Documentation) — CHANGELOG and ADR not required
- Runs: Implementation → Tests → Review → Ship Summary
- **Requires**: the change must not introduce any new public interfaces or modify existing API contracts. If it does, escalate to `--full`.

### `--hotfix`

For production-urgent fixes requiring the shortest possible path to deployment.

- Skips Stage 1 (Design)
- Skips Stage 5 (Documentation) — CHANGELOG entry required but ADR not
- Runs: Implementation → Tests → `security-engineer` targeted review (scope: the specific change only) → Ship Summary
- **Requires**: a linked incident ticket in the description (e.g. `/ship --hotfix fix null pointer in checkout #INC-142`)
- `security-engineer` reviews only the changed lines — not a full audit

## Usage

```
/ship <feature description>
/ship --fast <feature description>
/ship --hotfix <feature description> #<incident-ticket>
/ship --full <feature description>
```

### Examples

```
/ship user authentication with JWT — login, logout, refresh token
/ship --fast fix null check in order total calculation
/ship --hotfix fix null pointer in checkout #INC-142
/ship --full SearchBar component with debounced API calls and keyboard navigation
```

## Notes

- For a quick generation without the full team process, use `/generate` instead.
- Each stage can be paused for developer input — you do not have to run the full pipeline in one shot.
- If the feature is large, break it into sub-features and `/ship` each one separately.
