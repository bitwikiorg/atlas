# BITwiki Atlas

**Atlas is the durable repository knowledge layer produced by [BITwiki Foundry](https://github.com/bitwikiorg/foundry).**

Foundry analyzes a submitted public repository at a pinned Git commit. Atlas stores the generated, source-linked result: documentation, graphs, ratings, provenance, machine-readable surfaces, and version history.

```text
Foundry                           Atlas
analysis + orchestration   →     durable published knowledge
```

## Three-repository boundary

A Foundry job has three distinct repository roles:

- **Foundry** — `bitwikiorg/foundry`: control plane and public site; generated repository output does not belong there.
- **Source repository** — the user's submitted public GitHub repository: pinned and read only; never modified by Foundry.
- **Atlas** — `bitwikiorg/atlas`: the only Git destination for generated repository knowledge.

Vercel is presentation only. Once Atlas `index.json` advances, the Foundry site can display the repository without a per-job Vercel deployment.

## Repository boundary

This repository is intentionally lightweight. It contains:

- the shared Atlas output contract
- reusable documentation templates
- thin project-archetype overlays
- schemas for publication/index records
- the public repository index
- generated repository outputs under `repos/`
- minimal continuity state

Foundry operational state, queues, prompts, retries, and TODOs do not belong here.

## Canonical generated destination

**Every generated repository MUST be written below this exact path:**

```text
repos/<github-owner>/<github-repository>/
  latest.json
  versions/<source-sha>/
    README.md
    docs/
    graph.json
    scorecard.json
    provenance.jsonl
    llms.txt
    llms-full.txt
    manifest.json
    bundle.json
```

Do not place generated repository documents at repository root, inside `templates/`, or in ad-hoc directories.

Each `versions/<source-sha>/` directory is immutable. `latest.json` points to the currently published version.

See **[PUBLISHING.md](./PUBLISHING.md)** for the exact write transaction, credential boundary, and index contract.

## Template inheritance

Every generated repository starts from `templates/base.json`, then applies one archetype overlay when applicable:

- `library`
- `service`
- `application`
- `monorepo`
- `protocol`
- `research`

Overlays change emphasis and section ordering. They do not override source truth or force irrelevant sections.

## Public index

`index.json` is the canonical registry consumed by the Foundry public site.

A repository becomes visible in Foundry only after its complete version exists and its entry is committed to `index.json`.

## Rule

Repository source evidence is authoritative. Generated prose, diagrams, ratings, and inferred relationships must preserve provenance and uncertainty.

**Control plane:** [github.com/bitwikiorg/foundry](https://github.com/bitwikiorg/foundry)
