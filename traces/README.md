# Harness traces

Raw traces captured during the experiment. **Coverage is partial** — it is *not* one JSONL
per run (the runtime kept different artifacts for each harness):

- **`pi/traceedit03b.jsonl`** — a full **pi session in JSONL** (the harness-comparison edit
  test where the 2-bit truncated the file path: `.../typi` → directory → collapse). This is
  the only true JSONL session trace.
- **`hermes-kbd/`** — the winning EN run (`kbd`, 20/20). `scot-traces.log` = the model's
  per-turn reasoning (`reasoning_content`, the `ASSESS:/ACTION:` lines) dumped by the logging
  proxy; `logs/` = per-task Hermes stdout (tool calls).
- **`hermes-ralph-en/`** — the earlier EN run stopped at 3/20 (path-fail cascade).
- **`hermes-es-logs/ralph{9,11,12}/`** — per-task Hermes stdout for key Spanish runs
  (no reasoning dump: the proxy `scot-traces.log` for these was not preserved).

**Not available:**
- **atomic-agent** traces — deleted when that harness was uninstalled.
- **ralph2–ralph8, ralph10, ralph13, ralph14** — proxy reasoning traces were not preserved.

Format: Hermes files are plain-text logs (`.log`), not JSONL. The pi file is JSONL — one JSON
object per line, tree-structured via `id`/`parentId`.
