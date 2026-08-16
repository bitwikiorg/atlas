# Configuration

All configuration is environment-driven. There are no hardcoded secrets and no config file; the `Settings` class in `src/simple_thinking_tool/config.py` reads every value from the environment at startup [E003]. The README documents the primary variables in a table [E001]; the full set defined in `config.py` is listed below.

## Environment variables

| Env var | Default | Meaning | Source |
|---|---|---|---|
| `NUT_THINKING_STATE_DIR` | `state` | Directory where `thoughts.jsonl` lives | [E001][E003] |
| `NUT_THINKING_TRANSPORT` | `stdio` | Transport: `stdio`, `http`, `sse`, or `streamable-http` | [E001][E003][E006] |
| `NUT_THINKING_HOST` | `127.0.0.1` | Bind host for HTTP-family transports (localhost only by default) | [E001][E003] |
| `NUT_THINKING_PORT` | `9100` | Port for HTTP-family transports | [E001][E003] |
| `NUT_THINKING_TOKEN` | — | Optional bearer token; grants `agent` actor identity | [E001][E002][E003] |
| `NUT_THINKING_BOSS_TOKEN` | — | Optional bearer token; grants `boss` actor identity (read-only, cross-actor journal access) | [E001][E002][E003] |
| `NUT_THINKING_MAX_NOTE_CHARS` | `800` | Cap on stored note length | [E001][E003] |
| `NUT_THINKING_ROTATE_BYTES` | `5242880` (5 MB) | JSONL rotation threshold; `0` disables rotation | [E001][E003] |
| `NUT_THINKING_HOOKS` | — | Directory of background hook scripts (empty = disabled) | [E001][E003] |
| `NUT_THINKING_HOOK_TIMEOUT` | `10.0` | Max seconds a hook may run | [E001][E003] |
| `NUT_THINKING_GUARD_ENABLED` | `0` | Set `1` to enable the loop guard | [E001][E003] |
| `NUT_THINKING_GUARD_SAME_MODE_BLOCK` | `1` | Block same-mode twice-in-a-row | [E001][E003] |
| `NUT_THINKING_GUARD_WARN_AT` | `3` | Consecutive `think_*` calls before a warning | [E001][E003] |
| `NUT_THINKING_GUARD_BLOCK_AT` | `5` | Consecutive `think_*` calls before a hard block | [E001][E003] |

The following variables are also defined in `config.py` but are **not** documented in the README and are not referenced by the server wiring in `server.py` at the pinned revision [E003][E006]:

| Env var | Default | Meaning |
|---|---|---|
| `NUT_THINKING_BURST_LIMIT` | `3` | Defined in `Settings`; no usage found in `server.py` or `service.py` |
| `NUT_THINKING_ALLOW_MODES` | `goal_distill,understand` | Comma-separated mode list; defined in `Settings` but no enforcement found in the server wiring |

`NUT_THINKING_ACTOR` is **not** in the reserved list above: it is used by `server.py` in `_actor_from_request()` as the fallback actor identity when no HTTP request context is available (`except RuntimeError: return settings.actor`) [E006][E003]. It is simply not documented in the README's configuration table [E001].

Because `NUT_THINKING_BURST_LIMIT` and `NUT_THINKING_ALLOW_MODES` are defined but unused in the current wiring, treat them as reserved rather than functional knobs.

## Transports

The default transport is **stdio** — the server is spawned locally by the MCP client with no port and no network [E001]. `server.py` treats `http`, `sse`, and `streamable-http` as network transports and passes `host`/`port` to the FastMCP runtime; anything else runs over stdio [E006].

### Local (stdio — default)

```bash
mkdir -p state
NUT_THINKING_STATE_DIR="$PWD/state" python -m simple_thinking_tool.server
```

[E001]

### Remote (HTTP — optional)

```bash
NUT_THINKING_TRANSPORT=http NUT_THINKING_PORT=9100 python -m simple_thinking_tool.server
```

This binds `127.0.0.1` by default. If you expose it beyond localhost, protect it with bearer tokens (`NUT_THINKING_TOKEN`, `NUT_THINKING_BOSS_TOKEN`) [E001].

## Authentication

Authentication is optional and applies to the HTTP transport. Actor identity is derived **only** from the authenticated token, never from tool arguments — this is the spoofing blocker [E002][E001].

- `NUT_THINKING_TOKEN` → actor `agent`
- `NUT_THINKING_BOSS_TOKEN` → actor `boss`
- Tokens are compared with constant-time comparison (`hmac.compare_digest`) [E002].
- The bearer token is extracted from the `Authorization` header (`Bearer <token>`) [E002][E006].
- On the HTTP transport, a missing or invalid token raises `PermissionError` [E006].
- **Journal scoping:** a `boss` actor may read any actor's notes; an `agent` actor is scoped to its own notes [E006][E007].

## Storage

- Plain **JSONL**, one JSON object per line, append-only, with `fsync` after every write [E001][E008].
- No database, no in-memory reindex. Rotation is on by default (5 MB); tune with `NUT_THINKING_ROTATE_BYTES` or set it to `0` to disable [E001].
- On rotation, the current file is renamed to `<stem>.<timestamp>.bak.jsonl` and a new file is created for subsequent appends [E008].
- Notes are capped at `NUT_THINKING_MAX_NOTE_CHARS` (default 800); the store rejects notes over 800 characters [E008].
- The store enforces idempotency via `run_id`: a note with the same `actor_id` and `run_id` is rejected as a duplicate [E008].

> **Note-cap trap:** `store.py`'s `append_note` enforces a **hardcoded** 800-char rejection (`if len(note) > 800: raise ValueError`) independent of the configurable truncation [E008]. The server truncates rendered notes to `s.max_note_chars` before calling `append_note` [E006], and `service.py` hardcodes `note[:800]` [E007]. Do **not** set `NUT_THINKING_MAX_NOTE_CHARS` above 800: the server would truncate to the higher value, but the store would then reject the note. Setting it below 800 works (the server truncates first), but the store's hardcoded check becomes redundant.

## Loop guard (optional, off by default)

The guard prevents agent churn from repeated `think_*` calls. It is disabled by default; enable with `NUT_THINKING_GUARD_ENABLED=1` [E001][E004].

Decision points [E001][E004]:

- **Same-mode-in-a-row: hard block.** Repeating the identical mode is treated as pure churn; the server refuses and tells the agent.
- **Consecutive-count escalation:**
  - At `warn_at` (default **3**) consecutive `think_*` calls: inject state — a warning that the agent is approaching the loop threshold.
  - At `block_at` (default **5**): block until the agent performs a real, verifiable action (a non-thinking-tool call).
- The guard is per-actor: it counts trailing `think_*` notes for the calling actor only [E004].

When a call is blocked, the tool responds `{ "ok": false, "blocked": true, "status": "block", "message": "…" }` and nothing is stored [E001][E006]. When enabled and below `warn_at`, calls behave exactly as before [E001].

## Privacy

The thinking log is private by design — it is your reasoning notebook. Keep the state directory out of version control. Never commit the JSONL or any credentials. If you publish a fork, publish only public code and docs — never state, thoughts, or secrets [E001].

The server ships zero secrets, makes no outbound calls, and never calls an LLM [E001].