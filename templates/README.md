# Atlas templates

Templates describe **shape, priority, and required portable outputs**. They do not contain generated repository facts.

## Resolution

1. Load `base.json`.
2. Select one archetype overlay from deterministic repository evidence.
3. Merge overlay `section_order`, `emphasis`, and `optional_sections` onto base.
4. Omit sections that are not supported by evidence.
5. Generate portable Markdown/JSON artifacts before any rendered site.

## Archetypes

- `library.json` — packages/SDKs/libraries
- `service.json` — APIs, workers, daemons, backend services
- `application.json` — end-user or integrated applications
- `monorepo.json` — multiple packages/apps with shared infrastructure
- `protocol.json` — protocols, chains, smart-contract systems, standards implementations
- `research.json` — research/software hybrids, models, experiments, scientific tooling

If classification is uncertain, use `base.json` alone rather than forcing an overlay.
