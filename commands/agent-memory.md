# /agent-memory

Manage the agent memory system. Show status, search across memory, archive old entries, or toggle memory on and off.

## Usage

- `/agent-memory` or `/agent-memory status` — show current memory status and episodic index
- `/agent-memory search <term>` — search across episodic log, graph, and scratchpads
- `/agent-memory archive` — move episodic entries older than 30 days to the archive
- `/agent-memory off` — disable all agent memory operations for this and future sessions
- `/agent-memory on` — re-enable agent memory operations

---

## Behaviour

### `/agent-memory` or `/agent-memory status`

**Auto-init:** Before anything else, check whether `~/.supppeeerrr-harnes/agent-memory/config.md` exists.

If it does NOT exist, initialize the directory now:
1. Create `~/.supppeeerrr-harnes/agent-memory/scratchpad/supervised/`, `~/.supppeeerrr-harnes/agent-memory/scratchpad/team/`, and `~/.supppeeerrr-harnes/agent-memory/scratchpad/individual/`.
2. Copy the content from the plugin's own `agent-memory/config.md` to `~/.supppeeerrr-harnes/agent-memory/config.md`.
3. Create empty `~/.supppeeerrr-harnes/agent-memory/episodic.md` and `~/.supppeeerrr-harnes/agent-memory/graph.md` files.
4. Print: "Initialized ~/.supppeeerrr-harnes/agent-memory/ for this project."

Then proceed:

1. Read `~/.supppeeerrr-harnes/agent-memory/config.md` and report the current `MEMORY_ENABLED` value.
2. Read the **Index** table from `~/.supppeeerrr-harnes/agent-memory/episodic.md` and display it.
3. Report the node count from `~/.supppeeerrr-harnes/agent-memory/graph.md`.
4. List any active scratchpad files in `~/.supppeeerrr-harnes/agent-memory/scratchpad/supervised/` and `~/.supppeeerrr-harnes/agent-memory/scratchpad/team/`.

Output format:
```
Memory status: ENABLED / DISABLED

Episodic log: N entries
[index table]

Graph: N nodes, N edges

Active scratchpads: [list or "none"]
```

### `/agent-memory search <term>`

Search the full memory system for any entry mentioning the given term.

1. Search `~/.supppeeerrr-harnes/agent-memory/episodic.md` — return matching Index rows and the full entry body for each match.
2. Search `~/.supppeeerrr-harnes/agent-memory/graph.md` — return matching nodes and edges.
3. Search all files in `~/.supppeeerrr-harnes/agent-memory/scratchpad/supervised/` — return file name and matching lines.
4. Search all files in `~/.supppeeerrr-harnes/agent-memory/scratchpad/team/` — return file name and matching lines.
5. Search all files in `~/.supppeeerrr-harnes/agent-memory/scratchpad/individual/` — return file name and matching lines.

Output format:
```
Search: "<term>"

Episodic matches: N
  [entry ID] [date] [agent] — [summary line containing the term]

Graph matches: N
  [node or edge line containing the term]

Scratchpad matches: N
  [filename]: [matching line]
```

If no matches: "No memory entries found for '<term>'."

Search is case-insensitive. The term may be a developer name, ticket ID, codebase area, or any keyword.

### `/agent-memory archive`

Move episodic entries older than 30 days to the archive to keep the active Index table readable.

1. Read the **Index** table from `~/.supppeeerrr-harnes/agent-memory/episodic.md`.
2. Identify rows with a date older than 30 days from today.
3. For each identified row:
   - Read the full entry body from `~/.supppeeerrr-harnes/agent-memory/episodic.md`.
   - Append the entry to `~/.supppeeerrr-harnes/agent-memory/episodic-archive.md` (create the file if it does not exist, with a `# Episodic Archive` heading).
   - Remove the Index row and the full entry body from `~/.supppeeerrr-harnes/agent-memory/episodic.md`.
4. Report what was archived.

Output format:
```
Archived N entries older than [cutoff date]:
  [entry ID] [date] [agent] — [summary]

Active episodic log now contains N entries.
Archive: ~/.supppeeerrr-harnes/agent-memory/episodic-archive.md
```

If nothing is old enough to archive: "Nothing to archive. All episodic entries are within the 30-day window."

**Safety rule:** Never delete entries — always move them to the archive. Archived entries are still searchable with `/agent-memory search`.

### `/agent-memory off`

1. Read `~/.supppeeerrr-harnes/agent-memory/config.md`.
2. Set `MEMORY_ENABLED: false`.
3. Confirm: "Agent memory disabled. Agents will skip all memory reads and writes. Existing memory files are preserved. Run `/agent-memory on` to re-enable."

### `/agent-memory on`

1. Read `~/.supppeeerrr-harnes/agent-memory/config.md`.
2. Set `MEMORY_ENABLED: true`.
3. Confirm: "Agent memory enabled. Agents will read episodic index at task start and write entries on task complete."

---

## Notes

- Turning memory off does not delete any files. All episodic entries, graph relationships, and scratchpad files are preserved.
- Turning memory back on resumes from where it was left off — no data is lost.
- To permanently clear memory, manually delete the contents of `~/.supppeeerrr-harnes/agent-memory/episodic.md` entries, `~/.supppeeerrr-harnes/agent-memory/graph.md` edges, and any scratchpad files. The config file and README are never cleared.
- Archived entries remain part of the permanent record. They are not included in agent startup reads (which only read the active Index) but are available via `/agent-memory search`.
