# Teach Skill — Token Optimization Analysis (v2, simulation-backed)

**Supersedes** the v1 artifact (`teach_optimization_analysis.md`). v1's savings table
(`-58.8% / -74.0% / -33.3% / -64%`) was **hand-asserted, not simulated** — the v1 model
(`teach.zigsim`) carried no token dimension at all (only success/block/abort + abstract time).
This v2 rebuilds the model with an explicit token-accounting layer and reads every number
from a real `zigsim sweep`.

User preferences (unchanged): domain-knowledge · permissive (`w_revise=0.07`) · Anthropic
auto-route (→ `sonnet`) · **goal = minimize tokens**.

---

## 1. What changed in the model

Model: [teach_tokens.zigsim](file:///Users/vedmalex/work/agent-skills/repos/mattpocock-skills/.agents/flow-optimizer/teach/teach_tokens.zigsim)

The `teach` skill is **stateful across sessions** (its own SKILL.md says so), so the model
now simulates a **course of `n_sessions` lessons**, accumulating tokens per session:

```
session s (prior lessons on disk = s):
  prompt     = tok_skill                                  # SKILL.md, every session
             + format_load                                # see below
             + history_read(s)                            # ZPD: grows with prior lessons
  gate       = lesson-gen verdict loop (BLOCK/REVISE/PROCEED), rework-capped
  completion = attempts(=da_load) × (tok_lesson_base + inline_style·tok_style)
total course token cost = Σ_s (prompt + completion)
```

The three v1 proposals are encoded as **0/1 design flags** so savings come from the sweep:

| flag | 1 (current) | 0 (proposed) | proposal |
|---|---|---|---|
| `static_formats` | all 4 `*-FORMAT.md` loaded every session | on-demand (~1/session) | **A** |
| `full_history` | read ALL prior learning-records | last 3 + `NOTES.md` summary | **B** |
| `inline_style` | inline `<style>` per lesson | shared stylesheet link | **C** |

**Calibration** (declared as tunable params, from real word counts): `tok_skill=1600`
(SKILL.md = 1186 words), `tok_format=400` (×4 ≈ the "~2000" v1 claim), `tok_record=350`,
`tok_notes=300`, `tok_lesson_base=2000`, `tok_style=1500`.

**Validity checks (all pass):** every swept combo `status=ok`; petri capacity invariant holds;
`da_load = 1.077`/lesson matches the closed-form geometric expectation `1/(1−(1−0.02)·0.07) = 1.074`;
`tok_prompt = 35 750` at 10 sessions matches the analytic `10·2000 + 350·Σ₀..₉ = 35 750`.

---

## 2. ⚠️ Rec A is a non-fix (the v1 premise was factually wrong)

v1 claimed the `*-FORMAT.md` files add "~2,000 static tokens per call even if unused".
**They are not statically loaded** — `SKILL.md` only *links* them (`[...](./MISSION-FORMAT.md)`),
so they are already read on demand. The true current state is `static_formats=0`.

Sweep proof (10-lesson course, before-baseline `full_history=1, inline_style=1`):

| `static_formats` | tok_total | meaning |
|---|---|---|
| 1 (v1's assumed "current") | 85,409 | if formats *were* static |
| 0 (**actual current**) | 73,409 | already on-demand |

The 12,000-token gap is what A *would* save **if** the premise held — but since the skill is
already on-demand, **realizable saving from A = 0**. Drop recommendation A.

---

## 3. Simulated savings — design sweep (10-lesson course, 300 runs/combo)

Source: [sweeps_tok/sweep.csv](file:///Users/vedmalex/work/agent-skills/repos/mattpocock-skills/.agents/flow-optimizer/teach/sweeps_tok/sweep.csv). All `status=ok`.

| Config | static | full_hist | inline | tok_total | tok_prompt | tok_completion | vs BEFORE |
|---|:--:|:--:|:--:|--:|--:|--:|--:|
| **BEFORE** (current) | 0 | 1 | 1 | **73,409** | 35,750 | 37,659 | — |
| C only (stylesheet) | 0 | 1 | 0 | 57,269 | 35,750 | 21,519 | **−22.0%** |
| B only (history) | 0 | 0 | 1 | 69,059 | 31,400 | 37,659 | **−5.9%** |
| **AFTER** (B+C) | 0 | 0 | 0 | **52,919** | 31,400 | 21,519 | **−27.9%** |

At a typical 10-lesson course, the realistic total saving is **≈28%**, dominated by **C**
(shared stylesheet, −22%), with **B** marginal (−6%) and **A** zero. This is far below v1's
implied ~60–74%.

---

## 4. The levers SWAP with course length (the key finding v1 missed)

Source: [sweeps_growth/sweep.csv](file:///Users/vedmalex/work/agent-skills/repos/mattpocock-skills/.agents/flow-optimizer/teach/sweeps_growth/sweep.csv) — `n_sessions × full_history × inline_style`, all `status=ok`.

| course length | BEFORE | AFTER (B+C) | total saving | B-only | C-only |
|---|--:|--:|:--:|--:|--:|
| 5 lessons | 32,413 | 25,457 | **21.5%** | 33,563 ⚠️ | 24,307 |
| 10 lessons | 73,409 | 52,919 | **27.9%** | 69,059 | 57,269 |
| 20 lessons | 181,749 | 107,899 | **40.6%** | 140,149 | 149,499 |
| 40 lessons | 503,125 | 217,686 | **56.7%** | 282,025 | 438,786 |

Two simulation-derived insights:

1. **Rec B (history compaction) has a break-even and is NET-NEGATIVE for short courses.**
   At 5 lessons, B-only (33,563) > BEFORE (32,413): the `NOTES.md` summary costs 300 tok
   *every* session while there is little history to cap. B only pays off once the triangular
   full-history sum (`Σ s · 350`, quadratic) exceeds `3·350 + 300`/session — roughly **≥ 7–8 lessons**.

2. **Dominance flips.** Short courses → **C dominates** (per-lesson completion). Long courses
   (≥20) → **B dominates**, because uncapped history grows quadratically (at 40 lessons,
   history alone = `350·Σ₀..₃₉ = 273,000` tokens). Total saving is therefore **not a flat
   number** — it ranges 22% → 57% with course length, vs v1's single asserted figure.

---

## 5. Corrected recommendations

| # | Proposal | Verdict | Simulated effect |
|---|---|---|---|
| **A** | Dynamic format loading | ❌ **Drop** | 0 — formats are already on-demand (links, not static load) |
| **C** | Shared stylesheet (`reference/lesson-style.css`) | ✅ **Apply first** | −22% at 10 lessons; dominant for short courses; flat per-lesson win |
| **B** | History compaction (last 3 + `NOTES.md` summary) | ✅ **Apply for long courses** | net-negative < ~7 lessons; −41%→−57% of the gain at 20–40 lessons |

**Combined (B+C): 22%→57% token reduction**, scaling with course length. Apply C
unconditionally; gate B behind a length check (only compact history once ≥ ~8 learning-records exist).

**Model-$ context:** at `sonnet` ($3/$15 per Mtok), a 10-lesson course is ≈ $0.67 → $0.42
(−38%, completion-weighted). Per `decision-rules`, model-$ is <1% of total cost (supervision
dominates), so the token win is about context-window headroom and latency, not dollars.

**Caveats (honest):** (1) token-size constants are calibrated from word counts, not measured
token logs — re-run with `--param tok_*=…` if you have real numbers; (2) the model ignores
prompt-cache reuse of the static SKILL.md prefix, which would further shrink the *effective*
prompt delta; (3) lesson re-reads are assumed negligible ("lessons rarely revisited" per SKILL.md).
```
