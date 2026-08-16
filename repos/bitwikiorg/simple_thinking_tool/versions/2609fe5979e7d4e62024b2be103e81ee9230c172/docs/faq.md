# FAQ — simple_thinking_tool

This FAQ covers the most common questions about the `simple_thinking_tool` MCP server. Answers are grounded in the repository at commit `2609fe5979e7d4e62024b2be103e81ee9230c172` (v0.1.0, released 2026-08-05). Where a claim could not be verified from the pinned source, that is stated explicitly.

## Does the server call an LLM?

No. The server fills template strings from your fields and appends them to the log. It makes no model calls and no network egress, ever (E001, E006, E009). The "no-LLM invariant" is documented in the audit as a core guarantee: the server code contains no model calls and only fills templates and stores notes (E009).

## Why is there no router inside the server?

Because the LLM calling the tools is already the router. A second deterministic router would be redundant machinery that failed by gating modes behind priority lanes. The tool descriptions carry the routing logic (E001, E006). The server's own instructions tell the calling agent: "You are the router: read the tool descriptions and pick the mode that fits the problem" (E006).

## Why JSONL and not a database?

It is the readable notebook: open the file, read the audit. Append-only means no corruption risk, and a dump tool is trivial. This is the "notepad" use case, not a transactional system (E001). The store is a plain append-only JSONL log with `fsync` after every write (E008).

## How do I stop loop-calling?

One rule: never call the same tool twice in a row. That is the baseline loop-break rule (E001, E006). If loop-abuse appears, an optional loop guard can be enabled via `NUT_THINKING_GUARD_ENABLED=1`; it hard-blocks same-mode-in-a-row calls, issues escalating warnings at `warn_at` (default 3), and hard-blocks at `block_at` (default 5) (E001, E004, E009). The guard is off by default (E004, E009).

## Can any agent or harness use this?

Yes. It is a standard MCP server. Install it locally and register it in any MCP client — Claude Desktop, Cursor, Agent Zero, and others (E001). It is single-serve: each agent runs its own instance; nothing is shared or multi-tenant (E001).

## Can I build on top of a thinking call?

Yes. Set `NUT_THINKING_HOOKS` to a directory of scripts and the server fires a matching `<mode>.py` (or a generic `on_think.py`) as a background task after each note is stored (E001, E005). Hooks are fire-and-forget and best-effort; a hook failure never fails or slows the `think_*` call (E005). The raw JSONL is already your durable audit trail (E001).

## What tools does the server expose?

18 tools total: `health`, `definitions`, `journal`, and 15 mode tools named `think_<mode>` (E001, E009). The audit confirms 18 tools listed via `FastMCP.list_tools()` and one `think_<id>` tool for each of the 15 `modes.json` entries (E009).

## How do I add a new thinking mode?

Edit `src/simple_thinking_tool/modes.json` and add one entry with `id`, `label`, `purpose`, `when`, `level`, `typical_stage`, `fields`, and `template` (optionally `references`). It auto-registers as `think_<id>` and appears in `definitions`. No code change is needed (E001, E019, E020). The `_comment` field in `modes.json` instructs editors that no code changes are needed (E019).

## What happens when the loop guard blocks a call?

When a call is blocked, the tool responds `{ "ok": false, "blocked": true, "status": "block", "message": "…" }` and nothing is stored (E001, E006, E009). The server code returns `{'ok': False, 'blocked': True, 'status': guard.status, 'message': guard.message, 'consecutive': guard.consecutive, 'last_mode': guard.last_mode}` (E006).

## Is the thinking log private?

Yes, by design. The thinking log is your reasoning notebook. Keep the state directory out of version control, never commit the JSONL or any credentials, and if you publish a fork, publish only public code and docs — never state, thoughts, or secrets (E001). The `.gitignore` excludes `state/`, `*.bak.jsonl`, and env files (E009).

## What is the note size limit?

Notes are capped at `NUT_THINKING_MAX_NOTE_CHARS`, default 800 characters (E001, E003). The store raises `ValueError` if a note exceeds 800 chars (E008), and the server truncates rendered notes to `s.max_note_chars` before storing (E006).

## How is authentication handled?

Authentication is optional and applies to the HTTP transport. Bearer tokens (`NUT_THINKING_TOKEN` for the agent, `NUT_THINKING_BOSS_TOKEN` for a read-only boss) are verified with constant-time comparison (`hmac.compare_digest`). The actor identity is derived from the token, never from tool arguments (E001, E002, E009). The boss token grants read-only, cross-actor journal access (E009).

## Does the server make any outbound network calls?

No. The server makes zero outbound calls. It fills templates and appends to local JSONL (E001, E009). Hooks, if enabled, run as your own local subprocess; the server still makes no outbound calls and sends no secrets (E001, E005).

## What transports are supported?

The default transport is stdio (spawned locally). An optional HTTP transport is available via `NUT_THINKING_TRANSPORT=http` (default port 9100, binding `127.0.0.1` by default) (E001, E006). The server code also accepts `sse` and `streamable-http` as transport values (E006).

## How does storage rotation work?

Rotation is on by default at 5 MB, tunable with `NUT_THINKING_ROTATE_BYTES` or disabled by setting it to `0` (E001). When the file exceeds the threshold, the current file is renamed with a timestamp suffix (`.bak.jsonl`) and a new file is created for subsequent appends (E008).

## Are duplicate writes prevented?

Yes, via `run_id` idempotency. If a note with the same `actor_id` and `run_id` already exists, the append is rejected as a duplicate (E008). The service layer raises `ValueError` on a duplicate `run_id` (E016).

## What could not be verified from the pinned source?

- The README reports "32 passed, 1 skipped" for the test suite (E001), while AUDIT.md reports "43 passed, 1 skipped" (E009). The discrepancy is not reconciled in the pinned source.
- No CI pipeline, changelog, or release tags are present in the repository evidence.
- The `service.py` layer does not integrate the loop guard or hooks; whether this is intentional or an earlier interface is not documented in the source (E007).
- Session lifecycle methods exist in `store.py` (E008) but the server passes `actor_id` directly as `session_id` (E006), so session records are not created in normal operation; this is not explicitly documented.