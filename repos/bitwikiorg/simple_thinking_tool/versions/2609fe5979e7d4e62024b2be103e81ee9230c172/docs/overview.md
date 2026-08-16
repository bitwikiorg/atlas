# Overview — simple_thinking_tool

This document describes the `simple_thinking_tool` repository at pinned SHA [`2609fe5979e7d4e62024b2be103e81ee9230c172`](https://github.com/bitwikiorg/simple_thinking_tool/commit/2609fe5979e7d4e62024b2be103e81ee9230c172) (2026-08-05). [E001]

## Core idea

A small MCP server holds **a catalog of thinking methods**; each method is one tool; the calling LLM picks the tool that fits the problem. "The server does not think; it fills a template from your fields and stores the note in a plain, readable log." [E001]

Why it exists: agents act too fast — they retry broken calls, over-engineer, and burn compute — because nothing forces a deliberate thought between seeing a problem and acting on it. The baseline discipline is a single sentence: **"Never call the same tool twice in a row."** [E001]

## The tool surface (18 tools)

Per the README `The tools` table: `health`, `definitions`, `journal`, and 15 mode tools. [E001]

| Tool | Purpose |
|---|---|
| `health` | Server name, transport, state dir, tool count |
| `definitions` | The catalog — every mode, purpose, when-to-use, fields. Read first |
| `journal` | Read back stored notes (with filters) |
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

Each mode is backed by either an academic paper (arXiv) or an existing thinking-MCP server, or is honestly flagged as an operational pattern with no canonical citation. [E001] The catalog of 15 modes lives in `src/simple_thinking_tool/modes.json`. [E010]

### Adding a mode (no code change)

Modes are **data**. Editing `src/simple_thinking_tool/modes.json` and adding an entry — `id`, `label`, `purpose`, `when`, `level`, `typical_stage`, `fields`, `template`, and optional `references` — auto-registers a `think_<id>` tool and surfaces it in `definitions`. No Python and no redeploy logic is required. [E001]

## How to use / deploy

This is a **private, single-instance server**: each agent runs its own copy; nothing is shared or multi-tenant. It is a standard MCP server consumable by any MCP client (the README names Claude Desktop, Cursor, Agent Zero, and others). [E001]

Local (stdio, default):

```bash
pip install -e .
mkdir -p state
NUT_THINKING_STATE_DIR="$PWD/state" python -m simple_thinking_tool.server
```

Remote (HTTP, optional):

```bash
NUT_THINKING_TRANSPORT=http NUT_THINKING_PORT=9100 python -m simple_thinking_tool.server
```

HTTP binds `127.0.0.1` by default; if exposed beyond localhost, protect it with bearer tokens (`NUT_THINKING_TOKEN`, `NUT_THINKING_BOSS_TOKEN`). [E001]

Quick start (from README):

```bash
python -m venv .venv
. .venv/bin/activate
pip install -e .
python -m pytest tests/ -q          # 32 passed, 1 skipped
mkdir -p state
python -m simple_thinking_tool.server
```

[E001]

## Storage

- Plain **JSONL**, one JSON object per line, append-only, `fsync` after every write.
- No database, no in-memory reindex.
- Rotation is on by default (5 MB); tune with `NUT_THINKING_ROTATE_BYTES` or set `0` to disable.
- The state directory is your private reasoning notebook — keep it out of version control and never commit the JSONL or credentials. [E001]

## Events (hooks)

Thinking tools record; **hooks act**. Set `NUT_THINKING_HOOKS` to a directory of scripts; the server fires a matching script as a fire-and-forget background task after a note is persisted. [E001]

- A script named `<mode>.py` fires for that mode only (e.g. `tree.py` fires for `think_tree`).
- A script named `on_think.py` fires for every mode without a dedicated script.
- Mode-specific scripts win over the generic one.
- Each hook receives one argument: the path to a small JSON payload (`mode`, `note`, `actor`, `ts`) written to the system temp dir and removed after the run.
- Hooks are background and best-effort — treated as events, never a guarantee. [E001]

## Loop guard (optional, off by default)

Because a thinking call "looks like progress" and "cannot fail the way a search or execute can," the optional guard prevents loop churn while keeping the reasoning value. [E001]

- Same-mode-in-a-row → hard block.
- Consecutive-count escalation: warn at `NUT_THINKING_GUARD_WARN_AT` (default 3), hard block at `NUT_THINKING_GUARD_BLOCK_AT` (default 5).
- Enabled via `NUT_THINKING_GUARD_ENABLED=1`; off by default so the server stays simple out of the box.
- Blocked response shape: `{ "ok": false, "blocked": true, "status": "block", "message": "…" }`. [E001]

## Security & privacy posture

- **No egress.** Zero outbound calls.
- **No LLM calls.** It never thinks; it never queries a model.
- **Private by default.** The JSONL is your reasoning notebook.
- **Optional auth.** Bearer tokens protect HTTP transport; the actor derives from the token, never from tool args.
- **Bound localhost.** HTTP binds `127.0.0.1` unless overridden.
- **Public repo only.** Ship code and docs; never ship state, thoughts, keys, or internal context. [E001]

## Configuration reference (env vars)

| Env var | Default | Notes | Evidence |
|---|---|---|---|
| `NUT_THINKING_STATE_DIR` | `state` | where `thoughts.jsonl` lives | [E001] |
| `NUT_THINKING_TRANSPORT` | `stdio` | `stdio` or `http` | [E001] |
| `NUT_THINKING_HOST` | `127.0.0.1` | localhost only | [E001] |
| `NUT_THINKING_PORT` | `9100` | HTTP transport port | [E001] |
| `NUT_THINKING_TOKEN` | — | optional bearer token (agent) | [E001] |
| `NUT_THINKING_BOSS_TOKEN` | — | optional read-only token | [E001] |
| `NUT_THINKING_MAX_NOTE_CHARS` | `800` | note cap | [E001] |
| `NUT_THINKING_ROTATE_BYTES` | `5242880` | JSONL rotation threshold | [E001] |
| `NUT_THINKING_HOOKS` | — | directory of background hook scripts (empty = disabled) | [E001] |
| `NUT_THINKING_HOOK_TIMEOUT` | `10.0` | max seconds a hook may run | [E001] |
| `NUT_THINKING_GUARD_ENABLED` | `0` | set `1` to enable the loop guard | [E001] |
| `NUT_THINKING_GUARD_SAME_MODE_BLOCK` | `1` | block same-mode twice-in-a-row | [E001] |
| `NUT_THINKING_GUARD_WARN_AT` | `3` | consecutive calls before a warning | [E001] |
| `NUT_THINKING_GUARD_BLOCK_AT` | `5` | consecutive calls before a hard block | [E001] |

Configuration and guard/hook settings are read from environment variables by the configuration layer (`src/simple_thinking_tool/config.py`), per the module inventory. [E003]

## What could not be verified

- Full `pyproject.toml` and `requirements.txt` contents (dependencies, build config, any `console_scripts` beyond the `python -m simple_thinking_tool.server` invocation). [E002], [E022]
- `AUDIT.md` findings. [E020]
- Exact per-mode field lists and templates inside `modes.json`; this document lists the 15 mode names from the README but not each mode's exact schema. [E010], [E001]
- The exact JSONL note schema beyond what the README and hook payload imply (`mode`, `note`, `actor`, `ts`, unique `id`). [E001]
- License-detection mismatch (`NOASSERTION` on GitHub metadata vs. PolyForm Noncommercial in `LICENSE`) — the cause is unknown. [E021], [E001]

See [docs/architecture.md](architecture.md) for the internal module structure and request lifecycle.