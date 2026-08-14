# STATE.md — BITwiki Atlas

status: READY
schema_version: 1
template_version: 1
index_version: 1
generated_repositories: 0
last_updated: 2026-08-14

## Current truth

- `bitwikiorg/atlas` is the durable generated-output repository for BITwiki Foundry.
- `index.json` is the public repository registry.
- `templates/base.json` is inherited by every generated repository.
- Exactly one archetype overlay is selected when applicable.
- Generated versions are immutable by source SHA; `latest.json` advances to the current published version.
- Foundry retains planning, queue, orchestration, retry, and operational state.
