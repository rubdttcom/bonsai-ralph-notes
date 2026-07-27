# Bonsai · structured-CoT in the Ralph loop

Performance & methodology notes from an agentic coding experiment with
**[Ternary-Bonsai-27B](https://huggingface.co/prism-ml/Ternary-Bonsai-27B-gguf)** — a 2-bit
(~1.71 bits/weight) model — on an RTX 3080. The goal: get this tiny model to build a small
single-file app through a *Ralph loop* (one atomic task per clean process), and find which
strategies mitigate its failure modes. Each run documents what was tried, what got
better/worse, and why.

## Documents
- 📄 [Summary (English)](summary-en.md)
- 📄 [Resumen (Español)](resumen-es.md)

Each summary includes: a table of all runs with their configuration (+ legend), the path
traceability (`ralph2` → `kbd`, in 3 phases), the learnings (✅ / ➖ / ❌), and sections on the
task prompt, the task sets, and the AGENTS.md versions. Both summaries also list the papers
behind each mitigation strategy tested.

## Traces
[`traces/`](traces/) holds the raw harness traces we captured — Hermes per-turn reasoning
dumps and a pi JSONL session (absolute machine paths sanitized to `~`). Coverage is partial;
see [`traces/README.md`](traces/README.md).

## Key references
- **Model:** [prism-ml/Ternary-Bonsai-27B-gguf](https://huggingface.co/prism-ml/Ternary-Bonsai-27B-gguf) — 2-bit ternary GGUF (Qwen3.6-27B backbone).
- **structured-CoT (scot):** grammar-constrained chain-of-thought, our main lever —
  [andthattoo/structured-cot](https://github.com/andthattoo/structured-cot)
  ([write-up](https://andthattoo.dev/blog/structured_cot)). Related concept paper:
  [Structured Chain-of-Thought Prompting for Code Generation](https://arxiv.org/abs/2305.06599).
