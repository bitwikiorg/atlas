# simple_thinking_tool — BITwiki Atlas

**Pinned source:** [`bitwikiorg/simple_thinking_tool`](https://github.com/bitwikiorg/simple_thinking_tool) at commit [`2609fe5979e7d4e62024b2be103e81ee9230c172`](https://github.com/bitwikiorg/simple_thinking_tool/commit/2609fe5979e7d4e62024b2be103e81ee9230c172) (2026-08-05). [E001]

This atlas documents the repository as it exists at the pinned SHA. Claimed behavior is tied to evidence IDs (e.g. `[E001]`) and to files in the pinned tree; links below point at the `main` branch and should be read against the pinned SHA.

## What this repository is

`simple_thinking_tool` is a small **Model Context Protocol (MCP) server** written in Python (target 3.12) that holds **a catalog of thinking methods** ([E001], [E002]). Each method is exposed as one MCP tool. The design principle is **"the LLM is the router"**: the server contains no deterministic routing logic; the model that consumes the tools reads each tool's description and picks the fitting method. [E001]

The server itself **never thinks**:

- It never calls an LLM.
- It makes zero outbound network calls ("no egress").
- Its only operation is filling a string template from caller-supplied fields and appending the rendered note to a plain, readable JSONL log. [E001]

> [!NOTE]
> The README states the rationale directly: "For an LLM, the text it produces *is* the thinking. Forcing articulation forces the cognition. This tool inserts a required checkpoint between 'I could answer' and 'I act.'" [E001]

## Fast facts (verified from evidence)

| Item | Value | Evidence |
|---|---|---|
| Package name | `simple_thinking_tool` | [E001] |
| Language / target | Python 3.12 | [E002], [E001] |
| Protocol | MCP server; stdio (default) or HTTP transport | [E001], [E004] |
| Deployment model | single-serve, private (one instance per agent) | [E001] |
| Tool surface | 18 tools: `health`, `definitions`, `journal`, and 15 `think_<mode>` tools | [E001], [E010] |
| Persistence | append-only JSONL with `fsync` per write; rotation on by default (5 MB) | [E001], [E013] |
| LLM calls / egress | none by design | [E001] |
| Loop guard | optional, off by default (`NUT_THINKING_GUARD_ENABLED`) | [E001], [E008] |
| Hooks | optional fire-and-forget background scripts | [E001], [E009] |
| License | PolyForm Noncommercial 1.0.0 | [E001], [E021] |
| Test status (per README) | 32 passed, 1 skipped | [E001] |

## Repository layout (pinned SHA)

- `src/simple_thinking_tool/` — the package: `server.py`, `service.py`, `store.py`, `modes.py` (+ `modes.json`), `guard.py`, `hooks.py`, `auth.py`, `config.py`, `__init__.py`. [E003]–[E013]
- `tests/` — seven test modules mapped ~1:1 to source modules. [E005], [E014]–[E019]
- `README.md`, `AUDIT.md`, `LICENSE`, `pyproject.toml`, `requirements.txt`, `assets/`. [E001], [E002], [E020], [E021], [E022]

## Where to go next

- [docs/overview.md](docs/overview.md) — purpose, the tool catalog, deployment, storage, hooks, guard, configuration, and security posture.
- [docs/architecture.md](docs/architecture.md) — module map, request lifecycle, mode registration, guard/error paths, and test coverage.

## Verification limits

The following could **not** be verified from the provided evidence and should be read as unknown:

- The full contents of `pyproject.toml` ([E002]) and `requirements.txt` ([E022]) — manifest and dependency details are beyond what the evidence exposes.
- The contents and findings of `AUDIT.md` ([E020]) — the audit document exists but its scope is not disclosed in this evidence pack.
- Exact source-level details of `config.py`, `server.py`, `service.py`, `store.py`, `modes.py`, `guard.py`, `hooks.py`, `auth.py`, and `__init__.py` (whether `__init__.py` exports public APIs, exact import edges, and the full JSONL note schema) — these are inferred from the README and module naming and are not fully visible in the provided evidence. [E003]–[E013]
- GitHub's license detection field reports `NOASSERTION` despite the README and LICENSE file asserting PolyForm Noncommercial 1.0.0 ([E021], [E001]); the cause of the mismatch is unknown.

No CI configuration files were detected in the pinned tree (CI files: 0 in the repository summary).