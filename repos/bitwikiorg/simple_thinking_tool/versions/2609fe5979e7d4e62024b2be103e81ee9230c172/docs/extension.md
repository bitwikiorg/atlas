# Extension

`simple_thinking_tool` is designed for extension in two main ways:

1. **Adding a thinking mode** — pure data, no code change.
2. **Hooks** — background scripts that act after a note is stored.

The reader is referred to the repository README for the full architecture; this document captures what the pinned evidence supports.

## Adding a thinking mode

Modes are data. They live in `src/simple_thinking_tool/modes.json`. Add one entry to register a new mode; the server auto-registers it as a tool named `think_<id>` and it appears in the `definitions` catalog. No code change.

Schema of a mode entry (per the README):

```json
{
  "id": "my_mode",
  "label": "My Mode",
  "purpose": "One sentence on what it does.",
  "when": "Use when ...",
  "level": 1,
  "typical_stage": "any",
  "fields": ["input_a", "input_b"],
  "template": "Input A: {input_a}\nInput B: {input_b}",
  "references": ["https://arxiv.org/abs/XXXX"]
}
```

Notes:

- `id` must be a unique slug; it becomes `think_<id>`.
- `fields` are the required string fields the caller supplies.
- `template` is a string with `{field}` placeholders that the server fills with the caller's values.
- `references` (optional) is surfaced in `definitions` so an LLM can see provenance at call time.
- A single JSON entry produces a new callable MCP tool without touching Python.

The README states that 15 mode tools ship out of the box (plus `health`, `definitions`, and `journal` — 18 tools total).

## Hooks: background scripts

Thinking tools record; hooks act. A hook is a background script you drop into a directory; the server fires it as a fire-and-forget task after a matching note is persisted.

### Configure

Set `NUT_THINKING_HOOKS` to a directory containing scripts. Example:

```bash
NUT_THINKING_HOOKS=/path/to/hooks NUT_THINKING_STATE_DIR=/path/to/state python -m simple_thinking_tool.server
```

### Script resolution

- A script named `<mode>.py` fires for that mode only (e.g. `tree.py` fires for `think_tree`).
- A script named `on_think.py` fires for every mode without a dedicated script.
- Mode-specific scripts win over the generic one.
- Resolution order: `<mode>.py` → `on_think.py` → no-op.

### Payload

Each hook receives one argument: the path to a small JSON payload written to the system temp dir, removed after the run. The payload is exactly `{ "mode", "note", "actor", "ts" }` — no secrets, no request context, no caller arguments beyond the rendered note.

### When hooks fire

Hooks fire only in the **successful note-storage path**, as background, best-effort events:

| Event | Fires? | Notes |
|---|---|---|
| `think_<mode>` call, guard disabled/allowed | yes | after the note is fsync'd to JSONL |
| `think_<mode>` call, guard blocked | no | no note stored; response is `blocked: true` |
| `think_<mode>` call, guard warned (allowed) | yes | warn does not suppress storage or hooks |
| Hook script missing for mode + no `on_think.py` | no-op | nothing runs; call unaffected |
| Hook script throws / times out | best-effort | failure isolated; call already returned |
| `health`, `definitions`, `journal` | no | read-only tools, never fire hooks |
| Token/auth failure | no | rejected before any storage or hook |
| Note exceeds `NUT_THINKING_MAX_NOTE_CHARS` | no | rejected before storage; no hook |

### Design rules

- Caller side stays simple: no new tool args or required return fields.
- Fire-and-forget, best-effort: a hook failure never fails or slows the `think_*` call; hook runtimes are non-blocking in a daemon thread.
- No secrets, no egress: a hook is a local subprocess the operator owns.
- Off by default: no hooks dir means zero overhead.

### Example: from trees to a forest of thought

Point a `tree.py` hook at every `think_tree` call and have it append each pruned, scored branch to a `forest-of-thought.md`:

```python
# tree.py — dropped in NUT_THINKING_HOOKS
import json, sys
payload = json.load(open(sys.argv[1], encoding="utf-8"))
with open("forest-of-thought.md", "a") as f:
    f.write("## " + payload["mode"] + "\n" + payload["note"] + "\n")
```

The README generalizes the pattern: audit a `bias_check` stream, forward `goal_distill` to a planner, feed `metacognitive` output into a review queue.

## Caveats from the evidence

- The full contents of `modes.json` (all 15 mode definitions with exact field lists and templates) were not visible in the provided evidence, so only the schema structure and mode names from the README table are documented here.
- The exact JSONL note schema (`id`, `mode`, `note`, `actor`, `ts`) is inferred from the hook payload schema and README examples, not confirmed from `store.py` source, which was not fully visible.
- No hot-reloading of `modes.json`, no plugin manifest format, and no formal schema validation files are documented in the evidence.

---

Evidence references: [E001] (mode schema, hook configuration/resolution/payload/event table, design rules and example), [E009] (hooks module), [E010] (modes.json data file holding the 15 modes).