# Glossary — simple_thinking_tool

Terms are defined from repository evidence at pinned SHA `2609fe5979e7d4e62024b2be103e81ee9230c172`. Primary source is the README [E001]; definitions informed by the module layout and documented schemas. Items marked "inferred" could not be confirmed directly from source code in the provided evidence and are labeled as such.

## Terms

### Actor
The identity associated with a thinking call. Derived from the bearer token in HTTP mode (or a default identity in stdio mode). Used for guard state tracking and note attribution. **Never** derived from tool arguments. [E001, E007]

### Bearer token auth
Optional authentication for the HTTP transport, using `NUT_THINKING_TOKEN` (agent / full access) and `NUT_THINKING_BOSS_TOKEN` (read-only). The actor identity is derived from which token was used. [E001, E007]

### Definitions tool
One of three utility MCP tools (alongside `health` and `journal`). Returns the full catalog of modes — purpose, when-to-use, and required fields — so the LLM can learn the routing options. Intended to be called first. [E001, E011]

### Health tool
Utility MCP tool returning server metadata: name, transport, state directory, and tool count. [E001]

### Hook
An optional background script (a file in a configured directory) that the server fires as a fire-and-forget task after a note is successfully stored. Mode-specific scripts (`<mode>.py`) take precedence over the generic `on_think.py`. Hooks are best-effort, off by default, and receive a payload of `{mode, note, actor, ts}`. [E001, E009]

### Journal tool
Utility MCP tool that reads back stored notes from the JSONL store with optional filters, used for auditing and reviewing the thinking log. [E001]

### JSONL store
The persistence layer: a plain append-only JSON Lines file (`thoughts.jsonl` by default) where each rendered note is stored as one JSON object per line, written with `fsync` for durability. [E001, E013]

### LLM as router
The design principle that the server contains no deterministic routing logic. The LLM consuming the tools reads tool descriptions and picks the appropriate thinking mode. The server never decides which mode to use. [E001]

### Loop guard
An optional feature (off by default) that prevents agents from repeatedly calling thinking tools without producing real work. Enforces a same-mode-in-a-row hard block and consecutive-count escalation (warn at 3, block at 5 by default). [E001, E008]

### Loop-break rule
The baseline discipline enforced without the guard: never call the same thinking tool twice in a row. The optional guard extends this with consecutive-count escalation. [E001]

### MCP
Model Context Protocol — a standard for exposing tools to LLM agents. This project implements the MCP server side and can be consumed by any MCP client (Claude Desktop, Cursor, Agent Zero, and others). [E001]

### Mode level
A numeric field in `modes.json` (e.g., `level: 1`) indicating the cognitive depth tier of a mode. Referenced in mode definitions; routing is left to the LLM. [E001, E010]

### No egress
The security guarantee that the server makes zero outbound network calls. It only fills templates and writes to local files. Hooks, if enabled, are local subprocesses owned by the operator. [E001]

### Rotation
Log file rotation: when `thoughts.jsonl` exceeds `NUT_THINKING_ROTATE_BYTES` (default 5 MB / 5242880 bytes), a new file is created. Set to `0` to disable. [E001, E013]

### State directory
The filesystem location (`NUT_THINKING_STATE_DIR`, default `state`) where the JSONL log file resides. Must be kept out of version control for privacy. [E001, E003]

### Template filling
The server's only operation: take the caller's field values, substitute them into the mode's template string, and produce a rendered note. No reasoning, no LLM calls, no egress. [E001]

### Thinking mode
A named reasoning pattern (e.g., `goal_distill`, `tree`, `bias_check`) defined in `modes.json`, exposed as its own MCP tool named `think_<mode>`. Each mode has a template with required string fields the caller fills. [E001, E010]

### typical_stage
A field in `modes.json` (e.g., `any`, `planning`, `review`) indicating when in a problem-solving lifecycle a mode is typically used. [E001, E010]

## Schemas

### modes.json entry schema
```json
{
  "id": "unique slug (str)",
  "label": "display name (str)",
  "purpose": "one sentence (str)",
  "when": "when to use (str)",
  "level": "cognitive depth (int)",
  "typical_stage": "lifecycle stage (str)",
  "fields": ["required field names (str)"],
  "template": "template with {field} placeholders (str)",
  "references": ["optional citation URLs (str)"]
}
```
[E001, E010]

### Config environment variables

| Variable | Default | Meaning |
|---|---|---|
| `NUT_THINKING_STATE_DIR` | `state` | where `thoughts.jsonl` lives |
| `NUT_THINKING_TRANSPORT` | `stdio` | `stdio` or `http` |
| `NUT_THINKING_HOST` | `127.0.0.1` | localhost only |
| `NUT_THINKING_PORT` | `9100` | HTTP transport port |
| `NUT_THINKING_TOKEN` | — | optional bearer token (agent) |
| `NUT_THINKING_BOSS_TOKEN` | — | optional read-only token |
| `NUT_THINKING_MAX_NOTE_CHARS` | `800` | note cap |
| `NUT_THINKING_ROTATE_BYTES` | `5242880` | JSONL rotation threshold |
| `NUT_THINKING_HOOKS` | — | directory of background hook scripts (empty = disabled) |
| `NUT_THINKING_HOOK_TIMEOUT` | `10.0` | max seconds a hook may run |
| `NUT_THINKING_GUARD_ENABLED` | `0` | set `1` to enable the loop guard |
| `NUT_THINKING_GUARD_SAME_MODE_BLOCK` | `1` | block same-mode twice-in-a-row |
| `NUT_THINKING_GUARD_WARN_AT` | `3` | consecutive calls before a warning |
| `NUT_THINKING_GUARD_BLOCK_AT` | `5` | consecutive calls before a hard block |

[E001, E008]

### JSONL note schema (inferred)
The exact `store.py` implementation was not in the provided evidence, so this schema is **inferred** from the README and the documented hook payload:
```json
{
  "id": "unique note id",
  "mode": "mode id",
  "note": "rendered template output",
  "actor": "caller identity",
  "ts": "ISO timestamp"
}
```
[E001, E012, E013 — inference]

### Hook payload schema
```json
{
  "mode": "str",
  "note": "str",
  "actor": "str",
  "ts": "str"
}
```
Written to a temp JSON file; the path is passed as `argv[1]` to the hook script and removed after the run. [E001, E009]

### Tool response schemas
- Success (inferred): `{ ok: true, mode: str, note: str, id: str, hook: str|null }` [E001, E012]
- Guard block (documented): `{ ok: false, blocked: true, status: "block", message: "…" }` [E001, E008]
- Guard warn (documented as not suppressing storage): normal success plus a `status: "warn"` and injected state message [E001]

## Module map

| Module | Role |
|---|---|
| `src/simple_thinking_tool/config.py` | environment-driven configuration [E003] |
| `src/simple_thinking_tool/server.py` | MCP server entry point; tool registration, transport, dispatch [E004] |
| `src/simple_thinking_tool/service.py` | template filling, note rendering, guard + hook orchestration [E012] |
| `src/simple_thinking_tool/store.py` | JSONL append-only storage, fsync, rotation, journal read-back [E013] |
| `src/simple_thinking_tool/modes.py` | mode loading/validation, dynamic tool registration [E011] |
| `src/simple_thinking_tool/modes.json` | data file defining all 15 thinking modes [E010] |
| `src/simple_thinking_tool/guard.py` | optional loop guard [E008] |
| `src/simple_thinking_tool/auth.py` | optional bearer-token auth [E007] |
| `src/simple_thinking_tool/hooks.py` | optional background hook system [E009] |

## Notes on evidence limits

- The exact contents of `store.py`, `service.py`, `config.py`, and other source modules were **not** provided in the evidence; module roles above are taken from the README and the documented repository structure.
- The `AUDIT.md` file exists [E020] but its contents were not available; its scope and findings are unknown.
- `pyproject.toml` internals (dependencies, build config, entry points) were not visible in the evidence [E002].