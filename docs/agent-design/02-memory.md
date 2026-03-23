# Agent Memory Types

How agents store, retrieve, and use information across a task and across sessions.

**Primary sources:** Lilian Weng, "LLM Powered Autonomous Agents" (2023); Anthropic "Building Effective Agents"; Park et al. "Generative Agents" (2023); MemGPT / Letta project.

---

## The cognitive science foundation

Agent memory taxonomy maps directly onto human cognitive memory research. Understanding the analogy makes the technical choices intuitive.

```
Human memory                    Agent equivalent
─────────────────────────────────────────────────────
Sensory memory (milliseconds)   Raw input embeddings
Working / short-term memory     In-context window
Long-term memory:
  Episodic (events)          →  Episode store (vector DB of past runs)
  Semantic (facts)           →  Knowledge base / RAG corpus
  Procedural (how-to)        →  System prompt / skill library
Scratchpad (paper notes)        Scratchpad / working file
Shared memory (whiteboard)      Shared artifact store (multi-agent)
```

Source: Weng (2023) — "The design of generative agents combines all the components of an LLM-powered agent system."

---

## Memory type 1: In-Context (Working Memory)

**What it is:** Everything currently in the active context window — the system prompt, the conversation history, tool results, and any content the agent has explicitly loaded.

**Capacity:** Fixed and finite. GPT-4: ~128K tokens. Claude: 200K tokens. No matter the size, it is always bounded. This is the fundamental constraint of working memory.

**What it's best for:**
- The current task's active state.
- Reasoning chains that need to reference each other.
- Tool results from the current session.
- Short reference material explicitly loaded for this task.

**Limitations:**
- Forgotten the moment the context window is reset or the session ends.
- Degrades under load — performance drops as the context fills, especially in the middle of a long context (the "lost in the middle" problem).
- Cannot persist learnings across sessions without explicit write to external storage.

**How agents use it:** Everything in working memory is implicit — agents reason over it naturally. There is no explicit "read" operation; it is always present.

**Write pattern:** Content enters working memory through: system prompt (at session start), tool results (appended after each action), user messages, and explicit file loads.

---

## Memory type 2: External Long-Term Memory

**What it is:** Persistent storage outside the model — a vector database, key-value store, relational database, or file system — that the agent reads from and writes to via explicit tool calls.

**Storage mechanism:** Most commonly a vector database (Pinecone, Weaviate, Chroma, FAISS) using embedding-based similarity search (Maximum Inner Product Search — MIPS). The agent embeds a query, retrieves the top-k most semantically similar stored items, and injects them into context.

**Retrieval algorithms:** LSH (Locality Sensitive Hashing), ANNOY, HNSW (Hierarchical Navigable Small World), FAISS, ScaNN. Each trades off speed vs. recall.

**What it's best for:**
- Information that must persist across sessions.
- Knowledge too large to fit in context.
- Historical records of past work, decisions, or mistakes.

**Limitations:**
- Retrieval quality depends on embedding quality and query formulation.
- Stale data — long-term memory must be actively maintained or it becomes out of date.
- No guaranteed recall — similarity search may miss relevant items if they are not semantically close to the query.

**How agents use it:**
- **Read:** `memory_search(query)` → returns top-k relevant chunks → inject into context.
- **Write:** `memory_store(content, metadata)` → embeds and indexes the content.

**In this plugin:** Long-term memory is where lessons learned from past mistakes are stored (see `05-learning.md`). The `orchestrator` reads past project decisions before decomposing a new task.

---

## Memory type 3: Episodic Memory

**What it is:** A record of specific past events, interactions, and experiences — "what happened last time." Episodic memories are time-stamped, situated in context, and specific rather than generalised.

**Technical implementation:** A structured event log stored in a vector database or document store, where each entry records:
- What task was attempted.
- What actions were taken.
- What the outcome was (success, failure, partial).
- What the agent would do differently.

**Retrieval:** Queried by semantic similarity to the current situation. "I am about to do X — have I done something like X before?"

**What it's best for:**
- Avoiding repeated mistakes in similar situations.
- Recognising familiar patterns in new tasks.
- Providing relevant examples to few-shot prompting.

**The Generative Agents implementation (Park et al., 2023):**
- A memory stream stores all observations as natural language.
- At retrieval time, items are scored on three dimensions: recency (exponential decay), importance (LLM rates 1–10), and relevance (cosine similarity to current query).
- The top-k items by combined score are surfaced.

**Limitations:**
- Requires a running event log — not built into most LLM frameworks by default.
- Retrieval is probabilistic; a relevant episode may not surface if the query embedding is poorly constructed.
- Volume grows unbounded; periodic summarisation is needed (see Semantic Memory).

**In this plugin:** When the `reviewer` sees a finding it has flagged many times before and which keeps getting introduced, episodic memory surfaces "this pattern was flagged in the last 3 reviews — here is why it keeps appearing."

---

## Memory type 4: Semantic Memory

**What it is:** Generalised factual knowledge — "what is true about the world / this domain" — independent of any specific past event. The distillation of many episodic memories into stable facts.

**Technical implementation:**
- Static: a pre-built knowledge base, documentation corpus, or retrieval-augmented generation (RAG) index.
- Dynamic: summaries generated from episodic memory over time. The Generative Agents system generates semantic summaries by asking the LLM "what are the most important things I've learned from these 100 recent observations?"

**What it's best for:**
- Domain knowledge that doesn't change often (language specs, API standards, OWASP definitions).
- Project-specific facts that all agents in a team need (the domain model, the tech stack, the naming conventions).
- Compressed knowledge derived from many episodes.

**Limitations:**
- Static semantic memory goes stale as the domain evolves.
- Dynamic generation of semantic memory requires careful prompting to avoid compressing away important nuance.

**In this plugin:** The project-level facts (tech stack, conventions, architecture decisions) are semantic memory. The `backend-engineer` reads project semantic memory before every task ("the project uses `ruff`, `pytest`, `asyncio`").

---

## Memory type 5: Procedural Memory

**What it is:** Knowledge of *how to do things* — skills, action sequences, operating procedures. In humans this is implicit (you don't consciously recall how to type). In agents, it is encoded explicitly.

**Technical implementation:**
- **System prompt:** The agent's core procedures live here — the ReAct loop, the review checklist, the TDD cycle. This is the most direct form of procedural memory.
- **Skill library:** A collection of reusable prompt templates or tool-call sequences for known task types.
- **Retrieved procedures:** Procedures stored in long-term memory and retrieved when the agent recognises a task type.

**What it's best for:**
- Standard operating procedures that apply to a class of tasks (how to run a security review, how to write a pytest fixture).
- Reusable workflows that an agent invokes without needing to re-derive them.

**Limitations:**
- Procedures in the system prompt are fixed — they cannot adapt within a session.
- Procedural memory does not generalise well to novel situations; it is strong on known patterns, weak on new ones.

**In this plugin:** Every agent's `## Core standards` and `## ReAct loop` sections are procedural memory — the system prompt encodes how the agent performs its job.

---

## Memory type 6: Scratchpad

**What it is:** A temporary working area for in-progress reasoning — intermediate calculations, partial plans, draft outputs — that the agent doesn't want to commit to context permanently but needs while working through a complex task.

**Technical implementation:**
- **In-context scratchpad:** The agent writes thoughts in `<thinking>` or similar tags before producing a final answer. These are visible to the agent but filtered from the user.
- **External scratchpad:** A temporary file or note store that the agent writes to and reads from during a task, then discards.
- **Chain-of-thought:** The standard prompting technique ("think step by step") is essentially a structured scratchpad.

**What it's best for:**
- Complex multi-step reasoning that would overflow context if kept in-line.
- Draft outputs that need refinement before being committed.
- Hypothesis generation and elimination during debugging.

**Limitations:**
- In-context scratchpads consume tokens — deep reasoning chains can be expensive.
- External scratchpads require explicit read/write tool calls, adding latency.

**In this plugin:** The `debug` command explicitly uses scratchpad reasoning — the agent writes hypotheses, tests them, and eliminates them before committing to a root cause.

---

## Memory type 7: Shared Memory

**What it is:** A memory store that multiple agents in a team can all read from and write to. The team's common ground.

**Technical implementation patterns:**
- **Blackboard:** A shared document or database that all agents read and append to. No agent "owns" it.
- **Artifact store:** Each agent writes its outputs as named artifacts (spec, implementation, test suite, review findings). Other agents read artifacts by name.
- **Shared vector store:** All agents in a team query the same embedding index. Writes by one agent become retrievable by others.

**Coordination requirements:**
- **Write conflicts:** If two agents write simultaneously, the last write may overwrite the first. Solutions: append-only logs, versioned artifacts, or orchestrator-mediated writes.
- **Read freshness:** An agent may read stale shared memory if another agent hasn't written its update yet. Ordering constraints must be explicit.
- **Authority:** Which agent is the canonical owner of a given artifact? The `orchestrator` must define this per-task.

**What it's best for:**
- Artifacts that multiple specialists need to read (the API spec that both `security-engineer` and `frontend-engineer` need).
- Team-wide knowledge that should be consistent (the project's domain model).
- Accumulating team learnings across a session.

**In this plugin:** The `orchestrator` uses artifact-style shared memory — each specialist writes a structured finding artifact, and the orchestrator reads all of them during synthesis. Specialists do not read each other's artifacts directly.

---

## Memory type 8: Learned Memory

**What it is:** Memory that is actively updated based on feedback, corrections, and outcomes — not just stored, but refined over time.

**Technical implementation:**
- **Prompt mutation:** The agent's own instructions are updated based on accumulated feedback. (MemGPT / Letta implement this — the agent can edit its own system prompt based on what it learns.)
- **Feedback-tagged episodes:** Episodic memories tagged with "this worked" / "this failed" / "human corrected this to X" — the tag changes how strongly the episode influences future behaviour.
- **Constitutional refinement:** The agent checks its output against a set of principles and rewrites it if violations are found. The principles themselves can be updated by humans over time.

**What it's best for:**
- Agents that run repeatedly over time and should improve from session to session.
- Capturing domain-specific corrections that the base model doesn't know.
- Encoding team-specific conventions that emerged from practice rather than being defined upfront.

**Limitations:**
- Requires explicit infrastructure — by default, LLMs do not learn from feedback within or across sessions.
- Corrupt learning: if feedback is wrong, the agent learns the wrong thing. Human review of learned memory is important.
- MemGPT-style self-editing system prompts are powerful but risky — careful constraints needed.

**In this plugin:** See `05-learning.md` for the full implementation approach.

---

## Memory by agent

| Agent | Working | Long-term | Episodic | Semantic | Procedural | Scratchpad | Shared |
|-------|---------|-----------|----------|----------|------------|------------|--------|
| orchestrator | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ (writes + reads) |
| backend-engineer | ✓ | — | ✓ | ✓ | ✓ | ✓ | ✓ (reads artifacts) |
| frontend-engineer | ✓ | — | ✓ | ✓ | ✓ | ✓ | ✓ (reads artifacts) |
| accessibility-pm | ✓ | — | ✓ | ✓ | ✓ | — | ✓ (reads artifacts) |
| security-engineer | ✓ | — | ✓ | ✓ | ✓ | ✓ | ✓ (reads artifacts) |
| api-designer | ✓ | — | ✓ | ✓ | ✓ | ✓ | ✓ (writes spec) |
| performance-engineer | ✓ | — | ✓ | ✓ | ✓ | ✓ | ✓ (reads artifacts) |
| tech-writer | ✓ | — | — | ✓ | ✓ | — | ✓ (reads artifacts) |
| tdd-coach | ✓ | — | ✓ | ✓ | ✓ | — | — |
| reviewer | ✓ | — | ✓ | ✓ | ✓ | — | ✓ (reads artifacts) |
| product-manager | ✓ | — | ✓ | ✓ | ✓ | — | ✓ (writes stories) |

---

## Sources

| Source | URL |
|--------|-----|
| LLM Powered Autonomous Agents — Lilian Weng (2023) | lilianweng.github.io/posts/2023-06-23-agent/ |
| Generative Agents: Interactive Simulacra — Park et al. (2023) | arxiv.org/abs/2304.03442 |
| MemGPT / Letta — virtual context management | letta.ai |
| Building Effective Agents — Anthropic | anthropic.com/research/building-effective-agents |
