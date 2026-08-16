# Reference — simple_thinking_tool

This is the consolidated reference for the `simple_thinking_tool` repository at pinned source SHA `2609fe5979e7d4e62024b2be103e81ee9230c172` (commit date 2026-08-05). It covers the tool catalog, the thinking-mode definitions, data schemas, environment variables, the repository file map, and the test suite.

All claims below are grounded in the pinned repository evidence. Where a detail could not be verified from the pinned evidence, it is explicitly marked as unverified.

---

## 1. Project identity

| Field | Value | Evidence |
|---|---|---|
| Package name | `simple-thinking-tool` (import `simple_thinking_tool`) | [E011] |
| Version | `0.1.0` | [E011], [E018] |
| Python requirement | `>=3.11` | [E011] |
| License | PolyForm-Noncommercial-1.0.0 | [E011], [E010] |
| Runtime dependency | `fastmcp>=3.2,<4` | [E011], [E022] |
| Description | "Private per-agent reasoning ledger with a forced JIT digestion router. The server never thinks; it assembles templates and stores notes." | [E011] |

The project is a Model Context Protocol (MCP) server that exposes a catalog of named thinking methods as individual tools. The core design principle is "the LLM is the router": the calling agent reads each tool's description and selects the appropriate reasoning mode. The server itself never calls an LLM, makes no outbound network calls, and performs no reasoning — it fills a mode-specific template from caller-provided string fields and appends the rendered note to a durable, append-only JSONL log with `fsync` [E006], [E008].

---

## 2. Tool catalog

The server registers 18 tools: 3 fixed tools (`health`, `definitions`, `journal`) plus one `think_<mode>` tool per mode defined in `modes.json` [E006], [E019].

### 2.1 Fixed tools

| Tool | Signature | Purpose | Evidence |
|---|---|---|---|
| `health` | `health() -> dict` | Reports server health: `status`, `name`, `transport`, `state_dir`, `tools` (count = number of modes + 3). | [E006] |
| `definitions` | `definitions() -> list[dict]` | Returns the full catalog of every thinking mode: purpose, when-to-use, fields, and template. Intended to be called first when the LLM is unsure which tool to use. | [E006], [E020] |
| `journal` | `journal(after_ts: int \| None = None, actor_id: str \| None = None) -> dict` | Reads back stored thoughts. Optional `after_ts` (ms epoch) filters to new entries; optional `actor_id` filter is boss-only. Returns `{"records": [...], "stats": {...}}`. | [E006] |

### 2.2 Mode tools (`think_<mode>`)

Each mode in `modes.json` auto-registers as a tool named `think_<id>` with its `fields` as explicit required string arguments. The tool description is built from the mode's `purpose`, `when`, and `fields` [E006], [E019].

Success response shape (from the generated tool function):

```json
{"ok": true, "mode": "<mode_id>", "note": "<rendered note>", "id": "<record id>", "hook": "<script path, only if a hook fired>"}
```

Blocked response shape (when the loop guard rejects the call):

```json
{"ok": false, "blocked": true, "status": "block", "message": "<guard message>", "consecutive": <int>, "last_mode": "<mode or null>"}
```

[E006], [E004]

---

## 3. Thinking-mode definitions

Modes are defined as data in `src/simple_thinking_tool/modes.json`. Each entry becomes a tool named `think_<id>`. Editing this file adds or changes modes with no code change required [E019], [E020].

The `Mode` dataclass (frozen) fields: `id`, `label`, `purpose`, `when`, `level` (1–4), `typical_stage`, `fields` (tuple of str), `template`, `version` (default 1) [E020].

The 15 modes, with level, typical stage, fields, and template (newlines shown as `\n`):

| id | label | level | stage | fields | template | Evidence |
|---|---|---|---|---|---|---|
| `goal_distill` | Goal Distillation | 1 | plan | priority, why, action | `I am thinking about the highest-priority goal.\nCore priority: {priority}\nWhy this matters: {why}\nFirst concrete action: {action}` | [E019] |
| `understand` | Understand / Digest | 1 | understand | summary, risk, unknown | `What I now understand: {summary}\nKey constraint/risk: {risk}\nWhat remains unclear: {unknown}` | [E019] |
| `plan` | Plan | 1 | plan | objective, steps, criteria, blockers | `Objective: {objective}\nSteps: {steps}\nSuccess criteria: {criteria}\nBlockers: {blockers}` | [E019] |
| `creative` | Creative / Divergent | 2 | any | brief, directions, wildcard | `Creative brief: {brief}\nOpen directions: {directions}\nWildcard idea: {wildcard}` | [E019] |
| `deep` | Deep Think | 2 | execute | rephrase, parts, hypothesis, verify, synthesis | `Rephrase: {rephrase}\nDecompose: {parts}\nHypothesis: {hypothesis}\nVerify approach: {verify}\nSynthesis: {synthesis}` | [E019] |
| `tree` | Tree of Thoughts | 3 | any | branches, scores, pruned, chosen | `Branches: {branches}\nScores: {scores}\nPruned: {pruned}\nChosen branch: {chosen}` | [E019] |
| `graph` | Graph of Thoughts | 3 | any | nodes, edges, loop, answer | `Nodes/partials: {nodes}\nEdges/links: {edges}\nFeedback loop: {loop}\nAggregated answer: {answer}` | [E019] |
| `bias_check` | Bias Check | 3 | review | confirmation, survivorship, authority, counter | `Confirmation bias check: {confirmation}\nSurvivorship check: {survivorship}\nAuthority check: {authority}\nCounter-evidence: {counter}` | [E019] |
| `distill` | Distill / Compress | 2 | handoff | essence, durable, drop | `Essence: {essence}\nSurvives rephrasing: {durable}\nDrop: {drop}` | [E019] |
| `reinforced` | Reinforced / Red Team | 4 | review | takes, attack, conclusion | `Takes: {takes}\nRed team: {attack}\nRobust conclusion: {conclusion}` | [E019] |
| `reflect` | Reflect / Socratic | 2 | review | question, evidence, assumptions, truth | `Question: {question}\nEvidence: {evidence}\nAssumptions: {assumptions}\nTruth so far: {truth}` | [E019] |
| `chain_of_thought` | Chain of Thought | 1 | execute | premise, steps, conclusion | `Premise: {premise}\nIntermediate steps: {steps}\nConclusion: {conclusion}` | [E019] |
| `first_principles` | First Principles | 2 | any | fundamentals, rebuilt, difference | `Fundamental facts: {fundamentals}\nRebuilt reasoning: {rebuilt}\nWhat changed vs the inherited view: {difference}` | [E019] |
| `metacognitive` | Metacognitive / Self-Audit | 3 | review | bias, uncertainty, contradiction, correction | `Bias found: {bias}\nUncertainty: {uncertainty}\nContradiction: {contradiction}\nCorrection: {correction}` | [E019] |
| `adversarial_review` | Adversarial Review | 4 | review | claim, attack, survives | `Provisional claim: {claim}\nAttack: {attack}\nDoes it survive: {survives}` | [E019] |

Template rendering behavior: missing fields are filled with empty strings rather than raising errors; only the keys referenced by the template are substituted [E020]. The test suite confirms 15 modes are registered and that `all_modes()` sorts by `(level, id)` [E021].

---

## 4. Data schemas

### 4.1 Note record (JSONL line, `type: "note"`)

Written by `JSONLStore.append_note` [E008]:

| Field | Type | Notes |
|---|---|---|
| `type` | str | `"note"` |
| `id` | str | ULID-like: 12-hex-char ms timestamp + 16-hex-char uuid suffix |
| `session_id` | str | Server passes `actor_id` as the flat grouping key |
| `actor_id` | str | Actor identity |
| `parent_thought_id` | str \| None | Optional |
| `mode` | str | Mode id |
| `template_version` | int | Mode version at write time |
| `note` | str | Rendered note, capped at 800 chars (enforced in `append_note`) |
| `next_action` | str \| None | Optional |
| `run_id` | str \| None | Idempotency key; duplicate `(actor_id, run_id)` rejected |
| `input_hash` | str | `sha256(...)[:16]` over `{session_id, mode, note, next_action}` |
| `ts` | int | ms epoch |
| `meta` | dict | Optional; server sets `{mode, level, stage}` |

### 4.2 Session record (JSONL line, `type: "session"`)

Created by `JSONLStore.create_session` [E008]:

| Field | Type | Notes |
|---|---|---|
| `type` | str | `"session"` |
| `id` | str | ULID-like |
| `actor_id` | str | |
| `project_id` | str \| None | |
| `conversation_id` | str \| None | |
| `status` | str | `"open"` or `"finalized"` |
| `created_at` | int | ms epoch |
| `finalized_at` | int \| None | |

Note: the server passes `actor_id` directly as `session_id` and does not create session records in normal operation; the session lifecycle methods exist in the store but are not invoked by the server path [E006], [E008].

### 4.3 GuardResult

Dataclass returned by `Guard.evaluate` [E004]:

| Field | Type | Notes |
|---|---|---|
| `allowed` | bool | Default `True` |
| `status` | str | `"ok"` \| `"warn"` \| `"block"` |
| `consecutive` | int | Default 1 |
| `last_mode` | str \| None | |
| `message` | str | |

### 4.4 Hook payload

JSON file written to the temp directory and passed as a file-path argument to the hook script subprocess [E005]:

```json
{"mode": "<mode_id>", "note": "<note text>", "actor": "<actor_id>", "ts": <ms epoch>}
```

The payload contains only `{mode, note, actor, ts}` — no tokens, no secrets, no config [E005].

### 4.5 Settings

Environment-driven `Settings` class; all values come from environment variables, no hardcoded secrets [E003]. See the environment-variable table in section 5.

---

## 5. Environment variables

All configuration is via `NUT_THINKING_*` environment variables, read by `config.Settings` [E003].

| Variable | Default | Description | Evidence |
|---|---|---|---|
| `NUT_THINKING_TOKEN` | `""` | Agent bearer token; grants `"agent"` actor | [E003], [E002] |
| `NUT_THINKING_BOSS_TOKEN` | `""` | Boss bearer token; grants `"boss"` actor (cross-actor journal access) | [E003], [E002] |
| `NUT_THINKING_STATE_DIR` | `"state"` | Directory for the JSONL store | [E003] |
| `NUT_THINKING_HOST` | `"127.0.0.1"` | HTTP bind host | [E003] |
| `NUT_THINKING_PORT` | `9100` | HTTP bind port | [E003] |
| `NUT_THINKING_MAX_NOTE_CHARS` | `800` | Cap on rendered note length | [E003] |
| `NUT_THINKING_BURST_LIMIT` | `3` | Burst limit (defined in config; usage in server path not verified) | [E003] |
| `NUT_THINKING_TRANSPORT` | `"stdio"` | Transport: `stdio`, `http`, `sse`, `streamable-http` | [E003], [E006] |
| `NUT_THINKING_ACTOR` | `"agent"` | Fallback actor when no HTTP request context | [E003] |
| `NUT_THINKING_ALLOW_MODES` | `"goal_distill,understand"` | Comma-separated allowed modes | [E003] |
| `NUT_THINKING_ROTATE_BYTES` | `5242880` (5 MB) | JSONL rotation threshold | [E003], [E008] |
| `NUT_THINKING_HOOKS` | `""` | Hooks directory; empty = hooks disabled | [E003], [E005] |
| `NUT_THINKING_HOOK_TIMEOUT` | `"10.0"` | Hook subprocess timeout (seconds) | [E003], [E005] |
| `NUT_THINKING_GUARD_ENABLED` | `"0"` | `"1"` enables the loop guard | [E003], [E004] |
| `NUT_THINKING_GUARD_SAME_MODE_BLOCK` | `"1"` | `"1"` hard-blocks same-mode-in-a-row | [E003], [E004] |
| `NUT_THINKING_GUARD_WARN_AT` | `"3"` | Consecutive-call warning threshold | [E003], [E004] |
| `NUT_THINKING_GUARD_BLOCK_AT` | `"5"` | Consecutive-call hard ceiling | [E003], [E004] |

---

## 6. Loop guard behavior

The guard is off by default (`NUT_THINKING_GUARD_ENABLED=1` enables it). When enabled, `Guard.evaluate(store, actor_id, mode)` [E004]:

1. Collects trailing `note` records for the actor from `store.export()`.
2. Counts consecutive trailing notes that have a mode set, including the current call in the streak.
3. Hard-blocks if the last mode equals the current mode and `same_mode_block` is true (pure churn).
4. Hard-blocks if the consecutive count reaches `block_at` (default 5).
5. Otherwise warns (but allows) if the consecutive count reaches `warn_at` (default 3).

The guard never detects real work itself; it surfaces the LLM its own call state and hard-stops only as a backstop [E004].

---

## 7. Hooks system

`HookRunner` dispatches background hooks after a note is persisted [E005]:

- Enabled only when `NUT_THINKING_HOOKS` points to an existing directory.
- Resolution: a mode-specific script `<mode>.py` takes precedence over a generic `on_think.py`.
- Hooks run as fire-and-forget daemon-thread subprocesses with a timeout; a hook failure never fails or slows the `think_*` call.
- The hook receives a JSON payload file path as its single argument (schema in section 4.4).

---

## 8. Repository file map

| Path | Responsibility | Evidence |
|---|---|---|
| `src/simple_thinking_tool/__init__.py` | Package identity, `__version__ = "0.1.0"` | [E018] |
| `src/simple_thinking_tool/auth.py` | Token-derived actor identity (`verify_token`, `extract_bearer`); constant-time comparison | [E002] |
| `src/simple_thinking_tool/config.py` | Environment-driven `Settings` | [E003] |
| `src/simple_thinking_tool/guard.py` | Optional loop guard (`Guard`, `GuardResult`) | [E004] |
| `src/simple_thinking_tool/hooks.py` | Background event hooks (`HookRunner`) | [E005] |
| `src/simple_thinking_tool/modes.py` | Mode loader and template renderer (`Mode`, `load_modes`, `render_template`, `definitions`) | [E020] |
| `src/simple_thinking_tool/modes.json` | 15 thinking-mode definitions as data | [E019] |
| `src/simple_thinking_tool/server.py` | FastMCP server wiring (`build_server`, `main`); auto-registers tools | [E006] |
| `src/simple_thinking_tool/service.py` | Service layer (`ThinkingService`); alternative API without guard/hooks integration | [E007] |
| `src/simple_thinking_tool/store.py` | Append-only JSONL store (`JSONLStore`) with fsync, rotation, idempotency | [E008] |
| `tests/` | 7 test files (see section 9) | [E012]–[E017], [E021] |
| `README.md` | Project documentation | [E001] |
| `AUDIT.md` | Audit report (scope, change log, verified evidence, security review, open items) | [E009] |
| `LICENSE` | PolyForm Noncommercial 1.0.0 | [E010] |
| `pyproject.toml` | Build config, metadata, dependency, pytest config | [E011] |
| `requirements.txt` | Pinned dependency (`fastmcp`) | [E022] |
| `assets/` | Static assets (e.g., hero banner SVG) | context pack |

---

## 9. Test suite

Seven test files cover the source modules [E012]–[E017], [E021]:

| File | Coverage | Evidence |
|---|---|---|
| `tests/test_auth.py` | Token verification and bearer extraction | [E012] |
| `tests/test_guard.py` | Guard decision logic: disabled, same-mode block, warn, block ceiling | [E013] |
| `tests/test_hooks.py` | Hook resolution, payload writing, fire-and-forget dispatch | [E014] |
| `tests/test_modes.py` | Mode loading, template rendering, definitions surface, level sorting; asserts 15 modes | [E021] |
| `tests/test_server.py` | Server build, tool registration, health/definitions/journal tools | [E015] |
| `tests/test_service.py` | `ThinkingService` append, journal, definitions, health | [E016] |
| `tests/test_store.py` | JSONL store: append, load, rotate, idempotency, export, stats | [E017] |

Pytest is configured via `pyproject.toml` with `pythonpath = ["src"]` and `testpaths = ["tests"]` [E011]. The README reports "32 passed, 1 skipped" [E001] while AUDIT.md reports "43 passed, 1 skipped" [E009]; neither count matches the 46 test functions visible in the pinned source [E012]–[E017], [E021], suggesting both counts are stale. AUDIT.md also miscounts the test files as 6 when the pinned source contains 7 [E009].

---

## 10. Security, privacy, and known caveats

- **No-LLM invariant**: the server code contains no model calls, no API calls to any LLM, and no outbound network requests; it only fills templates and appends notes locally [E006], [E011].
- **Auth**: actor identity is derived from the authenticated bearer token via constant-time comparison (`hmac.compare_digest`), never from tool input arguments — a spoofing blocker [E002].
- **Durability**: every write is flushed and `fsync`'d; partial tail lines are tolerated on load; rotation archives the log when it exceeds `NUT_THINKING_ROTATE_BYTES` [E008].
- **Idempotency**: a duplicate `(actor_id, run_id)` append is rejected [E008].
- **Dynamic tool generation**: `server.py` generates mode tool functions via `exec()` from fixed `modes.json` field names (marked `noqa S102`). This is safe only while `modes.json` is operator-controlled; if the file could be externally modified, it would require auditing [E006].
- **Test count discrepancy**: the README reports "32 passed, 1 skipped" [E001] and AUDIT.md reports "43 passed, 1 skipped" [E009]; neither matches the 46 test functions visible in the pinned source [E012]–[E017], [E021], suggesting both counts are stale. AUDIT.md also miscounts the test files as 6 when there are 7 [E009]. These counts should be reconciled against an actual test run.
- **Session lifecycle methods** (`create_session`, `get_session`, `list_sessions`, `finalize_session`) exist in `store.py` but are not used by the server path, which treats `actor_id` as a flat grouping key [E008], [E006].

---

## 11. Source references

All evidence is from the pinned commit `2609fe5979e7d4e62024b2be103e81ee9230c172`:

- [E001] README.md
- [E002] src/simple_thinking_tool/auth.py
- [E003] src/simple_thinking_tool/config.py
- [E004] src/simple_thinking_tool/guard.py
- [E005] src/simple_thinking_tool/hooks.py
- [E006] src/simple_thinking_tool/server.py
- [E007] src/simple_thinking_tool/service.py
- [E008] src/simple_thinking_tool/store.py
- [E009] AUDIT.md
- [E010] LICENSE
- [E011] pyproject.toml
- [E012] tests/test_auth.py
- [E013] tests/test_guard.py
- [E014] tests/test_hooks.py
- [E015] tests/test_server.py
- [E016] tests/test_service.py
- [E017] tests/test_store.py
- [E018] src/simple_thinking_tool/__init__.py
- [E019] src/simple_thinking_tool/modes.json
- [E020] src/simple_thinking_tool/modes.py
- [E021] tests/test_modes.py
- [E022] requirements.txt

Repository URL: https://github.com/bitwikiorg/simple_thinking_tool