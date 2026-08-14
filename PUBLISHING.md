# BITwiki Atlas — Publishing Contract

This file defines how BITwiki Foundry publishes generated repository knowledge into Atlas.

## Identity

For a source repository:

```text
https://github.com/<owner>/<repo>
```

Foundry derives:

```text
repo_id = <owner>/<repo>   # normalized lowercase operational ID
source_sha = pinned full Git commit SHA
```

The human-facing directory retains the normalized GitHub owner/repository path supplied by the publisher.

## Destination

A generated version MUST be written to:

```text
repos/<owner>/<repo>/versions/<source_sha>/
```

Required portable outputs come from `templates/base.json` plus the selected archetype overlay.

The minimum version directory is:

```text
README.md
docs/
graph.json
scorecard.json
provenance.jsonl
llms.txt
llms-full.txt
manifest.json
```

`docs/` contains the applicable generated Markdown sections. Unsupported sections are omitted rather than fabricated.

## Write order

Publication is complete only after all four steps succeed:

```text
1. immutable version directory
2. repo latest.json pointer
3. root index.json upsert
4. Git commit/push verification
```

Prefer one Git commit containing steps 1–3.

### 1. Version

Write:

```text
repos/<owner>/<repo>/versions/<source_sha>/...
```

A published SHA directory is immutable. If the same SHA already exists with a matching manifest/content hashes, publication is idempotently complete. If content differs, stop rather than overwrite it.

### 2. Latest pointer

Write:

```text
repos/<owner>/<repo>/latest.json
```

The record MUST validate against `schemas/latest.schema.json` and point to the published version directory.

### 3. Public registry

Upsert exactly one entry for `repo_id` in root:

```text
index.json
```

The complete file MUST validate against `schemas/index.schema.json`.

Foundry renders its public repository index from this file. If the repository is absent from `index.json`, it is not publicly indexed by Foundry even if version files exist.

### 4. Verification

After the Git commit/push:

- fetch `github_docs_url` or `manifest_uri`
- verify it resolves
- only then mark the Foundry job published and delete the queue row

## Public index fields

The publisher should populate, when available:

```text
repo_id
owner
repo
repo_url
source_sha
last_indexed_at
template_id
language
foundry_score
score_confidence
evidence_grade
doc_coverage
github_stars
github_forks
github_docs_url
download_url
atlas_url
manifest_uri
```

`atlas_url` is optional rendered presentation. `github_docs_url` and the underlying repository files are the durable output.

## Template resolution

```text
templates/base.json
  +
templates/<archetype>.json
```

Supported overlays:

`library | service | application | monorepo | protocol | research`

If classification is uncertain, use `base` only.

Templates control document shape and emphasis. Repository evidence controls truth.

## Ownership boundary

**Foundry:** intake, analysis, RepoGraph, agents, queue, retries, validation, publication orchestration.

**Atlas:** templates, schemas, public index, generated immutable versions, current pointers.

Do not place Foundry operational files in Atlas. Do not place generated Atlas versions in Foundry.
