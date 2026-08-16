# FAQ — simple_thinking_tool

This FAQ is compiled from the repository at pinned source SHA `2609fe5979e7d4e62024b2be103e81ee9230c172` (main branch, commit dated 2026-08-05). Where a claim could not be verified from the pinned evidence, that is stated explicitly.

## Design & behavior

### Does the server call an LLM?
No. The server fills template strings from your fields and appends the rendered notes to the JSONL log. There are no model calls and no network egress, ever. [E001]

### Why is there no router inside the server?
Because the LLM that calls the tools is already a router. The authors judged a second deterministic router to be redundant machinery — one that previously "failed by gating modes behind priority lanes." The tool descriptions carry the routing logic. [E001]

### What does "the LLM is the router" mean?
Each `think_*` tool has a description stating what it does and *when* to use it. The model's native tool-picker chooses which mode fits the problem. The server never decides which mode to use. [E001]

### Does the server "think"?
No. The server only fills templates and appends notes. Zero LLM calls, zero egress. The note in the README: "forcing articulation forces the cognition" — the text the LLM produces *is* the thinking; the tool is a required checkpoint between "I could answer" and "I act." [E001]

### Why JSONL and not a database?
Because it is the readable notebook: open the file, read the audit. Append-only means no corruption risk from partial writes, and a dump tool is trivial. This is a "notepad" use case, not a transactional system. [E001]

## Usage & integration

### Can any agent / harness use this?
Yes. It is a standard MCP server. Install it locally and register it in any MCP client (the README names Claude Desktop, Cursor, Agent Zero, and others). It is single-serve: each agent runs its own instance; nothing is shared or multi-tenant. [E001]

### How do I stop loop-calling?
One baseline rule: **never call the same tool twice in a row.** That is the loop-break. If loop-abuse appears, enable the optional loop guard for a togglable escalation (warn at 3 consecutive calls, hard block at 5). See "Loop guard" below. [E001, E008]

### What should I call first?
Call the **`definitions`** tool. It lists every mode, what it does, when to use it, and its required fields. The README recommends reading this catalog before the first call — it is how the LLM learns to route. [E001]

### Can I build on top of a thinking call?
Yes. Set `NUT_THINKING_HOOKS` to a directory of scripts and the server fires a matching `<mode>.py` (or the generic `on_think.py`) as a background task after each note is stored. This is for processing and automation; the raw JSONL is already the durable audit trail. [E001, E009]

### How do I add a new thinking mode?
Edit `src/simple_thinking_tool/modes.json` and add one entry (`id`, `label`, `purpose`, `when`, `level`, `typical_stage`, `fields`, `template`, optional `references`). It auto-registers as `think_<mode>(...)` and appears in `definitions`. No code change is needed. [E001, E010, E011]

## Storage & privacy

### Where are notes stored?
In a plain JSONL file (`thoughts.jsonl` by default) inside the state directory (`NUT_THINKING_STATE_DIR`, default `state`). The file is append-only with `fsync` after every write. Each mode call appends one JSON object per line. [E001, E013]

### How do I audit or read back stored notes?
Call the **`journal`** tool, which reads back stored notes with filters. The raw log is a plain JSONL file you can open in any editor. [E001]

### Can I keep my thinking log private?
The thinking log is private by design — it is your reasoning notebook. Keep the state directory out of version control. Never commit the JSONL or any credentials. If you publish a fork, publish only public code and docs — never state, thoughts, or secrets. [E001]

### Is log rotation configurable?
Yes. Rotation is on by default at 5 MB (5242880 bytes). Tune with `NUT_THINKING_ROTATE_BYTES` or set it to `0` to disable. [E001, E013]

## Security

### Does the server make outbound network calls?
No. The server makes zero outbound calls. It fills templates and appends to local JSONL. It also makes no LLM calls. [E001]

### Are hooks a security or egress concern?
No. Hooks (if enabled) run as your own local subprocesses. The server still makes no outbound calls and sends no secrets. [E001]

### Is authentication available?
Optional bearer tokens protect the HTTP transport: `NUT_THINKING_TOKEN` (agent, full access) and `NUT_THINKING_BOSS_TOKEN` (read-only). The actor identity is derived from the token, never from tool arguments. [E001, E007]

### Why does HTTP bind to localhost by default?
`NUT_THINKING_HOST` defaults to `127.0.0.1`. If you expose the server beyond localhost, protect it with bearer tokens. [E001, E003]

## Configuration

### What environment variables configure the server?

| Env var | Default | Notes |
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

[E001]

### Loop guard variables

| Env var | Default | Meaning |
|---|---|---|
| `NUT_THINKING_GUARD_ENABLED` | `0` | set `1` to enable the loop guard |
| `NUT_THINKING_GUARD_SAME_MODE_BLOCK` | `1` | block same-mode twice-in-a-row |
| `NUT_THINKING_GUARD_WARN_AT` | `3` | consecutive calls before a warning |
| `NUT_THINKING_GUARD_BLOCK_AT` | `5` | consecutive calls before a hard block |

[E001, E008]

## Loop guard

### When is a call blocked?
Two conditions trigger a hard block: (1) same-mode-in-a-row (a hard block by default), and (2) consecutive `think_*` calls at or above `block_at` (default 5). When blocked, the tool responds `{ "ok": false, "blocked": true, "status": "block", "message": "…" }` and nothing is stored. [E001, E008]

### What happens on a warning?
At `warn_at` consecutive calls (default 3), the guard injects state — telling the agent how many times it has called thinking tools in a row and prompting real work or a conscious mode change. Warn does not suppress storage or hooks. [E001, E008]

### Is the guard on by default?
No. It is off by default (`NUT_THINKING_GUARD_ENABLED` defaults to `0`) so the server stays simple out of the box. When enabled and below `warn_at`, calls behave exactly as before. [E001, E008]

## Hooks

### When do hooks fire?
Hooks fire only in the **successful note-storage path**, as background best-effort events — never as a guarantee.

| Event | Fires? | Notes |
|---|---|---|
| `think_<mode>` call, guard disabled / allowed | ✅ yes | after the note is fsync'd to JSONL |
| `think_<mode>` call, guard blocked | ❌ no | no note stored ⇒ no hook; response is `blocked:true` |
| `think_<mode>` call, guard warned (but allowed) | ✅ yes | warn does not suppress storage or hooks |
| Hook script missing for mode + no `on_think.py` | no-op | nothing runs; call unaffected |
| Hook script throws / times out | ⚠️ best-effort | failure isolated; call already returned |
| `health`, `definitions`, `journal` | ❌ no | read-only tools, never fire hooks |
| Token/auth failure | ❌ no | rejected before any storage or hook |
| Note exceeds `NUT_THINKING_MAX_NOTE_CHARS` | ❌ no | rejected before storage; no hook |

Resolution order: `<mode>.py` → else `on_think.py` → else no-op. Mode-specific scripts win. The payload is `{mode, note, actor, ts}` only — no secrets, no request context. [E001, E009]

### What is the baseline loop-break rule?
"Never call the same tool twice in a row." That single sentence is enough to start; the guard is a documented, optional escalation. [E001]

## Licensing

### Under what license is this released?
The README declares **PolyForm Noncommercial 1.0.0** with SPDX badge `PolyForm-Noncommercial-1.0.0`, and a LICENSE file is present in the repository. Note: the repository metadata used in this analysis reported GitHub's automated license detection as "NOASSERTION," a discrepancy between the README declaration and GitHub's detection. The license is source-available but not OSI-approved, and restricts commercial use. [E001, E021]

## Things not verified from pinned evidence

- **Exact JSONL note schema** — the repository's `store.py` content was not part of the provided evidence. The README and hook payload imply fields `id`, `mode`, `note`, `actor`, `ts`, but the exact `store.py` implementation and any additional fields could not be confirmed.
- **`AUDIT.md` contents** — the file exists (security/code audit document) but its contents were not provided in the evidence, so its scope and findings are unknown. [E020]
- **`pyproject.toml` internals** — build configuration, dependencies, and entry points beyond the documented `python -m simple_thinking_tool.server` invocation were not visible in the evidence. [E002]
- **Test suite results** — the README reports 32 passed, 1 skipped from `python -m pytest tests/ -q`, but this could not be independently re-run from the provided evidence. [E001]