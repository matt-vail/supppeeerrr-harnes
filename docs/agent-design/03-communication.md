# Agent Communication Patterns and Team Structures

How agents in a multi-agent system send information to each other, what they send, and how the team is organised.

**Primary sources:** Anthropic "Building Effective Agents" and multi-agent engineering blog; CAMEL paper (Li et al., 2023); MetaGPT (Hong et al., 2023); AutoGen (Wu et al., 2023).

---

## The fundamental question

When Agent A needs Agent B to do work, how does that communication happen? The answer determines the system's reliability, debuggability, and failure modes.

There are two axes:
1. **Topology:** who can talk to whom?
2. **Content:** what format does the message take?

---

## Communication topologies

### 1. Orchestrator-mediated (hub-and-spoke)

All communication flows through a central orchestrator. Specialists do not talk to each other directly.

```
         ┌──────────────┐
         │ Orchestrator │
         └──────┬───────┘
      ┌─────────┼─────────┐
      ▼         ▼         ▼
  Agent A    Agent B    Agent C
  (writes    (writes    (writes
  artifact)  artifact)  artifact)
```

**How it works:** The orchestrator assigns a task to Specialist A, receives the artifact, then uses it to construct the task for Specialist B. B never sees A's internal reasoning — only the artifact the orchestrator chose to pass.

**Rationale (Anthropic):** "Subagents do not communicate with each other. All flow goes through the orchestrator." This is deliberate — it prevents cascading errors (A's wrong assumption propagating into B's work), keeps the orchestrator as the single source of truth, and makes the system's behaviour auditable.

**When to use:**
- Tasks where quality control between steps matters.
- When specialists might produce conflicting outputs that need arbitration.
- When you need a clear audit trail of what information each agent had.

**Failure mode:** The orchestrator becomes a bottleneck. If it misunderstands A's artifact, B gets a bad input. The orchestrator's comprehension quality gates the whole system.

**Used in this plugin:** The primary communication topology. All specialist agents write artifacts; the `orchestrator` reads and routes them.

---

### 2. Direct agent-to-agent

Agents call each other directly without an intermediary.

```
  Agent A ──────────────► Agent B
     ▲                       │
     └───────────────────────┘
```

**How it works:** Agent A invokes Agent B as a tool call, passing a task and receiving a result. B can also call A back (bidirectional).

**When to use:**
- Tight collaboration between two known specialists (e.g., `api-designer` consulting `security-engineer` during spec design).
- When the orchestration overhead exceeds the coordination benefit.
- Short-lived, well-defined sub-tasks with a single clear consumer.

**Failure modes:**
- Circular calls — A calls B, B calls A, infinite loop.
- Error propagation — A's wrong assumption enters B directly without a quality gate.
- Debugging difficulty — the call chain is harder to inspect than orchestrator logs.

**Anthropic's guidance:** Use sparingly. "Agents generally should not communicate directly — all flow should go through the orchestrator" in production systems. Direct agent-to-agent is appropriate for sub-tasks within a well-understood bounded scope.

---

### 3. Blackboard / shared artifact store

Agents read from and write to a shared store. No agent calls another directly; coordination happens through the shared state.

```
  Agent A ──writes──► [Blackboard] ◄──reads── Agent B
  Agent C ──writes──► [Blackboard] ◄──reads── Agent D
```

**How it works:** Each agent monitors or polls the blackboard for items relevant to its role. When it sees relevant input, it processes it and writes its output back to the blackboard.

**When to use:**
- Loosely coupled teams where the set of agents and their order of execution isn't known upfront.
- Asynchronous workflows where agents operate at different speeds.
- Audit-heavy contexts where a complete record of all intermediate states is needed.

**Failure modes:**
- Write conflicts — two agents write to the same slot simultaneously.
- Stale reads — an agent reads before another has written its update.
- Coordination complexity — without explicit ordering, the system can reach inconsistent states.

**In this plugin:** The `orchestrator`'s artifact collection pattern approximates a blackboard — each specialist writes a named artifact, the orchestrator reads all of them. The orchestrator controls write ordering, avoiding the conflict problem.

---

### 4. Pipeline (sequential handoff)

The output of one agent is the direct input to the next. A strict linear chain.

```
  Agent A ──artifact──► Agent B ──artifact──► Agent C ──► Final output
```

**How it works:** Each agent in the chain receives the previous agent's full output as its input. No branching, no parallelism.

**When to use:**
- Tasks with known, fixed sequential dependencies.
- When each step must complete fully before the next can start.
- Simple workflows where the output format is stable and well-defined.

**Failure modes:**
- Error amplification — an early agent's mistake propagates through every subsequent agent.
- No recovery path — if Agent B fails, the whole pipeline stalls.
- Brittleness — any change to an intermediate output format breaks downstream agents.

**In this plugin:** `/ship` uses a pipeline for its stages: `product-manager` → `api-designer` → implementation → `tdd-coach` → `reviewer` → `tech-writer`. Each stage is a gate before the next.

---

### 5. Broadcast

One agent sends the same message to all other agents simultaneously.

```
         ┌──────────────┐
         │  Broadcaster  │
         └──────┬────────┘
      ┌─────────┼─────────┐
      ▼         ▼         ▼
  Agent A    Agent B    Agent C
  (each processes independently)
```

**How it works:** The orchestrator sends the same task context to all relevant specialists in parallel. Each processes independently and returns their artifact.

**When to use:**
- Independent reviews of the same artifact (security + performance + accessibility reviewing the same PR simultaneously).
- When you want multiple perspectives without agents influencing each other.

**This is Fan-Out.** See `01-orchestration-patterns.md` for detail.

---

### 6. Debate / adversarial

Two or more agents take opposing positions on a question and argue. A judge agent (or the orchestrator) evaluates the debate.

```
  Agent A (advocate) ──┐
                       ├──► Judge ──► Decision
  Agent B (critic)  ──┘
```

**How it works:** One agent proposes a solution or position. A second agent critiques it. A third evaluates which argument is stronger. Multiple rounds of debate may occur before the judge decides.

**Source:** CAMEL paper (Li et al., 2023) — "Communicative Agents for Mind Exploration of Large Scale Language Model Society." Also used in constitutional AI approaches.

**When to use:**
- High-stakes architectural decisions where a single agent's bias could be costly.
- Security reviews where you want an adversarial "attacker" perspective against a "defender."
- Contentious scope or design questions where stakeholders disagree.

**Failure modes:**
- Both agents may converge on the same wrong answer (mode collapse).
- Without a strong judge, the better-articulated argument wins regardless of correctness.
- Expensive — multiple full reasoning traces per decision.

**In this plugin:** Not currently implemented as a formal pattern but applicable to: `security-engineer` (attacker perspective) vs `api-designer` (usability perspective) debates on API design decisions, adjudicated by `orchestrator`.

---

## Team structures

### Hierarchical (most common in this plugin)

```
              Orchestrator
             /      |      \
          PM    API-Designer  Reviewer
                   |
             Python-Engineer
```

A clear authority hierarchy. The orchestrator delegates to sub-orchestrators or specialists. Sub-orchestrators may delegate further.

**Best for:** Complex multi-domain tasks where different specialists own different layers of the work.

---

### Flat / peer

All agents are equal peers. No central coordinator. Each agent decides what to work on and who to consult.

**Best for:** Experimental or research contexts where emergent coordination is acceptable. Not recommended for production — the lack of a coordinator makes error recovery and audit very hard.

---

### Swarm (specialised flat)

Many lightweight agents, each handling a narrow task, self-organising around a shared goal. No central coordinator.

**Source:** Anthropic's swarm research.

**Best for:** Tasks that decompose into many small independent units with no complex dependencies. Each unit can fail without affecting others.

---

## What agents actually pass to each other

The format of communication matters as much as the topology.

### Structured artifacts (recommended)

```json
{
  "agent": "security-engineer",
  "task": "security-review",
  "status": "complete",
  "findings": [
    {
      "priority": "CRITICAL",
      "category": "A03 Injection",
      "file": "src/db/queries.py",
      "line": 42,
      "description": "String-concatenated SQL query",
      "fix": "Use parameterised query: cursor.execute('SELECT * FROM users WHERE id = %s', (user_id,))"
    }
  ],
  "summary": "1 CRITICAL, 2 HIGH findings. Review cannot proceed to merge."
}
```

**Why:** Structured artifacts are machine-readable, can be parsed and routed by the orchestrator, and make the system's state inspectable. Natural language outputs require the orchestrator to parse meaning — a source of error.

**Anthropic's guidance:** Workers should return artifacts, not conversation. The orchestrator reads artifacts, not chat.

### Natural language (for human-facing output only)

Natural language is appropriate at the final synthesis stage — when the orchestrator produces output for the human. Not for inter-agent communication.

### Code and files

Specialist agents may produce actual code files as artifacts. These are passed by reference (file path) rather than inline content to avoid context window bloat.

---

## Communication content checklist

Every inter-agent message should include:

- [ ] **Agent identity** — who sent this.
- [ ] **Task reference** — which task this is a response to.
- [ ] **Status** — complete / partial / blocked.
- [ ] **Artifact** — the actual output (findings, spec, code reference, story list).
- [ ] **Confidence** — any caveats or assumptions the receiving agent should know.
- [ ] **Escalation flag** — does this finding require Human-in-the-Loop?

---

## Sources

| Source | URL |
|--------|-----|
| Building Effective Agents — Anthropic | anthropic.com/research/building-effective-agents |
| Multi-Agent Research System — Anthropic Engineering | anthropic.com/engineering/multi-agent-research-system |
| CAMEL: Communicative Agents — Li et al. (2023) | arxiv.org/abs/2303.17760 |
| MetaGPT: Meta Programming for Multi-Agent — Hong et al. (2023) | arxiv.org/abs/2308.00352 |
| AutoGen: Enabling Next-Gen LLM Applications — Wu et al. (2023) | arxiv.org/abs/2308.08155 |
