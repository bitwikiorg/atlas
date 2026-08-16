# Architecture — simple_thinking_tool

This document describes the internal architecture of `simple_thinking_tool` as of the pinned source commit `2609fe5979e7d4e62024b2be103e81ee9230c172` (2026-08-05). All claims are grounded in the pinned repository evidence.

## Component map

The package `src/simple_thinking_tool/` contains 8 modules plus the `modes.json` data file. Each module has a single responsibility. [E001][E009]

| Module | Responsibility | Evidence |
|---|---|---|
| `server.py` | FastMCP wiring — `build_server()` orchestrates all components, auto-registers 15 `think_*` tools plus `health`/`definitions`/`journal` | [E006] |
| `service.py` | Lean service layer (`ThinkingService`) wrapping store, modes, and auth | [E007] |
| `store.py` | JSONL append-only store with `fsync`, rotation, idempotency, `input_hash` | [E008] |
| `modes.py` | Mode loader + template renderer + definitions surface | [E020] |
| `modes.json` | 15 thinking-mode definitions as data | [E019] |
| `hooks.py` | Background event hooks (fire-and-forget, off by default) | [E005] |
| `config.py` | Environment-driven `Settings` class; no hardcoded secrets | [E003] |
| `auth.py` | Token-derived actor identity (agent / boss) | [E002] |
| `guard.py` | Optional loop guard (anti-churn control, off by default) | [E004] |

Dependencies flow in one direction: `server.py` composes `auth`, `config`, `guard`, `hooks`, `modes`, and `store`. The only third-party runtime dependency is `fastmcp>=3.2,<4`. [E011]

## Request lifecycle of a `think_*` call

When an LLM invokes a `think_<mode>` tool, the server orchestrates the following sequence [E006]:

1. **Actor resolution** — `_actor_from_request()` extracts a bearer token from the HTTP `Authorization` header (if present) and resolves the actor via `verify_token()`. On HTTP transport, an invalid or missing token raises `PermissionError`. On stdio, the configured default actor is used. [E006][E002]
2. **Guard evaluation** — `guard_runner.evaluate(store, actor, mode.id)` checks anti-churn rules. If the guard is disabled (the default), the call proceeds. If blocked, the tool returns `{ok: false, blocked: true, status: "block", message, consecutive, last_mode}` and nothing is stored. [E006][E004]
3. **Template rendering** — `render_template(mode, provided)` fills the mode's template from the caller's field values. Missing fields are filled with empty strings rather than raising errors. [E020]
4. **Note append** — `store.append_note(...)` writes one JSON line to `thoughts.jsonl` with `fsync`, capped at `max_note_chars` (default 800). [E006][E008]
5. **Hook dispatch** — `hooks.fire(mode.id, note, actor)` fires a background hook if configured. [E006][E005]
6. **Response** — on success, returns `{ok: true, mode, note, id}` plus `hook` if a hook fired. [E006]

The mode tool functions are generated dynamically from `modes.json` via `_make_mode_function()`, which builds a function body with `exec()` using field names from the modes file. The generated functions receive `store`, `s` (settings), `hooks`, and `guard_runner` via `__globals__` injection. [E006]

## Storage design

`JSONLStore` in `store.py` is the canonical store: a plain append-only JSON Lines log. [E008]

- **Record format:** one JSON object per line. Records are discriminated by a `type` field (`note` or `session`). [E008]
- **Durability:** `fsync` after every write. [E008]
- **Idempotency:** `run_id` + `actor_id` dedupe — a duplicate `run_id` raises `ValueError`. [E008]
- **Integrity:** `input_hash` is a SHA-256 digest (first 16 hex chars) of the record's key fields. [E008]
- **Rotation:** when the file exceeds `rotate_bytes` (default 5 MB), the current file is renamed to `<stem>.<timestamp>.bak.jsonl` and a new file is created. [E008]
- **In-memory cache:** all records are loaded into `_records` on startup (tolerating partial tail lines) and appended to in memory on each write. A `threading.Lock` protects concurrent writes. [E008]
- **Stats:** `stats()` reports record/note/session counts and a by-mode breakdown. [E008]

> **Note-cap trap:** `store.py`'s `append_note` enforces a **hardcoded** 800-char rejection (`if len(note) > 800: raise ValueError`) independent of the configurable truncation [E008]. The server truncates rendered notes to `s.max_note_chars` before calling `append_note` [E006], and `service.py` hardcodes `note[:800]` [E007]. If `NUT_THINKING_MAX_NOTE_CHARS` is set above 800, the server truncates to the higher value but the store rejects the note; if set below 800, the server truncates first and the store's hardcoded check is redundant. Keep `NUT_THINKING_MAX_NOTE_CHARS` at or below 800.

Note: the store includes session lifecycle methods (`create_session`, `get_session`, `list_sessions`, `finalize_session`), but the server passes `actor_id` directly as `session_id`, treating it as a flat grouping key rather than a session lifecycle. Session records are therefore not created in normal server operation. [E008][E006]

## Loop guard

`Guard` in `guard.py` is an optional anti-churn control, off by default (`NUT_THINKING_GUARD_ENABLED=1` enables it). [E004]

Decision logic in `Guard.evaluate(store, actor_id, mode)` [E004]:

1. If disabled, return `allowed=True, status="ok"`.
2. Filter `store.export()` for trailing notes by this actor; count consecutive trailing notes that have a mode set, including the current call (`consecutive = consec + 1`).
3. **Same-mode hard block:** if the last note's mode equals the current mode and `same_mode_block` is on, block (pure churn).
4. **Hard ceiling:** if consecutive count ≥ `block_at` (default 5), block until the agent interleaves real work.
5. **Soft warning:** if consecutive count ≥ `warn_at` (default 3), allow but inject an escalating warning message.

`GuardResult` is a dataclass with `allowed`, `status` (`ok` | `warn` | `block`), `consecutive`, `last_mode`, and `message`. [E004]

The guard never detects real work itself; it surfaces the LLM its own call state and hard-stops only as a backstop. [E004]

## Hooks system

`HookRunner` in `hooks.py` dispatches background hooks. [E005]

- **Resolution:** a script named `<mode>.py` fires for that mode only; `on_think.py` fires for every mode without a dedicated script; mode-specific wins. [E005]
- **Payload:** a JSON file `{mode, note, actor, ts}` written to the system temp dir and removed after the run. No tokens, no secrets, no request context. [E005]
- **Execution:** fire-and-forget in a daemon thread via `subprocess.run` with a timeout (default 10.0 s). Hook failures are logged and never break the server. [E005]
- **Enabled:** only when `NUT_THINKING_HOOKS` points at an existing directory. [E005]

Hooks fire only in the successful note-storage path. They do not fire when the guard blocks, on read-only tools (`health`, `definitions`, `journal`), on auth failure, or when a note exceeds the char cap. [E001]

## Authentication

`auth.py` provides token-derived actor identity. [E002]

- `verify_token(token, agent_token, boss_token)` returns `"agent"` or `"boss"` using constant-time comparison (`hmac.compare_digest`). [E002]
- `extract_bearer(authorization)` parses a `Bearer <token>` header. [E002]
- Actor identity comes from the authenticated token, never from tool input — a spoofing blocker. [E002]
- The `boss` actor has read-only, cross-actor `journal` access; the `agent` actor is scoped to its own notes. [E006]

## Configuration

`config.py` defines a `Settings` class populated entirely from environment variables, with no hardcoded secrets. [E003]

Key settings and defaults [E003]:

| Field | Env var | Default |
|---|---|---|
| `token` | `NUT_THINKING_TOKEN` | `""` |
| `boss_token` | `NUT_THINKING_BOSS_TOKEN` | `""` |
| `state_dir` | `NUT_THINKING_STATE_DIR` | `state` |
| `host` | `NUT_THINKING_HOST` | `127.0.0.1` |
| `port` | `NUT_THINKING_PORT` | `9100` |
| `max_note_chars` | `NUT_THINKING_MAX_NOTE_CHARS` | `800` |
| `burst_limit` | `NUT_THINKING_BURST_LIMIT` | `3` |
| `transport` | `NUT_THINKING_TRANSPORT` | `stdio` |
| `actor` | `NUT_THINKING_ACTOR` | `agent` |
| `allow_modes` | `NUT_THINKING_ALLOW_MODES` | `goal_distill,understand` |
| `rotate_bytes` | `NUT_THINKING_ROTATE_BYTES` | `5242880` |
| `hooks_dir` | `NUT_THINKING_HOOKS` | `""` |
| `hooks_timeout` | `NUT_THINKING_HOOK_TIMEOUT` | `10.0` |
| `guard_enabled` | `NUT_THINKING_GUARD_ENABLED` | `0` |
| `guard_same_mode_block` | `NUT_THINKING_GUARD_SAME_MODE_BLOCK` | `1` |
| `guard_warn_at` | `NUT_THINKING_GUARD_WARN_AT` | `3` |
| `guard_block_at` | `NUT_THINKING_GUARD_BLOCK_AT` | `5` |

Note: `burst_limit` and `allow_modes` are parsed in `config.py` [E003] but are not referenced by the server wiring and are not documented in the README's configuration table [E001]. `actor` (`NUT_THINKING_ACTOR`) **is** used by `server.py` as the fallback actor in `_actor_from_request()` when no HTTP request context is available [E006], but is likewise not documented in the README table [E001].

## Transports

The server supports two transports, selected by `NUT_THINKING_TRANSPORT` [E006][E003]:

- **stdio** (default) — spawned locally by the MCP client; no port, no network.
- **http** (also accepts `sse` / `streamable-http` in `main()`) — binds `127.0.0.1` by default; bearer-token auth recommended when exposed beyond localhost.

## Service layer

`service.py` provides `ThinkingService`, an alternative programmatic API that wraps the store, modes, and auth without FastMCP. It supports `append`, `definitions`, `journal`, and `health`. It does not integrate the loop guard or hooks. [E007]

## Known limitations and notes

- The guard's `evaluate()` calls `store.export()` and filters in Python — an O(n) scan of all in-memory records on every `think_*` call when the guard is enabled. [E004][E008]
- The in-memory `_records` list loads all records on startup and grows with each append; rotation archives the file but does not bound the in-memory list. [E008]
- Mode tool functions are generated with `exec()` from `modes.json` field names; the field names are operator-controlled (not user input), but the file should be treated as trusted. [E006]
- `AUDIT.md` reports 43 passed / 1 skipped for the pytest suite, while the README reports 32 passed / 1 skipped; the counts are not reconciled in the pinned source, and neither matches the 46 test functions visible in the source. AUDIT.md also miscounts the test files as 6 when there are 7. [E009][E001]
- `AUDIT.md` notes FastMCP 3.2.4 client bugs (streamable_http rejecting a headers kwarg) as CLI-side, not server-side. [E009]