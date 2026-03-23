# /tdd — Test-Driven Development

Enforce the strict **red → green → refactor** cycle for a given piece of functionality.

## Cycle

### Phase 1 — Red (failing test)

1. Write the smallest possible failing test for the next behaviour.
2. Run the test suite and **confirm the new test fails** with a meaningful failure message.
   - If the test passes immediately, the test is wrong — stop and diagnose.
3. Show the failure output to the user before proceeding.

### Phase 2 — Green (minimum passing code)

1. Write the minimum production code needed to make the test pass.
   - No gold-plating. No extra abstraction. No unrequested behaviour.
2. Run the test suite and confirm **all tests pass**.
3. Show the green output to the user.

### Phase 3 — Refactor

1. Only now improve structure, naming, duplication, or readability.
2. The test suite must stay green throughout every refactor step.
3. Commit (or checkpoint) the clean passing state.

### Loop

- Ask: "Add another test case, or are we done?"
- If another case: return to Phase 1 with the next behaviour.
- If done: run the `skills/definition-of-done.md` **Code quality and Tests gates only** before declaring done. Documentation, Security, and Operations gates are out of scope for a TDD session — those belong in `/ship`.

  ```
  Definition of Done — [behaviour name]
  ---
  Code quality:  PASS (N/N checks)
  Tests:         PASS (N/N checks)
  ---
  Overall: PASS | BLOCKED
  Blocking items: [list, or "none"]
  ```

  A BLOCKED result means the session is not done. Resolve the failing check, then return here.

- If PASS: summarise the full cycle — tests written, code produced, refactors made, DoD result.

## Usage

```
/tdd <description of the behaviour to implement>
```

### Examples

```
/tdd user login returns a JWT on success
/tdd stack.pop() raises IndexError when empty
/tdd calculateDiscount applies 10% for orders over $100
```

## Rules

- **Never skip the red phase.** Running a test that was never seen failing is not TDD.
- **Never refactor on red.** Structural changes only happen when the suite is green.
- **Never add behaviour during refactor.** Behaviour changes belong in Phase 1 of the next loop.
- Delegate to the `tdd-coach` agent for a full session with multiple features.
