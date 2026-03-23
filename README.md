<div align="center">

# 🛠 supppeeerrr-harnes

**A full expert engineering team as a Claude Code plugin**

*17 commands · 18 specialist agents · 16 prompt skills*

![Version](https://img.shields.io/badge/version-0.15.0-blue)
![License](https://img.shields.io/badge/license-MIT-green)
![Claude Code](https://img.shields.io/badge/Claude%20Code-plugin-orange)

</div>

---

Built on Anthropic agentic patterns — **Orchestrator-Workers**, **ReAct**, **Evaluator-Optimizer**, **Fan-Out/Fan-In**, and **Human-in-the-Loop** — this plugin gives you a coordinated team of specialists that work together through a single orchestrator. Every task is routed, reviewed, and documented by the right expert.

---

## 🚀 Getting Started

Clone and load — that's it:

```bash
git clone https://github.com/matt-vail/supppeeerrr-harnes
claude --plugin-dir ./supppeeerrr-harnes
```

The repo ships with the full `agent-memory/` directory structure and `.gitignore` already in place. Nothing to configure.

Then run these two commands once in your project if you want to have a reference map and confirm memory:

```
/map
```
Indexes your codebase so agents don't re-explore on every task.

```
/agent-memory status
```
Confirms memory is active and shows the episodic log.

After that, pick the command that matches your task. All commands route through the orchestrator — you never call agents directly.

---

## 📋 Commands

Commands are the entry points. Each one orchestrates a multi-agent pipeline behind the scenes. The orchestrator routes work, specialists execute, the reviewer gates quality.

### Delivery

| Command | What it does |
|---|---|
| `/generate` | Generate code matching project conventions — detects language, framework, and naming patterns first |
| `/ship` | Full team delivery: design → implement → test → review → document → Definition of Done |
| `/tdd` | Strict **red → green → refactor**. Never skips the failing test phase |
| `/code-review` | Four-lens review of a file or diff: correctness, clarity, structure, test coverage |
| `/debug` | Hypothesis-testing loop — forms a hypothesis before reading a single file |
| `/docs` | Generate or improve README, API docs, inline comments, ADRs, changelog |

#### `/ship` profiles

| Flag | Behaviour |
|---|---|
| `--full` | All 6 stages: design, implement, test, review, document, DoD check |
| `--fast` | Skip design + docs — implement, test, review only |
| `--hotfix` | Targeted security review, CHANGELOG entry required |

**Example:**
```
/ship --fast Add pagination to the /users endpoint
/ship --hotfix Patch the SQL injection in the search query
```

---

### Analysis & Audit

| Command | What it does |
|---|---|
| `/audit` | Full project health check: security, performance, accessibility, code quality, dependencies |
| `/accessibility` | WCAG 2.1 AA audit with before/after fixes and verification steps |
| `/dependencies` | CVE scanning, licence compliance, outdated packages, upgrade plan |
| `/map` | Generate and cache a structural repo map — agents read this instead of re-exploring |

#### `/dependencies` flags

| Flag | Scope |
|---|---|
| `--scope security` | CVE scan only |
| `--scope licences` | Licence compliance only |
| `--scope outdated` | Version drift only |
| `--fix` | Include an upgrade plan in the output |

#### `/map` flags

| Flag | Behaviour |
|---|---|
| *(none)* | Standard map: entry points, domain model, conventions |
| `--quick` | Entry points and directory structure only |
| `--full` | Includes tech debt scan |
| `--refresh` | Rebuild map from scratch, discard cache |

---

### Planning

| Command | What it does |
|---|---|
| `/breakdown` | Vague brief → confirmed scope, user stories, estimates, engineering handoff |
| `/sprint` | Sprint planning: dependency ordering, risk sequencing, velocity-based selection |
| `/roadmap` | Strategic idea → phased engineering roadmap. Kicks off `/research` for unknowns |
| `/research` | Specialist team investigates a complex technical question in parallel |
| `/spike` | Time-boxed exploration with a pursue/abandon recommendation |

#### `/research` flags

| Flag | Behaviour |
|---|---|
| `--quick` | Single specialist, 10-minute box |
| `--deep` | Full fan-out with cross-referencing and confidence ratings |
| `--compare A vs B` | Direct technology/approach comparison |

**Examples:**
```
/research --deep What are the tradeoffs between event sourcing and CQRS for our use case?
/research --compare Celery vs RQ for background job processing in Django
/roadmap Build a multi-tenant billing system with Stripe
```

---

### Operations

| Command | What it does |
|---|---|
| `/incident` | Active incident triage (observability-engineer as commander) or post-mortem |
| `/agent-memory` | Manage persistent memory: `status` · `search <term>` · `archive` · `on` · `off` |

#### `/incident` modes

```
/incident          # Active triage — provide symptoms, get structured response
/incident postmortem TICKET-123   # Generate post-mortem from incident record
```

---

## 💾 Agent Memory

The plugin maintains persistent memory across sessions — agents pick up where they left off. Memory lives in `agent-memory/` inside the plugin repo — it's already there when you clone.

### Memory types

#### Episodic log (`agent-memory/episodic.md`)
An indexed log of every completed task. Each entry records: what agent did it, what changed, which files were affected, and the outcome. Agents read the index at task start to pick up prior context.

```
| # | Date | Agent | Ticket | Category | Summary | Outcome |
|---|------|-------|--------|----------|---------|---------|
| 042 | 2026-03-22 | backend-engineer | AUTH-17 | feature | Added JWT refresh token rotation | ✅ |
```

Entries are **immutable** — to correct one, write a new entry with `Corrects: #NNN`.

#### Graph (`agent-memory/graph.md`)
Tracks relationships between tasks, findings, and decisions. When a security finding is resolved, an edge connects the finding node to the task that fixed it. Useful for understanding why something was done.

```
| From | Relation | To | Note |
|------|----------|----|------|
| TASK-042 | resolves | FINDING-003 | JWT rotation closes the session hijack risk |
```

#### Scratchpads (`agent-memory/scratchpad/`)

Three types of working notes — automatically created and cleaned up by agents:

| Type | Path | Who uses it |
|---|---|---|
| `individual/` | `{agent}-{YYYYMMDD-HHMM}.md` | One agent, one task — private working notes |
| `team/` | `{agent-type}.md` | All instances of the same agent type on a fan-out task |
| `supervised/` | `{TASK-ID}.md` | Orchestrator-monitored — **every write is echoed to your terminal** |

> **Supervised scratchpad writes are surfaced to your terminal by the PostToolUse hook.** This is intentional — it lets you see the orchestrator's working state on complex tasks without having to ask.

#### Config (`agent-memory/config.md`)
Controls whether memory is active. Agents check this before every memory operation.

```
MEMORY_ENABLED: true
```

### Managing memory

```
/agent-memory status              # See enabled state + episodic index + active scratchpads
/agent-memory search auth         # Search across episodic log, graph, and scratchpads
/agent-memory archive             # Move entries older than 30 days to episodic-archive.md
/agent-memory off                 # Disable all memory operations
/agent-memory on                  # Re-enable memory
```

### What's gitignored

Runtime memory files are never committed to your repo:

```
agent-memory/episodic.md
agent-memory/episodic-archive.md
agent-memory/graph.md
agent-memory/repo-map.md
agent-memory/scratchpad/
```

Only `config.md` and `README.md` (the templates) are committed to the plugin repo. Your project's memory stays in your project.

---

## 🤖 Agents

All agents are routed by the `orchestrator` — you never call them directly. Every command resolves to the right specialists for the task.

### Core

| Agent | Role |
|---|---|
| `orchestrator` | Routes work, fans out, synthesises results, manages Human-in-the-Loop gates |
| `reviewer` | Pre-merge quality gate — plan compliance, four-lens review, specialist escalation |
| `code-reviewer` | Developer coaching reviewer — guides understanding, not just fixes |
| `tdd-coach` | Enforces red → green → refactor. 10-cycle evaluator-optimizer |
| `architect` | System design, ADRs, C4 diagrams, technology selection, trade-off analysis |
| `product-manager` | Ambiguous briefs → confirmed scope, user stories, acceptance criteria |

### Engineering

| Agent | Specialisation |
|---|---|
| `backend-engineer` | REST APIs, FastAPI, Django, Flask, Python — N+1, O(n²) fundamentals |
| `frontend-engineer` | TypeScript, React, component architecture, bundle performance |
| `database-engineer` | Schema design, zero-downtime migrations, indexing, query optimisation |
| `api-designer` | REST/GraphQL contracts, OpenAPI specs, versioning, error responses |
| `devops-engineer` | CI/CD, containerisation, IaC (Terraform, Pulumi, CDK), deployment strategies |
| `performance-engineer` | Profiling, caching strategy, N+1 elimination, Web Vitals |

### Quality & Security

| Agent | Specialisation |
|---|---|
| `security-engineer` | OWASP Top 10, auth/authz, input validation, secrets, threat modelling |
| `accessibility-pm` | WCAG 2.1 — keyboard, screen reader, contrast, ARIA |
| `observability-engineer` | Logging, metrics, traces, SLI/SLO, alerting, incident command |
| `dependency-engineer` | CVE scanning, licence compliance, version drift — npm/pip/go/cargo/dotnet |

### Support

| Agent | Specialisation |
|---|---|
| `tech-writer` | README, API docs, inline comments, ADRs, changelog, onboarding guides |
| `code-archaeologist` | Maps unfamiliar codebases: domain model, entry points, conventions, debt |

---

## 🧠 Skills

Skills are prompt libraries loaded into agent context automatically. Not invocable by users — they enforce consistent behaviour across every agent so you don't have to ask for it.

### Always loaded

| Skill | Enforces |
|---|---|
| `communication-protocol` | HANDOFF, ARTIFACT, HUDDLE XML formats between agents; model routing hints (haiku/sonnet/opus per task type) |
| `memory-protocol` | Episodic read/write, graph updates, failure pattern recording, Task Continuity Protocol |
| `human-in-the-loop` | 10 hard-stop gate conditions — never bypassed |
| `react-loop` | Thought → Action → Observation before every conclusion (3-cycle cap, then HitL) |

#### Model routing

The orchestrator automatically selects a model tier for each specialist task:

| Task type | Model |
|---|---|
| Orientation, documentation, single-file review | haiku |
| Standard implementation, multi-file review, TDD | sonnet |
| Security threat modelling, architecture/ADR, complex multi-domain | opus |

### Domain skills

| Skill | Enforces |
|---|---|
| `review-lens` | Four-lens code review: correctness, clarity, structure, tests |
| `findings-template` | Structured finding format: level, location, asset, finding, fix, verify |
| `tdd-discipline` | Red → green → refactor rules and anti-patterns |
| `codegen-patterns` | Convention detection, TODO placement, no gold-plating |
| `adr-format` | Architecture Decision Record structure and lifecycle |
| `definition-of-done` | Five-gate DoD: code quality, tests, documentation, security, operations |
| `session-protocol` | Session resume snapshots, stale scratchpad detection |
| `fastapi-standards` | FastAPI patterns and conventions |
| `django-standards` | Django patterns and conventions |
| `flask-standards` | Flask patterns and conventions |
| `python-standards` | Python language standards |
| `git-workflow` | Branch naming, commit conventions, PR protocol |

---

## 🪝 Hooks

The plugin ships with two PostToolUse hooks in `hooks/hooks.json`:

| Hook | Trigger | What it does |
|---|---|---|
| Test runner | Any `Write` or `Edit` | Placeholder — replace `YOUR_TEST_COMMAND_HERE` with your project's test command |
| Scratchpad echo | Write to `scratchpad/supervised/` | Prints supervised scratchpad content to your terminal so you can monitor orchestrator state |

To activate hooks, copy `hooks/hooks.json` content into your project's `.claude/settings.json` under `"hooks"`. Update the test command placeholder with your actual test runner (e.g. `pytest`, `npm test`).

---

## 👁 What You'll See While It Works

Understanding what the orchestrator surfaces during a task prevents confusion.

### PAUSE gates

When a hard-stop trigger is hit, you'll see this format before any action is taken:

```
PAUSE — Human review required
Trigger:  [which condition was met]
Context:  [what the agent was about to do]
Risk:     [what could go wrong]
Options:  [what you can choose]
Question: [the specific decision needed]
```

Respond to continue, adjust scope, or cancel. The decision is recorded in memory regardless of outcome.

For ambiguous requirements you'll see a lighter version:

```
PAUSE — Clarification needed
Question: [the specific unknown]
Context:  [why it matters]
Options:  [discrete choices if applicable]
```

### HUDDLE in your terminal

On cross-domain decisions (e.g. security vs performance trade-off, novel architecture choice), the orchestrator convenes a multi-agent discussion. You'll see specialists writing structured positions to the supervised scratchpad, each echoed to your terminal by the hook:

```
=== SUPERVISED SCRATCHPAD UPDATE ===
[security-engineer | 14:22]
  <contribution agent="security-engineer">
    <position>Store tokens in httpOnly cookies, not localStorage</position>
    ...
  </contribution>
=== END SUPERVISED SCRATCHPAD ===
```

The orchestrator collects all contributions and closes with a `<decision>` block. If the decision warrants an ADR, it routes to `architect` and `tech-writer` automatically.

### Direction checks on every artifact

Every specialist artifact is checked against the original task intent. You'll see a line at the bottom of each artifact:

```
Direction check: ALIGNED — output fully serves the stated goal
Direction check: PARTIAL — covers goal but missing X
Direction check: DRIFT — output diverges from intent anchor
```

A `DRIFT` result triggers a Human-in-the-Loop gate before the artifact is accepted. The orchestrator won't silently absorb off-track work.

### Findings format

All review agents (security, accessibility, performance, code review) use the same structured output:

```
CRITICAL  | src/auth/jwt.py:42   | Missing signature verification → add verify=True to decode()
HIGH      | api/users.py:18      | N+1 query in list endpoint → prefetch related in queryset
MEDIUM    | components/Form.tsx:7 | No ARIA label on input → add aria-label="Email address"
LOW       | utils/helpers.py:33  | Unused import → remove
NIT       | models/user.py:12    | Variable name 'u' is unclear → rename to 'user'
```

**CRITICAL** — block merge/deploy, fix immediately
**HIGH** — fix before merge, can go to staging
**MEDIUM** — fix next iteration, track in backlog
**LOW** — non-blocking follow-up
**NIT** — style preference, optional

After every findings list, the agent offers to apply CRITICAL and HIGH fixes directly.

### Session resume on long tasks

When a complex task (like `/ship --full` or `/audit`) spans multiple Claude sessions, the orchestrator writes a `SESSION-RESUME` snapshot to the supervised scratchpad at session end:

```
## SESSION-RESUME — TASK-042 — 2026-03-23 17:45
Command: /ship
Profile: --full
Status: IN PROGRESS

Completed stages:
- Design: API contract agreed, OpenAPI spec written
- Implementation: backend complete, frontend in progress

Current stage: Implementation — frontend
Next action: Route to frontend-engineer with HANDOFF referencing OpenAPI spec at artifacts.md

Key decisions made:
- JWT refresh rotation over session cookies (security requirement)
Open HANDOFFs: frontend-engineer (in progress)
```

In the next session, the orchestrator reads only the snapshot — not the full scratchpad — and resumes from "Next action" exactly as specified. You don't need to re-explain the task.

### Context budget adaptation

The orchestrator monitors token usage and adjusts automatically:

| Token usage | Behaviour |
|---|---|
| Under 400k | Normal operation — full artifacts, full orientation |
| 400k–600k | Requests summary-only artifacts from pending specialists |
| Over 600k | Stops starting new specialist invocations, compresses to summaries |
| Over 800k | Writes recovery snapshot, surfaces HitL gate — task may need to continue in a new session |

### Failure mode memory

When a task is blocked or fails, the exact cause is recorded in episodic memory:

```
**Failure mode:** Missing auth middleware on new endpoint — not a dependency issue
```

The next agent touching the same domain or ticket reads this before starting. It won't repeat the same mistake without knowing it was already tried.

---

## 🔍 Reviewer vs Code-Reviewer

These are two different agents — pick the right one:

| Agent | Use when |
|---|---|
| `reviewer` | Automated by commands — pre-merge quality gate, plan compliance, four-lens findings, DoD check. Invoked by `/ship`, `/code-review` |
| `code-reviewer` | Developer coaching — explains *why* something is wrong, teaches, asks questions. Use when you want to learn from the review, not just get a list of fixes |

The orchestrator defaults to `reviewer` for pipeline gates. If you want a coaching session, run `/code-review` and the orchestrator will route to `code-reviewer`.

---

## ✅ Definition of Done

The DoD runs automatically at the end of `/ship` and `/tdd`. It has five gates — not all apply to every change:

| Gate | Always required? | Applies when |
|---|---|---|
| **Code quality** | Yes | Every change |
| **Tests** | Yes | Every change |
| **Documentation** | Conditional | Change is user-visible or modifies a public interface |
| **Security** | Conditional | Change touches auth, input handling, or data persistence |
| **Operations** | Conditional | Change will be deployed to a live environment |

Output looks like:

```
Definition of Done — Add JWT refresh rotation
---
Code quality:    PASS (4/4 checks)
Tests:           PASS (4/4 checks)
Documentation:   PASS (3/3 checks)
Security:        PASS (4/4 checks)
Operations:      N/A (local dev change — no deployment target)
---
Overall: PASS
Blocking items: none
```

A `BLOCKED` result prevents the Ship Summary and the `/tdd` done declaration until all failing checks are resolved. The orchestrator cannot mark a task done with a blocked DoD.

---

## 🏗 Agentic Patterns

| Pattern | Applied in |
|---|---|
| **Orchestrator-Workers** | All commands — single entry point, specialist execution |
| **ReAct** | All agents — Thought → Action → Observation loop (3-cycle cap, then HitL) |
| **Fan-Out / Fan-In** | `/audit`, `/research`, `/ship` Stage 3 — parallel specialists, synthesised output |
| **Evaluator-Optimizer** | `reviewer` (3 cycles max) · `tdd-coach` (10 cycles max) |
| **Human-in-the-Loop** | 10 hard-stop conditions — orchestrator never self-approves |
| **Task Continuity** | Intent Anchor → Checkpoint → Decision Log → Recovery Snapshot |
| **Session Resume** | End-of-session snapshot for multi-session tasks |

### Human-in-the-Loop gates

The orchestrator will **always stop and ask you** before:

1. Any action touching more than 5 files
2. Any file or data deletion
3. Breaking API or interface change
4. CRITICAL security finding from `security-engineer`
5. Architecture change flagged by any specialist
6. Any production environment action
7. Unresolved specialist conflict after one round
8. Task scope expands mid-execution
9. Migration modifying or deleting existing data
10. Deployment without staging validation

These are non-negotiable — the orchestrator cannot self-approve any of them.
