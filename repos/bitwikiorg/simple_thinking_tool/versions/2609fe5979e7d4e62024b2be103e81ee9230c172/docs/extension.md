# Extending the Tool

There are two primary extension points: **adding thinking modes** (a data edit, no code) and **writing background hooks** (scripts that act on stored notes). Both are designed so the caller side stays unchanged.

## Add a thinking mode

Thinking modes are data, not code. They live in `src/simple_thinking_tool/modes.json`; the server loads them at startup and auto-registers each entry as a tool named `think_<id>` with its fields as explicit required string arguments [E001][E019][E020]. No Python changes and no redeploy of logic are needed [E001].

### The entry schema

Each mode entry in the `modes` array supports these keys [E019][E020]:

| Key | Required | Meaning |
|---|---|---|
| `id` | yes | Becomes the tool name `think_<id>` |
| `label` | no (falls back to `id`) | Human-readable name |
| `purpose` | no | One sentence on what it does |
| `when` | no | When to use this mode |
| `level` | no (default `1`) | Difficulty/depth indicator (1–4) |
| `typical_stage` | no (default `any`) | Workflow stage: `plan`, `understand`, `execute`, `review`, `handoff`, or `any` |
| `fields` | no (default `[]`) | Template input names the caller must supply |
| `template` | no | Template string with `{field}` placeholders |
| `version` | no (default `1`) | Template version, stored on each note |
| `references` | optional | Surfaced in `definitions` so the LLM can see provenance at call time |

The `references` key is not part of the `Mode` dataclass in `modes.py` (which reads `id`, `label`, `purpose`, `when`, `level`, `typical_stage`, `fields`, `template`, `version`) [E020]; the README recommends it as an optional per-mode field surfaced in `definitions` [E001].

### Example

From the README [E001]:

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

It auto-registers as `think_my_mode(input_a: str, input_b: str)` and appears in `definitions`. No code changed [E001].

### Template rendering behavior

Rendering uses Python string formatting. Missing fields are filled with empty strings rather than raising errors — `render_template` substitutes `""` for any declared placeholder that was not provided [E020]. The server truncates the rendered note to `NUT_THINKING_MAX_NOTE_CHARS` (default 800) before storage [E006][E008].

### Notes on the dynamic registration

The server generates each mode function dynamically from the `modes.json` field names using `exec()` [E006]. The field names are operator-controlled (from the JSON file, not user input), but this means `modes.json` should be treated as trusted configuration — an externally modifiable `modes.json` could be a code-injection vector. Keep the file under your control.

## Write background hooks

Thinking tools record; **hooks act**. A hook is a background script you drop into a directory; the server fires it as a fire-and-forget task after a matching note is persisted. The caller side never changes — a mode tool signs and stores exactly as before; the hook runs out of band [E001][E005].

### Configure

Set `NUT_THINKING_HOOKS` to a directory containing scripts:

```bash
NUT_THINKING_HOOKS=/path/to/hooks NUT_THINKING_STATE_DIR=/path/to/state python -m simple_thinking_tool.server
```

[E001]

### Resolution rules

- A script named `<mode>.py` fires for that mode only (e.g. `tree.py` fires for `think_tree`).
- A script named `on_think.py` fires for every mode without a dedicated script.
- Mode-specific scripts win over the generic one.

[E001][E005]

### Payload

Each hook receives one argument: the path to a small JSON payload file written to the system temp dir and removed after the run. The payload contains only `{mode, note, actor, ts}` — no secrets, no request context, no caller arguments beyond the rendered note [E001][E005]. `ts` is a millisecond epoch timestamp [E005].

### When hooks fire (disclosed)

Hooks fire only in the **successful note-storage path**. They are background and best-effort — treat them as events, never as a guarantee [E001].

| Event | Fires? | Notes |
|---|---|---|
| `think_<mode>` call, guard **disabled / allowed** | yes | fires after the note is fsync'd to JSONL |
| `think_<mode>` call, guard **blocked** | no | no note stored ⇒ no hook; response is `blocked:true` |
| `think_<mode>` call, guard **warned** (but allowed) | yes | warn does not suppress storage or hooks |
| Hook script missing for mode + no `on_think.py` | no-op | nothing runs; call unaffected |
| Hook script throws / times out | best-effort | failure isolated; call already returned |
| `health`, `definitions`, `journal` | no | read-only tools, never fire hooks |
| Token/auth failure | no | rejected before any storage or hook |
| Note exceeds `NUT_THINKING_MAX_NOTE_CHARS` | no | rejected before storage; no hook |

[E001]

Ordering: resolve `<mode>.py` → else `on_think.py` → else no-op. Mode-specific wins [E001][E005].

### Design rules

- **Caller side stays simple.** No new tool args, no new return fields required — hooks are advanced background configuration, not part of the normal call path [E001].
- **Fire-and-forget, best-effort.** A hook failure never fails or slows the `think_*` call. Hook runtimes are non-blocking in a daemon thread [E001][E005].
- **No secrets, no egress.** A hook is a local subprocess the operator owns. The payload carries only the thought itself. The server ships nothing anywhere [E001].
- **Off by default.** No hooks dir = zero overhead [E001].
- **Timeout.** Each hook runs via `subprocess.run` with a timeout from `NUT_THINKING_HOOK_TIMEOUT` (default 10.0 seconds); a timeout or non-zero return is logged, never fatal [E005].

### Example: from trees to a forest of thought

Point a `tree.py` hook at every `think_tree` call and have it append each pruned, scored branch to a `forest-of-thought.md` [E001]:

```python
# tree.py  # dropped in NUT_THINKING_HOOKS
import json, sys
payload = json.load(open(sys.argv[1], encoding="utf-8"))
with open("forest-of-thought.md", "a") as f:
    f.write("## " + payload["mode"] + "\n" + payload["note"] + "\n")
```

[E001]

The same pattern generalizes: audit a `bias_check` stream, forward `goal_distill` to a planner, feed `metacognitive` output into a review queue [E001].