# BITwiki Atlas — Publication Contract

This file is the canonical write contract for generated repository knowledge.

## Three repository roles

A Foundry job always involves three distinct repositories/surfaces:

1. **Foundry** — `bitwikiorg/foundry`: orchestration and public UI. A repository job never writes generated output here.
2. **Source repository** — the user's submitted public GitHub repository: immutable, read-only evidence input pinned to one commit SHA. Never modify it.
3. **Atlas** — `bitwikiorg/atlas`: the only per-job Git write destination.

Vercel is presentation only. Publishing an Atlas version does **not** deploy Vercel. Foundry reads this repository's root `index.json`, so a successful Atlas commit becomes visible to the site without modifying Foundry.

## Exact destination

For source repository `OWNER/REPO` at source commit `SOURCE_SHA`, generated output belongs only at:

```text
repos/OWNER/REPO/
  latest.json
  versions/SOURCE_SHA/
    README.md
    docs/
    navigation.json
    graph.json
    visuals/
      index.json
      views/
        <visual-id>.json
    scorecard.json
    provenance.jsonl
    llms.txt
    llms-full.txt
    manifest.json
    bundle.json
```

Do not place generated repository output at Atlas root, under `templates/`, under `schemas/`, or in the Foundry repository.

## Identity and immutability

- `SOURCE_SHA` is the Git commit SHA pinned before analysis.
- `versions/SOURCE_SHA/` is immutable once published.
- If the same version already exists, never overwrite it with newly generated content.
- A new source SHA creates a new version directory.
- `latest.json` is a mutable pointer to the currently published source version.

## Templates

Every build inherits `templates/base.json` plus at most one archetype overlay:

`library | service | application | monorepo | protocol | research`

Templates shape output; source evidence remains authoritative.

## Navigation contract

`navigation.json` is the repository-specific documentation topology consumed by the Foundry reader.

It must:

- identify the same `repo_id` and `source_sha` as the version manifest;
- select one generated document as `primary_page`;
- group generated pages into a concise reading hierarchy derived from the repository's actual concepts;
- reference only files that exist in the immutable version directory;
- validate against `schemas/navigation.schema.json`.

Templates provide documentation grammar. Repository analysis determines documentation topology.

## Visual data contract

`graph.json` is the canonical repository graph. It contains the broad deterministic graph and curated semantic graph information for the pinned source version.

**A graph is data before it is a drawing.** Atlas therefore stores multiple bounded visual projections of that canonical graph rather than one monolithic rendered diagram.

`visuals/index.json` catalogs all published visual views. Each view is stored at:

```text
visuals/views/<visual-id>.json
```

Each view must contain typed nodes, typed edges, evidence references, semantic grouping, and layout intent. The canonical visual files must **not** contain model-generated Mermaid, SVG, HTML, CSS, JavaScript, or fixed coordinates.

The Foundry presentation layer renders these visual datasets interactively. Portable Mermaid may be generated deterministically as a fallback/export, but Mermaid is never the source of truth.

Default renderer policy:

- interactive graph renderer: Cytoscape.js;
- directed/hierarchical layout: ELK layered;
- portable diagram fallback: Mermaid;
- Mermaid security level: strict.

The base template requires at least four useful visual views and allows up to ten. Three deterministic views are expected when data exists:

- repository topology;
- dependency surface;
- evidence map.

Additional views should answer distinct repository-specific questions such as architecture, execution flow, state flow, lifecycle, ontology, testing, deployment, or reading path. Cosmetic variants of the same topology do not count as distinct views.

`visuals/index.json` must validate against `schemas/visual-index.schema.json`; each visual view must validate against `schemas/visual.schema.json`.

## Publication transaction

The deterministic publisher—not an AI agent—must:

1. Read the current Atlas `main` ref and tree.
2. Check whether `repos/OWNER/REPO/versions/SOURCE_SHA/manifest.json` already exists.
3. If new, add the complete immutable version directory.
4. Write/update `repos/OWNER/REPO/latest.json`.
5. Upsert the repository record in root `index.json`.
6. Create one Git tree from the current base tree.
7. Create one commit whose parent is the current `main` commit.
8. Advance `refs/heads/main` without force.
9. Fetch the published `manifest.json` from the exact new Atlas commit SHA.
10. Verify source SHA and required-file inventory before Foundry marks the job published or deletes its queue row.

Prefer one Git commit for the version, latest pointer, and root index so `index.json` cannot point at incomplete output.

## Deterministic validation before release

Before Release Adjudicator may approve publication, deterministic code should verify at minimum:

- all required files exist;
- all navigation paths resolve;
- every visual ID is unique;
- every visual edge references declared nodes;
- visual count satisfies the template policy;
- visual/index source SHA matches the version source SHA;
- evidence references use the published evidence namespace;
- manifest file inventory includes navigation and visual artifacts.

Renderer-specific syntax validation belongs to deterministic rendering/export code, not to language-model agents.

## Activity telemetry

Publication should record deterministic activity metrics in `manifest.json` and copy the current totals needed by the public index into `index.json`.

- `version_count` — number of immutable published source-SHA versions for this repository after this publication.
- `words_generated` — word count of the canonical human Atlas corpus: `README.md` plus `docs/**/*.md`. Do not count `llms-full.txt` or other duplicate machine representations.
- `source_files_mapped` — number of source-repository file nodes represented in `graph.json`.
- `graph_nodes` — canonical graph node count.
- `graph_edges` — canonical graph edge count.
- `graph_entities` — `graph_nodes + graph_edges`.
- `visual_count` — number of published bounded visual datasets for this version.
- `agent_runs` — number of AI-agent invocations actually executed for this publication job.
- `generation_seconds` — elapsed worker time from claim through verified publication.
- `model_tokens` — optional sum of provider-reported model tokens. Record only when actual usage metadata exists; never estimate it.

These values describe generation work. **Human impressions are not generated by the publisher.** Foundry measures impressions from pageviews of the canonical public route:

```text
/atlas/OWNER/REPO
```

Vercel Web Analytics is the live source for those pageviews. An `impressions` value in `index.json`, if present, is only a snapshot/fallback and must not be presented as more current than live analytics.

## Credential boundary

AI agents never receive GitHub write credentials.

Source repository reads are public/read-only. The deterministic publisher uses a **separate GitHub credential restricted to `bitwikiorg/atlas`** with the minimum repository permission required for Git writes (`Contents: write`). That credential must not be reused for the intake webhook.

## `latest.json`

Must validate against `schemas/latest.schema.json` and identify:

- `repo_id`
- `source_sha`
- `version_path`
- `manifest_uri`
- `published_at`

## Root `index.json`

This is the canonical public registry consumed by Foundry. Each published repository has exactly one current entry, validated against `schemas/index.schema.json`.

The entry must point at the version that `latest.json` identifies and include durable GitHub docs/download/manifest links plus current score/evidence metadata and deterministic activity metrics when available.

## Version manifest

Each version's `manifest.json` must validate against `schemas/manifest.schema.json` and list the generated artifacts. Material claims in human-facing files should remain traceable through evidence IDs/source URLs represented in the version's provenance artifacts.

## Failure rule

If any Git write or post-write verification fails:

- do not mark the Foundry job published;
- do not delete the queue row;
- retry from the current Atlas head;
- never force-push over concurrent Atlas changes.
