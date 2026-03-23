# Agent Tools

The tools an agent has access to define what it can actually do. An agent without the right tools is a planner with no hands.

**Primary sources:** Anthropic tool use documentation; Anthropic "Building Effective Agents" (ACI section); Weng (2023) tool taxonomy; Anthropic SWE-bench agent case study.

> "In our experience, the most important factor in agent performance is not the model or the prompt — it's the tool design."
> — Anthropic, "Building Effective Agents"

---

## The Agent-Computer Interface (ACI)

Anthropic introduced the concept of the **Agent-Computer Interface** — the design of tools for agents is as important as API design for developers. A poorly designed tool produces agent errors. A well-designed tool prevents them.

### ACI design principles (Anthropic)

**1. Document like a senior dev writing for a junior colleague.**
Every tool definition needs:
- What it does.
- When to use it vs. a similar tool.
- Example usage.
- Edge cases and failure modes.
- Input format requirements.

**2. Use Poka-yoke design — make wrong usage hard to express.**
> "Change the arguments so that it is harder to make mistakes."
Example: absolute file paths instead of relative paths. The agent cannot accidentally reference the wrong file.

**3. Keep formats close to what the model has seen in training.**
Unusual formats require the agent to translate. Standard formats (JSON, markdown, Python) are natively understood. Use them.

**4. Give the agent enough tokens to think before it commits.**
A tool that forces an immediate action without room for reasoning produces hasty errors. Design tools that accept deliberate inputs, not reactive ones.

**5. Don't make the agent count or track large numbers.**
Asking an agent to "edit line 4,721" is error-prone. Tools that operate on semantic anchors (function names, patterns) are more reliable than line numbers.

**Anthropic case study:** In their SWE-bench agent, the team spent more time optimising tool design than overall prompts. The biggest single improvement was switching from relative to absolute file paths — it eliminated a whole class of "wrong file" errors.

---

## Tool taxonomy

### Category 1: File system tools

Read, write, search, and navigate the project's files.

| Tool | What it does | When agents use it |
|------|-------------|-------------------|
| `read_file(path)` | Read file contents | Before any edit; reading conventions |
| `write_file(path, content)` | Create or overwrite a file | Generating new files |
| `edit_file(path, old, new)` | Surgical string replacement | Targeted edits without full rewrites |
| `list_directory(path)` | List files in a directory | Understanding project structure |
| `search_files(pattern, glob)` | Regex/glob search across files | Finding usages, patterns, conventions |
| `get_file_info(path)` | File metadata, last modified | Checking recency for convention detection |

**ACI note:** Always use absolute paths. Never relative paths. The agent should never have to calculate what the current working directory is.

---

### Category 2: Code execution tools

Run code and get results. The most powerful and most dangerous tool category.

| Tool | What it does | Security considerations |
|------|-------------|------------------------|
| `run_command(cmd)` | Execute a shell command | Sandbox required; never allow arbitrary input to reach here |
| `run_tests(suite)` | Execute the test suite | Low risk; read the output carefully |
| `run_linter(files)` | Execute linter on files | Read-only; safe |
| `run_type_checker(files)` | Run mypy, tsc, etc. | Read-only; safe |
| `run_formatter(files)` | Auto-format code | Writes files; confirm before running on large sets |
| `execute_python(code)` | Run a Python snippet | Sandboxed only; high risk if unsandboxed |

**Security rule:** Code execution tools must be sandboxed. An agent that can run arbitrary code can access the file system, network, and environment. Never give an agent a raw shell execution tool without constraints.

**Anthropic guidance:** "Prefer reversible over irreversible actions." Linting and type-checking are reversible (read-only). File writes are reversible if version controlled. Shell commands may not be.

---

### Category 3: Search and retrieval tools

Find information the agent doesn't have in context.

| Tool | What it does | Best for |
|------|-------------|---------|
| `web_search(query)` | Search the web | Current documentation, CVEs, API references |
| `vector_search(query, store)` | Semantic search over a knowledge base | Project history, past decisions, episodic memory |
| `fetch_url(url)` | Fetch and parse a web page | Reading specific documentation pages |
| `search_codebase(pattern)` | Find usages across the codebase | Convention detection, refactoring impact |

**Retrieval quality note:** The quality of retrieval tools determines the quality of retrieved context. A semantic search that returns irrelevant chunks misleads the agent. Monitor retrieval precision.

---

### Category 4: Code analysis tools (static and dynamic)

Inspect code without running it (static) or with controlled execution (dynamic).

| Tool | What it does | Agents that use it |
|------|-------------|-------------------|
| `get_ast(file)` | Parse file into AST | Structural analysis, refactoring |
| `get_call_graph(entry)` | Map function call dependencies | Performance, security surface |
| `get_test_coverage(suite)` | Coverage report per file | reviewer, tdd-coach |
| `explain_query(sql)` | EXPLAIN ANALYZE output | performance-engineer |
| `diff(before, after)` | Show changes between two states | reviewer, orchestrator |

---

### Category 5: Memory tools

Read from and write to the agent's persistent memory stores.

| Tool | What it does |
|------|-------------|
| `memory_store(content, tags)` | Write to long-term memory |
| `memory_search(query)` | Semantic search over memory |
| `memory_get(key)` | Retrieve a specific memory by key |
| `episode_log(task, outcome, lesson)` | Record an episode for future retrieval |
| `episode_search(situation)` | Find past episodes similar to current situation |

---

### Category 6: Agent spawning and delegation tools

The orchestrator's coordination tools.

| Tool | What it does |
|------|-------------|
| `spawn_agent(agent_name, task, inputs)` | Delegate a subtask to a specialist |
| `await_artifact(agent_name, artifact_name)` | Wait for and retrieve a specialist's output |
| `broadcast(agents, task, inputs)` | Fan-out: send the same task to multiple agents simultaneously |
| `request_human_review(context, question)` | Trigger the Human-in-the-Loop gate |

---

### Category 7: Specialised domain tools

Domain-specific tools that give specialist agents capabilities beyond general coding.

| Tool | Agent | What it does |
|------|-------|-------------|
| `run_axe(url_or_html)` | accessibility-pm | Automated WCAG accessibility scan |
| `check_contrast(fg, bg)` | accessibility-pm | Compute colour contrast ratio |
| `run_dependency_audit()` | security-engineer | Check for known CVEs in dependencies |
| `scan_secrets(path)` | security-engineer | Detect hardcoded credentials |
| `run_sast(files)` | security-engineer | Static application security testing |
| `validate_openapi(spec)` | api-designer | Validate OpenAPI 3.1 spec |
| `profile_endpoint(url)` | performance-engineer | Measure response time and resource usage |
| `analyze_bundle(entrypoint)` | performance-engineer | Bundle size and composition report |
| `check_links(doc)` | tech-writer | Verify all links in a document are valid |

---

## Per-agent tool mapping

What each agent in this plugin needs access to.

### orchestrator

```
Core:          read_file, list_directory, search_files
Delegation:    spawn_agent, await_artifact, broadcast, request_human_review
Memory:        memory_search, episode_search
Communication: diff (to compare before/after artifacts)
```

The orchestrator does not need code execution tools — it does not run code itself.

---

### backend-engineer

```
File system:   read_file, write_file, edit_file, search_files
Execution:     run_tests (pytest), run_linter (ruff), run_type_checker (mypy),
               run_formatter (black/ruff format), run_command (scoped)
Analysis:      get_test_coverage, search_codebase
Memory:        memory_search (for past Python patterns), episode_search
```

---

### frontend-engineer

```
File system:   read_file, write_file, edit_file, search_files
Execution:     run_tests (jest/vitest), run_linter (eslint), run_type_checker (tsc),
               run_formatter (prettier), run_command (npm/yarn, scoped)
Analysis:      analyze_bundle, get_test_coverage, search_codebase
Memory:        memory_search (for React patterns), episode_search
```

---

### accessibility-pm

```
File system:   read_file, search_files
Specialised:   run_axe, check_contrast
Analysis:      search_codebase (find ARIA attributes, form labels, focus handlers)
Memory:        memory_search (past a11y findings), episode_search
```

Does not need write or execution tools — it reviews and advises.

---

### security-engineer

```
File system:   read_file, search_files
Execution:     run_dependency_audit, scan_secrets, run_sast
Analysis:      search_codebase (find injection surfaces, auth checks, crypto usage),
               get_call_graph (map trust boundaries)
Memory:        memory_search (past security findings), episode_search
Web:           web_search (CVE lookup, OWASP references)
```

---

### api-designer

```
File system:   read_file, write_file (OpenAPI spec), search_files
Specialised:   validate_openapi
Analysis:      search_codebase (find existing endpoints and patterns)
Web:           fetch_url (OpenAPI spec references, RFC documents)
Memory:        memory_search (past API design decisions)
```

---

### performance-engineer

```
File system:   read_file, search_files
Execution:     run_command (profiler, benchmark), profile_endpoint
Specialised:   explain_query, analyze_bundle
Analysis:      search_codebase (find N+1 patterns, missing indexes),
               get_call_graph (identify hot paths)
Memory:        memory_search (past performance baselines)
```

---

### tech-writer

```
File system:   read_file, write_file, edit_file, search_files
Specialised:   check_links
Analysis:      diff (compare docs to code for accuracy), search_codebase
Memory:        memory_search (past doc decisions, ADR numbering)
```

Does not need code execution tools.

---

### tdd-coach

```
File system:   read_file, write_file, edit_file, search_files
Execution:     run_tests (with full output capture), get_test_coverage
Analysis:      diff (before/after each cycle)
Memory:        episode_search (past TDD sessions for this codebase)
```

---

### reviewer

```
File system:   read_file, search_files
Execution:     run_tests, run_linter, run_type_checker, get_test_coverage
Analysis:      diff (the change under review), search_codebase, get_call_graph
Memory:        episode_search (past review findings for this codebase)
```

---

### product-manager

```
File system:   read_file, write_file (stories, acceptance criteria)
Analysis:      search_codebase (understand existing domain model)
Memory:        memory_search (past project decisions), episode_search
```

Does not need execution tools.

---

## Tool security principles

1. **Minimal permissions.** Each agent gets only the tools it needs. The `tech-writer` does not need `run_command`. The `accessibility-pm` does not need `write_file`.

2. **Sandboxed execution.** Any tool that runs code does so in a sandbox with no access to production systems, secrets, or the network except through controlled APIs.

3. **Reversibility preference.** Read tools before write tools. Write tools before execute tools. When in doubt, prefer the more reversible option.

4. **Human gate for destructive tools.** Any tool that deletes files, drops database tables, or executes arbitrary shell commands should trigger Human-in-the-Loop before running.

5. **Audit logging.** Every tool call is logged with: which agent called it, what arguments were passed, and what the result was. This is the foundation of the error recovery and learning system.

---

## Sources

| Source | URL |
|--------|-----|
| Building Effective Agents (ACI section) — Anthropic | anthropic.com/research/building-effective-agents |
| Tool use overview — Anthropic docs | docs.anthropic.com/en/docs/tool-use |
| LLM Powered Autonomous Agents (tools section) — Weng (2023) | lilianweng.github.io/posts/2023-06-23-agent/ |
| ChemCrow: Augmenting LLMs with Chemistry Tools | arxiv.org/abs/2304.05332 |
