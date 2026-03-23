# Agent Memory Configuration

MEMORY_ENABLED: true

---

## Settings

**MEMORY_ENABLED** — controls whether agents read from or write to any memory file.

- `true` — agents read episodic index at task start, write entries on task complete, maintain graph relationships, and use scratchpads as needed.
- `false` — agents skip all memory operations. No reads, no writes, no scratchpad creation. Existing memory files are preserved but ignored.

To change: edit `MEMORY_ENABLED` above, or run `/memory on` / `/memory off`.
