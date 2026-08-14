# AGENTS.md — Atlas publication rules

This repository is an output substrate, not an orchestration workspace.

## Rules

1. Do not add TODO, PLAN, workflow, queue, prompt, credential, or internal orchestration files here.
2. Generated repository facts must trace to the pinned source commit.
3. Never overwrite a published `versions/<sha>/` directory with different content.
4. Update `latest.json` only after the complete version passes publication checks.
5. Update `index.json` only after the generated version is published successfully.
6. Keep generated artifacts portable: Markdown/JSON/JSONL/text first; rendered sites are secondary.
7. Archetype templates guide presentation only. They never force unsupported claims or irrelevant sections.
8. External research must be labeled separately from repository-derived facts.
9. Keep root files and templates small. Generated repository content belongs under `repos/`.
