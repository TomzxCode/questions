# Can you inspect sessions with agentsview (CLI)?

## Answer

Not directly at first: the `agentsview` binary was not on PATH, but the daemon was running with a live SQLite archive at `~/.agentsview/sessions.db`, so sessions could be inspected by querying that database with `sqlite3` (tables: `sessions`, `messages`, `messages_fts`, `tool_calls`, etc.).

## Follow-up: running via uvx

`uvx agentsview` works (v0.40.1). The first call failed with "daemon API version 6 is incompatible with client API version 5"; running `uvx agentsview daemon restart` upgraded the database (full resync of 2508 sessions) and afterwards `uvx agentsview session list` worked.

Useful commands:

- `uvx agentsview session list [--limit N]` — list sessions (one-shot sessions excluded by default, `--include-one-shot` to include)
- `uvx agentsview session get <id>` — metadata and signals
- `uvx agentsview session messages <id>` — message window
- `uvx agentsview session search` — search message/tool content
- `uvx agentsview session usage <id>` — token usage and cost
- `uvx agentsview daemon status` — background server status (web UI at http://127.0.0.1:8080)

## Follow-up: what else can be done from the CLI

Grouped from `--help` (v0.40.1):

- Browsing/analysis: `session get|messages|search|tool-calls|usage|export|watch` (per-session detail, streaming NDJSON, raw JSONL export), `projects` (session counts per project), `stats` and `activity report` (window-scoped analytics and concurrency), `health` (per-session grade A-F and outcome signals).
- Cost tracking: `usage statusline` (one-line today cost), `usage daily` (tokens/cache/cost per day per model), `usage cursor` (ingest Cursor admin usage events).
- Knowledge/recall: `recall extract|list|query|brief|stats` (build a recallable knowledge base from past transcripts), `embeddings build|list|activate|retire` (semantic search index).
- Sync/export/remote: `sync`, `daemon start|stop|status|restart`, `mcp` (read-only MCP server so agents can query sessions), `export day|hour|digest|sessions`, `pg push|serve|service` and `duckdb push|serve` (PostgreSQL/DuckDB mirrors), remote hosts via `[[remote_hosts]]` in `~/.agentsview/config.toml`.
- Maintenance/safety: `secrets scan|list` (leaked secret detection, redacted output), `prune` (delete sessions by filter), `import`, `parse-diff` (re-parse and diff transcripts), `doctor`, `openapi`.
