# Quick Start

This guide gets a local, private instance of `simple_thinking_tool` — an MCP server that holds a catalog of thinking methods — up and running. The server itself never thinks: it fills templates from your fields and appends the rendered note to a plain JSONL log. The LLM that consumes the server is the router that picks the right thinking tool.

## Prerequisites

- Python 3.12 (per project metadata).
- `pip` and the ability to create a virtual environment.

## Install

From the repository root:

```bash
python -m venv .venv
. .venv/bin/activate
pip install -e .
```

Run the test suite to confirm the install (per the README this reports 32 passed, 1 skipped):

```bash
python -m pytest tests/ -q
```

## First run (stdio transport, default)

Create a state directory, set it via `NUT_THINKING_STATE_DIR`, and start the server:

```bash
mkdir -p state
NUT_THINKING_STATE_DIR="$PWD/state" python -m simple_thinking_tool.server
```

The default transport is stdio, spawned locally — no port, no network.

## Register in an MCP client

Point your MCP client at the server. A generic client config that spawns it locally:

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

Any harness that speaks MCP can consume it (examples named in the README include Claude Desktop, Cursor, and Agent Zero). Each agent runs its own copy — this is a single-serve, private server.

## Call the catalog first

Call the **`definitions`** tool. It lists every mode, what it does, when to use it, and its required fields. Read this catalog before the first call; this is how an LLM learns to route.

## Pick a mode and fill its fields

Each mode is its own tool with explicit string fields. Example from the README:

```
think_goal_distill(
  priority="ship v2",
  why="users blocked on auth",
  action="write PRD"
)
```

The server fills the template and appends the rendered note to the log.

## The loop rule

- Never call the same mode tool twice in a row.
- A thinking call is a checkpoint *before* work, not work itself. After 3–5 thinking calls, produce a real, verifiable action.

## Audit

Call **`journal`** to read back stored notes. The raw log is a plain JSONL file openable in any editor.

## Serve over HTTP (optional)

For harnesses that need a network endpoint:

```bash
NUT_THINKING_TRANSPORT=http NUT_THINKING_PORT=9100 python -m simple_thinking_tool.server
```

Binds `127.0.0.1` by default. If you expose it beyond localhost, protect it with bearer tokens (`NUT_THINKING_TOKEN`, `NUT_THINKING_BOSS_TOKEN`). See the configuration guide for details.

## What the server does not do

- It makes no outbound network calls (no egress).
- It never calls an LLM.
- It keeps your thinking log private: keep the state directory out of version control, and never commit the JSONL or credentials.

> Confidence note: this guide reflects the pinned repository evidence (README at commit `2609fe5` and supporting source layout). Exact tool response fields beyond what the README shows could not be verified from the provided source blobs.

---

Evidence references: [E001] (all commands, the loop rule, the `think_goal_distill` example, and the transport/config env vars are sourced from the README).