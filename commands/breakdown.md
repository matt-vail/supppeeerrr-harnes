# /breakdown — Abstract Brief to Engineering Spec

Take a vague, ambiguous, or high-level client brief and produce a structured specification that the engineering team can execute. No assumptions. No guessing. Everything surfaced and confirmed before a line of code is written.

## When to use

- A client or stakeholder has described something they want but hasn't specified what to build.
- You have a brief that uses business language rather than technical language.
- A feature request exists but nobody has written acceptance criteria.
- You are about to start work and realise you don't actually know what "done" looks like.
- Scope is creeping and you need to draw a hard line.

## Process

### Phase 1 — Capture the raw brief

Paste or describe the client's request in their exact words. Do not sanitise it. The raw language contains signal:
- Which words they emphasise.
- What they assume is already understood.
- What they don't mention (often the hardest constraint).

The `product-manager` listens without interrupting at this stage.

### Phase 2 — Structured elicitation

The `product-manager` asks clarifying questions across five dimensions. Questions are tailored to what was and wasn't said in the raw brief.

**Who is the user?**
- Primary persona: who uses this feature, in what role, with what goals?
- Secondary actors: who else touches it?
- Who is explicitly not the user for this phase?

**What is the real problem?**
- What does the user do today to achieve this outcome?
- What is broken or painful about that?
- How often does the pain occur, and how severe is it?

**What does success look like?**
- How will we measure success 30–90 days after launch?
- What would a user say when they first experience it working?
- What would make this a failure despite shipping on time?

**What are the constraints?**
- Hard deadline and the reason it's hard.
- Technology, compliance, or integration constraints.
- Budget in effort, cost, or both.

**What is the scope boundary?**
- What is explicitly out of scope for this phase?
- What is intentionally deferred to a later phase?
- What has already been ruled out and why?

Elicitation continues in rounds until all five dimensions are answered. No story is written before this is complete.

### Phase 3 — Restate and confirm

Before writing any stories, the `product-manager` reflects back a structured summary:

```
## What I heard

User: [primary persona and their context]
Problem: [the real problem being solved]
Current state: [what they do today]
Desired state: [what they want to do after this ships]
Success metric: [how we'll know it worked]

In scope (Phase 1): [list]
Deferred (Phase 2+): [list]
Out of scope: [list]

Hard constraints: [list]
Assumptions we're making: [list]
```

**The client must confirm this before Phase 4 begins.** This is the contract. Changes after this point are scope changes.

### Phase 4 — User stories with acceptance criteria

Each confirmed requirement becomes one or more user stories:

```
Story: [short title]
As a [persona],
I want to [action],
so that [outcome].

Acceptance criteria:
- Given [context], when [action], then [expected result]
- Given [edge case], when [action], then [expected result]
- [Error state]: [expected behaviour]

Out of scope for this story: [explicit exclusions]
Size: XS | S | M | L | XL
```

Stories are ordered by dependency (what must exist before what) and classified:

| Class | Meaning |
|-------|---------|
| MVP   | Required for the thing to function at all |
| V1    | Required for the first real-user release |
| V2    | Valuable, not blocking |
| Later | Backlog |

### Phase 5 — Technical decomposition

For each story, name:
- Which engineering agents own the work (`backend-engineer`, `frontend-engineer`, `api-designer`, etc.).
- Whether any unknowns require a `/spike` before estimation.
- API surfaces that need `api-designer` to sign off before implementation.
- Security or compliance considerations that need `security-engineer` review.
- Accessibility requirements that belong in the acceptance criteria (not bolted on after).

### Phase 6 — Risk and assumption register

Every assumption that, if wrong, would change the architecture or timeline gets logged:

```
ASSUMPTION: [what we're treating as true]
Risk if wrong: [consequence]
Validation: [how to prove or disprove it — spike, interview, prototype]

UNKNOWN: [what we don't know]
Impact: [how it blocks or changes the plan]
Owner: [who resolves it, by when]
```

### Phase 7 — Handoff package

The output is a complete handoff to the engineering team:

```
## Brief: [feature name]
Version: 1.0  Date: [today]

## Confirmed scope
[confirmed in/out/deferred lists]

## User stories
[all stories with acceptance criteria and sizes]

## Technical routing
[which agents own which stories]

## Spikes needed before starting
[any unknowns requiring investigation]

## Risk register
[assumptions and unknowns]

## Definition of done
[what the team ships, how we verify it, how we measure success post-launch]
```

## Usage

```
/breakdown "<raw client brief>"
/breakdown               # paste the brief interactively
```

### Examples

```
/breakdown "we need a way for customers to track their orders"
/breakdown "the app is too slow, can you make it faster"
/breakdown "I want a dashboard that shows how the business is doing"
/breakdown "we need to add login"
```

## Flags

```
/breakdown --skip-elicitation    # brief is already detailed; jump straight to stories
/breakdown --mvp-only            # classify everything into MVP vs later; skip full story writing
/breakdown --technical           # skip Phase 7 handoff; just produce the spec for developer review
```

## Notes

- The most important output of `/breakdown` is not the user stories — it is the confirmed scope boundary. Stories can be refined; scope agreed in Phase 3 is the contract.
- If the client cannot answer the success metric question, the feature is not ready to build. Name this explicitly rather than inventing a metric.
- "We'll know it when we see it" is not an acceptance criterion. Push until it becomes measurable.
- For complex multi-feature briefs, run `/breakdown` once per feature, not once for the whole project.
