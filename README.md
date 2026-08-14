# BITwiki Atlas

**Atlas is the durable repository knowledge layer produced by [BITwiki Foundry](https://github.com/bitwikiorg/foundry).**

Foundry analyzes a submitted public repository at a pinned Git commit. Atlas stores the generated, source-linked result: documentation, graphs, ratings, provenance, machine-readable surfaces, and version history.

```text
Foundry                           Atlas
analysis + orchestration   →     durable published knowledge
```

## Repository boundary

This repository is intentionally lightweight. It contains:

- the shared Atlas output contract
- reusable documentation templates
- thin project-archetype overlays
- schemas for publication/index records
- the public repository index
- generated repository outputs under `repos/`
- minimal continuity state

Foundry owns orchestration, queueing, agents, retries, operational state, public presentation, and TODOs. Those do not belong here.

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

Example:

```text
repos/bitwikiorg/foundry/versions/abc123.../
```

Do not place generated repository documents at repository root, inside `templates/`, or in ad-hoc directories.

Each `versions/<source-sha>/` directory is immutable. `latest.json` points to the currently published version.

See **[PUBLISHING.md](./PUBLISHING.md)** for the exact write order, telemetry definitions, and index contract.

## Template inheritance

Every generated repository starts from `templates/base.json`, then applies one archetype overlay when applicable:

- `library`
- `service`
- `application`
- `monorepo`
- `protocol`
- `research`

Overlays change emphasis and section ordering. They do not override source truth or force irrelevant sections.

## Public index and Foundry pages

`index.json` is the canonical registry consumed by the Foundry public site.

A repository becomes visible in Foundry only after its complete version exists and its entry is committed to `index.json`.

Foundry exposes a canonical public page for each published repository at:

```text
/atlas/<owner>/<repo>
```

Those pages read this repository as their durable source. Human page impressions belong to Foundry/Vercel analytics; generation telemetry belongs in Atlas manifests/index entries.

## Rule

Repository source evidence is authoritative. Generated prose, diagrams, ratings, and inferred relationships must preserve provenance and uncertainty.

**Control plane:** [github.com/bitwikiorg/foundry](https://github.com/bitwikiorg/foundry)
