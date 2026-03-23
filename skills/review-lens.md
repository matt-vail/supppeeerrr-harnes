---
name: review-lens
description: >
  Four-lens code review framework — Correctness & Edge Cases, Clarity &
  Readability, Structural Concerns, Test Coverage. Use for every code review
  task where executable code is being evaluated for quality, correctness,
  security, or maintainability. Produces prioritised findings (CRITICAL through
  NIT) with file:line references and concrete fixes.
---

# Skill: review-lens

When this skill is active, every review in the conversation follows the four-lens framework below. Do not give ad-hoc feedback — always structure findings by lens and priority.

## The four lenses

Apply them in this order. Do not skip a lens even if it seems unlikely to surface issues.

### Lens 1 — Correctness & Edge Cases
- Logic errors, including subtle ones (precedence, short-circuit evaluation, mutation).
- Boundary conditions: empty collections, zero, negative, max values, nulls/undefined.
- Concurrency: shared state, missing locks, TOCTOU races.
- Error propagation: swallowed errors, missing error returns, wrong error types.

### Lens 2 — Clarity & Readability
- Names that require reading the implementation to understand (variables, functions, types).
- Functions doing more than one thing.
- Comments explaining *what* instead of *why*.
- Excessive nesting or branching that can be flattened.

### Lens 3 — Structural Concerns
- New patterns introduced without justification when existing patterns would work.
- Duplication that crosses the extraction threshold (≥3 occurrences, non-trivial logic).
- Coupling to implementation details that should be hidden.
- Security: unvalidated external input, missing auth/authz, hardcoded secrets, unsafe deserialization.

### Lens 4 — Test Coverage
- Missing tests for the happy path.
- Missing tests for known edge cases and error paths.
- Tests that exist but assert too little (test always passes) or too much (brittle to unrelated changes).
- Public API surface with no tests.

## Output format

```
CRITICAL  | <file>:<line> | <problem> → <fix>
HIGH      | <file>:<line> | <problem> → <fix>
MEDIUM    | <file>:<line> | <problem> → <fix>
LOW       | <file>:<line> | <problem> → <fix>
NIT       | <file>:<line> | <problem> → <fix>
```

After the list:
- State which lens produced the most findings (signal for where the code is weakest).
- Offer to apply fixes for any CRITICAL or HIGH findings immediately.

## Quality gate

Before presenting output, verify:
- [ ] All four lenses were applied.
- [ ] Every finding has a file+line reference.
- [ ] Every finding has a concrete suggested fix, not just a description of the problem.
- [ ] Findings are ordered by priority within the list.
