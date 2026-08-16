# Glossary — simple_thinking_tool

Terms are grounded in the repository at commit `2609fe5979e7d4e62024b2be103e81ee9230c172` (v0.1.0). Evidence IDs reference the pinned source files.

## MCP
Model Context Protocol — a standard protocol for connecting AI agents to external tools and data sources. This server is a standard MCP server consumable by any MCP client (Claude Desktop, Cursor, Agent Zero, etc.) (E001).

## FastMCP
The Python framework (v3.2+, `<4`) used to build the MCP server. It provides tool registration, transport handling, and the server runtime (E006, E011, E022).

## Thinking Mode
A named reasoning method (e.g., `goal_distill`, `tree`, `bias_check`) defined as a JSON entry in `modes.json`. Each mode auto-registers as a tool named `think_<id>` with its fields as explicit required string arguments (E001, E019, E020). There are 15 modes in v0.1.0 (E019, E021).

## LLM-as-Router
The core design principle where the calling LLM selects which thinking mode to use by reading each tool's description, rather than relying on a server-side deterministic router. The server provides no routing logic (E001, E006). The server instructions state: "You are the router: read the tool descriptions and pick the mode that fits the problem" (E006).

## Modes-as-Data
The architecture where thinking modes are defined in `modes.json` (a data file) rather than in code. Adding, editing, or removing a mode requires only a JSON edit — no Python changes, no redeploy of logic. Tools and definitions auto-register from the file (E001, E019, E020).

## JSONL Store
The append-only JSON Lines log file (`thoughts.jsonl`) that stores notes (and, in the store API, session records). Each record is one JSON object per line. Writes are `fsync`'d for durability. Rotation occurs when the file exceeds a configurable size threshold (default 5 MB) (E001, E008).

## Loop Guard
An optional anti-churn mechanism (off by default, enabled via `NUT_THINKING_GUARD_ENABLED=1`) that blocks same-mode-in-a-row calls (hard block), issues escalating warnings at a consecutive-call threshold (`warn_at`, default 3), and hard-blocks at a ceiling threshold (`block_at`, default 5). It is per-actor isolated (E001, E004, E009).

## Hook
A background script dropped into a hooks directory (`NUT_THINKING_HOOKS`) that the server fires as a fire-and-forget task after a matching note is persisted. Mode-specific scripts (`<mode>.py`) take precedence over a generic `on_think.py`. Hooks receive a JSON payload file with `{mode, note, actor, ts}` (E001, E005).

## Actor
The identity derived from the bearer token at authentication time — either `agent` (`NUT_THINKING_TOKEN`) or `boss` (`NUT_THINKING_BOSS_TOKEN`). Actor identity never comes from tool arguments, preventing spoofing (E001, E002, E009).

## Note
A rendered thinking record stored in the JSONL log. Contains the mode id, template output (capped at `NUT_THINKING_MAX_NOTE_CHARS`, default 800), actor id, timestamp, and metadata. The `type` field distinguishes notes from session records (E001, E008).

## Template Rendering
The process of filling a mode's template string with caller-provided field values using Python string formatting. Missing fields are filled with empty strings rather than raising errors (E020, E009). The audit notes this was a fixed bug: missing fields fill `''` instead of raising `KeyError` (E009).

## Mode Level
A difficulty/depth indicator (1–4) assigned to each thinking mode. Level 1 modes (`goal_distill`, `understand`, `plan`, `chain_of_thought`) are the cheapest and most-used; level 4 modes (`reinforced`, `adversarial_review`) are for high-stakes conclusions (E019).

## Typical Stage
The workflow stage a thinking mode is designed for: `plan`, `understand`, `execute`, `review`, `handoff`, or `any`. Used for organization in the definitions catalog (E019).

## Rotation
The process of archiving the JSONL log file when it exceeds `NUT_THINKING_ROTATE_BYTES` (default 5 MB). The current file is renamed with a timestamp suffix (`.bak.jsonl`) and a new empty file is created for subsequent appends (E008).

## Idempotency
The `run_id`-based deduplication mechanism in the store. If a note with the same `actor_id` and `run_id` already exists, the append is rejected as a duplicate, preventing accidental double-writes (E008, E016).

## Bearer Token
An optional authentication token passed in the HTTP `Authorization` header. `NUT_THINKING_TOKEN` grants `agent` identity; `NUT_THINKING_BOSS_TOKEN` grants `boss` identity (read-only, cross-actor journal access). Verified using constant-time comparison (`hmac.compare_digest`) (E001, E002, E009).

## Definitions Tool
One of three non-mode tools (alongside `health` and `journal`). Returns the full catalog of every thinking mode — its purpose, when-to-use guidance, fields, and template. The LLM is expected to call this first to learn the catalog before routing (E001, E006, E020).

## Journal Tool
A read-only tool that reads back stored notes from the JSONL log. Supports optional `after_ts` (millisecond epoch) and `actor_id` filters. Boss actors can read any actor's notes; agent actors are scoped to their own (E006, E009).

## No-LLM Invariant
A core security guarantee: the server code contains no model calls, no API calls to any LLM, and no outbound network requests. It only fills templates and appends notes locally (E001, E009).

## Loop-Break Rule
The baseline rule: never call the same tool twice in a row. This is the default anti-churn discipline before the optional loop guard is enabled (E001, E006).

## GuardResult
A dataclass returned by `Guard.evaluate()` with fields `allowed` (bool), `status` (`ok` | `warn` | `block`), `consecutive` (int), `last_mode` (str | None), and `message` (str) (E004).

## Settings
The environment-driven configuration class with fields for tokens, state directory, host/port, transport, max note chars, rotation bytes, hooks directory/timeout, and guard thresholds. All values come from environment variables; no hardcoded secrets (E003).

## Mode
A frozen dataclass representing a thinking mode with fields `id`, `label`, `purpose`, `when`, `level`, `typical_stage`, `fields` (tuple of str), `template`, and `version` (E020).

## input_hash
A SHA-256 hash (first 16 hex chars) of the note's key fields, stored on each note record for integrity tracing (E008).

## ULID-like ID
The record identifier format generated by the store: a 12-hex-digit millisecond timestamp prefix plus a 16-hex-character UUID suffix (E008).