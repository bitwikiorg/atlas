# simple_thinking_tool

A small, private Model Context Protocol (MCP) server that holds a **catalog of thinking methods**. Each method is one tool. You pick the tool that fits the problem — **the LLM is the router**. The server does not think; it fills a template from your fields and stores the note in a plain, readable log.

- **Repository:** https://github.com/bitwikiorg/simple_thinking_tool
- **Pinned source SHA:** `2609fe5979e7d4e62024b2be103e81ee9230c172`
- **Version:** 0.1.0 [E018]
- **License:** PolyForm Noncommercial 1.0.0 [E010]
- **Python:** `>=3.11` [E011]

> For an LLM, the text it produces *is* the thinking. Forcing articulation forces the cognition. This tool inserts a required checkpoint between "I could answer" and "I act." [E001]

---

## The core idea

A small MCP server that holds a catalog of thinking methods. Each method is one tool. The LLM reads each tool's description and picks the mode that fits the problem — the server provides no routing logic. The server fills a mode-specific template from caller-provided string fields and appends the rendered note to a durable, append-only JSONL log with `fsync`. [E001][E006]

The baseline loop-break rule is a single sentence:

> **Never call the same tool twice in a row.** [E001]

## Why this exists

Agents act too fast — they retry broken calls, over-engineer, and burn compute, because nothing forces a deliberate thought between *seeing a problem* and *acting on it*. A thinking tool must be **trivial to call** or it never gets called. So this one is: one tool per mode, one string per field, no sessions, no guards, no config by default. [E001]

## Features

- **The LLM is the router.** Every tool has a description saying what it does and *when* to use it. The model's native tool-picker chooses. [E001]
- **One tool per mode.** `think_goal_distill`, `think_tree`, `think_bias_check`, … each with its own required string fields. [E001]
- **Modes are data.** They live in `modes.json`. Add or edit a mode there — no Python, no redeploy logic; tools and definitions auto-register. [E001][E019]
- **The server never thinks.** It only fills templates and appends notes. Zero LLM calls, zero egress. [E001][E006]
- **Optional loop guard.** Off by default; blocks same-mode-in-a-row churn and escalates warnings for consecutive `think_*` calls. [E004]
- **Background event hooks.** Fire-and-forget scripts after a note is stored, for post-processing automation. [E005]
- **Private by default.** Single-serve instance; the JSONL log is your reasoning notebook. [E001]

## Quick start

```bash
python -m venv .venv
. .venv/bin/activate
pip install -e .
python -m pytest tests/ -q

mkdir -p state
python -m simple_thinking_tool.server
```

That's it. Register the server in your MCP client, call `definitions`, then call a mode tool. [E001]

> Note: the README reports "32 passed, 1 skipped" while `AUDIT.md` reports "43 passed, 1 skipped" for the pytest suite — the two counts are not reconciled in the pinned source. [E001][E009]

## How to use

This is a **private, single-instance server**: each agent runs its own copy. It is a standard MCP server, so any agent on any harness can consume it — Claude Desktop, Cursor, Agent Zero, and others. Nothing is shared or multi-tenant. [E001]

1. **Install locally** — `pip install -e .`
2. **Register it in your MCP client** — point your client at the server. Default transport is stdio (spawned locally); an optional HTTP transport is available.
3. **Learn the catalog first** — call the `definitions` tool. It lists every mode, what it does, when to use it, and its required fields.
4. **Pick a mode and fill its fields** — each mode is its own tool with explicit string fields.
5. **Follow the loop rule** — never call the same mode tool twice in a row; a thinking call is a checkpoint *before* work, not work itself.
6. **Audit** — call `journal` to read back stored notes. The raw log is a plain JSONL file. [E001]

Example call:

```
think_goal_distill(
  priority="ship v2",
  why="users blocked on auth",
  action="write PRD"
)
``` [E001]

## How to serve

### Local (stdio — default)

```bash
pip install -e .
mkdir -p state
NUT_THINKING_STATE_DIR="$PWD/state" python -m simple_thinking_tool.server
```

Generic client config that spawns it locally:

```json
{
  "mcpServers": {
    "simple-thinking-tool": {
      "command": "python3",
      "args": ["-m", "simple_thinking_tool.server"],
      "env": {
        "NUT_THINKING_STATE_DIR": "/absolute/path/to/your/state"
      }
    }
  }
}
``` [E001]

### Remote (HTTP — optional)

```bash
NUT_THINKING_TRANSPORT=http NUT_THINKING_PORT=9100 python -m simple_thinking_tool.server
```

Binds `127.0.0.1` by default. If you expose it beyond localhost, protect it with bearer tokens (`NUT_THINKING_TOKEN`, `NUT_THINKING_BOSS_TOKEN`). [E001]

## The tools

18 tools total: `health`, `definitions`, `journal`, and 15 mode tools. [E001]

| Tool | Purpose |
|---|---|
| `health` | Server name, transport, state dir, tool count |
| `definitions` | The catalog — every mode, purpose, when-to-use, fields. Read first. |
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

The full mode catalog with fields, templates, levels, and academic references is defined in `src/simple_thinking_tool/modes.json` [E019]. See `docs/overview.md` for the catalog organized by level and stage.

## Add a thinking mode

Edit `src/simple_thinking_tool/modes.json`. Add one entry:

```json
{
  "id": "my_mode",
  "label": "My Mode",
  "purpose": "One sentence on what it does.",
  "when": "Use when ...",
  "level": 1,
  "typical_stage": "any",
  "fields": ["input_a", "input_b"],
  "template": "Input A: {input_a}\nInput B: {input_b}",
  "references": ["https://arxiv.org/abs/XXXX"]
}
```

It auto-registers as `think_my_mode(input_a: str, input_b: str)` and appears in `definitions`. No code changed. [E001][E019]

## Storage

- Plain **JSONL**, one JSON object per line, append-only, `fsync` after every write. [E001][E008]
- Open it in any editor — the audit trail is just a file. [E001]
- No database, no in-memory reindex. Rotation is on by default (5 MB) — tune with `NUT_THINKING_ROTATE_BYTES` or set it to `0` to disable. [E001][E008]

## Events (hooks)

Thinking tools record; **hooks act**. A hook is a background script you drop into a directory; the server fires it as a fire-and-forget task after a matching note is persisted. [E001][E005]

- A script named `<mode>.py` fires for that mode only (e.g. `tree.py` fires for `think_tree`).
- A script named `on_think.py` fires for every mode without a dedicated script.
- Mode-specific scripts win over the generic one.

Each hook receives one argument: the path to a small JSON payload (`mode`, `note`, `actor`, `ts`) written to the system temp dir and removed after the run. [E001][E005]

Hooks fire only in the **successful note-storage path**. They are background and best-effort — treat them as events, never as a guarantee. [E001]

## Loop guard (optional, off by default)

Thinking tools are oddly addictive to agents. A call *looks like* progress and *cannot fail* the way a search or execute can. This optional guard prevents that churn while keeping the reasoning value. [E001]

- **Same-mode-in-a-row: hard block.** No reasoning benefit comes from repeating the identical mode.
- **Consecutive-count escalation (soft → hard):** at `warn_at` (default **3**) consecutive `think_*` calls, inject state; at `block_at` (default **5**), block until the agent performs a real, verifiable action.
- **Lightweight + togglable:** enable/disable and configure thresholds via env vars, off by default. [E001][E004]

When a call is blocked, the tool responds `{ "ok": false, "blocked": true, "status": "block", "message": "…" }` and nothing is stored. [E001][E004]

## Configuration

All configuration is environment-driven; no hardcoded secrets. [E003]

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
| `NUT_THINKING_GUARD_ENABLED` | `0` | set `1` to enable the loop guard |
| `NUT_THINKING_GUARD_SAME_MODE_BLOCK` | `1` | block same-mode twice-in-a-row |
| `NUT_THINKING_GUARD_WARN_AT` | `3` | consecutive calls before a warning |
| `NUT_THINKING_GUARD_BLOCK_AT` | `5` | consecutive calls before a hard block |

The `Settings` class in `config.py` also reads `NUT_THINKING_BURST_LIMIT`, `NUT_THINKING_ACTOR`, and `NUT_THINKING_ALLOW_MODES`; these are parsed in code [E003] but are not documented in the README table [E001].

## Security & privacy

- **No egress.** The server makes zero outbound calls. It fills templates and appends to local JSONL. [E001]
- **Hooks are local.** Background hooks (if enabled) run as your own subprocess; the server still makes no outbound calls and sends no secrets. [E001]
- **No LLM calls.** It never thinks; it never queries a model. [E001]
- **Private by default.** The JSONL is your reasoning notebook. Keep it out of git; never commit state or secrets. [E001]
- **Optional auth.** Bearer tokens (`NUT_THINKING_TOKEN`, `NUT_THINKING_BOSS_TOKEN`) protect HTTP transport; actor derives from the token, never from tool args. [E001][E002]
- **Bound localhost.** HTTP binds `127.0.0.1` unless you explicitly override. [E001]

## References

Each mode's idea comes from academic papers (arXiv) or existing thinking-MCP servers. Where no canonical paper exists, that is stated honestly rather than invented. Full citation tables are in the README of the pinned source [E001] and summarized in `docs/overview.md`.

## Repository layout

```text
simple_thinking_tool/
├── src/simple_thinking_tool/
│   ├── server.py      # FastMCP wiring — registers all tools; the LLM is the router
│   ├── service.py     # lean service layer (append/definitions/journal/health)
│   ├── store.py       # JSONL append-only store, fsync
│   ├── modes.py       # modes loader + template render + definitions
│   ├── modes.json     # edit this to add/change modes (data, not code)
│   ├── hooks.py       # background event hooks (fire-and-forget, off by default)
│   ├── config.py      # env-driven settings, no hardcoded secrets
│   └── auth.py        # optional bearer token → actor
├── tests/             # pytest suite
├── assets/            # inline SVGs (hero banner)
├── pyproject.toml
└── requirements.txt
``` [E001]

## License

PolyForm Noncommercial 1.0.0. See [LICENSE](https://github.com/bitwikiorg/simple_thinking_tool/blob/2609fe5979e7d4e62024b2be103e81ee9230c172/LICENSE) [E010]. The license permits noncommercial use and modification but does not permit distribution of the software or derivative works. [E010]

---

> A thinking tool must feed work, not replace it. Shell steady. 🌰 [E001]