# Quickstart

This guide gets a private instance of **simple-thinking-tool** running and makes your first thinking-tool call. The project is a small Model Context Protocol (MCP) server that exposes a catalog of named thinking methods as individual tools. The calling agent (the LLM) is the router: it reads each tool's description and picks the mode that fits the problem. The server itself never calls an LLM and makes no outbound network calls — it fills a mode template from your fields and appends the rendered note to a local JSONL log [E001].

## Prerequisites

- **Python 3.11+** — declared in `pyproject.toml` (`requires-python = ">=3.11"`) [E011].
- A single third-party dependency: `fastmcp>=3.2,<4` [E011][E022].

## Install

From the repository root:

```bash
python -m venv .venv
. .venv/bin/activate
pip install -e .
```

The editable install uses the setuptools build backend configured in `pyproject.toml` [E011].

## Run the test suite (optional)

```bash
python -m pytest tests/ -q
```

The README reports `32 passed, 1 skipped` for this command [E001], while `AUDIT.md` reports `43 passed, 1 skipped` [E009]. The two documents disagree on the exact count; run the suite yourself to confirm the current number. The suite covers all eight source modules across seven test files [E012–E017][E021].

## Start the server

The default transport is **stdio** — the server is spawned locally by your MCP client, with no port and no network [E001].

```bash
mkdir -p state
python -m simple_thinking_tool.server
```

The state directory defaults to `state` (relative to the working directory) and is where `thoughts.jsonl` lives [E001][E003]. You can point it elsewhere with `NUT_THINKING_STATE_DIR`; see [configuration.md](configuration.md).

## Register in your MCP client

This is a standard MCP server, so any MCP client can consume it — the README names Claude Desktop, Cursor, and Agent Zero as examples [E001]. Generic client config that spawns it locally over stdio:

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
```

[E001]

## Make your first call

The server registers three non-mode tools plus one tool per thinking mode [E006]:

| Tool | Purpose |
|---|---|
| `health` | Server name, transport, state dir, tool count |
| `definitions` | The catalog — every mode, purpose, when-to-use, fields. Read first. |
| `journal` | Read back stored notes (with optional filters) |
| `think_<mode>` | One tool per thinking mode, e.g. `think_goal_distill` |

### 1. Learn the catalog

Call **`definitions`** first. It lists every mode, what it does, when to use it, and its required fields. This is how the LLM learns to route [E001][E006].

### 2. Pick a mode and fill its fields

Each mode is its own tool with explicit string fields. Example from the README:

```
think_goal_distill(
  priority="ship v2",
  why="users blocked on auth",
  action="write PRD"
)
```

The server fills the mode template from your fields and appends the rendered note to the log [E001].

### 3. Follow the loop rule

- **Never call the same mode tool twice in a row.** This is the baseline loop-break rule [E001].
- A thinking call is a checkpoint *before* work, not work itself. After 3–5 thinking calls, produce a real, verifiable action [E001].
- Depth is chosen at call time: the tool forces the *form*, not the *depth* [E001].

### 4. Audit

Call **`journal`** to read back stored notes. The raw log is a plain JSONL file you can open in any editor [E001].

## What happens on a call

When you call a `think_<mode>` tool, the server: resolves the actor from the bearer token (HTTP transport only), evaluates the optional loop guard, renders the mode template from your fields, appends the note to the JSONL log with `fsync`, fires any configured background hook, and returns `{ok, mode, note, id, hook?}` [E006][E008][E005].

## Next steps

- All environment variables and transport/auth options: [configuration.md](configuration.md)
- Adding your own thinking modes and writing hooks: [extension.md](extension.md)