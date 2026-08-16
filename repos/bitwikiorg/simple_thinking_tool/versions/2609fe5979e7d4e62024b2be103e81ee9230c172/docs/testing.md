# Testing

This document describes the automated test suite for `simple_thinking_tool`. It is based on the repository at commit `2609fe5979e7d4e62024b2be103e81ee9230c172` (https://github.com/bitwikiorg/simple_thinking_tool).

## Overview

The repository ships a pytest suite under `tests/` with seven test files covering every source module in `src/simple_thinking_tool/`. The suite is configured in `pyproject.toml` (E011):

```toml
[tool.pytest.ini_options]
pythonpath = ["src"]
testpaths = ["tests"]
```

- `pythonpath = ["src"]` lets tests import the package without installation.
- `testpaths = ["tests"]` scopes the default pytest run to the `tests/` directory.

The only declared runtime dependency is `fastmcp>=3.2,<4` (E011). The server tests guard against its absence with `pytest.importorskip("fastmcp")` (E015), so the rest of the suite can still run in environments where FastMCP is not installed.

## Running the tests

From the repository root:

```bash
pytest
```

Because of the `pyproject.toml` configuration, this discovers and runs all tests under `tests/` with `src/` on the import path. No additional flags are required for a standard run.

> **Note:** The pinned evidence does not include a CI configuration file, and the repository tree contains no CI workflow files (see the context pack summary). Tests are expected to be run manually via `pytest`. This could not be verified further from the pinned source.

## Test file inventory

The following table lists each test file, the module it exercises, and the behaviors it verifies. All paths are relative to the repository root at the pinned SHA.

| Test file | Module under test | What it covers |
| --- | --- | --- |
| `tests/test_auth.py` (E012) | `src/simple_thinking_tool/auth.py` | Token verification (`verify_token`) returning `agent`, `boss`, or `None`; bearer-token extraction (`extract_bearer`) including case-insensitivity and rejection of non-Bearer schemes. |
| `tests/test_guard.py` (E013) | `src/simple_thinking_tool/guard.py` | Loop-guard decision logic: disabled guard allows everything, same-mode-in-a-row hard block, different-mode allowance, no-prior-notes allowance, warn threshold, hard ceiling block, per-actor isolation, `same_mode_block` disable, `block_at` floor, and `GuardResult` defaults. Uses a `FakeStore` stand-in. |
| `tests/test_hooks.py` (E014) | `src/simple_thinking_tool/hooks.py` | Hook runner behavior: disabled no-op, mode-specific script resolution, precedence of `<mode>.py` over `on_think.py`, no-fire when no script exists, payload writing on fire, and isolation of failing hooks. |
| `tests/test_modes.py` (E021) | `src/simple_thinking_tool/modes.py` | Mode registry size (15 modes), presence of core modes, template rendering, missing-field empty fill, unknown-mode `None`, level-sorted ordering, definitions surface, and mode summary. |
| `tests/test_server.py` (E015) | `src/simple_thinking_tool/server.py` | Server build surface: presence of `health`, `definitions`, `journal`; absence of `session_create` and a flat `think` dispatcher; one tool per mode; health tool reporting. Skips if `fastmcp` is unavailable. |
| `tests/test_service.py` (E016) | `src/simple_thinking_tool/service.py` | `ThinkingService` operations: append with `goal_distill`, unknown-mode `ValueError`, note truncation to 800 chars, definitions surface, journal actor filtering, health, and duplicate `run_id` rejection. |
| `tests/test_store.py` (E017) | `src/simple_thinking_tool/store.py` | `JSONLStore` behavior: session CRUD, note append/read, 800-char note limit, `run_id` idempotency, `last_next_action`, file rotation to `.bak.jsonl`, export filters (`after_ts`, `actor_id`), and reload consistency. |

## Test counts

Counting the `test_*` functions defined in the pinned source files yields the following per-file totals:

| File | Test functions |
| --- | --- |
| `tests/test_auth.py` | 4 |
| `tests/test_guard.py` | 11 |
| `tests/test_hooks.py` | 6 |
| `tests/test_modes.py` | 8 |
| `tests/test_server.py` | 3 |
| `tests/test_service.py` | 7 |
| `tests/test_store.py` | 7 |
| **Total** | **46** |

These counts reflect the number of test functions visible in the source at the pinned SHA. The README reports "32 passed, 1 skipped" (E001) while AUDIT.md reports "43 passed, 1 skipped" (E009); neither count matches the 46 test functions visible in the pinned source, suggesting both counts are stale. AUDIT.md also miscounts the test files as 6 when the pinned source contains 7 (E009). The exact runtime pass/skip totals should be confirmed by running the suite.

## What the suite covers

- **Identity and security** — constant-time token comparison and actor derivation, plus bearer-header parsing (E012).
- **Anti-churn guard** — the full decision tree of the optional loop guard, including per-actor isolation and threshold behavior, tested against a fake store (E013).
- **Background hooks** — script resolution precedence, payload delivery, and failure isolation (E014).
- **Mode catalog** — loading, rendering, missing-field tolerance, and the definitions surface (E021).
- **Server wiring** — the lean tool surface and the one-tool-per-mode invariant (E015).
- **Service layer** — append, truncation, journal filtering, and idempotency (E016).
- **Persistence** — append, load, rotation, idempotency, export filters, and stats (E017).

## Known gaps and unverified claims

The following could not be verified from the pinned repository evidence and are noted for accuracy:

- **No CI configuration** — the repository tree contains no CI workflow files, so there is no automated test runner in the pinned source.
- **No HTTP transport integration tests** — the server tests (E015) verify tool registration and surface but do not exercise the HTTP transport or token-authenticated request path end to end.
- **Exact pass/skip totals** — the runtime pass/skip counts reported in `README.md` (32 passed, 1 skipped, E001) and `AUDIT.md` (43 passed, 1 skipped, E009) are not reconciled in the pinned source, and neither matches the 46 test functions visible in the source. AUDIT.md also miscounts the test files as 6 when there are 7 (E009). Run the suite to confirm the current totals.
- **`modes.json` field-name validation** — no test verifies that `modes.json` field names are safe Python identifiers, which is relevant to the `exec()`-based tool generation in `server.py` (E006).