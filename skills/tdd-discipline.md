---
name: tdd-discipline
description: >
  Red/green/refactor phase gate rules for strict TDD sessions. Use whenever
  writing tests, driving implementation with tests, or running a TDD coaching
  session. Enforces that every test is seen failing before any production code
  is written, prevents phase-skipping, and keeps refactor phases free of new
  behaviour.
---

# Skill: tdd-discipline

When this skill is active, enforce the TDD contract on every test-writing task in the conversation.

## Phase gate rules

These rules are non-negotiable while the skill is active:

| Phase    | Entry condition          | Exit condition                        |
|----------|--------------------------|---------------------------------------|
| Red      | No test exists yet       | New test written AND seen failing     |
| Green    | Test is failing          | All tests pass with minimal new code  |
| Refactor | All tests green          | Suite still green after changes       |

**If any gate is violated**, stop, name the violation, and redirect to the correct phase.

## What counts as "seen failing"

- The test runner was executed.
- The new test produced a failure (not a compilation error, not a skipped test).
- The failure message reveals what the test is actually checking.

A test that was never seen red is not a TDD test — it is a test that was written after the fact and may be testing nothing.

## What counts as "minimum code"

The smallest change that makes the failing test pass without breaking existing tests. Specifically:
- No new abstractions unless the test requires them.
- No defensive code for cases not yet covered by a test.
- No refactoring mixed in.

## Refactor phase constraints

- Run the suite after every individual structural change.
- A suite that goes red during refactor means the refactor introduced a bug — revert the last change.
- No new behaviour during refactor. If a refactor reveals missing behaviour, note it as a future test case.

## Prompts to keep the cycle moving

After red: "Test is failing as expected. Ready to write minimum passing code?"
After green: "All tests pass. Any refactoring to do before the next case?"
After refactor: "Still green. Add another test case or wrap up?"
