# Operations — simple_thinking_tool

This document covers running, configuring, and operating the `simple_thinking_tool` MCP server. It is written against the pinned source at commit `2609fe5979e7d4e62024b2be103e81ee9230c172` (2026-08-05). Claims are tied to evidence IDs; where the repository evidence does not support a statement, that is noted explicitly.

## Scope and architecture in brief

`simple_thinking_tool` is a FastMCP server that exposes 15 named thinking methods as individual MCP tools (`think_<id>`), plus three non-mode tools: `health`, `definitions`, and `journal` (E006, E009). The server never calls an LLM and makes no outbound network calls; it fills a mode template from caller-provided string fields and appends the rendered note to a durable JSONL log (E006, E009).

The server is designed as a single-serve private instance — one copy per agent — with no multi-tenancy or shared state (E001, E009).

## Installation and dependencies

- The package uses a standard `src/` layout with setuptools packaging via `pyproject.toml` (E011).
- The only declared third-party runtime dependency is `fastmcp` (E011, E022). `requirements.txt` pins this dependency (E022).
- Python 3.11+ is required (E011, E001).
- Development install is `pip install -e .` (E001, E011).

## Configuration

All configuration is read from environment variables; there are no hardcoded secrets (E003). The `Settings` class in `src/simple_thinking_tool/config.py` reads the following variables (E003):

| Environment variable | Default | Purpose |
|---|---|---|
| `NUT_THINKING_TOKEN` | `""` | Agent bearer token (E003) |
| `NUT_THINKING_BOSS_TOKEN` | `""` | Boss bearer token (E003) |
| `NUT_THINKING_STATE_DIR` | `state` | Directory for the JSONL store (E003) |
| `NUT_THINKING_HOST` | `127.0.0.1` | Bind host for HTTP transports (E003) |
| `NUT_THINKING_PORT` | `9100` | Bind port for HTTP transports (E003) |
| `NUT_THINKING_MAX_NOTE_CHARS` | `800` | Note length cap (E003) |
| `NUT_THINKING_BURST_LIMIT` | `3` | Burst limit setting (E003) |
| `NUT_THINKING_TRANSPORT` | `stdio` | Transport: `stdio`, `http`, `sse`, `streamable-http` (E003, E006) |
| `NUT_THINKING_ACTOR` | `agent` | Fallback actor when no HTTP request context (E003, E006) |
| `NUT_THINKING_ALLOW_MODES` | `goal_distill,understand` | Comma-separated allowed modes (E003) |
| `NUT_THINKING_ROTATE_BYTES` | `5242880` (5 MB) | JSONL rotation threshold (E003, E008) |
| `NUT_THINKING_HOOKS` | `""` | Directory for background hook scripts (E003, E005) |
| `NUT_THINKING_HOOK_TIMEOUT` | `10.0` | Hook subprocess timeout in seconds (E003, E005) |
| `NUT_THINKING_GUARD_ENABLED` | `0` | Set to `1` to enable the loop guard (E003) |
| `NUT_THINKING_GUARD_SAME_MODE_BLOCK` | `1` | Hard-block same-mode-in-a-row when guard enabled (E003) |
| `NUT_THINKING_GUARD_WARN_AT` | `3` | Consecutive-call warning threshold (E003) |
| `NUT_THINKING_GUARD_BLOCK_AT` | `5` | Consecutive-call hard ceiling (E003) |

Note: the `Settings` class reads all of the above; the source lists 17 environment reads (E003). Treat the table above as authoritative.

## Running the server

`server.main()` loads settings and runs the MCP server (E006):

- If `NUT_THINKING_TRANSPORT` is `http`, `sse`, or `streamable-http`, the server runs with `host` and `port` from settings (defaults `127.0.0.1:9100`) (E003, E006).
- Otherwise it runs with the configured transport directly (default `stdio`) (E003, E006).

The default transport is `stdio`, intended for local spawn by an MCP client (E003, E006, E001).

## Health and diagnostics

The `health` tool reports: `status`, `name`, `transport`, `state_dir`, and tool count (`len(modes) + 3`) (E006). The `journal` tool reads back stored thoughts with optional `after_ts` (ms epoch) and `actor_id` filters; `actor_id` filtering is boss-only (E006). `store.stats()` provides record/note/session counts and a per-mode breakdown (E008).

## State and storage

The store is an append-only JSONL log at `<state_dir>/thoughts.jsonl` (E006, E008).

- Every record is one JSON line; `type` distinguishes `note` and `session` records (E008).
- Writes are flushed and `os.fsync`'d after every append (E008).
- Rotation: when the file exceeds `rotate_bytes` (default 5 MB), the current file is renamed to `<stem>.<UTC timestamp>.bak.jsonl` and a new file is created (E008).
- Note length is capped at 800 characters; exceeding it raises `ValueError` (E008).
- Idempotency: a note with the same `actor_id` and `run_id` is rejected as a duplicate (E008).
- Each note carries an `input_hash` (SHA-256 of selected fields, first 16 hex chars) (E008).
- On startup the store loads existing lines into an in-memory list, tolerating a partial tail line (E008).

Operational note: the server passes `actor_id` directly as `session_id` when appending notes, so in normal operation the store's session lifecycle methods (`create_session`, `finalize_session`, `list_sessions`) are not exercised by the server path (E006, E008). The analysis graph flags this as a likely dead-code surface (confidence 0.85).

## Background hooks

Hooks are an optional, off-by-default feature for post-processing automation (E005).

- Point `NUT_THINKING_HOOKS` at a directory; the server fires a script named `<mode>.py`, or a generic `on_think.py` if no mode-specific script exists (E005).
- Hooks are fire-and-forget, best-effort, run in a daemon thread as a subprocess with a timeout (`NUT_THINKING_HOOK_TIMEOUT`, default 10.0 s) (E005).
- A hook failure never fails or slows the `think_*` call (E005).
- The hook receives a JSON payload file written to the system temp directory containing only `{mode, note, actor, ts}` — no tokens, secrets, or config (E005).
- With no hooks directory configured, hook dispatch is a no-op with zero overhead (E005).

## Optional loop guard

The loop guard is off by default and enabled with `NUT_THINKING_GUARD_ENABLED=1` (E003, E009).

- When enabled, it evaluates trailing `think_*` calls per actor (E004, E009).
- Same-mode-in-a-row is hard-blocked when `NUT_THINKING_GUARD_SAME_MODE_BLOCK=1` (default) (E003, E009).
- Escalating warning at `warn_at` (default 3); hard ceiling at `block_at` (default 5) (E003, E009).
- Blocked calls return `{ok: false, blocked: true, status: block, ...}` and store nothing (E009).
- Isolation is per-actor (E009).

The exact decision flow is directly verifiable from `guard.py` (E004): enabled check → filter trailing notes for the actor → count consecutive trailing notes with a truthy `mode` field, including the current call (`consecutive = consec + 1`) → same-mode hard block → hard-ceiling block at `block_at` → soft warning at `warn_at`.

## Testing

- The test suite covers all eight source modules across seven test files (E012–E017, E021).
- AUDIT.md reports `43 passed, 1 skipped` at the 2026-08-05 audit, plus `py_compile` OK and an 18-tool surface verified via `FastMCP.list_tools()` (E009).
- Note: the analysis graph flags a discrepancy — README reports `32 passed, 1 skipped` while AUDIT.md reports `43 passed, 1 skipped` (E001, E009). This was not reconciled in the pinned source.
- There is no CI pipeline in the repository; tests are run manually (E009, E011).

## Known limitations and unverified items

- No CI workflow, pre-commit config, or automated release pipeline is present in the repository (E009, E011).
- The `service.py` module provides an alternative programmatic API (`ThinkingService`) that wraps store, modes, and auth but does not integrate the guard or hooks; the analysis graph infers it predates the server's guard/hooks integration (confidence 0.8) (E007, E006). Its behavior is fully verifiable from the `service.py` source (E007).
- The guard's `evaluate()` calls `store.export()` and filters in Python, which is O(n) per call; the analysis graph flags this as a potential bottleneck for large logs (confidence 0.7) (E004, E008).
- AUDIT.md lists an open item: FastMCP 3.2.4 client bugs with `streamable_http` rejecting a headers kwarg are CLI-side, not server-side (E009).
- AUDIT.md notes that a full external audit by "Boss" is pending (E009).