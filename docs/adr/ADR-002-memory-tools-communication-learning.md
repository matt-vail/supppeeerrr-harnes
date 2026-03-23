# ADR-002: Document Memory, Tools, Communication, and Learning as Foundational Design Dimensions

Date: 2026-03-17
Status: Accepted

---

## Context

After establishing the orchestration patterns (ADR-001), a deeper review of the agent team identified four additional dimensions that are not yet formally addressed in any agent definition:

1. **Memory** — agents have no defined memory strategy beyond in-context window.
2. **Tools** — tool requirements are implied but not specified per agent.
3. **Communication** — inter-agent communication format and topology are defined by convention, not documented.
4. **Learning** — no mechanism exists for agents to improve from past mistakes within or across sessions.

Research was conducted against primary sources (Anthropic, Lilian Weng's 2023 agent survey, Shinn et al. Reflexion paper, Anthropic Constitutional AI paper, Park et al. Generative Agents paper) before writing any documentation.

---

## Decisions

### Memory

Adopt the cognitive science taxonomy as the reference framework (Weng, 2023):

| Type | Role in this plugin |
|------|-------------------|
| In-context (working) | Active task state — always present |
| External long-term | Persistent cross-session storage via vector DB |
| Episodic | Past task logs — retrieved at task start for relevant lessons |
| Semantic | Project facts and distilled lessons — loaded into context |
| Procedural | Agent standards and ReAct loop — encoded in system prompt |
| Scratchpad | Temporary reasoning workspace during complex tasks |
| Shared | Artifact store — orchestrator mediates all writes |
| Learned | Memory updated from feedback — enabled by Reflexion + consolidation |

The `orchestrator` uses all eight types. Specialists use the subset relevant to their domain.

### Tools

Adopt Anthropic's ACI (Agent-Computer Interface) design principles as the standard for all tool definitions. Each agent is assigned only the tools it needs (minimal permissions). Tool categories: file system, code execution, search/retrieval, code analysis, memory, agent spawning, and specialised domain tools.

Full per-agent tool mapping is documented in `agent-design/04-tools.md`.

### Communication

Adopt the orchestrator-mediated topology as the primary communication pattern (all specialist output flows through the orchestrator). Direct agent-to-agent is permitted for tightly-coupled sub-tasks within a bounded scope. All inter-agent messages use structured artifact format, not natural language.

### Learning

Adopt a three-layer learning stack:
1. **Within-session:** ReAct observation loop (already implemented in all agents).
2. **Post-task:** Reflexion — every rejected output triggers a written reflection stored to episodic memory.
3. **Across-session:** Episode retrieval at task start + periodic consolidation of episodes into semantic lessons.

Constitutional AI self-critique (agent checks own output against its standards before returning) is recommended as a pre-return quality gate for all agents.

---

## Consequences

**Easier:**
- New agents can be defined against a clear checklist: what memory types, what tools, what communication topology, what learning mechanisms.
- The per-agent tool table makes security review straightforward (minimal permissions are explicit, not assumed).
- Reflexion gives every agent a defined recovery path for repeated mistakes.

**Harder:**
- Full implementation requires infrastructure: a persistent vector database, episode logging at task boundaries, and a periodic consolidation job.
- Learned memory requires human oversight to prevent corrupt learning from bad feedback.
- The tool taxonomy adds complexity to agent setup — each agent needs its specific tools configured.

**Not yet implemented:**
- Persistent episodic memory store (requires infrastructure beyond the plugin itself).
- Automated Reflexion writing (currently a convention, not enforced by tooling).
- Memory consolidation scheduler.

These are future implementation milestones. The documentation establishes the target architecture.

---

## Reference

- `docs/agent-design/02-memory.md`
- `docs/agent-design/03-communication.md`
- `docs/agent-design/04-tools.md`
- `docs/agent-design/05-learning.md`
