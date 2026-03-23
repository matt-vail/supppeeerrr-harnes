# Agent Learning and Adaptation

How agents recognise mistakes, recover from them within a session, and improve across sessions.

**Primary sources:** Anthropic "Building Effective Agents"; Shinn et al. "Reflexion" (2023); Anthropic Constitutional AI paper (2022); Park et al. "Generative Agents" (2023); MemGPT / Letta project; Weng (2023).

---

## The learning problem

By default, LLM-based agents do not learn. Each session starts from the same base model. Any mistake made today will be made again tomorrow unless something explicitly captures and feeds back the correction.

There are two distinct learning horizons:

| Horizon | What it means | What it requires |
|---------|---------------|-----------------|
| **Within-session** | Agent improves during the current task | ReAct observations, scratchpad, evaluator feedback |
| **Across-sessions** | Agent improves in future tasks from past ones | Persistent memory, structured episode logging |

Both are necessary. Within-session learning keeps the agent on track during complex tasks. Across-session learning is what makes the agent genuinely better over time.

---

## Mechanism 1: ReAct observation loop (within-session)

**Source:** Yao et al., 2022; Anthropic

The foundational within-session learning mechanism. Every Action produces an Observation — external ground truth — that the agent must incorporate before the next Thought. This breaks the error-compounding chain.

**How it prevents mistakes from snowballing:**
Without observations, a wrong assumption at step 2 propagates silently through steps 3, 4, and 5. With observations, the wrong assumption is contradicted at step 3 before it travels further.

**Recovery pattern:**
```
Thought 3: I assumed the function exists. Let me verify.
Action 3:  search_files("def process_payment")
Observation 3: No matches found.
Thought 4: My assumption was wrong. The function doesn't exist yet.
           I need to create it before calling it.
```

**Limitation:** Only corrects mistakes the agent is designed to check. Blind spots in the ReAct loop remain blind.

---

## Mechanism 2: Reflexion — verbal reinforcement learning

**Source:** Shinn et al., "Reflexion: Language Agents with Verbal Reinforcement Learning" (2023)

Reflexion adds a formal reflection step after task completion or failure. The agent evaluates its own trajectory, generates a verbal summary of what went wrong, and stores it as a memory that is prepended to future attempts.

**The Reflexion loop:**
```
1. Attempt task → get result (success or failure signal)
2. Reflect: "What went wrong? What should I have done differently?"
3. Store reflection as a short verbal memory
4. On the next attempt: prepend the reflection to the context
5. Attempt again — agent now has the lesson explicitly available
```

**What Reflexion stores (from the paper):**
- Detection of inefficiency: "this trajectory took too many steps — there was a shorter path."
- Detection of hallucination: "I repeated the same action three times because I wasn't checking the observation."
- Corrective instruction: "Next time, verify the file exists before trying to edit it."

**Working memory constraint:** Reflexion stores up to 3 recent reflections in working memory. Older reflections are moved to long-term memory to prevent context overflow.

**Experimental results (from paper):**
- AlfWorld (interactive decision making): 97% success with Reflexion vs 75% baseline.
- HotpotQA: +22% accuracy over chain-of-thought baseline.
- Programming tasks: significant improvement in first-attempt success rate.

**Implementation for this plugin:**
After every failed task or rejected output (reviewer sends back CRITICAL findings, tdd-coach runs fail), the responsible agent writes a reflection:
```
REFLECTION — [agent] — [date]
Task: [what was attempted]
Failure: [what went wrong]
Root cause: [why it went wrong]
Correction: [what to do differently next time in this situation]
```
This reflection is stored to episodic memory and retrieved when a similar task is attempted.

---

## Mechanism 3: Constitutional AI — principle-based self-correction

**Source:** Bai et al., "Constitutional AI: Harmlessness from AI Feedback" — Anthropic (2022)

An agent evaluates its own output against a fixed set of principles ("the constitution") and revises if violations are found. The principles themselves are explicitly stated — not learned, but defined.

**The self-critique cycle:**
```
1. Agent produces initial output.
2. Agent critiques it: "Does this output violate any of the following principles?"
3. If yes: agent revises the output.
4. Repeat until no violations found (or max iterations).
```

**For domain agents, the "constitution" is the agent's standards section:**
- `backend-engineer`'s constitution: PEP 8, type annotations required, no mutable defaults, etc.
- `security-engineer`'s constitution: OWASP Top 10, no hardcoded secrets, parameterised queries always.
- `reviewer`'s constitution: every finding needs a file:line reference and a concrete fix.

**The self-critique prompt structure:**
```
Here is my output: [output]
Here are the standards I must meet: [agent's standards section]
Does my output violate any standard? If yes, which ones and how?
Revised output that fixes the violations: [revised output]
```

**When to apply:** After every output, before returning it to the orchestrator. This is the quality gate the agent applies to itself.

**Limitation:** An agent can only critique against standards it's been given. Novel failure modes not covered by the constitution are not caught.

---

## Mechanism 4: Evaluator-Optimizer feedback loop

**Source:** Anthropic "Building Effective Agents"

An external evaluator (the `reviewer` or `tdd-coach`) produces structured feedback. The generator receives this feedback and revises. The feedback itself is the learning signal within the session.

**What makes feedback effective (Anthropic):**
- Specific: "Line 42 uses string-concatenated SQL" not "there are security issues."
- Actionable: paired with a concrete fix, not just a problem description.
- Prioritised: the generator knows which issues must be fixed before the next cycle vs. which are optional.

**Feedback format used in this plugin:**
```
[CRITICAL] security-engineer | src/db/queries.py:42
Problem: String-concatenated SQL — injectable.
Fix: cursor.execute('SELECT * FROM users WHERE id = %s', (user_id,))
Must fix: YES — blocks cycle completion.
```

The generator receives this, applies the fix, and the evaluator re-runs. Three cycles maximum before escalation.

---

## Mechanism 5: Episodic memory retrieval — learning from the past

**Source:** Park et al. "Generative Agents" (2023); Weng (2023)

Before starting a task, the agent queries its episodic memory for similar past situations. If a relevant past episode is found — especially one where a mistake was made and corrected — the lesson is surfaced explicitly in context.

**Retrieval query structure:**
```
"I am about to [task description]. Have I done this before?
What mistakes were made? What was the corrective action?"
```

**Scoring (from Generative Agents paper):** Episodes are scored on:
- **Recency:** more recent = higher score (exponential decay).
- **Importance:** the episode was tagged as high-importance at write time.
- **Relevance:** cosine similarity between the retrieval query and the episode embedding.

**What to store in an episode:**
```
EPISODE
Agent:       backend-engineer
Date:        2026-03-17
Task:        implement async database connection pool
Situation:   greenfield project, no existing patterns
Action:      applied asyncio with threading.Lock — wrong for async context
Outcome:     FAILURE — deadlock in production
Correction:  use asyncio.Lock, not threading.Lock, in async contexts
Tags:        python, async, concurrency, greenfield
```

**Retrieval trigger:** Before any task involving async Python, the `backend-engineer` queries episodic memory with "async Python" and retrieves this episode — preventing the same mistake.

---

## Mechanism 6: Chain of Hindsight — learning from annotated history

**Source:** Liu et al., "Chain of Hindsight Aligns Language Models with Feedback" (2023); Weng (2023)

A sequence of past outputs is presented to the agent with human feedback annotations attached to each. The agent is asked to predict the optimal output given this annotated history.

**Format:**
```
Attempt 1: [output] | Feedback: "Too verbose, missing type annotations."
Attempt 2: [output] | Feedback: "Better, but still missing error handling."
Attempt 3: [output] | Feedback: "Correct."
Now generate the optimal output for this new task: [new task]
```

**When to use:** When the agent has accumulated several failed attempts on a class of task and you want it to reason over the full improvement history rather than just the most recent feedback.

**Limitation:** Requires storing and structuring the full feedback history. More expensive than single-episode retrieval.

---

## Mechanism 7: Self-reflection and memory consolidation

**Source:** Park et al. "Generative Agents" (2023)

Agents periodically synthesise their episodic memories into higher-level semantic lessons. This prevents episodic memory from becoming an unwieldy log and ensures the most important learnings are preserved as stable facts.

**The consolidation process:**
1. After every N episodes (e.g., 10), or when a session ends:
2. Agent queries its most recent episodes.
3. Agent generates: "What are the 3 most important things I've learned from these experiences?"
4. These generalisations are written to semantic memory.
5. The raw episodes are archived (or summarised further).

**Example consolidation:**
```
Recent episodes (10):
- 3 episodes where relative imports caused circular import errors
- 2 episodes where missing type annotations caused mypy failures at review
- 1 episode where forgetting asyncio.Lock caused a deadlock

Generated semantic lessons:
1. "Always use absolute imports in this project — relative imports consistently cause circular dependency errors."
2. "Type annotations are required; mypy is configured in strict mode. Any unannotated function will fail review."
3. "async context managers must use asyncio.Lock, not threading.Lock."
```

These lessons are then loaded into context for every future task by this agent.

---

## Across-session learning: what it requires

| Requirement | Implementation |
|-------------|---------------|
| Persistent episode store | Vector database (Chroma, FAISS, Pinecone) persisted to disk |
| Episode write on task completion | Every agent writes a structured episode at the end of each task |
| Episode retrieval at task start | Every agent queries episodic memory before beginning a new task |
| Periodic consolidation | Scheduled or triggered consolidation of N episodes into semantic lessons |
| Human review of learned memory | Humans can inspect, correct, or delete stored memories |

**MemGPT / Letta implementation:** The most complete open-source implementation of persistent cross-session agent memory. Agents manage their own memory tiers (in-context, recall storage, archival storage) and can edit their own system prompt based on what they learn.

---

## Error taxonomy — structured mistake logging

To learn from mistakes, you must first categorise them consistently. An uncategorised mistake is just noise.

**Mistake categories for this plugin's agents:**

| Category | Description | Example |
|----------|-------------|---------|
| `convention-mismatch` | Code doesn't match project conventions | Used camelCase in a snake_case project |
| `missing-context` | Agent proceeded without sufficient information | Generated code for a function that already existed |
| `wrong-scope` | Agent did more or less than requested | Refactored unrelated code during a bug fix |
| `assumption-unverified` | Agent assumed something without checking | Assumed the test suite existed; it didn't |
| `tool-error-unhandled` | Tool returned an error the agent didn't respond to | Linter failed; agent continued as if it passed |
| `escalation-missed` | Agent should have triggered Human gate but didn't | Deleted files without confirmation |
| `cycle-limit-exceeded` | Evaluator loop ran to max without passing | 3 review cycles, still CRITICAL findings |

Every episode log includes a `mistake_category` field. This enables:
- Pattern detection: "This agent makes `convention-mismatch` errors 60% of the time in greenfield projects."
- Targeted recovery: "If category is `missing-context`, always run the context-gathering ReAct step first."

---

## Practical implementation for this plugin

### What to build now (minimal viable learning)

1. **Reflexion after every rejected output.** When `reviewer` returns CRITICAL findings, the responsible agent writes a Reflexion entry to episodic memory before revising.

2. **Episode log at task end.** Every completed task writes a structured episode (task, outcome, mistakes, lessons) to the persistent store.

3. **Episode retrieval at task start.** Every agent queries episodic memory with the current task description before beginning. Top-3 relevant episodes are injected into context.

### What to build next (full cross-session learning)

4. **Periodic consolidation.** After every 10 episodes per agent, run the consolidation process and update semantic memory.

5. **Human-reviewable memory.** Build a simple interface for inspecting and correcting stored memories before they influence future behaviour.

6. **Mistake pattern reporting.** Periodically surface "agent X makes category Y mistakes most often" to the human, so the agent's procedural memory (system prompt) can be updated to address the pattern.

---

## Sources

| Source | URL |
|--------|-----|
| Reflexion: Language Agents with Verbal Reinforcement Learning — Shinn et al. (2023) | arxiv.org/abs/2303.11366 |
| Constitutional AI — Bai et al., Anthropic (2022) | arxiv.org/abs/2212.08073 |
| Generative Agents: Interactive Simulacra — Park et al. (2023) | arxiv.org/abs/2304.03442 |
| Chain of Hindsight — Liu et al. (2023) | arxiv.org/abs/2302.02676 |
| Building Effective Agents — Anthropic | anthropic.com/research/building-effective-agents |
| LLM Powered Autonomous Agents — Weng (2023) | lilianweng.github.io/posts/2023-06-23-agent/ |
| MemGPT / Letta | letta.ai |
