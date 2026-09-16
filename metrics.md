# MMAD-ToM Evaluation Metrics

Four metrics used to evaluate answer-revision behavior in confidence-informed multi-agent debate, designed to isolate *when* and *how much* debate helps (Recovery) versus harms (Drift), separate from aggregate accuracy.

---

## 1. Drift Rate

### Definition

A **drift event** occurs when a debater agent's answer is correct at round `r-1` but becomes incorrect at round `r`. Drift Rate measures, among agent-example pairs that were correct at the previous round, what fraction get flipped to incorrect.

The denominator is conditioned on "correct at r-1" (not the full agent-example population) so the rate reflects the *fragility of a correct answer under peer pressure*, independent of how many answers happened to be correct at that round.

### Formula

For round transition `r-1 → r`:

```
Drift Rate(r) = # {agent-example pairs: correct at r-1, incorrect at r}
                ─────────────────────────────────────────────────────
                # {agent-example pairs: correct at r-1}
```

Computed separately for every consecutive round transition (1→2, 2→3, 3→4, ...), over debater agents only (the Judge is excluded — it arbitrates rather than holding a revisable personal stance).

### Expected Results

- **Vanilla / confidence-based MAD**: Drift Rate should show a visible peak in the early-to-middle rounds — the point where an agent first sees a peer's confidently-stated counterargument. This is the direct empirical signature of sycophancy: an agent abandoning a correct answer because of peer confidence rather than genuine reasoning correction.
- **iMAD (triggered subset only)**: since iMAD reuses a vanilla MAD engine once debate is triggered, its conditional Drift Rate within the triggered subset should be statistically indistinguishable from vanilla MAD's. iMAD reduces *exposure* to drift (by debating less often) but does not reduce the *per-debate* drift probability — this is the key evidence that iMAD addresses a different axis of the problem (§3 of `iMAD_summary.md`).
- **MMAD-ToM**: the ToM peer-reasoning model and teacher guidance are expected to flatten or lower this peak — i.e., a visibly lower Drift Rate in the rounds where vanilla MAD peaks, without needing to suppress revision altogether (see the joint reading with Recovery Rate below).

---

## 2. Recovery Rate

### Definition

A **recovery event** occurs when a debater agent's answer is incorrect at round `r-1` and becomes correct at round `r`. Recovery Rate measures, among agent-example pairs that were incorrect at the previous round, what fraction get corrected.

Symmetric to Drift Rate: the denominator is conditioned on "incorrect at r-1," isolating the *fraction of recoverable errors that debate actually recovers*.

### Formula

For round transition `r-1 → r`:

```
Recovery Rate(r) = # {agent-example pairs: incorrect at r-1, correct at r}
                    ────────────────────────────────────────────────────
                    # {agent-example pairs: incorrect at r-1}
```

Same scope as Drift Rate: debater agents only, computed per consecutive round transition.

### Expected Results

- All debate methods should show Recovery Rate meaningfully above 0% — this is the baseline justification for running debate at all.
- **The critical comparison is not Recovery Rate in isolation, but Drift Rate vs. Recovery Rate together.** A trivial way to "reduce drift" is to make agents stubborn (never revise), which would also collapse Recovery Rate toward 0% — that is not a success case, it's just disabling debate. The success criterion for MMAD-ToM is a **decoupling**: Drift Rate goes down while Recovery Rate stays flat or improves, showing the ToM mechanism distinguishes "peer reasoning that genuinely reveals an error" (should be kept, drives Recovery) from "peer confidence alone" (should be filtered out, drives Drift).
- Expect MMAD-ToM's Recovery Rate to be comparable to or higher than vanilla MAD's, not lower — a drop here alongside a Drift Rate drop would indicate the mechanism is just suppressing all revision, undermining the paper's contribution claim.

---

## 3. Accuracy1 (Post-Debate Accuracy)

### Definition

The overall fraction of examples answered correctly by the **final, debate-concluded answer** (the Judge's finalized decision, or the consensus/majority answer if there is no explicit Judge role). This is the standard headline accuracy metric, comparable across single-agent, vanilla MAD, iMAD, and MMAD-ToM.

### Formula

```
Accuracy1 = # {examples: final debate answer is correct}
            ──────────────────────────────────────────────
            # {total examples}
```

### Expected Results

- Ordering expectation: single-agent baseline ≤ vanilla/confidence-based MAD ≤ MMAD-ToM.
- Against iMAD: expect comparable or better Accuracy1 for MMAD-ToM on the same backbone and benchmark, but this must be reported **alongside token/inference cost**, since iMAD's design goal is efficiency, not maximal accuracy — a fair write-up states the accuracy-cost trade-off explicitly rather than treating Accuracy1 alone as the deciding metric.

---

## 4. Accuracy2 (Per-Round Accuracy Trajectory)

### Definition

The fraction of agent-example pairs that are correct **at each individual round** (not just the final round), tracked across the full debate. This captures the *shape* of the debate process rather than only its endpoint — something neither iMAD nor vanilla MAD baselines typically report.

### Formula

For round `r`:

```
Accuracy2(r) = # {agent-example pairs: correct at round r}
               ───────────────────────────────────────────
               # agents × # examples
```

Plotted as a curve over `r = 1, 2, ..., R` (R = max debate rounds).

### Expected Results

- **Vanilla / confidence-based MAD**: expect a "dip-and-partial-rebound" trajectory — accuracy drops in the early-to-middle rounds (Drift outpacing Recovery, driven by sycophancy) before partially recovering by the final round as stronger arguments eventually win out. A method evaluated only on Accuracy1 would completely miss this dip.
- **MMAD-ToM**: expect a flatter, non-decreasing (or much shallower dip) trajectory, demonstrating that the ToM + teacher-guidance mechanism prevents the mid-debate wobble rather than merely cleaning it up by the end. This curve, read together with the Drift/Recovery peak location, is the main empirical evidence for the paper's central claim.

---

## Notes on Measurement Validity

1. **Scope**: Drift Rate, Recovery Rate, and Accuracy2 are computed over debater agents' individual stated positions per round; Accuracy1 is computed over the debate's single finalized/consensus answer. Mixing the Judge's arbitration output into the per-round agent statistics would conflate two different behaviors (individual susceptibility to peer influence vs. arbitration quality) and should be avoided.
2. **Early termination / sample shrinkage**: if the debate protocol allows early stopping (e.g., the Judge ends the debate once one side is clearly favored), later round transitions (e.g., round 4→5) are computed over a shrinking, non-random subset — only the hardest, unresolved cases remain. Report the denominator count (N) alongside each round's Drift/Recovery Rate so trends across rounds aren't misread as a mechanism effect when they are actually a composition/selection effect.
3. **Cross-method comparability**: because Drift Rate and Recovery Rate are conditioned on the previous round's correctness, they remain comparable across methods and datasets even when the underlying round-by-round accuracy trajectories differ — this is the reason for conditioning the denominators rather than using the full agent-example population for both.
