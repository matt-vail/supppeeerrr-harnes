# /spike — Rapid Exploration

Build a minimal, throwaway prototype to answer a specific technical question. Time-boxed. No production standards. Ends with a clear "pursue" or "abandon" recommendation.

## When to use

- You don't know if a technical approach is feasible before committing to it.
- You need to understand a new library, API, or pattern before integrating it.
- Two approaches seem viable and you need evidence to choose one.
- Estimating a piece of work is impossible without trying it.

## The spike contract

A spike is not production code. Everyone must agree on this before starting:

- **Time-boxed**: agree on a fixed duration before writing a line of code (30 minutes, 2 hours, 1 day).
- **Throwaway**: spike code is deleted or isolated after the learning is captured. It does not get merged as-is.
- **Question-driven**: the spike answers a specific, pre-stated question. Scope creep invalidates the spike.
- **Learning-first**: messy code is fine. Hardcoded values are fine. Missing tests are fine. The goal is the answer, not the artefact.

## Process

### Step 1 — Define the question

State the spike question precisely before writing any code:
- "Can we integrate `<library>` with our existing `<framework>` setup in a way that supports `<constraint>`?"
- "Is approach A or approach B faster for `<operation>` at `<scale>`?"
- "Does `<third-party API>` return the data shape we need for `<feature>`?"

If the question is vague, sharpen it before proceeding. A vague question produces a vague answer.

### Step 2 — Define the time box

Agree on a duration. When the time box expires, stop and report — even if the question is unanswered. An unanswered spike is itself useful data ("this is harder than expected").

### Step 3 — Build the minimal probe

Write only what is needed to answer the question:
- Hardcoded values where configuration would be needed in production.
- No error handling unless the question is specifically about error handling.
- No tests unless the question is about testability.
- No documentation.
- Single file if possible.

### Step 4 — Record findings as you go

As you learn things, note them immediately — do not rely on memory at the end:
- What worked.
- What didn't work and why.
- Surprises (good and bad).
- Questions that arose during the spike (potential future spikes).

### Step 5 — Produce the spike report

At the end of the time box, produce:

```
## Spike: <question>
Time boxed: <duration>
Time used: <actual>

## Answer
<direct answer to the spike question — one paragraph maximum>

## Evidence
<key code or output that supports the answer>

## Recommendation
[ ] Pursue — <next concrete step>
[ ] Abandon — <reason and alternative>
[ ] Needs more investigation — <what is still unknown and a refined question>

## Risks and open questions
<what we still don't know that matters>

## Spike artefacts
<where the spike code lives, and confirmation that it will not be merged>
```

### Step 6 — Clean up

Delete or archive spike code. Transfer any reusable patterns into a note or task for the production implementation. Do not let spike code linger in the working branch.

## Usage

```
/spike <question to answer>
```

### Examples

```
/spike can we use react-query for our data fetching without rewriting all existing fetch hooks?
/spike is Redis fast enough for session storage at 10k concurrent users?
/spike does the Stripe webhook signature validation work with our current Express middleware setup?
```

## Notes

- A spike that concludes "abandon" is a success. You saved the team from building the wrong thing.
- If the spike reveals the question was wrong, that is the finding — document it and reframe.
- Do not skip the report. The learning dies if it isn't written down.
