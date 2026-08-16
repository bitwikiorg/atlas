# Architecture — simple_thinking_tool

Pinned source: [`bitwikiorg/simple_thinking_tool`](https://github.com/bitwikiorg/simple_thinking_tool) at commit [`2609fe5979e7d4e62024b2be103e81ee9230c172`](https://github.com/bitwikiorg/simple_thinking_tool/commit/2609fe5979e7d4e62024b2be103e81ee9230c172) (2026-08-05). [E001]

> **Scope and confidence note:** the source files exist at the pinned paths below, but the provided evidence exposes the README ([E001]) plus module/file identity ([E003]–[E013], [E014]–[E019]) rather than full source text. The import topology and request flow are **inferred** from the README architecture description, the module file map, and the preplan's module inventory, and are marked as inferred where they are not directly quoted. Do not treat un-shown implementation details (exact function signatures, exact JSONL field set, exact import edges) as verified.

## Design philosophy (from README)

- **The LLM is the router.** Every tool has a description saying what it does and *when* to use it; the model's native tool-picker chooses. There is no second deterministic router. The README explicitly rejects a deterministic router because it "failed by gating modes behind priority lanes." [E001]
- **One tool per mode.** `think_goal_distill`, `think_tree`, `think_bias_check`, … each with its own required fields. [E001]
- **Modes are data.** They live in `modes.json`; adding or editing a mode requires no Python and no redeploy logic. [E001]
- **The server never thinks.** It fills templates and appends notes; zero LLM calls, zero egress. [E001]

## Module map (inferred topology)

The package is `src/simple_thinking_tool/` ([E003]–[E013]) with tests in `tests/` ([E005], [E014]–[E019]). The repository summary lists 10 files under `src/` and 7 under `tests/`.

| Module | Responsibility | Evidence |
|---|---|---|
| `__init__.py` | Package init and version metadata | [E006] |
| `config.py` | Environment-driven configuration (state dir, transport, port, tokens, rotation, guard, hooks settings) | [E003] |
| `server.py` | MCP server entry point; tool registration, transport setup (stdio/http), request dispatch | [E004] |
| `service.py` | Core service logic: template filling, note rendering, guard integration, hook firing orchestration | [E012] |
| `store.py` | JSONL append-only storage with fsync, log rotation, journal read-back with filters | [E013] |
| `modes.py` | Mode loading from `modes.json`, validation, dynamic tool/definition registration | [E011] |
| `modes.json` | Data file defining all 15 thinking modes | [E010] |
| `guard.py` | Optional loop guard: same-mode block, consecutive-count warn/block escalation, per-actor state | [E008] |
| `auth.py` | Optional bearer-token authentication for HTTP transport; actor derivation from token | [E007] |
| `hooks.py` | Optional background hook system: script resolution (`<mode>.py` vs `on_think.py`), payload writing, fire-and-forget daemon thread execution | [E009] |

Inferred dependency shape (from the preplan's architecture analysis): `server.py` is the central orchestration hub importing all six functional modules (auth, config, guard, hooks, modes, store); `service.py` is the secondary business-logic hub importing auth, modes, and store. `config.py` appears to be consumed at the server dispatch layer. These edges are inferred, not directly quoted from source.

## Request lifecycle (a `think_<mode>` call)

Per the README's sequence diagram and storage/hook documentation [E001], the runtime path is:

1. **Call** — the LLM calls `think_<mode>(fields…)` into the MCP server (`server.py`, [E004]).
2. **Auth gate (HTTP only, if enabled)** — `auth.py` validates the bearer token; the actor derives from the token, never from tool args ([E007], [E001]). Token/auth failure is rejected before any storage or hook. [E001]
3. **Guard check** — `guard.py` evaluates the same-mode block and consecutive-count thresholds before processing ([E008], [E001]). A blocked call returns `{ "ok": false, "blocked": true, "status": "block", "message": "…" }` and stores nothing. [E001]
4. **Template fill** — `service.py` fills the mode's template from the caller's fields, using the mode data from `modes.py` / `modes.json` ([E012], [E011], [E010]).
5. **Store append** — `store.py` appends the rendered note to the JSONL with `fsync` ([E013], [E001]).
6. **Hook firing (optional)** — if hooks are configured, a matching script fires as a fire-and-forget background task after the note is persisted ([E009], [E001]).
7. **Response** — the server returns `{ ok, mode, note, id, hook? }` (schema from README). [E001]

Hooks fire **only in the successful note-storage path**. They do not fire on guard-blocks, on auth failures, on note-size rejection (`NUT_THINKING_MAX_NOTE_CHARS`), or for the read-only tools `health` / `definitions` / `journal`. A `warn` (guard allowed-but-warned) does not suppress storage or hooks. [E001]

## Mode registration pipeline (data → tool)

1. `modes.json` holds the 15 mode entries (schema: `id`, `label`, `purpose`, `when`, `level`, `typical_stage`, `fields`, `template`, optional `references`) [E010], [E001].
2. `modes.py` loads and validates entries and drives dynamic tool/definition registration [E011].
3. `server.py` exposes each as `think_<id>` MCP tool and the `definitions` tool surfaces the catalog for the LLM to read first [E004], [E001].
4. At call time, `service.py` renders the template from the registered mode data [E012]. Adding a JSON entry yields a new tool with no code change. [E001]

## Storage & rotation

- Append-only JSONL, one JSON object per line, `fsync` after every write; no database, no in-memory reindex. [E001]
- Rotation: when the file exceeds `NUT_THINKING_ROTATE_BYTES` (default 5 MB / 5242880), a new file is created; set `0` to disable. [E001]
- `journal` reads stored notes back with filters; the raw log is plain JSONL openable in any editor. [E001]
- The exact JSONL field set is **not fully verified**; the README and hook payload imply at least `mode`, `note`, `actor`, `ts` and a unique `id`. [E001]

## Guard enforcement path

1. `service.py` consults `guard.py` before processing [E012], [E008].
2. **Same-mode-in-a-row: hard block** (default on via `NUT_THINKING_GUARD_SAME_MODE_BLOCK=1`). [E001]
3. **Consecutive-count escalation:** at `warn_at` (default 3) inject state ("you have called thinking tools 3× in a row…"); at `block_at` (default 5) block until a real, non-thinking-tool action occurs. [E001]
4. Guard off by default (`NUT_THINKING_GUARD_ENABLED=0`). [E001]

The README describes the design as a middle path: "context injection + escalating warnings + a hard ceiling — keeps the reasoning value while killing the loop economics." [E001]

## Hooks execution model

- Resolution: `<mode>.py` → else `on_think.py` → else no-op. Mode-specific wins. [E001]
- Payload: `{mode, note, actor, ts}` only — no secrets, no request context, no caller arguments beyond the rendered note. [E001]
- Fire-and-forget, best-effort: hook failures never fail or slow the `think_*` call; hook runtimes are non-blocking in a daemon thread; timeouts governed by `NUT_THINKING_HOOK_TIMEOUT` (default 10.0 s). [E001], [E009]
- Off by default: no hooks dir = zero overhead. [E001]

## Config foundation

`config.py` reads the `NUT_THINKING_*` environment variables (state dir, transport, host, port, tokens, note cap, rotation, hooks, guard settings) [E003]. The full var table with defaults is in [docs/overview.md](overview.md); the README is the authoritative table [E001].

## Test suite

Seven test modules under `tests/` with a near 1:1 mapping to source modules [E005], [E014]–[E019]:

| Test file | Targets | Evidence |
|---|---|---|
| `tests/test_server.py` | `server.py` (also modes integration) | [E005] |
| `tests/test_auth.py` | `auth.py` | [E014] |
| `tests/test_guard.py` | `guard.py` | [E015] |
| `tests/test_hooks.py` | `hooks.py` | [E016] |
| `tests/test_modes.py` | `modes.py` / `modes.json` | [E017] |
| `tests/test_service.py` | `service.py` (also store integration) | [E018] |
| `tests/test_store.py` | `store.py` | [E019] |

The README reports the suite as **32 passed, 1 skipped** (`python -m pytest tests/ -q`). [E001] No CI configuration files were detected in the pinned tree.

## Unverified / uncertain

- **Exact import graph** between modules — inferred from file inventory and README, not confirmed from source text.
- **Exact JSONL note schema** in `store.py` — inferred; the README's hook payload and response schemas are the only confirmed shape.
- **Exact `modes.json` content** (per-mode fields and templates) — only mode names and the entry schema are confirmed from README.
- **`pyproject.toml` / `requirements.txt` contents** and any `console_scripts` beyond `python -m simple_thinking_tool.server`. [E002], [E022]
- **`AUDIT.md` findings** — the file exists ([E020]) but its contents are not exposed in this evidence pack.
- **`__init__.py` API surface** ([E006]) — whether it exports public API is unknown.
- The README's tools table and mermaid diagrams describe behavior; diagrams themselves are documentation, and the rendered behavior should be treated as inferred unless independently confirmed from source.