# Skills Architecture — Research Findings

**Source:** Official Claude Code and Anthropic documentation review, 2026-03-19.

---

## What skills are designed for

Skills are reusable, filesystem-based resources that provide Claude with domain-specific expertise — workflows, context, and best practices that transform general-purpose agents into specialists.

Skills load **on-demand** and run **inline** in the active conversation context. They are not isolated execution environments.

**Skills are the right tool for:**
- Domain knowledge, conventions, and reference material
- Patterns and anti-patterns specific to a technology or framework
- Procedural workflows and checklists
- Standards that apply across multiple sessions and projects

**Skills are NOT designed for:**
- Personality instructions or role-play — those belong in the agent body
- Isolated execution (use subagents for that)
- Tool access restriction (use subagents for that)
- One-off behaviors (use main conversation for that)

---

## What agents (subagents) are designed for

Subagents run in **isolated context windows** with their own system prompt, tool access, and permissions.

**Subagents are the right tool for:**
- Tasks that produce verbose output you don't need in the main context
- Enforcing tool restrictions
- Using a different model for cost or speed
- Self-contained work that returns a summary

**Hard constraint:** Subagents cannot spawn other subagents. Nested delegation must be handled through skills or chaining from the main conversation.

---

## How skills and agents relate

Skills can be preloaded into subagents via the `skills` field in the subagent frontmatter. When preloaded, the full skill content is injected at startup — not just made available for invocation.

The relationship is:
- **Agent body** = persona, identity, core behavior, tone
- **Skills** = domain knowledge, expertise, pluggable patterns

---

## Validated design decisions

| Decision | Status | Notes |
|----------|--------|-------|
| Skills as pluggable expertise enhancers | ✅ Correct use | Explicitly supported by documentation |
| Personality/tone in agent body, not skills | ✅ Correct | Docs: "Skills should contain domain knowledge, not role-play" |
| Stack-specific skills (`fastapi-expert`, `django-expert`) | ✅ Correct | Primary use case for skills |
| `memory-protocol`, `react-loop` as shared skills | ✅ Correct | Reusable domain knowledge pattern |
| Orchestrator routing to specialist agents | ✅ Correct | Matches documented subagent patterns |

---

## Anti-patterns to avoid

1. **Length over clarity** — "More lines ≠ better instructions. Ruthlessly prune." Important rules get lost in noise.
2. **Too many options** — Provide a default with an escape hatch, not a list of equals.
3. **Deep reference nesting** — Keep references one level deep. Claude may partially read deeply nested files.
4. **Personality in skills** — Tone and character belong in the agent system prompt, not in a skill.
5. **Assuming tools are installed** — State dependencies explicitly.

---

## Python engineer design conclusion

| Layer | Content |
|-------|---------|
| `agents/backend-engineer.md` | Core identity + personality (high standards, precise, direct, encouraging) |
| `skills/fastapi-expert.md` | FastAPI patterns, Pydantic, Depends(), router structure, anti-patterns |
| `skills/django-expert.md` | ORM N+1, reversible migrations, MTV, settings split |
| `skills/python-async-expert.md` | asyncio patterns, gather vs sequential, sync-in-async violations |

The agent is the person. The skills are what they have studied.
