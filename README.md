# BITwiki Atlas

**Atlas is the durable repository knowledge layer produced by BITwiki Foundry.**

Foundry analyzes a public repository. Atlas stores the generated, source-linked result: documentation, graphs, ratings, provenance, machine-readable surfaces, and version history.

## Repository role

This repository is intentionally lightweight. It contains:

- the shared Atlas output contract
- reusable documentation templates
- thin project-archetype overlays
- the public repository index
- generated repository outputs under `repos/`
- a minimal continuity state

Foundry owns orchestration, queueing, agents, retries, operational state, and TODOs. Those do not belong here.

## Generated layout

```text
repos/<owner>/<repository>/
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
```

Each generated version is tied to an immutable source commit. `latest.json` points to the currently published version.

## Template inheritance

Every generated repository starts from `templates/base.json`, then applies one archetype overlay:

- `library`
- `service`
- `application`
- `monorepo`
- `protocol`
- `research`

Overlays change emphasis and section ordering. They do not override source truth or force irrelevant sections.

## Public index

`index.json` is the canonical lightweight registry consumed by Foundry's public repository index.

## Rule

Repository source evidence is authoritative. Generated prose, diagrams, ratings, and inferred relationships must preserve provenance and uncertainty.
