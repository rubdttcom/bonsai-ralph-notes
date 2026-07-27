# Performance & methodology notes — structured-CoT in the Ralph loop

Date: 2026-07-23. Context: test whether structured-CoT (pre-trigger grammar)
speeds up Hermes's agentic loop over Ternary-Bonsai-27B on the RTX 3080.

## Summary table — all runs and their configuration

The whole project in one table; below it, the numbered **legend** (the ¹–⁸ superscripts on headers/values link to it).

**Common to all rows:** model Ternary-Bonsai-27B-Q2_0 (2-bit) on RTX 3080 · served context `-c` = **64K** (uniform) · `reasoning_budget` = -1 (OFF) except `ralph13` · Harness = Hermes except the `atomic-agent` and `pi` rows.

| Run | Harness | Binary¹ | scot (template)² | AGENTS.md reinf.³ | Key config / change⁴ | GPU cap (W)⁵ | max_tok cap⁶ | Lang | Tasks | Total time | Turns | Spiral⁷ | Result / finding |
|--|--|--|--|--|--|--|--|--|--|--|--|--|--|
| ralph2 | Hermes | Prebuilt | — (no-scot) | — | historical baseline · no proxy | 280 | 8192 | ES | 10 | ~20m | — | — | 10/10, working game |
| ralph3 | Hermes | Patched | GOAL/PLAN/CHECK | — | 1st scot · no proxy | 280 | 8192 | ES | 10 | 44m24s | — | B | 10/10; **death-spiral** t9=1168s (bad template) |
| ralph4 | Hermes | Patched | ASSESS/ACTION | — | scot baseline | 280 | 8192 | ES | 10 | **15m06s** | 100 | — | 10/10 — turned out to be **luck** (see ralph8) |
| ralph5 | Hermes | Patched | — (no-scot) | — | control (no scot) | 280 | 8192 | ES | 10 | 34m06s | 202 | — | 10/10; scot-A ≈ ½ turns vs this |
| ralph6 | Hermes | Patched | ASSESS/ACTION | v1 | — | 280 | 8192 | ES | 10 | 22m58s | 122 | B? | v1 fragmented → worse; stop adherence 77% |
| ralph7 | Hermes | Patched | ASSESS/ACTION | v2 | — | 280 | 8192 | ES | 10 | 18m32s | 117 | B | v2 > v1 but doesn't beat ralph4 |
| ralph8 | Hermes | Patched | ASSESS/ACTION | — | replica of ralph4 | 280 | 8192 | ES | 10 | 29m56s | 131 | B | **variance ~2×**: ralph4 was luck |
| ralph9 | Hermes | Patched | ASSESS/ACTION | — | **tail split** (init→3, verify→3) | 280 | 8192 | ES | 14 | **15m33s** | 105 | — | 14/14; kills the tail spiral (structural) |
| ralph10 | Hermes | Patched | ASSESS/ACTION | — | replica of ralph9 | 280 | 8192 | ES | 14 | 22m26s | 116 | B | verify stable 83s, but spiral **migrates** to logic |
| ralph11 | Hermes | Patched | ASSESS/ACTION | — | logic atomized | 280 | 8192 | ES | 18 | 22m30s | 149 | B | 18/18; spiral migrates to **sentences** (368s) |
| ralph12 | Hermes | Patched | ASSESS/ACTION | — | (trace reading) | 280 | 8192 | ES | 20 | — | — | A | **Spiral A**: turn "Done."×4094 = 24k chars |
| ralph13 | Hermes | Patched | ASSESS/ACTION | — | **DRY⁸ + reasoning_budget=64** | 280 | 8192 | ES | 20 | (aborted) | — | B | budget=64 **backfire** (thrash task1) |
| ralph14 | Hermes | Patched | ASSESS/ACTION | — | **DRY⁸ only** (budget -1) | 280 | 8192 | ES | 20 | (stopped) | — | B | DRY kills Spiral A; **Spiral B persists** (task3) |
| atomic-agent | **atomic-agent** | Patched :8080 | — (own grammar) | — | scot doesn't port | 280 | 8192 | ES | task 01 | — | — | B | **doesn't even create the file** (Spiral B ×3) |
| pi | **pi** | Patched :8080 | — (own grammar) | — | PTY · thinkingFormat deepseek | 280 | 8192 | ES | task 01 | 166s (create) | — | B | creates OK; **edit fails** on paths (trace) |
| ralph-en | Hermes | Patched | ASSESS/ACTION | — | dir with name **COLLISION** | 320 | 8192 | **EN** | 20 (stopped 3/20) | — | — | B | path-fail cascade, ~185s/task |
| **kbd** | Hermes | Patched | ASSESS/ACTION | PATH | **neutral dir + decoys moved aside** | 280 | 8192 | **EN** | 20 | **60m45s** | 290 ct / 133 stop | B | **20/20, 0 failures, 0 path-fails**; slow (sentences+cap) |
| kbd / hot-01 | Hermes | Patched | ASSESS/ACTION | PATH | **hot** GPU | 280 | 8192 | EN | task 01 | **196s** | — | — | vs **638s cold** → 3.3× (isolates warmup) |

### Legend

**¹ Binary** — `Prebuilt` = historical release of the fork; `Patched` = PrismML fork recompiled with the `pre-trigger-grammar` patch (what enables scot). Speed difference ≈5% (within noise).

**² scot (template)** — *structured-CoT*: a GBNF grammar that forces reasoning into a fixed format. `ASSESS/ACTION` (**decision**-based, good → ½ turns) · `GOAL/PLAN/CHECK` (**planning**-based, bad → death-spiral) · `— (no-scot)` = no grammar · `— (own grammar)` = the harness (atomic-agent/pi) uses its own tool grammar and **doesn't allow scot**.

**³ AGENTS.md reinforcement** — *prompt-engineering* edits of `AGENTS.md` (a separate axis from scot and the proxy): `v1` ("1 tool/turn" → **made it worse**, fragmented) · `v2` ("min. turns" → better than v1, doesn't beat the baseline) · `PATH` (path rule: "use the relative name, never absolute paths") · `—` = no reinforcement.

**⁴ Key config / change** — the run's distinctive change. Vocabulary:
- *proxy* = `scot_proxy.py` inserted in between (:8080→:8081) that **only observes** (dumps `reasoning_content` to a log so we can read the traces); it **does not influence** generation, not even in the control. `no proxy` = no trace capture.
- *control* = run **without scot**, for comparison.
- *tail split · logic atomized · replica* = task **decomposition** decisions.
- *DRY⁸ · reasoning_budget* = **sampler** tweaks (see ⁸).
- *PTY* = a `script -qfec` wrapper that gives a pseudo-terminal (pi hangs without it).
- *collision · neutral dir + decoys moved aside* = fixes for the **path attractor** (a working dir the 2-bit confused with another that had a decoy `typing-game.html`).

**⁵ GPU cap (W)** — power limit of the 3080 (`nvidia-smi -pl`). **280 W** in all except `ralph-en` (**320**: the reboot reset the cap to default; that's why that run was stopped and 280 was re-set in `kbd`).

**⁶ max_tok cap** — per-response **output** limit (`max_tokens`) = **8192** in all. It's the "8192 cap" that triggers **Spiral A** (the model generates until it hits it).

**⁷ Spiral** — the type of spiral when the 2-bit fails to finish a task: **A** = degenerate repetition in ONE generation (e.g. `"Done."`×4094; killed by DRY) · **B** = agentic thrash (many turns, different tools, file-state/path confusion; the **2-bit's intelligence ceiling**, no inference lever cures it). Trace-verified only in `ralph8/11/12` and `kbd`; the rest **inferred**. `B?` = probable · `—` = no spiral. **A is only possible without DRY** (≤`ralph12`).

**⁸ DRY** — *Don't Repeat Yourself*, a llama.cpp sampler that **penalizes repeating sequences** already generated (exponential penalty past `allowed-length`). Config: `--dry-multiplier 0.8 --dry-base 1.75 --dry-allowed-length 2`. Kills **Spiral A** at zero cost; doesn't touch B. ON by default since `ralph13`.

## The experiment's path (traceability)

Read top to bottom: each test **mitigates the previous one's error**. Result: 🟢 better · 🔴 worse/fail · 🟡 mixed/same · ⚪ baseline/diagnostic.

Convention: **`ralphN`** = iteration N of the *Ralph loop* (one task per clean process, queued in `todo/done/fail` folders); each run's name is also its working directory. Rows are grouped into the project's **3 phases**.

| # | Run | What's tested (mitigation) | Against the error of | Result | Learning |
|--|--|--|--|--|--|
| | **▸ Phase 1 · scot experiment (Hermes, ES)** | | | | |
| 1 | ralph2 | baseline (prebuilt, no scot) | — (starting point) | ⚪ ~20m, 10/10 | initial reference |
| 2 | ralph3 | add **scot** (GOAL/PLAN/CHECK template) | baseline slowness/variance | 🔴 44m, death-spiral | *planning* template → spiral |
| 3 | ralph4 | **ASSESS/ACTION** template (decision) | GOAL/PLAN/CHECK death-spiral | 🟢 15m, ½ turns | scot helps **if it's decision-based** |
| 4 | ralph5 | **control without scot** (same binary/proxy) | isolate scot's real effect | ⚪ 34m, 202 turns | confirms scot ≈ **½ turns** |
| 5 | ralph6 | `AGENTS.md` reinforcement **v1** ("1 tool/turn") | low stop adherence in ralph4 | 🔴 23m | micromanagement **fragments** → worse |
| 6 | ralph7 | reinforcement **v2** ("min. turns") | v1 fragmentation | 🟡 18.5m | better than v1, **doesn't beat baseline** |
| 7 | ralph8 | **replicate** ralph4 exactly | were the 15m real? | 🟡 30m | ralph4 was **luck**; variance ~2× |
| 8 | ralph9 | **split the tail** (init/verify → 6 tasks) | tail spiral (verify 459s) | 🟢 15.5m, structural | atomizing **kills the local spiral** |
| 9 | ralph10 | replicate ralph9 | confirm the tail fix | 🟡 22.5m | tail stable, but **spiral migrates** to logic |
| 10 | ralph11 | **atomize the logic** (18 tasks) | render-finish spiral | 🟡 22.5m | spiral migrates to **sentences** (whack-a-mole) |
| 11 | ralph12 | **read the traces** from the proxy | what is the spiral inside? | ⚪ diagnostic | identifies **Spiral A** (repetition) |
| 12 | ralph13 | **DRY** + reasoning_budget=64 | Spiral A + B | 🔴 backfire | budget=64 **worsens B**; aborted |
| 13 | ralph14 | **DRY only** (budget -1) | budget backfire | 🟢🟡 | DRY **kills A**; **Spiral B persists** (= ceiling) |
| | **▸ Phase 2 · Harness comparison (task 01 test)** | | | | |
| 14 | atomic-agent | switch **harness** (atomic-agent) | does another harness avoid the ceiling? | 🔴 doesn't even create the file | same ceiling B, worse surface |
| 15 | pi | switch **harness** (pi) | atomic-agent's failure | 🟡 creates, edit fails | same ceiling B; **doesn't recover** |
| | **▸ Phase 3 · English + path fix (Hermes, EN)** | | | | |
| 16 | ralph-en | **translate to English** (back to Hermes) | does Spanish degrade the 2-bit? | 🔴 path-fails | dir with **collision** = attractor (our mistake) |
| 17 | **kbd** | **neutral dir + decoys moved + PATH rule** | ralph-en's path-fails | 🟢 20/20, 0 path-fails | real, transferable **path fix** |
| 18 | kbd / hot-01 | task 01 with **hot GPU** | were the 638s a cold start? | 🟢 196s | confirms **cold-start** (3.3×) |

**The through-line:** each green step exposed the next bottleneck —
template → variance → tail → logic → sentences → repetition (Spiral A) → thrash (Spiral B) →
harness → language → paths. The *strategy* fixes (scot, atomizing, DRY, clean dir) run out at
**Spiral B**, which is the 2-bit model's intelligence ceiling.

## Learnings

### ✅ Positive — the strategy DOES mitigate the error

| Strategy | Mitigates (error) | Evidence |
|--|--|--|
| **scot ASSESS/ACTION template** | the 2-bit rambles → doubles turns | ~½ turns vs no-scot (robust, n=4) |
| **Atomic tasks** (small edits < 8192 cap) | death-spirals from big/open-ended tasks | tail 738s → 174s; avoids the 1168s case |
| **DRY sampler** | **Spiral A** (degenerate repetition) | `"Done."`×4094 doesn't recur; one flag, cost 0 |
| **Verify/counting OUTSIDE the loop** (deterministic) | tasks that trigger **Spiral B** | removes over-verify (459s) by design |
| **Neutral dir + decoys moved + PATH rule** | path-fail cascade (Spiral B via paths) | `kbd`: **0 path-fails** in 20 tasks |
| **Hermes loop-detection** (`same_tool_failure`) | a path error that doesn't resolve | lets it recover and **complete** (vs pi) |

### ➖ Neutral — no clear impact (or confounded by noise)

| Change | Impact | Note |
|--|--|--|
| **Translate tasks to English** | **none measurable** in speed/quality | confounded by thermals + spirals + n=1; reliability OK |
| **AGENTS.md reinforcement v2** | doesn't beat the no-reinforcement baseline | uncertain return, masked by ~2× variance |
| **Thinking length per turn** | scot **doesn't** shorten it | the saving is from turn count, not from compressing |
| **Logging proxy** | **zero** (only observes) | transparent; forwards byte for byte |
| **Patched vs prebuilt binary** | ~5% tok/s (53 vs 56) | within measurement noise |

### ❌ Negative — makes it worse or doesn't work

| Change | Effect | Why |
|--|--|--|
| **Switching harness** (atomic-agent / pi) | **worse**: they don't complete | they lose the **scot grammar** (decisive turns) and loop recovery → the 2-bit is left unaided |
| **scot GOAL/PLAN/CHECK template** | death-spiral (44m) | *planning* each turn instead of *deciding* → doesn't finish |
| **Reinforcement v1** ("1 tool/turn") | +turns, worse | micromanagement → fragments tool use |
| **Aggressive reasoning_budget** (=64) | worsens **Spiral B** | cuts reasoning short → can't recover from an error |
| **Any inference lever vs Spiral B** | doesn't cure it | DRY/budget/temp don't touch the agentic *thrash* = the 2-bit's ceiling |
