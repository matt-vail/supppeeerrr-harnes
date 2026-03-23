# ADR-003: Adopt Compress-and-Externalize, Context Budget Thresholds, and Task Continuity Checkpointing as the Plugin's Context Management Strategy

**Date:** 2026-03-22
**Status:** Accepted
**Deciders:** orchestrator, architect
**Supersedes:** N/A

---

## Context

A structured research session (2026-03-22) investigated Claude model limits, how context fills in agentic sessions, and what the plugin's current design does or does not do to manage context effectively. Four findings drove this decision:

**Finding 1 — Context rot is the operative risk, not the hard limit.**

Claude Sonnet 4.6's 1M token context window is rarely the binding constraint on task quality. Anthropic documents "context rot": accuracy and recall degrade measurably as context depth increases, well before the 1M ceiling is reached. A `/audit` or `/ship` run on a medium codebase accumulates approximately 32,000 tokens across orchestrator instructions, orientation reads, specialist HANDOFFs, ARTIFACTs, and synthesis — comfortably within the window. But on a large codebase where each specialist reads many source files, totals of 100,000–200,000 tokens are reached. At this depth, context rot produces degraded synthesis quality before any hard limit is hit. The plugin had no mechanism for managing context density proactively.

**Finding 2 — A four-specialist fan-out costs ~10,500 tokens in ARTIFACTs alone.**

Each specialist ARTIFACT in a `/audit` fan-out (security-engineer, performance-engineer, accessibility-pm, reviewer) averages 1,400–5,500 tokens. Without compression, all four ARTIFACTs accumulate in the orchestrator's working context and remain there for the rest of the session. The orchestrator does not need all four ARTIFACT bodies in context simultaneously — it needs each body during evaluation and specific sections during synthesis. Retaining full ARTIFACTs in working context is pure overhead after those moments.

**Finding 3 — The `triggers` frontmatter key is not a documented Anthropic Skills architecture feature.**

The plugin's skill files use a custom `triggers` key with keyword phrase lists intended to auto-activate skills. The official Anthropic Skills architecture uses the `description` field as Level 1 metadata (~100 tokens, always loaded) and loads the skill body (Level 2) when Claude determines relevance. The `triggers` key has no documented operational meaning as a pre-load filter — trigger phrases are read only after the skill body has already been loaded. Three skills that define constitutional behaviors — `communication-protocol.md`, `memory-protocol.md`, and `human-in-the-loop.md` — rely on this undocumented mechanism for activation. The failure mode is silent omission: if trigger phrases do not appear in the session context, the constitutional behaviors are not applied. An agent that does not load `communication-protocol.md` returns prose instead of a structured ARTIFACT, and the orchestrator's evaluation gate cannot detect this.

**Finding 4 — No within-task continuity mechanism existed.**

For long-running multi-phase tasks, the orchestrator held all directional state — the original goal, key decisions, completed phases — exclusively in the context window. Context rot and server-side compaction could both degrade or lose this state without any signal to the orchestrator that direction had been lost. The peer review confirmed this as a structural gap: agents drift from original goals, forget decisions, and repeat work on complex tasks. No mechanism existed to maintain direction across context pressure or session boundaries.

---

## Decision

We will adopt three complementary strategies as the plugin's context management approach:

1. **Compress-and-externalize.** After receiving each specialist ARTIFACT, the orchestrator extracts a ~30-token summary to working context, writes the full ARTIFACT to the supervised scratchpad on disk, and discards the verbose content from active working memory. Full ARTIFACT sections are re-read from disk during synthesis only when needed. This reduces a four-specialist fan-out from ~10,500 tokens in orchestrator context to ~120 tokens.

2. **Context budget thresholds.** The orchestrator acts on the `Token usage: N/1,000,000; N remaining` signal injected by the API after each tool call. Four tiers define graduated responses: under 400k (standard operation), 400k–600k (request summary ARTIFACTs from pending specialists), 600k–800k (complete current fan-out only, synthesize from summaries, surface context constraint in output), over 800k (write recovery snapshot, Human-in-the-Loop gate).

3. **Task continuity checkpointing.** The orchestrator writes an intent anchor to the supervised scratchpad before any specialist is invoked. Progress checkpoints are written after each major phase. Key decisions are logged immediately when made. At the 800k threshold, a context recovery snapshot is written. These records are persistent, filesystem-based, and readable by any future context window — they maintain directional continuity regardless of context pressure.

---

## Rationale

**Why compress-and-externalize over keeping full ARTIFACTs in context:** The orchestrator evaluates each ARTIFACT once (did it meet the definition of done?) and references specific sections during synthesis. The full body is not needed continuously. Keeping 10,500 tokens of ARTIFACT body in context for the duration of a session serves no purpose after evaluation is complete and deepens context rot for all subsequent reasoning. Compress-and-externalize retains everything on disk while reducing working context cost by ~99% for the ARTIFACT component.

**Why context budget thresholds over reactive management:** Without explicit thresholds, the orchestrator has the token usage signal but no instruction for what to do with it. The thresholds convert an ambient signal into a graduated action protocol. The four-tier model is calibrated for Sonnet 4.6's 1M window: the 600k threshold leaves 400k for remaining specialist work and synthesis output; the 800k threshold leaves 200k of headroom before the hard limit, which is sufficient for recovery snapshot writing and Human-in-the-Loop gate interaction but not for a new full fan-out. The thresholds deliberately avoid triggering too early — under 400k, no special action is required because context rot at that depth is minimal.

**Why filesystem-based task continuity over relying on context retention:** The context window is subject to context rot and compaction. The filesystem is not. State that must survive context pressure must go to disk. The task continuity hierarchy (CLAUDE.md → episodic → graph → supervised scratchpad → individual scratchpad) is already present in the plugin's architecture; what was missing was the operational protocol for what to write, when to write it, and how to use it for direction recovery. Checkpointing formalizes this protocol without requiring new infrastructure.

**Why explicit loading for constitutional skills over trigger-based activation:** The failure mode of trigger-based activation for constitutional behaviors (silent omission) is categorically worse than the alternative (deterministic loading). A deterministically loaded `communication-protocol.md` costs ~1,400 tokens on every invocation; a silently-not-loaded `communication-protocol.md` costs the orchestrator an ARTIFACT it cannot evaluate and a specialist invocation it must repeat. The token cost of deterministic loading is a fixed, predictable expense. The cost of silent omission is an unpredictable failure of inter-agent coordination.

---

## Alternatives considered

| Option | Why rejected |
|--------|-------------|
| Rely on server-side compaction only | Server-side compaction is a beta feature as of March 2026. It operates automatically and does not give the orchestrator visibility into what was compacted. Compaction that removes a key decision or intent anchor from context without the orchestrator knowing produces the exact goal drift the task continuity protocol is designed to prevent. Compaction is a safety net, not a strategy. |
| Reduce agent file sizes only | The research identified specific files that can be trimmed (reviewer.md at 5,500 tokens, code-reviewer.md at 4,000 tokens). These are valuable improvements, but trimming agent files addresses startup token cost, not the accumulation cost of running a multi-specialist fan-out. After 6 specialists return ARTIFACTs, even perfectly-sized agent files do not help with the 18,000 tokens of ARTIFACT content in working context. File size reduction is complementary, not a substitute. |
| Split complex tasks into smaller independent sessions | Splitting tasks avoids context accumulation but eliminates the orchestrator's ability to synthesize across specialists. A `/audit` that fires security-engineer in one session and performance-engineer in another cannot produce a unified risk-prioritized synthesis — there is no orchestrator context that holds both sets of findings. The architectural value of the fan-out/fan-in pattern is precisely that synthesis happens in a single orchestrator context. Task splitting sacrifices this value. |
| Do nothing — context is large enough | Sonnet 4.6's 1M window is large enough that most tasks will complete without hitting the hard limit. However, the research finding is that context rot — not the hard limit — is the operative risk. A task that completes but produces degraded synthesis output because the orchestrator was reasoning at 400k context depth is a task failure. The 1M window does not protect against context rot. Additionally, as the episodic index grows and as the plugin is used on larger codebases, the margin will shrink. Establishing the protocol now, while the margin is comfortable, is lower risk than waiting until the margin is tight. |

---

## Consequences

**Positive:**
- Orchestrator context costs for ARTIFACT content drop by ~99% per specialist when compress-and-externalize is applied, reducing context rot on complex multi-specialist tasks.
- Goal drift and decision amnesia on long tasks are addressed by the task continuity protocol — the intent anchor, decision log, and progress checkpoints maintain direction externally.
- Constitutional skill failures (silent omission of HANDOFF/ARTIFACT format, memory protocol, Human-in-the-Loop gates) are converted from probabilistic failures to deterministic, detectable failures by moving to explicit loading.
- Tasks that approach the 800k threshold do not fail silently — the recovery snapshot and Human-in-the-Loop gate provide a defined recovery path.
- The context budget thresholds give the orchestrator a graduated response protocol that degrades gracefully: full output → summary output → synthesize from current ARTIFACTs → recovery gate.

**Negative / Trade-offs:**
- Compress-and-externalize adds orchestrator steps (write full ARTIFACT to disk, replace with summary) that were not previously required. This is additional orchestrator work per specialist return.
- Deterministic loading of constitutional skills adds ~3,800 tokens per agent session startup (communication-protocol: ~1,400; memory-protocol: ~800; human-in-the-loop: ~1,600). This is a fixed cost increase on every invocation.
- Progress checkpointing and decision logging add orchestrator writes during task execution. These are low-cost individually (~200 tokens each) but are additional steps that agents must reliably perform.
- The task continuity protocol requires the orchestrator to maintain discipline across a long session — writing checkpoints, logging decisions, re-reading the intent anchor before synthesis. This relies on the orchestrator's instructions being specific enough to enforce this discipline.

**Risks:**
- If the orchestrator's compressed ARTIFACT summary omits a critical finding, that finding may not receive appropriate weight in synthesis. Mitigation: the compress-and-externalize format explicitly includes the highest-severity findings by level (CRITICAL, HIGH) in the summary.
- If the 400k threshold for requesting summary ARTIFACTs is reached early in a complex task (e.g., on a very large codebase with dense orientation reads), all specialists may return summary ARTIFACTs. For tasks where full evidence is required (e.g., a security audit report going to an external party), the orchestrator must override the summary ARTIFACT default and explicitly request full output.
- If the supervised scratchpad grows very large (many phases, many decisions, many specialists), re-reading it for orientation may itself become a context cost. Mitigation: the recovery snapshot is designed to be a self-contained compact summary that replaces full scratchpad re-reads when context is pressured.

---

## Review trigger

- Significant model capability improvements that raise the threshold at which context rot becomes measurable (e.g., if a future model maintains quality to 500k with no degradation, the 400k threshold should be re-evaluated).
- New compaction features from Anthropic that provide orchestrator visibility into what was compacted — if compaction becomes transparent and resumable, the task continuity protocol may be simplified.
- When the episodic index reaches 100 entries — review the 50-row threshold rule and adjust the filtered-read strategy if needed.
- Observed failure of the compress-and-externalize pattern to preserve critical findings through the summary (two or more instances where a CRITICAL finding was in a full ARTIFACT but absent from synthesis because the summary omitted it).
