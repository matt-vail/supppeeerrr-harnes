# Agentic AI Patterns — Reference Guide

A practical reference for the patterns that govern how this plugin's agent team is designed. Every architectural decision in this plugin traces back to one of these patterns.

**Primary source:** Anthropic ("Building Effective Agents", multi-agent research engineering blog)
**Secondary source:** MongoDB ("7 Practical Design Patterns for Agentic Systems")
**Foundational research:** ReAct paper — Yao et al., 2022 (arXiv:2210.03629)

> "Start with the simplest architecture that works. Add complexity only when there is clear evidence it is needed."
> — Anthropic

---

## The patterns

### 1. ReAct — Reasoning + Acting

**Source:** Yao et al., 2022 / Anthropic production systems

The foundational loop for any agent that uses tools or faces uncertainty. Prior to ReAct, reasoning (chain-of-thought) and acting (tool use) were separate. ReAct interleaves them so each observation immediately informs the next thought.

#### The loop

```
Thought  →  Action  →  Observation  →  Thought  →  ...  →  Final answer
```

- **Thought:** Reason about current state. What do I know? What am I assuming? What is the highest-risk unknown right now?
- **Action:** The smallest action that resolves the highest-risk unknown. Read a file. Run a command. Call a tool. Ask one question.
- **Observation:** The real-world result — external ground truth, not the agent's internal belief. Append to context immediately. Update the model before the next thought.

The loop repeats until the task is complete or a hard stop is reached.

#### Why it matters

Without the observation step, reasoning errors compound silently. With it, every cycle is a correction opportunity. The agent can:

- **Reformulate:** if a search returns irrelevant results, the observation makes the mismatch visible and the next thought can try different terms.
- **Invalidate assumptions:** if an intermediate result contradicts a prior assumption, the observation surfaces it before the error propagates.
- **Pivot strategy:** if approach A fails, the observation records that failure and the next thought can switch to approach B.

This is the mechanism that gives agents graceful degradation under uncertainty.

#### Experimental results (from the paper)

- +34% success on ALFWorld (interactive decision environment) vs prior methods.
- +10% success on WebShop (web interaction benchmark).
- Finetuned smaller models using ReAct outperformed prompted larger models — the pattern improves efficiency, not just accuracy.

#### When to use

Every agent that uses tools, reads files, runs commands, or calls other agents. In this plugin: every specialist agent uses ReAct as its internal reasoning loop.

#### When NOT to use

Purely deterministic pipelines where every step is known in advance and no tool result can change the plan. In those cases, a simple prompt chain is sufficient and more predictable.

---

### 2. Orchestrator-Workers

**Source:** Anthropic — "Building Effective Agents"

A central agent (the orchestrator) dynamically decomposes a task, delegates bounded subtasks to specialist workers, and synthesizes their outputs. Workers do not communicate with each other — all flow passes through the orchestrator.

#### Structure

```
                    ┌─────────────────┐
                    │   Orchestrator  │
                    │  (decomposes,   │
                    │   delegates,    │
                    │   synthesizes)  │
                    └────────┬────────┘
                             │
           ┌─────────────────┼─────────────────┐
           ▼                 ▼                 ▼
    ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
    │ Specialist A│  │ Specialist B│  │ Specialist C│
    │  (artifact) │  │  (artifact) │  │  (artifact) │
    └─────────────┘  └─────────────┘  └─────────────┘
```

#### The orchestrator's job

1. Classify the incoming task.
2. Decompose it into bounded subtasks — each with a clear goal, input, expected artifact, and scope constraint.
3. Assign each subtask to the appropriate specialist.
4. Collect and evaluate artifacts.
5. Synthesize into a final output.

The orchestrator does **not** do the detailed implementation work. That is the specialists' job.

#### The workers' job

- Receive a self-contained task description.
- Operate independently using their own tools and context window.
- Return a structured artifact — not a conversational response.

#### Key design guidance (Anthropic)

- Workers should be given **self-contained assignments** so they can operate without back-and-forth with the orchestrator.
- Workers return **artifacts** that persist independently. This minimises token overhead from re-passing large content.
- Invest as much in **Agent-Computer Interface (ACI) design** — tool documentation, parameter naming, error messages — as in prompt engineering.

#### When to use

- Complex problems where the required subtasks cannot be predicted upfront.
- Subtasks require different expertise or toolsets.
- Scope may change based on intermediate findings.

#### When NOT to use

- Simple, single-domain tasks. Use a direct specialist agent instead.
- Tasks where all steps are known and deterministic. Use prompt chaining.

---

### 3. Parallelization — Fan-Out / Fan-In

**Source:** Anthropic — "Building Effective Agents"

Execute multiple independent tasks simultaneously, then aggregate results. Two modes:

#### Mode A: Sectioning (Fan-Out)

Break a task into independent subtasks, run all in parallel, merge results.

```
         ┌──────────────────────────────────────┐
         │            Orchestrator              │
         └──────┬──────────┬──────────┬─────────┘
                │ fan-out  │          │
                ▼          ▼          ▼
          ┌─────────┐ ┌─────────┐ ┌─────────┐
          │Security │ │  Perf   │ │  A11y   │
          │ Review  │ │  Audit  │ │  Audit  │
          └────┬────┘ └────┬────┘ └────┬────┘
               └───────────┴───────────┘
                           │ fan-in
                           ▼
                    ┌─────────────┐
                    │  Synthesized│
                    │   findings  │
                    └─────────────┘
```

Use when: subtasks are genuinely independent (their outputs don't depend on each other) and you want to reduce wall-clock time.

In this plugin: `/audit` fans out `security-engineer`, `performance-engineer`, `accessibility-pm`, and `reviewer` simultaneously against the same codebase.

#### Mode B: Voting

Run the same task with multiple independent agents. Require threshold agreement for the result to be accepted.

```
   Task ──┬──► Agent 1 ──► Finding A
          ├──► Agent 2 ──► Finding A    ─► 2/3 agree ─► Accept
          └──► Agent 3 ──► Finding B
```

Use when: a single agent's judgement is insufficient for a high-stakes decision. The agreement threshold adds robustness.

#### Anthropic's production result

Anthropic's Research feature uses fan-out with 3–5 parallel subagents, each with its own independent context window. This architecture cut research time by up to **90%** for complex queries by exploiting parallel independent context windows.

The key insight: each subagent compresses its findings before returning them to the orchestrator. This prevents the orchestrator from being overwhelmed by raw data.

#### When to use

- Multiple specialists need to evaluate the same artifact (sectioning).
- A high-stakes decision needs confidence beyond a single agent (voting).
- Latency matters and subtasks are independent.

#### When NOT to use

- Subtasks have dependencies (Specialist B needs Specialist A's output). Use sequential orchestration instead.
- The overhead of coordination exceeds the time saved by parallelism (small tasks).

---

### 4. Evaluator-Optimizer

**Source:** Anthropic — "Building Effective Agents"

A generator produces output. An evaluator scores it against defined criteria. The loop continues until the score meets a threshold — or a maximum iteration count is reached.

```
┌──────────┐    output    ┌───────────┐
│Generator │ ──────────► │ Evaluator │
│          │ ◄────────── │           │
└──────────┘   feedback   └───────────┘
     │                          │
     │        (loop)            │ score ≥ threshold
     │                          │ OR max iterations
     └──────────────────────────┘
                                │
                                ▼
                          Final output
```

#### Critical requirement: explicit exit conditions

An evaluator loop without an exit condition will run forever. Always define both:

1. **Quality threshold:** what score or state constitutes "done." (All CRITICAL and HIGH findings resolved. All tests green. Score ≥ 0.8.)
2. **Maximum iterations:** the loop terminates even if the threshold is not met, and the unresolved state is escalated. (3 review cycles. 10 TDD cycles.)

#### In this plugin

| Agent | Generator | Evaluator | Exit condition |
|-------|-----------|-----------|----------------|
| `tdd-coach` | Implementation code | Test suite | All tests pass + max 10 cycles |
| `reviewer` | Code artifact | Reviewer findings | All CRITICAL/HIGH resolved + max 3 cycles |
| `orchestrator` | Specialist artifacts | Orchestrator evaluation gate | Quality bar met + max 3 refinement loops |

#### When to use

- Output quality is critical and iterative improvement demonstrably helps.
- Clear evaluation criteria exist (a linter, a test suite, a rubric).
- The cost of a bad output exceeds the cost of additional iterations.

#### When NOT to use

- No clear evaluation criteria — if you can't define "done," the loop has no exit.
- The generator and evaluator are not meaningfully independent (the evaluator will just agree with the generator).

---

### 5. Human-in-the-Loop

**Source:** Anthropic — "Building Effective Agents" / MongoDB

The system pauses at a pre-defined checkpoint, presents its findings and intended action to a human, and waits for approval before proceeding. Workflow context is preserved across the pause.

#### Structure

```
Agent action ──► Trigger check ──► [no trigger] ──► Continue
                      │
                 [trigger met]
                      │
                      ▼
               Present to human:
               - What the agent intends to do
               - Relevant findings
               - Risk if wrong
               - Explicit question
                      │
               Human approves / adjusts
                      │
                      ▼
                   Proceed
```

#### Trigger conditions in this plugin

| Trigger | Reason |
|---------|--------|
| Action touches more than 5 files | Blast radius requires human judgement |
| Any file deletion | Irreversible |
| Breaking API change | Downstream consumers affected |
| CRITICAL security finding | Risk too high to proceed without sign-off |
| Architecture change | Long-term structural consequences |
| Specialist conflict unresolved after one round | Human is the tiebreaker |
| Scope expands mid-execution | Re-confirm before continuing |

#### Why this threshold

Five files is the point at which the risk of unintended side effects becomes material. Deletions are never reversible once committed. Breaking changes have costs beyond the current session. These are not arbitrary numbers — they reflect where human judgement adds more value than agent autonomy.

#### When to use

- Consequences of errors are significant (financial, legal, security, irreversible).
- Trust in the agent's judgement is not yet established for this class of decision.
- The action affects people or systems outside the current session.

#### When NOT to use

- Routine, reversible, low-blast-radius actions — human gates here create friction without value.
- Explicitly autonomous workflows where the human has pre-approved a class of action.

---

### 6. LLM as Router

**Source:** MongoDB — "7 Practical Design Patterns for Agentic Systems"

An LLM classifies incoming requests and routes them to the appropriate specialist handler. The router does not execute the task — it determines who should.

#### Structure

```
Incoming task ──► Router (classifies) ──► Specialist A
                                     ──► Specialist B
                                     ──► Specialist C
                                     ──► Orchestrator (if multi-domain)
```

#### In this plugin

The `orchestrator` performs this classification as Step 1 of every task:

| Task type | Routing decision |
|-----------|-----------------|
| Single-domain, simple | Route to one specialist directly |
| Single-domain, complex | Delegate to one specialist with full artifact spec |
| Multi-domain | Decompose → fan out → synthesize |
| Ambiguous | ReAct loop to reduce ambiguity before routing |

The key rule: **never route an ambiguous task**. Routing ambiguity to a specialist doesn't resolve it — the specialist will either guess or stall. Reduce ambiguity first.

#### When to use

- Multiple distinct processing paths exist.
- Accurate classification is achievable (the task types are meaningfully different).
- You want intelligent dispatching without requiring full agent autonomy.

---

## How the patterns compose

Patterns are not mutually exclusive. In production systems, they are nested:

```
Incoming request
       │
       ▼
[LLM as Router]  ←── Step 1: classify
       │
       ├── simple ──► [Single specialist + ReAct loop]
       │
       └── complex ──► [Orchestrator-Workers]
                              │
                    ┌─────────┴──────────┐
                    │  Fan-Out (parallel) │
                    │  for independent   │
                    │  specialist tasks  │
                    └─────────┬──────────┘
                              │
                    [Evaluator-Optimizer]
                    (per specialist output)
                              │
                    [Human-in-the-Loop]
                    (at trigger thresholds)
                              │
                    Synthesized final output
```

Each specialist internally runs a **ReAct loop** regardless of which outer pattern invoked it.

The `tdd-coach` and `reviewer` are **Evaluator-Optimizer** loops embedded within the larger Orchestrator-Workers flow.

---

## Pattern selection guide

| Situation | Pattern |
|-----------|---------|
| Agent needs tools and faces uncertainty | ReAct |
| Task needs multiple specialist domains | Orchestrator-Workers |
| Multiple specialists can work simultaneously | Fan-Out |
| Need confidence beyond one agent's judgement | Voting |
| Output quality requires iterative refinement | Evaluator-Optimizer |
| Action is irreversible or high blast radius | Human-in-the-Loop |
| Multiple task types need different handlers | LLM as Router |

---

## What this plugin uses and where

| Pattern | Applied in |
|---------|-----------|
| **ReAct** | Every agent — internal reasoning loop + hard situation recovery |
| **Orchestrator-Workers** | `orchestrator` agent + all specialist agents |
| **Fan-Out (Sectioning)** | `/audit` command, `/ship` stage 4 |
| **Evaluator-Optimizer** | `reviewer` (3-cycle max), `tdd-coach` (10-cycle max), `orchestrator` evaluation gate (3-cycle max) |
| **Human-in-the-Loop** | `orchestrator` — 7 defined trigger conditions |
| **LLM as Router** | `orchestrator` — Step 1 of every task |

---

## Key principles from Anthropic (verbatim)

> "Start with the simplest architecture that works, carefully evaluate its performance, and add additional components only if there is clear evidence that they are needed."

> "In agentic contexts, it's important to try to maintain a minimal footprint where possible. Request only necessary permissions, avoid storing sensitive information beyond immediate needs, prefer reversible over irreversible actions."

> "Invest as much in Agent-Computer Interface (ACI) design — tool documentation, parameter naming, error messages — as in prompt engineering."

> "By tightly coupling each action to explicit reasoning, and allowing each observation to immediately inform the next reasoning step, the agent maintains flexibility while preserving strategic coherence." — on ReAct

---

## Sources

| Source | Authority | URL |
|--------|-----------|-----|
| Building Effective Agents | Anthropic (primary) | anthropic.com/research/building-effective-agents |
| Multi-Agent Research System | Anthropic Engineering | anthropic.com/engineering/multi-agent-research-system |
| ReAct: Synergizing Reasoning and Acting | Yao et al., 2022 | arxiv.org/abs/2210.03629 |
| 7 Practical Design Patterns for Agentic Systems | MongoDB | mongodb.com/resources/basics/artificial-intelligence/agentic-systems |
| What is a ReAct Agent? | IBM | ibm.com/think/topics/react-agent |
