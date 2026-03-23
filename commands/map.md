# /map — Repository Map

Generate a structured orientation report for the current codebase and cache it for agent use. Other agents read this map before exploring the codebase — it replaces ad-hoc file reading with a shared, validated index.

## When to use

- First time working in a new codebase — run this before any other command.
- After a significant architectural change that makes the cached map stale.
- Before running `/audit`, `/ship`, or any command that touches multiple parts of the codebase.
- When an agent reports needing orientation and no current map exists.

## What it produces

`code-archaeologist` explores the codebase and writes a structured report to `agent-memory/repo-map.md`. This file is the shared codebase index — all agents check it at task start before doing their own exploratory reads.

The report contains:
- What the system does
- Entry points
- Domain model (core entities and relationships)
- Architecture shape
- Technology map
- Conventions in force
- Test coverage summary
- Known debt and risk signals
- Where to start for common task types

## Depth levels

| Flag | Depth | When to use |
|------|-------|-------------|
| *(none)* | Standard (Steps 1–4) | Default — orientation, entry points, domain model, conventions |
| `--quick` | Quick (Steps 1–2) | Fast orientation for small or well-understood repos |
| `--full` | Full (Steps 1–5) | Includes debt and risk scan — use before `/audit` or a large refactor |
| `--refresh` | Standard + invalidates cache | Forces a new map even if one already exists |

## Cache behaviour

On every run:
1. Check if `agent-memory/repo-map.md` exists.
2. If it exists and `--refresh` is not set: report the map age and ask whether to refresh or use the existing map.
3. If it does not exist or `--refresh` is set: run the full exploration and write the new map.

The map file includes a `Generated:` timestamp at the top so agents and developers can judge freshness.

## Usage

```
/map
/map --quick
/map --full
/map --refresh
```

### Examples

```
/map                   # standard map, use cache if it exists
/map --refresh         # force regeneration even if a map exists
/map --full            # full map including debt scan, before a major audit
/map --quick           # fast map for a small or familiar repo
```

## Output

The map is written to `agent-memory/repo-map.md` and a summary is shown in the terminal. Subsequent agent runs read from that path instead of re-exploring the codebase.

## Notes

- Running `/map` before `/audit`, `/ship`, or `/code-review` significantly reduces context usage — agents read the cached map instead of re-exploring independently.
- The map is a snapshot. After a large refactor or when the codebase adds new modules, run `/map --refresh`.
- `agent-memory/repo-map.md` is gitignored — it is a local working file, not a committed artefact.
- Agents that receive a HANDOFF for an unfamiliar codebase should check whether `agent-memory/repo-map.md` exists before starting their own exploration. If a recent map exists, read it instead of running a full Step 1–5 sequence.
