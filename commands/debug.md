# /debug — Structured Debugging Session

Work through a bug or unexpected behaviour using a disciplined hypothesis-testing loop. No random changes; no shotgun debugging. Find the root cause, fix it, and verify it stays fixed.

## When to use

- A test is failing and you do not know why.
- A production behaviour does not match what the code appears to do.
- An intermittent error that you cannot reproduce reliably.
- Performance degradation with no obvious cause.

## Process

### Step 1 — Describe the symptom precisely

Before looking at any code, capture:
- **Observed behaviour**: what actually happens.
- **Expected behaviour**: what should happen.
- **Reproduction steps**: the exact sequence that triggers the issue.
- **Environment**: language version, framework, OS, relevant config.
- **Frequency**: always, sometimes, under specific conditions only.

Do not proceed until the symptom is precise. Vague bugs produce vague fixes.

### Step 2 — Locate the blast radius

Read the relevant code and map:
- The entry point (where does the triggering input enter the system?).
- The exit point (where does the incorrect output leave the system?).
- The path between them.

Narrow the search space before forming hypotheses.

### Step 3 — Form hypotheses (max 3 at a time)

Generate a short list of candidate root causes. For each:
- State the hypothesis as a falsifiable claim: "The bug occurs because X does Y when Z".
- Predict what evidence would confirm or deny it.
- Rank by probability.

Do not start fixing until you have at least one testable hypothesis.

### Step 4 — Test hypotheses with instrumentation

For each hypothesis, add the minimal instrumentation to confirm or deny it:
- Logging or print statements at key points.
- A targeted test that would pass if the hypothesis is wrong.
- An assertion or invariant check in the suspect code path.

Run the code with the instrumentation. Interpret the output. Eliminate hypotheses that the evidence contradicts.

### Step 5 — Identify root cause

When one hypothesis survives all evidence:
- State the root cause precisely: "The bug is in `function:line` because `reason`."
- Explain why the symptom manifests from this cause.
- Identify whether this is a symptom of a deeper issue (e.g. missing validation at a boundary means all callers could have the same bug).

### Step 6 — Fix

Apply the minimal change that corrects the root cause:
- Do not fix symptoms; fix causes.
- Do not refactor while fixing — that is a separate step after the bug is verified gone.
- Add a regression test that would have caught this bug before committing.

### Step 7 — Verify

Run the full test suite. Confirm:
- [ ] The failing test now passes.
- [ ] No previously passing tests regressed.
- [ ] The reproduction steps from Step 1 no longer produce the bug.

### Step 8 — Post-mortem (brief)

One paragraph:
- Root cause.
- Why it wasn't caught earlier (missing test? wrong assumption?).
- What was added to prevent recurrence.

## Usage

```
/debug <description of the symptom>
```

### Examples

```
/debug login returns 401 for valid credentials after the recent auth refactor
/debug the order total is calculated incorrectly when a discount code is applied
/debug intermittent timeout on the /export endpoint under load
```

## Routing

If debugging reveals:
- A security vulnerability: route findings to `security-engineer`.
- A performance root cause: route to `performance-engineer` for deeper analysis.
- An accessibility regression: route to `accessibility-pm`.
- A test strategy gap (no regression possible): involve `tdd-coach` to add coverage.
