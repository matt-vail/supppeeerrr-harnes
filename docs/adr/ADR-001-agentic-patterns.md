# ADR-001: Adopt Anthropic Agentic Patterns as the Architectural Foundation

Date: 2026-03-17
Status: Accepted

---

## Context

The plugin started with three agents (code-generator, tdd-coach, reviewer) written from intuition. As the team grew to eleven agents, two problems became clear:

1. **No shared recovery protocol.** When an agent hit uncertainty — ambiguous input, missing context, conflicting constraints — there was no structured path forward. Agents would either stall or hallucinate through the problem.

2. **No architectural pattern.** The `code-generator` was trying to be both a router and a worker. The `reviewer` and `tdd-coach` had implicit loops with no exit conditions. The team had no defined communication model.

A quality review against three criteria (clear instructions, domain standards, happy path for hard situations) confirmed that the "happy path" dimension was failing across every agent.

---

## Decision

Adopt the following patterns from Anthropic's published research as the architectural foundation, supplemented by MongoDB's pattern taxonomy:

| Pattern | Source | Applied to |
|---------|--------|-----------|
| ReAct (Thought → Action → Observation) | Yao et al. 2022 / Anthropic | Every agent — internal reasoning + hard-situation recovery |
| Orchestrator-Workers | Anthropic | `orchestrator` + all specialists |
| Fan-Out / Fan-In | Anthropic | `/audit`, `/ship` multi-specialist stages |
| Evaluator-Optimizer | Anthropic | `reviewer`, `tdd-coach`, orchestrator evaluation gate |
| Human-in-the-Loop | Anthropic | `orchestrator` — 7 defined trigger conditions |
| LLM as Router | MongoDB | `orchestrator` — Step 1 classification |

**`code-generator` was renamed `orchestrator`** and rebuilt to embody the Orchestrator-Workers pattern. It no longer writes code — it decomposes, routes, evaluates, and synthesizes.

**Human-in-the-Loop triggers** were set at: >5 files, any deletion, breaking API change, CRITICAL security finding, architecture change, unresolved specialist conflict, mid-task scope expansion.

---

## Consequences

**Easier:**
- Every agent has a defined recovery path when it gets stuck (ReAct hard situations).
- Evaluator loops have explicit exit conditions (no infinite loops).
- The orchestrator's role is unambiguous — strategic coordination, not implementation.
- New agents can be added following the same structural template.
- Human oversight is defined in advance, not ad hoc.

**Harder:**
- More complex to reason about for simple tasks — a single `/gen` command now routes through the orchestrator before reaching a specialist.
- The fan-out pattern requires the orchestrator to manage parallel workstreams, which adds coordination overhead.
- Evaluator-Optimizer loops mean some tasks take more cycles than a single-pass approach.

**Trade-off accepted:**
The added complexity is justified for the quality gain. A plugin that silently produces wrong output on hard tasks is worse than one that takes an extra cycle and gets it right.

---

## Reference

See `docs/agentic-patterns.md` for the full pattern reference and source citations.
