# AGENTS.md — BITwiki Atlas

Atlas is an output substrate, not an orchestration workspace.

## Before writing

Read `PUBLISHING.md`. It is the canonical publication contract.

A Foundry job has three separate repositories:

- `bitwikiorg/foundry` — control plane; do not place generated repository docs there.
- the submitted source repository — read-only evidence; never modify it.
- `bitwikiorg/atlas` — the only destination for generated repository knowledge.

## Exact generated location

For source `OWNER/REPO` pinned at `SOURCE_SHA`, write only under:

```text
repos/OWNER/REPO/versions/SOURCE_SHA/
```

Then update:

```text
repos/OWNER/REPO/latest.json
index.json
```

Never dump generated output at repository root.

## Rules

1. Do not add TODO, PLAN, workflow, queue, prompt, credential, or internal Foundry orchestration files here.
2. Generated repository facts must trace to the pinned source commit.
3. Never overwrite an existing `versions/SOURCE_SHA/` directory with different content.
4. Update `latest.json` and root `index.json` only as part of a validated publication.
5. Keep artifacts portable: Markdown/JSON/JSONL/text first; rendered sites are secondary.
6. Templates guide presentation only and never override evidence.
7. External research must be labeled separately from repository-derived facts.
8. AI agents never receive Atlas Git write credentials. The deterministic publisher owns the final Git transaction.
9. A per-repository publication never writes Foundry and never deploys Vercel; the Foundry site reads Atlas `index.json`.
