# Configuration

`simple_thinking_tool` is configured entirely through environment variables with a consistent `NUT_THINKING_*` prefix. Configuration is read by `src/simple_thinking_tool/config.py` and applied via server dispatch.

## Environment variables

| Env var | Default | Notes |
|---|---|---|
| `NUT_THINKING_STATE_DIR` | `state` | where `thoughts.jsonl` lives |
| `NUT_THINKING_TRANSPORT` | `stdio` | `stdio` or `http` |
| `NUT_THINKING_HOST` | `127.0.0.1` | localhost only |
| `NUT_THINKING_PORT` | `9100` | HTTP transport port |
| `NUT_THINKING_TOKEN` | — | optional bearer token (agent) |
| `NUT_THINKING_BOSS_TOKEN` | — | optional read-only token |
| `NUT_THINKING_MAX_NOTE_CHARS` | `800` | note cap |
| `NUT_THINKING_ROTATE_BYTES` | `5242880` | JSONL rotation threshold (5 MB); set `0` to disable |
| `NUT_THINKING_HOOKS` | — | directory of background hook scripts (empty = disabled) |
| `NUT_THINKING_HOOK_TIMEOUT` | `10.0` | max seconds a hook may run |

These defaults are taken verbatim from the README storage table. [E001]

## Loop guard variables

The optional loop guard is off by default. It prevents agents from churning on repeated thinking calls.

| Env var | Default | Meaning |
|---|---|---|
| `NUT_THINKING_GUARD_ENABLED` | `0` | set `1` to enable the loop guard |
| `NUT_THINKING_GUARD_SAME_MODE_BLOCK` | `1` | block same-mode twice-in-a-row |
| `NUT_THINKING_GUARD_WARN_AT` | `3` | consecutive calls before a warning |
| `NUT_THINKING_GUARD_BLOCK_AT` | `5` | consecutive calls before a hard block |

[E001], [E008]

Behavior:

- Same-mode-in-a-row is a hard block: the server refuses and returns `{ "ok": false, "blocked": true, "status": "block", "message": "…" }`; nothing is stored.
- At `warn_at` consecutive calls, the server injects state about the loop threshold (a warning does not suppress storage or hooks).
- At `block_at`, it blocks until the agent performs a real, non-thinking-tool action.
- Below `warn_at`, calls behave exactly as before.

## Serving examples

### HTTP transport with hooks and auth

```bash
NUT_THINKING_TRANSPORT=http \
NUT_THINKING_PORT=9100 \
NUT_THINKING_TOKEN=secret-agent-token \
NUT_THINKING_BOSS_TOKEN=readonly-token \
NUT_THINKING_HOOKS=/path/to/hooks \
NUT_THINKING_STATE_DIR=/path/to/state \
python -m simple_thinking_tool.server
```

### stdio transport with the guard enabled

```bash
NUT_THINKING_GUARD_ENABLED=1 \
NUT_THINKING_STATE_DIR=/path/to/state \
python -m simple_thinking_tool.server
```

## Notes and caveats

- With HTTP transport, the actor identity is derived from the bearer token, never from tool arguments. `NUT_THINKING_BOSS_TOKEN` is described as a read-only token; the exact privilege boundary could not be verified from the provided source blobs beyond the README description.
- Rotation threshold applies to the JSONL log file: when it is exceeded a new file is created.
- Python 3.12 is specified by project metadata; exact dependency pinning in `pyproject.toml` and `requirements.txt` could not be verified from the provided evidence (the manifest contents were not visible).
- There is no visible CI or `.env.example` in the pinned tree; config validation at startup (for invalid combinations) is not documented in the evidence.

---

Evidence references: [E001] (env var tables, guard behavior, serving examples from the README), [E003] (configuration layer is read via `config.py`), [E008] (guard env vars and behavior).