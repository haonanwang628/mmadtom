# iMAD Baseline 2 — Mistral-7B-Instruct-v0.3, Cross-Benchmark Summary

Consolidates all four completed iMAD Baseline-2 runs for the Mistral-7B-Instruct-v0.3
backbone: **GSM8K**, **CommonsenseQA**, **MMLU-Pro**, and **CS1QA**. All four runs use
the same frozen iMAD protocol (self-critique → 41-feature extraction → gate → 3-role
debate, max 5 rounds, τ=0.7) and, per the paper's own zero-shot design, the *same*
PubMedQA-trained classifier (`classifier_training_mistral/model_v1_pubmedqa964`),
**never retrained** across any of the four datasets. GSM8K's result folders were
relocated by the user after that run completed; this summary reuses only its already-
published `experiment_summary`/consolidated-report figures, not a rerun.

## Main Results

| Metric | GSM8K | CommonsenseQA | MMLU-Pro | CS1QA |
|---|---:|---:|---:|---:|
| Task format | Open-ended numeric | Native 5-way MCQ | Native 3–10-way MCQ | Open-ended free text (code QA) |
| N | 1319 | 1221 | 12032 | 1847 |
| Official Single-Agent Accuracy (Baseline 1, reused not rerun) | 0.5110 | 0.6970 | 0.3125 | 0.2826 |
| iMAD Acc(0) (pre-debate, post-self-critique) | 0.3192 | 0.6577 | 0.2818 | 0.2426 |
| iMAD FinalAcc (post-debate) | 0.3616 | 0.6609 | 0.2832 | 0.2420 |
| Self-Critique Delta (Acc(0) − Single-Agent) | −0.1918 | −0.0393 | −0.0307 | −0.0401 |
| Debate Delta (FinalAcc − Acc(0)) | +0.0425 | +0.0033 | +0.0014 | **−0.0005** |
| Total Delta (FinalAcc − Single-Agent) | −0.1494 | −0.0361 | −0.0293 | −0.0406 |
| Overall DR | 0.02047 | 0.00573 | 0.01513 | 0.01462 |
| Overall RR | 0.06293 | 0.00901 | 0.01654 | 0.01408 |
| Trigger rate | 47.7% (629/1319) | 47.26% (577/1221) | 63.8% (7682/12032) | 34.38% (635/1847) |
| Self-critique JSON schema-valid rate | 65.3% | 89.35% | 72.43% | 98.11% |
| Debate role-call error rate (per triggered example) | — | 0.0017 | 0.0309 | 0.0110 |
| Avg. debate rounds (triggered) | 1.75 | 1.33 | — | 2.008 |

## Cross-Benchmark Findings

1. **Self-critique is the dominant source of accuracy loss on every dataset, but its
   severity is highly task-dependent.** GSM8K takes by far the largest hit (−19.18pp)
   — a single forced counter-argument is enough to talk Mistral out of a large share
   of correct numeric derivations. CS1QA (−4.01pp) and CommonsenseQA (−3.93pp) are
   comparable to each other; MMLU-Pro is the mildest (−3.07pp). CS1QA's open-ended,
   code-grounded answers make it more vulnerable to this effect than the two MCQ
   datasets, despite having no fixed answer set to anchor confusion around — likely
   because free-text answers offer far more surface area for the counter-argument
   step to introduce a plausible-sounding but wrong alternative.

2. **CS1QA is the only one of the four datasets where debate is net-negative for
   Mistral**, even if only marginally (−0.05pp). Every other dataset shows debate
   recovering at least part of the self-critique loss (+0.14pp to +4.25pp). CS1QA's
   correct→incorrect and incorrect→correct transition counts are nearly balanced (27
   vs. 26 out of 1847), so this is closer to "debate does nothing net" than "debate
   actively harms" — but it is the first dataset in this series where selective
   debate does not even partially offset the self-critique penalty.

3. **Trigger rate does not track task difficulty in an obviously monotonic way.**
   MMLU-Pro (hardest, 28.2% pre-debate accuracy) triggers most (63.8%); CS1QA
   (also hard, 24.3% pre-debate accuracy) triggers least (34.4%) — well below even
   GSM8K's and CommonsenseQA's ~47%. Since the classifier was trained purely on
   PubMedQA and never sees any of these four datasets, this reflects how differently
   each domain's self-critique text maps onto PubMedQA-derived confidence/hedging
   features, not any deliberate re-calibration.

4. **Self-critique reliability (JSON schema-valid rate) is CS1QA's strongest result
   in this series**: 98.11%, comfortably the highest of the four datasets (vs. 65.3%
   on GSM8K, the weakest). The frozen self-critique schema generalizes cleanly to a
   free-text, code-grounded answer format for this backbone, even though the
   underlying accuracy dynamics (points 1–2 above) are less favorable.

5. **Debate role-call error rate on CS1QA (0.0110/triggered example)** sits between
   CommonsenseQA's best-observed rate (0.0017) and MMLU-Pro's worst (0.0309),
   confirming the free-text `debate_engine_cs1qa.py` adaptation of the post-bugfix
   JSON-template pattern did not reintroduce the doubled-brace pathology discovered
   and fixed earlier in this series (which, at its worst, produced 4.96
   errors/triggered-example on the first, discarded Qwen/CommonsenseQA attempt).

6. **τ=0.7 was never re-tuned for any of the four datasets or this backbone** — an
   explicitly disclosed limitation carried through every run in this series. Gate
   selectivity (the gap between wrong-answer trigger recall and the false-trigger
   rate on already-correct examples) varies by dataset without a consistent pattern,
   consistent with this being a genuinely fixed, not dataset-adapted, threshold.

## Per-Benchmark Reports

- GSM8K: `../GSM8K/iMAD_GSM8K_summary_en.md` (consolidated 3-backbone report; result
  folders relocated by the user after completion, per-example artifacts not
  independently re-verifiable from this repo)
- CommonsenseQA: `CommenseQA/experiment_summary.md`
- MMLU-Pro: `MMLUpro/experiment_summary.md`
- CS1QA: `CS1QA/experiment_summary.md`

## Status

All four Mistral-7B-Instruct-v0.3 iMAD Baseline-2 runs are complete. Per instruction,
Qwen2.5-7B-Instruct and Phi-4-mini-instruct experiments on CS1QA have **not** been
started automatically and will only begin on explicit request.
