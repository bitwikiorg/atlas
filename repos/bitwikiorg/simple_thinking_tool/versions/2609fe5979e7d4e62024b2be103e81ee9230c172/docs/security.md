# Security — simple_thinking_tool

This document describes the security and privacy posture of `simple_thinking_tool` as evidenced at commit `2609fe5979e7d4e62024b2be103e81ee9230c172` (2026-08-05). Claims are tied to evidence IDs; where the repository evidence does not support a statement, that is noted explicitly.

## Security model overview

The project's security review (AUDIT.md, 2026-08-05) enumerates seven areas: authentication, authorization, abuse surface, data integrity, loop-guard, network, and the no-LLM invariant (E009). The design is private-by-default: no outbound network calls, no LLM calls, localhost binding by default, and optional bearer-token authentication (E009, E003, E006).

## Authentication

Authentication is optional and token-based (E002, E009).

- `verify_token(token, agent_token, boss_token)` returns `"agent"`, `"boss"`, or `None` (E002).
- Comparison uses `hmac.compare_digest` (constant-time) (E002).
- `extract_bearer(authorization)` parses an `Authorization` header of the form `Bearer <token>` (case-insensitive on the scheme); other schemes (e.g., `Basic`) return `None` (E002).
- Tokens come from environment variables only: `NUT_THINKING_TOKEN` (agent) and `NUT_THINKING_BOSS_TOKEN` (boss); there are no hardcoded secrets (E003).
- In the HTTP path, the server resolves the actor from the bearer token; if the token is invalid or missing, it raises `PermissionError` (E006).
- In non-HTTP contexts (no HTTP request available), the server falls back to the configured `NUT_THINKING_ACTOR` (default `"agent"`) (E003, E006).

## Authorization and spoofing prevention

- Actor identity is derived from the authenticated token, never from tool input arguments — this is the explicit anti-spoofing design (E002, E009).
- The `journal` tool scopes reads by actor: a boss actor may read any actor's notes (via `actor_id` filter); an agent actor is scoped to its own notes (E006).
- The boss token is described as read-only in the audit (E009).

## Data integrity and durability

- Every append is flushed and `os.fsync`'d before the record is considered written (E008).
- The store is append-only JSONL; rotation archives the file to `<stem>.<timestamp>.bak.jsonl` (E008).
- Idempotency: a note with the same `actor_id` and `run_id` is rejected as a duplicate, preventing accidental double-writes (E008).
- Each note carries an `input_hash` (SHA-256 of selected fields, first 16 hex chars) for integrity tracing (E008).
- On load, a partial tail line is tolerated rather than failing the whole store (E008).

## Abuse surface

- Note length is capped at 800 characters (`NUT_THINKING_MAX_NOTE_CHARS`); exceeding it raises `ValueError` (E003, E008).
- The audit states there are no outside network calls in the code (E009).
- The optional loop guard limits agent churn: same-mode-in-a-row hard block, escalating warning at `warn_at` (default 3), hard ceiling at `block_at` (default 5), per-actor isolation; blocked calls store nothing (E009, E003).

## Network posture

- The HTTP transport binds to `127.0.0.1` by default (`NUT_THINKING_HOST`), with port default `9100` (E003, E009).
- The default transport is `stdio` (E003).
- The audit notes no outbound calls (E009).

## No-LLM invariant

- The server code contains no model calls; it fills templates and appends notes only (E009, E006). This is an architectural invariant, not just a configuration default.

## Hook payload hygiene

- Background hooks receive a JSON payload file containing only `{mode, note, actor, ts}` — explicitly no tokens, no secrets, no config (E005).
- Hooks run as local subprocesses owned by the operator (E005).

## Audit status and open items

- AUDIT.md (2026-08-05) records a self-audit status of PASS, with a full external audit by "Boss" pending (E009).
- The audit's security review items are listed above (E009).
- The audit reports no secrets in the repository; `.gitignore` excludes `state/`, `*.bak.jsonl`, and env files (E009).

## Caveated items

- The loop guard's full decision logic is directly verifiable from `guard.py` (E004): it counts trailing notes with a truthy `mode` field for the actor, includes the current call in the streak (`consecutive = consec + 1`), hard-blocks same-mode-in-a-row, hard-blocks at `block_at`, and warns at `warn_at`. The README's security section is fully present in E001 and consistent with the reviewed modules.
- The analysis graph flags the `exec()`-based dynamic tool generation in `server.py` as a code smell: field names come from `modes.json` (operator-controlled, not user input), but if `modes.json` were ever externally modifiable, this could be an injection vector (confidence 0.75) (E006). The source marks the `exec` with `noqa: S102` and notes it is generated from fixed `modes.json` field names (E006).
- The analysis graph notes there is no input sanitization on template field values, but assesses the risk as low for local string fills (E006, E020).
- The analysis graph recommends validating that `modes.json` field names are safe Python identifiers before use in `exec()`; this validation is not present in the pinned source (E006).
- The audit's "no outside network calls" claim (E009) is consistent with the reviewed modules, including the full README (E001) and `guard.py` (E004) contents.