# supppeeerrr-harnes — Project Context

## What this is
A Claude Code plugin providing a full expert engineering team built on Anthropic agentic patterns. 18 agents, 17 commands, 16 prompt skills, and a structured agent memory system.

## Team structure
`orchestrator` is the single entry point for all tasks. It routes to 17 specialists. Specialists do not communicate with each other — all flow goes through the orchestrator.

**Specialists:** architect, database-engineer, devops-engineer, observability-engineer, code-archaeologist, backend-engineer, frontend-engineer, accessibility-pm, security-engineer, api-designer, performance-engineer, tech-writer, tdd-coach, reviewer, code-reviewer, product-manager, dependency-engineer.

## Agentic patterns in use
- ReAct — all agents
- Orchestrator-Workers — orchestrator routes, workers execute
- Fan-Out/Fan-In — parallel independent specialists, synthesized by orchestrator
- Evaluator-Optimizer — reviewer (3 cycles max), tdd-coach (10 cycles max)
- Human-in-the-Loop — orchestrator, 10 trigger conditions, non-negotiable

## Key directories
- `agents/` — 18 agent definition files
- `commands/` — 17 slash commands
- `skills/` — 16 prompt skill files
- `hooks/` — PostToolUse automation
- `agent-memory/` — template files (config.md, README.md) committed to the plugin repo; runtime memory goes to `~/.supppeeerrr-harnes/agent-memory/` in the user's project
- `docs/` — ADRs and agentic design references

## Memory system
Agents read `~/.supppeeerrr-harnes/agent-memory/episodic.md` (Index table only) at task start and write one entry on task complete. Relationships go in `~/.supppeeerrr-harnes/agent-memory/graph.md`. Full format spec in `agent-memory/README.md`.

## Non-negotiables
- Human-in-the-Loop gates are hard stops — never proceed past a trigger without explicit human confirmation
- Episodic entries are immutable once written — add a correction entry, never edit a prior one
- Supervised scratchpad writes always surface to terminal
- Specialists receive work from orchestrator only — no direct agent-to-agent routing
