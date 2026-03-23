# Documentation

Reference documentation for the agent team design, architectural decisions, and agentic AI patterns.

---

## Agent Design — Core Concepts

These documents cover the foundational theory behind how agents in this plugin are built. Read them in order if you are new to agentic systems.

| Document | What it covers |
|----------|---------------|
| [agent-design/01-orchestration-patterns.md](agent-design/01-orchestration-patterns.md) | ReAct, Orchestrator-Workers, Fan-Out, Evaluator-Optimizer, Human-in-the-Loop, LLM-as-Router |
| [agent-design/02-memory.md](agent-design/02-memory.md) | In-context, long-term, episodic, semantic, procedural, scratchpad, shared, learned |
| [agent-design/03-communication.md](agent-design/03-communication.md) | Agent-to-agent patterns, team structures, artifact types |
| [agent-design/04-tools.md](agent-design/04-tools.md) | Tool taxonomy, ACI design principles, per-agent tool mapping |
| [agent-design/05-learning.md](agent-design/05-learning.md) | Reflection, feedback loops, mistake memory, cross-session learning |
| [agent-design/07-skills-architecture.md](agent-design/07-skills-architecture.md) | Skills vs subagents, on-demand loading, validated design decisions, anti-patterns |
| [agent-design/08-context-management.md](agent-design/08-context-management.md) | Claude model limits, 13-step accumulation model, compress-and-externalize pattern, context budget thresholds, skills trigger architecture finding, lazy orientation, evaluator-optimizer loop cost, anti-patterns |
| [agent-design/09-task-continuity.md](agent-design/09-task-continuity.md) | Task continuity hierarchy, intent anchoring, progress checkpointing, decision logging, direction verification, context recovery protocol, memory index management |

---

## Architectural Decision Records

Records of significant decisions made about how this plugin is designed and why.

| ADR | Decision |
|-----|---------|
| [adr/ADR-001-agentic-patterns.md](adr/ADR-001-agentic-patterns.md) | Adopt Anthropic agentic patterns as architectural foundation |
| [adr/ADR-002-memory-tools-communication-learning.md](adr/ADR-002-memory-tools-communication-learning.md) | Document memory, tools, communication, and learning as foundational design dimensions |
| [adr/ADR-003-context-management-strategy.md](adr/ADR-003-context-management-strategy.md) | Adopt compress-and-externalize, context budget thresholds, and task continuity checkpointing as the plugin's context management strategy |

---

## How to use these docs

- **Building a new agent?** Start with `01-orchestration-patterns.md` for the structural pattern, `02-memory.md` for what memory the agent needs, and `04-tools.md` for its tool set.
- **Debugging agent behaviour?** `05-learning.md` covers how agents recognise and recover from mistakes.
- **Designing a multi-agent workflow?** `03-communication.md` covers how agents communicate and what team structure fits your scenario.
- **Questioning an architectural choice?** Check the ADRs for the documented rationale.
