# Overview — simple_thinking_tool

This document provides a high-level overview of `simple_thinking_tool`, a private Model Context Protocol (MCP) server that exposes a catalog of thinking methods as individual tools. It is written from the pinned source at commit `2609fe5979e7d4e62024b2be103e81ee9230c172` (2026-08-05).

- **Repository:** https://github.com/bitwikiorg/simple_thinking_tool
- **Version:** 0.1.0 [E018]
- **License:** PolyForm Noncommercial 1.0.0 [E010]
- **Python:** `>=3.11` [E011]
- **Runtime dependency:** `fastmcp>=3.2,<4` [E011]

## What it is

`simple_thinking_tool` is a lightweight MCP server that holds a catalog of 15 named thinking methods. Each method is exposed as one MCP tool named `think_<id>`. The calling agent (the LLM) reads each tool's description and selects the appropriate reasoning mode — **the LLM is the router**. The server itself never calls an LLM, makes no outbound network calls, and performs no reasoning: it fills a mode-specific template from caller-provided string fields and appends the rendered note to a durable, append-only JSONL log with `fsync`. [E001][E006]

The project is designed as a **single-serve private instance** — one copy per agent or user — with no multi-tenancy or shared state. [E001]

## Core design principles

### 1. The LLM is the router

There is no server-side deterministic router. The routing logic lives in the tool descriptions: each mode tool's description states its purpose, when to use it, and its required fields. The model's native tool-picker chooses the mode. A second deterministic router is described in the project's own FAQ as "redundant machinery." [E001][E006]

### 2. Modes are data

Thinking modes are defined in `src/simple_thinking_tool/modes.json` as JSON entries, not in code. Adding, editing, or removing a mode requires only a JSON edit — tools and definitions auto-register from the file. [E001][E019][E020]

### 3. The server never thinks

The server fills template strings from caller fields and appends them to the log. No model calls, no network egress, ever. This is an explicit invariant documented in the audit report. [E001][E009]

### 4. Forcing articulation forces cognition

For an LLM, the text it produces *is* the thinking. The tool inserts a required checkpoint between "I could answer" and "I act." [E001]

## Why this exists

Agents act too fast — they retry broken calls, over-engineer, and burn compute, because nothing forces a deliberate thought between *seeing a problem* and *acting on it*. A thinking tool must be trivial to call or it never gets called. The baseline loop-break rule is a single sentence: **never call the same tool twice in a row.** [E001]

## The thinking-methods catalog

The 15 modes are defined in `modes.json` [E019]. Each mode has an `id`, `label`, `purpose`, `when` (when-to-use guidance), `level` (1–4 depth indicator), `typical_stage` (plan / understand / execute / review / handoff / any), `fields`, and a `template`. [E019][E020]

### Level 1 — Core (cheapest, most-used)

| Mode | Typical stage | Purpose |
|---|---|---|
| `goal_distill` | plan | Reduce the entire turn to one objective, its core priority, and the first action. Use by default at the start of any task. |
| `understand` | understand | Digest context and content: code, tools, documents, user intents. |
| `plan` | plan | Structure ordered steps before execution. |
| `chain_of_thought` | execute | Walk a linear chain of intermediate conclusions from premise to answer. |

### Level 2 — Structured

| Mode | Typical stage | Purpose |
|---|---|---|
| `creative` | any | Generate open directions and wildcard ideas without premature selection. |
| `deep` | execute | Rephrase, decompose, hypothesize, verify, then synthesize. |
| `distill` | handoff | Compress what survives rephrasing into its essence. |
| `reflect` | review | Socratic self-interrogation: what is actually true here? |
| `first_principles` | any | Reduce to fundamental facts and rebuild without inherited assumptions. |

### Level 3 — Advanced

| Mode | Typical stage | Purpose |
|---|---|---|
| `tree` | any | Branch multiple paths, score them, prune weak ones, pick a branch. |
| `graph` | any | Build partial solutions as a graph with feedback loops, then aggregate. |
| `bias_check` | review | Scan for confirmation, survivorship, and authority bias before a high-stakes call. |
| `metacognitive` | review | Audit the reasoning itself for bias, uncertainty, and contradictions. |

### Level 4 — High-stakes

| Mode | Typical stage | Purpose |
|---|---|---|
| `reinforced` | review | Multiple complete takes plus an adversarial red-team attack. |
| `adversarial_review` | review | Attempt to disprove the provisional conclusion. |

[E019]

## The tool surface

18 tools total: `health`, `definitions`, `journal`, and 15 mode tools. [E001][E009]

- `health` — server name, transport, state dir, tool count.
- `definitions` — the catalog; every mode, purpose, when-to-use, fields. The LLM is expected to call this first.
- `journal` — read back stored notes, with optional `after_ts` and `actor_id` filters.
- `think_<mode>` — one tool per mode, each with explicit string fields. [E001][E006]

## Provenance and references

Each mode's idea comes from academic papers (arXiv) or existing thinking-MCP servers. The project is explicit about provenance: where a mode maps to a real paper or server, it is cited; where it is an operational pattern with no canonical source, that is stated out loud rather than inventing a citation. [E001]

Papers backing modes include Chain-of-Thought ([#15], arXiv 2201.11903), Tree of Thoughts ([#17], arXiv 2305.10601), Graph of Thoughts ([#18], arXiv 2308.09687), and Plan, Verify and Switch ([#26], arXiv 2310.14628). Operational patterns with no canonical citation include `goal_distill`, `understand`, `distill`, `adversarial_review`, and `reflect`. [E001]

## Extensibility

- **Add a mode:** edit `modes.json` — no code change. It auto-registers as `think_<id>` and appears in `definitions`. [E001][E019]
- **Add a hook:** drop a script named `<mode>.py` (or `on_think.py`) into the hooks directory; the server fires it as a background task after a matching note is stored. [E001][E005]
- **Configure the guard:** enable and tune the optional loop guard via `NUT_THINKING_GUARD_*` env vars. [E001][E003]

## Security posture

- No outbound calls; no LLM calls. [E001][E009]
- Optional bearer-token auth for HTTP; actor identity derives from the token, never from tool args (constant-time comparison via `hmac.compare_digest`). [E002]
- HTTP binds `127.0.0.1` by default. [E001]
- Note cap of 800 chars by default. [E003][E008]
- `.gitignore` excludes `state/`, `*.bak.jsonl`, and env files. [E009]

## Audit status

`AUDIT.md` documents a self-audit of the 0.1.0 release (auditor: Peanutoshi, status PASS, external audit pending). Verified evidence includes a pytest suite reported as **43 passed, 1 skipped**, `py_compile` OK, an 18-tool surface via `FastMCP.list_tools()`, and a prod smoke test of `think_goal_distill`. Note that the README reports "32 passed, 1 skipped" — the two counts are not reconciled in the pinned source. [E009][E001]

## Where to go next

- **Architecture:** see `docs/architecture.md` for the component map, request lifecycle, storage design, loop guard, hooks system, and authentication.
- **Source of truth:** the repository README at the pinned SHA [E001] contains the full usage guide, configuration table, references, and FAQ.