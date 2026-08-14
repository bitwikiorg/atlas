# AGENTS.md — BITwiki Atlas publication rules

Atlas is the durable output repository for **[BITwiki Foundry](https://github.com/bitwikiorg/foundry)**. It is an output substrate, not an orchestration workspace.

## Canonical write target

For a source repository `OWNER/REPO` at pinned commit `SHA`, generated output belongs only here:

```text
repos/<owner>/<repo>/versions/<sha>/
```

The owner and repository directory names are the normalized GitHub owner/repository names used by Foundry.

After the immutable version is complete and validated:

```text
repos/<owner>/<repo>/latest.json
index.json
```

must be advanced to reference that published version.

**Never write generated repository output at root, under `templates/`, under `schemas/`, or to a new ad-hoc top-level directory.**

## Required publication sequence

1. Resolve `templates/base.json` + the applicable archetype overlay.
2. Write the complete immutable version to `repos/<owner>/<repo>/versions/<sha>/`.
3. Validate `manifest.json` against `schemas/manifest.schema.json`.
4. Update `repos/<owner>/<repo>/latest.json` according to `schemas/latest.schema.json`.
5. Upsert exactly one repository record in root `index.json` according to `schemas/index.schema.json`.
6. Prefer steps 2–5 in one Git commit so the public index cannot point at incomplete output.

See `PUBLISHING.md` for the full contract.

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
10. Foundry is the control plane; Atlas is the published memory. Do not duplicate Foundry operational state here.
