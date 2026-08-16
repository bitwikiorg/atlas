# simple_thinking_tool — Reference

This is the technical reference for the `simple_thinking_tool` package, an MCP (Model Context Protocol) server that exposes a catalog of structured thinking methods as individual MCP tools. It is a private, single-serve, per-agent reasoning ledger.

> **Scope and confidence note:** this document was produced from the pinned repository state at commit `2609fe5979e7d4e62024b2be103e81ee9230c172` (2026-08-05) of `bitwikiorg/simple_thinking_tool`. The provided evidence exposes the README ([E001]) plus module/file identity ([E003]–[E013], [E014]–[E019]); the full source text of the modules is **not** part of the visible evidence. Where a claim is directly quoted from the README it is verified against [E001]; everything about internal implementation (function names, class names, exact signatures, exact return fields, exact JSONL schema) is **inferred** and labeled as such. Do not treat un-shown implementation details as verified.

## 1. Overview

The server's core idea: hold a catalog of thinking methods, each exposed as one MCP tool. The LLM that consumes the server acts as the router — it reads tool descriptions and picks the appropriate thinking tool. The server never calls an LLM, makes zero outbound network calls, and performs no reasoning; it fills string templates from caller-supplied fields and appends the rendered note to a durable JSONL log. [E001]

Design principles stated in the README:

- **"The LLM is the router"** — the server has no deterministic routing logic; the README explicitly rejects a second deterministic router because it "failed by gating modes behind priority lanes." [E001]
- **"The server never thinks"** — it fills templates and appends notes; zero LLM calls, zero egress. [E001]
- **"Modes are data"** — adding a thinking mode is a single JSON entry in `modes.json` with no code change. [E001]

The repository summary counts 24 files: 10 source files under `src/simple_thinking_tool/`, 7 test files, `pyproject.toml`, `requirements.txt`, `README.md`, `AUDIT.md`, `LICENSE`, `.gitignore`, and `assets/`. No CI configuration files were detected (CI files: 0). [repository summary, E001]

## 2. Design Principles

### 2.1 LLM as router

The server does not decide which thinking mode to use. Each tool's description states what it does and *when* to use it; the model's native tool-picker chooses. [E001]

### 2.2 Server never thinks

Operations performed by the server are limited to: filling a mode template from caller fields and appending the note to a JSONL log. No LLM calls, no egress. [E001]

### 2.3 Modes as data

Thinking modes live in `modes.json`. Each entry auto-registers as a tool named `think_<id>` with its fields as explicit required arguments. Adding or editing a mode requires no Python and no redeploy logic — the README states "no code changed" when adding a JSON entry. [E001]

## 3. Architecture

The README's architecture description and the module file map imply a hub-and-spoke layout where `server.py` is the central entry point and `service.py` handles business logic. These relationships are **inferred** from the README, the file inventory, and module naming; the exact import graph is not confirmed from source text. [E001], [E003]–[E013]

### 3.1 Module map (inferred roles)

| Module | Responsibility | Evidence |
| --- | --- | --- |
| `__init__.py` | Package init; version metadata (exact value unverified) | [E006] |
| `config.py` | Environment-driven configuration (state dir, transport, port, tokens, rotation, guard, hooks settings) | [E003] |
| `server.py` | MCP server entry point; tool registration; transport setup (stdio/http); request dispatch | [E004] |
| `service.py` | Core service logic: template filling, note rendering, guard/hook/store orchestration | [E012] |
| `store.py` | JSONL append-only storage with fsync, rotation, journal read-back with filters | [E013] |
| `modes.py` | Mode loading from `modes.json`; dynamic tool/definition registration | [E011] |
| `modes.json` | Data file defining all 15 thinking modes | [E010] |
| `guard.py` | Optional loop guard: same-mode block, consecutive-count warn/block escalation, per-actor state | [E008] |
| `auth.py` | Optional bearer-token authentication for HTTP transport; actor derivation from token | [E007] |
| `hooks.py` | Optional background hook system: script resolution, payload writing, fire-and-forget execution | [E009] |

Roles are inferred from the README architecture description and module naming; exact function-level responsibility is **not verified** from source text.

### 3.2 Request lifecycle (inferred)

Per the README's sequence diagram and storage/hook documentation [E001], the runtime path of a `think_<mode>` call is inferred to be:

1. **Call** — the LLM calls `think_<mode>(fields…)` into the MCP server (`server.py`, [E004]).
2. **Auth gate (HTTP only, if enabled)** — the bearer token is validated; the actor derives from the token, never from tool args. Token/auth failure is rejected before any storage or hook. [E001], [E007]
3. **Guard check** — the optional guard evaluates the same-mode block and consecutive-count thresholds. A blocked call returns `{ ok, blocked, status, message }` and stores nothing. [E001], [E008]
4. **Template fill** — the mode's template is filled from the caller's fields, using mode data from `modes.json` / `modes.py`. [E001], [E011], [E012]
5. **Store append** — the rendered note is appended to the JSONL with `fsync`. [E001], [E013]
6. **Hook firing (optional)** — if hooks are configured, a matching script fires as a fire-and-forget background task after the note is persisted. [E001], [E009]
7. **Response** — the server returns `{ ok, mode, note, id, hook? }` (schema from the README sequence diagram). [E001]

The exact function names, call order, and internal data flow are **inferred**; only the outer behavior above is documented in the README.

### 3.3 Critical paths

The following paths are **inferred** from the README architecture description and module roles, not verified from source text:

- **MCP tool call path:** `server.py` receives the call → template fill → optional guard check → store append with fsync → optional hook fire → response.
- **Mode registration path:** `modes.json` (data) → `modes.py` (load/validate/register) → `server.py` (expose as `think_<mode>` tools) → `definitions` tool (catalog exposure).
- **Configuration foundation path:** `config.py` reads `NUT_THINKING_*` env vars; the server uses them for transport/state/tokens.
- **HTTP auth gate path:** `auth.py` validates the bearer token → actor identity derived from token → guard/store attribute the note to that actor.
- **Guard enforcement path:** same-mode block first, then consecutive count against `warn_at`/`block_at` thresholds.
- **Storage rotation path:** append → fsync → check file size against rotation threshold → rotate if exceeded.

## 4. Configuration (Environment Variables)

All configuration comes from environment variables; there are no hardcoded secrets. Defaults are documented in the README storage table [E001]. The table below is taken verbatim from the README.

| Variable | Default | Description |
| --- | --- | --- |
| `NUT_THINKING_STATE_DIR` | `state` | Directory for the JSONL log file |
| `NUT_THINKING_TRANSPORT` | `stdio` | Transport: `stdio` or `http` |
| `NUT_THINKING_HOST` | `127.0.0.1` | HTTP transport bind host (localhost only) |
| `NUT_THINKING_PORT` | `9100` | HTTP transport bind port |
| `NUT_THINKING_TOKEN` | — | Optional bearer token (agent) |
| `NUT_THINKING_BOSS_TOKEN` | — | Optional read-only token |
| `NUT_THINKING_MAX_NOTE_CHARS` | `800` | Maximum characters per stored note |
| `NUT_THINKING_ROTATE_BYTES` | `5242880` (5 MB) | JSONL rotation threshold; `0` disables |
| `NUT_THINKING_HOOKS` | — | Directory for background hook scripts (empty = disabled) |
| `NUT_THINKING_HOOK_TIMEOUT` | `10.0` | Max seconds a hook may run |

### Loop guard variables

| Variable | Default | Description |
| --- | --- | --- |
| `NUT_THINKING_GUARD_ENABLED` | `0` | Set to `1` to enable the loop guard |
| `NUT_THINKING_GUARD_SAME_MODE_BLOCK` | `1` | `1` enables hard block on same-mode repeat |
| `NUT_THINKING_GUARD_WARN_AT` | `3` | Consecutive count to begin warning |
| `NUT_THINKING_GUARD_BLOCK_AT` | `5` | Consecutive count to hard-block |

[E001]

> **Note:** transport is `stdio` or `http` per the README; `sse` and `streamable-http` are **not** documented in the evidence and are not listed here. The env vars `NUT_THINKING_BURST_LIMIT`, `NUT_THINKING_ACTOR`, and `NUT_THINKING_ALLOW_MODES` are **not** present in the README tables and are therefore omitted until verified from source.

## 5. Transport and Serving

The README documents two serving modes [E001]:

- **Local (stdio — default):** most desktop MCP clients spawn a local server over stdio; no port, no network.
  ```bash
  NUT_THINKING_STATE_DIR="$PWD/state" python -m simple_thinking_tool.server
  ```
- **Remote (HTTP — optional):** for harnesses that need a network endpoint.
  ```bash
  NUT_THINKING_TRANSPORT=http NUT_THINKING_PORT=9100 python -m simple_thinking_tool.server
  ```

HTTP binds `127.0.0.1` by default; if exposed beyond localhost, protect it with bearer tokens (`NUT_THINKING_TOKEN`, `NUT_THINKING_BOSS_TOKEN`). [E001]

Three utility tools are always registered alongside the mode tools [E001]:

- **`health`** — returns server name, transport, state dir, tool count.
- **`definitions`** — the catalog: "every mode, what it does, when to use it, and its required fields. Read first." Intended to be called before the first mode call so the LLM learns to route.
- **`journal`** — reads back stored notes with filters; the raw log is a plain JSONL file.

Exact return-field sets for these tools beyond the README descriptions are **not verified** from source.

## 6. Thinking Modes and Tools

Modes are defined in `src/simple_thinking_tool/modes.json`. Each registers as a tool named `think_<id>`. The catalog contains 15 mode tools (plus `health`, `definitions`, `journal` — 18 tools total). [E001], [E010]

The following table lists the 15 mode tools and their README-described purposes [E001]. Per-mode field lists, levels, and `typical_stage` values are **not** exposed in the README beyond the `think_goal_distill(priority, why, action)` example and the general `modes.json` entry schema; they are therefore **unverified** here.

| Tool | Purpose (from README) |
| --- | --- |
| `think_goal_distill` | Reduce a turn to one objective, priority, first action (default) |
| `think_understand` | Digest new or complex material before acting |
| `think_plan` | Order steps + success criteria for a multi-step task |
| `think_chain_of_thought` | Linear step-by-step reasoning |
| `think_first_principles` | Strip inherited assumptions, rebuild from fundamentals |
| `think_creative` | Divergent idea generation |
| `think_deep` | Rephrase → decompose → hypothesize → verify → synthesize |
| `think_tree` | Branch, score, prune, pick |
| `think_graph` | Interacting pieces with feedback loops, aggregated |
| `think_bias_check` | Confirmation / survivorship / authority scan |
| `think_metacognitive` | Audit the reasoning itself |
| `think_adversarial_review` | Try to disprove your own conclusion |
| `think_reinforced` | Multiple takes + red-team attack |
| `think_distill` | Compress a long chain to its essence |
| `think_reflect` | Socratic: what is actually true? |

### 6.1 Template filling

Each mode has a template with `{field}` placeholders. The server fills the template from the caller's fields and appends the rendered note to the log. The `modes.json` entry schema (from the README) is: `id`, `label`, `purpose`, `when`, `level` (int), `typical_stage`, `fields` (list), `template` (with `{field}` placeholders), and optional `references`. [E001] The exact per-mode templates and field lists inside `modes.json` are **not** visible in the evidence. [E010]

### 6.2 Tool response schema

Success response (schema shown in the README sequence diagram) [E001]:

```json
{ "ok": true, "mode": "<id>", "note": "<rendered>", "id": "<id>", "hook": "<path>" }
```

The `hook` field is present only when a hook fired.

Guard block response (documented in the README) [E001]:

```json
{ "ok": false, "blocked": true, "status": "block", "message": "…" }
```

Any additional fields beyond these four on a guard block are **unverified**.

### 6.3 Adding modes

Edit `modes.json` to add or change a mode. A single JSON entry (`id`, `label`, `purpose`, `when`, `level`, `typical_stage`, `fields`, `template`, optional `references`) auto-registers as `think_<id>` and appears in `definitions`; no code change is required. [E001] Whether `modes.json` supports hot-reloading without a server restart is **not** documented in the evidence and is **unverified**.

## 7. Storage (JSONL Store)

Per the README [E001]:

- Plain **JSONL**, one JSON object per line, append-only, `fsync` after every write.
- The file is `thoughts.jsonl` by default, inside the state directory (`NUT_THINKING_STATE_DIR`, default `state`).
- No database, no in-memory reindex.
- Rotation is on by default (5 MB / 5242880 bytes), tuned with `NUT_THINKING_ROTATE_BYTES` or disabled by setting `0`.
- `journal` reads stored notes back with filters; the raw log is openable in any editor.

The exact JSONL note record field set is **not** visible in the evidence. The README response schema and the hook payload imply at least `{id, mode, note, actor, ts}`, but the precise record schema and class/API shape of `store.py` are **unverified**. [E013]

## 8. Authentication (HTTP Transport)

Optional bearer-token authentication protects the HTTP transport [E001]:

- `NUT_THINKING_TOKEN` — agent token (full access).
- `NUT_THINKING_BOSS_TOKEN` — read-only token.

The actor identity is derived from the token, never from tool arguments. [E001], [E007] The specific token-validation functions and the constant-time comparison implementation are **not visible** in the evidence and are **unverified**.

## 9. Loop Guard

The optional loop guard is **off by default** (`NUT_THINKING_GUARD_ENABLED=0`). Enable via `NUT_THINKING_GUARD_ENABLED=1`. [E001]

Behavior when enabled [E001]:

1. **Same-mode hard block:** if same-mode-in-a-row blocking is on (default `NUT_THINKING_GUARD_SAME_MODE_BLOCK=1`), repeating the identical mode is blocked as pure churn.
2. **Consecutive-count escalation:** at `NUT_THINKING_GUARD_WARN_AT` (default 3) consecutive `think_*` calls, the guard injects state (a warning that does not suppress storage or hooks); at `NUT_THINKING_GUARD_BLOCK_AT` (default 5), it blocks until the agent performs a real, verifiable action (a non-thinking-tool call).

When blocked, the tool responds `{ "ok": false, "blocked": true, "status": "block", "message": "…" }` and nothing is stored. [E001] When enabled and below `warn_at`, calls behave exactly as before. [E001]

The README characterizes the design as "context injection + escalating warnings + a hard ceiling — keeps the reasoning value while killing the loop economics." [E001] Internal state structures, per-actor tracking details, and exact warning message text are **not visible** in the evidence and are **unverified**. [E008]

## 10. Hooks (Background Processing)

Hooks are **off by default**: no hooks directory means zero overhead. Set `NUT_THINKING_HOOKS` to a directory containing scripts. [E001]

- **Resolution:** a script named `<mode>.py` fires for that mode only; a script named `on_think.py` fires for every mode without a dedicated script; mode-specific scripts win. Resolution order: `<mode>.py` → `on_think.py` → no-op. [E001]
- **Payload:** each hook receives one argument: the path to a small JSON payload (`{mode, note, actor, ts}`) written to the system temp dir and removed after the run. The payload carries no secrets, no request context, no caller arguments beyond the rendered note. [E001]
- **Execution:** fire-and-forget, best-effort; a hook failure never fails or slows the `think_*` call; timeouts governed by `NUT_THINKING_HOOK_TIMEOUT` (default 10.0 s). [E001]
- **When fires:** hooks fire only in the **successful note-storage path** — not on guard-blocks, auth failures, or note-size rejection, and never for `health` / `definitions` / `journal`. [E001]

The internal hook-runner class shape, subprocess invocation details, and return shape of the dispatch are **not visible** in the evidence and are **unverified**. [E009]

## 11. Security and Privacy Posture

From the README [E001]:

- **No egress:** the server makes zero outbound network calls; it fills templates and appends to local JSONL.
- **No LLM calls:** it never thinks; it never queries a model.
- **Private by default:** the JSONL is your reasoning notebook; keep the state directory out of version control.
- **Optional auth:** bearer tokens protect HTTP transport; the actor derives from the token, never from tool args.
- **Bound localhost:** HTTP binds `127.0.0.1` unless overridden.
- **Public repo only:** ship code and docs; never ship state, thoughts, keys, or internal context.
- **Hooks are local:** background hooks (if enabled) run as your own subprocess; the server still makes no outbound calls and sends no secrets.

## 12. Testing

The repository contains 7 test modules with a near 1:1 mapping to source modules [E005], [E014]–[E019]:

| Test file | Target module(s) | Evidence |
| --- | --- | --- |
| `tests/test_server.py` | `server.py` (also modes integration) | [E005] |
| `tests/test_auth.py` | `auth.py` | [E014] |
| `tests/test_guard.py` | `guard.py` | [E015] |
| `tests/test_hooks.py` | `hooks.py` | [E016] |
| `tests/test_modes.py` | `modes.py` / `modes.json` | [E017] |
| `tests/test_service.py` | `service.py` (also store integration) | [E018] |
| `tests/test_store.py` | `store.py` | [E019] |

The README reports the suite as **32 passed, 1 skipped** (`python -m pytest tests/ -q`); a per-test breakdown is not independently verifiable from the evidence. [E001] No CI configuration files were detected in the pinned tree.

## 13. Licensing and Metadata

- **License:** the README declares **PolyForm Noncommercial 1.0.0** with an SPDX-style badge, and a `LICENSE` file is present ([E001], [E021]). The exact contents of `LICENSE` were not provided in the evidence, and GitHub's automated license detection reports `NOASSERTION`; the cause of the mismatch is unknown.
- **Version:** a specific `__version__` value in `__init__.py` is **not** present in the README and was not visible in the evidence; it is **unverified**. [E006]
- **Python:** README and project metadata reference Python 3.12. The `pyproject.toml` contents (dependencies, build config, entry points) were not visible in the evidence; treat the declared Python requirement as supported by the README badge but the manifest internals as **unverified**. [E002]

## 14. Unverified Items

The following could not be verified from the pinned repository evidence and are intentionally not detailed here:

- The exact source of `config.py`, `server.py`, `service.py`, `store.py`, `modes.py`, `guard.py`, `hooks.py`, `auth.py`, and `__init__.py` — function names, class names, exact signatures, and return fields are inferred, not verified. [E003]–[E013]
- The exact JSONL note record schema from `store.py`.
- The full `pyproject.toml` configuration (dependencies, entry points, build system). [E002]
- The contents of `AUDIT.md` ([E020]) and `LICENSE` ([E021]).
- The contents of `requirements.txt` ([E022]).
- The exact per-mode field lists, levels, `typical_stage` values, and templates inside `modes.json` beyond the README's entry schema and `think_goal_distill` example.
- Whether `modes.json` supports hot-reload without a server restart.
- `assets/hero-banner.svg` contents.

## 15. Rough Gotchas

- The guard, hooks, and auth are all optional and off by default. Do not assume they are active without setting the corresponding env vars. [E001]
- Transport is `stdio` or `http` per the README; do not assume `sse` / `streamable-http` support. [E001]
- The `health` tool reports a tool count; the README totals 18 tools (15 modes + 3 utilities). [E001]
- Reading back history may require consideration of rotated files; read-back implementation details in `store.py` are not visible. [E013]