---
name: definition-of-done
description: >
  Machine-readable Definition of Done checklist applied at the end of /ship
  and /tdd sessions. Five gate categories: code quality, tests, documentation,
  security, and operations. Produces a structured PASS/BLOCKED result. A BLOCKED
  result prevents Ship Summary and TDD done declaration until all failing checks
  are resolved.
---

# Skill: definition-of-done

When this skill is active, every completion gate in the conversation follows the five-category DoD framework below. Do not declare a feature done, ship it, or close a TDD session without running this checklist and producing the structured output.

## Gate categories

### Code quality (always apply)

- [ ] All CRITICAL and HIGH review findings from `reviewer` are resolved
- [ ] No new linter errors introduced (compare pre/post)
- [ ] Type annotations present on all new public functions/methods
- [ ] No `TODO` or `FIXME` comments in new code without a linked ticket reference

### Tests (always apply)

- [ ] Test suite: all tests pass (green)
- [ ] New functionality has test coverage: at minimum happy path + one error path
- [ ] No tests were deleted to make the suite pass
- [ ] If a bug fix: a regression test that would have caught the bug is present

### Documentation (apply when the change is user-visible or changes a public interface)

- [ ] CHANGELOG entry written for user-visible changes
- [ ] API docs updated if any endpoint contract changed
- [ ] Inline comments on non-obvious logic
- [ ] ADR written if an architectural decision was made

### Security (apply when the change touches auth, input handling, or data persistence)

- [ ] Input validation present at all trust boundaries
- [ ] No new secrets hardcoded
- [ ] Auth/authz checks present on new endpoints
- [ ] `security-engineer` has reviewed if threat model indicated security surface

### Operations (apply when the change will be deployed)

- [ ] Deployment runbook updated or confirmed current
- [ ] Rollback procedure documented
- [ ] No CRITICAL CVEs in dependency audit (`/dependencies` clean)

## Output format

```
Definition of Done — [feature name]
---
Code quality:    PASS (N/N checks)
Tests:           PASS (N/N checks)
Documentation:   PASS (N/N checks) | N/A (no public interface changes)
Security:        PASS (N/N checks) | N/A (no security surface)
Operations:      PASS (N/N checks) | N/A (local/non-deployed change)
---
Overall: PASS | BLOCKED
Blocking items: [list any failed checks, or "none"]
```

A BLOCKED DoD result prevents Ship Summary in `/ship` and prevents `/tdd` from declaring done. The specific failed check must be addressed before proceeding.

## How to apply

### Which checks are always required vs conditional

**Always required:** Code quality and Tests gates apply to every change regardless of scope. There are no exceptions.

**Conditional gates:**

| Gate | Apply when |
|------|-----------|
| Documentation | The change is user-visible, or it modifies or introduces a public interface (API endpoint, component props, exported function signature). |
| Security | The change touches authentication, authorisation, user input handling, or data persistence (database writes, file I/O, external service calls). |
| Operations | The change is intended for deployment to a live environment (staging or production). Local-only tooling changes and dev-only scripts are exempt. |

### How to determine which gates are in scope

Before running the checklist, answer these three questions:

1. Does the change expose anything to a user or consumer? → Documentation gate applies.
2. Does the change touch a trust boundary or sensitive data? → Security gate applies.
3. Will this change be deployed? → Operations gate applies.

When in doubt, apply the gate. Marking a gate N/A when it should have applied is a more serious error than running an unnecessary check.

### What to do when a gate is genuinely not applicable

Mark the gate `N/A` in the output with a one-line reason in parentheses. The reason must be specific to the change — generic phrases like "not applicable" or "doesn't apply here" are not acceptable.

Examples of valid N/A reasons:

- `N/A (internal refactor — no public interface changed)`
- `N/A (CLI tool, no deployment target)`
- `N/A (no auth, input handling, or persistence in this change)`

The orchestrator is responsible for challenging N/A claims that appear unjustified. If a specialist marks a gate N/A and the orchestrator disagrees, it is a Human-in-the-Loop trigger — surface to the developer before proceeding.
