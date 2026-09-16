# iMAD: Paper Summary and Reproduction Feasibility Analysis

> Source: *iMAD: Intelligent Multi-Agent Debate for Efficient and Accurate LLM Inference* (Fan, Yoon, Ji — Virginia Tech, AAAI'26)
> arXiv: 2511.11306v2　Code: https://github.com/Fanwei100/iMAD

---

## Part 1: Project Summary

### 1.1 The Problem

Multi-Agent Debate (MAD) lets multiple LLM agents argue with and critique each other, in theory correcting a single agent's mistakes. The authors ran empirical measurements across six QA/VQA datasets and found two counterintuitive issues:

- **High token cost**: MAD consumes 3-5x more tokens than single-agent CoT (Table 1), since every debate round resends the full conversation history to every agent.
- **Gains concentrated in a tiny subset of cases**: bucketing each instance by (single-agent correct/incorrect) × (MAD correct/incorrect), the fraction where MAD actually fixes a wrong answer (✗→✓) is small (4.9% on OKVQA, 19.1% on GSM8K at the high end). Much of the debate is either **redundant** (already correct, ✓→✓) or **harmful** (flips a correct answer to incorrect, ✓→✗).

Further analysis (Insight 3) shows confidence scores are unreliable signals for deciding when to debate: incorrect answers are often still assigned high confidence (overconfidence), and confidence frequently misaligns with hesitation cues in the reasoning text (hedging, contradictions, shallow reasoning).

### 1.2 iMAD's Approach

iMAD (intelligent Multi-Agent Debate) does not redesign the debate itself — it adds a lightweight **gating layer that decides whether debate is even worth running**, in three steps:

1. **Structured self-critique prompt**: a single agent produces, in one pass, an initial CoT answer → a forced counterargument against itself → a final answer, with confidence scores for both the initial and final stance. This adds almost no extra tokens but surfaces the model's internal hesitation signals.
2. **41 interpretable features**: extracted from the question, initial reasoning, and self-critique text — surface-level statistics, readability metrics (Flesch, Coleman-Liau), syntactic parse depth, part-of-speech counts, and hedge/certainty/contrast word counts.
3. **A lightweight MLP classifier trained with FocusCal loss**: a shared encoder feeds two heads — a correctness head producing score p and a hesitation head producing score u. The loss combines an Asymmetric Focal Loss (heavily penalizing "high score but wrong" overconfident cases), a Confidence Penalty (penalizing misalignment between p and the semantic hesitation score u), and an ECE term (calibration). At inference, a threshold τ=0.7 decides: if p is below the threshold, trigger the full three-agent MAD; otherwise keep the single-agent answer.

The classifier is trained on **only two datasets** (PubMedQA, GQA) and then used zero-shot on six unseen evaluation datasets (MedQA/MMLU/GSM8K/OKVQA/VQA-v2/ScienceQA) — this is the paper's core selling point: it learns generalizable "model hesitation behavior," not dataset-specific quirks.

### 1.3 Results

- vs. full-debate MAD: up to 68-92% fewer tokens, with comparable or higher accuracy (8.4% higher than MAD on GSM8K).
- vs. single-agent CoT: up to 13.5% accuracy improvement.
- vs. DOWN, a comparable selective-triggering baseline (which needs a confidence threshold tuned on labeled evaluation data, violating the zero-shot assumption): iMAD is about 4.1% more accurate on average, at a modest extra token cost that is well justified by the accuracy gain.
- An ablation confirms all 41 features contribute (removing the bottom 20% drops accuracy by 0.5% while token usage actually rises 6.9%).

### 1.4 Positioning

iMAD **is not a new debate architecture** — it is a general-purpose gating module for "when to debate," which could in principle sit in front of any MAD engine. Its scope is clearly bounded: save tokens, avoid flipping correct answers into wrong ones during debate. It does not address the quality of the debate protocol itself.

---

## Part 2: The MAD Structure

The debate engine iMAD reuses underneath is a standard three-role, round-based state machine (Appendix A.1). Details below:

### 2.1 The Three Roles

| Role | Input | Function |
|---|---|---|
| **Debater (Affirmative)** | The answer produced by the single-agent self-critique stage, taken as a **fixed initial stance** `{AFF_ANS}` | Supports this answer with reasons/evidence; it is not regenerated — the answer is directly reused from step 1, which is another source of token savings |
| **Debater (Negative)** | The Affirmative side's latest message `{AFF_ANS}` | Looks for problems; if it disagrees, it picks an alternative from the candidate set and provides a counter-justification |
| **Judge** | Both sides' current messages plus the full transcript | Evaluates the quality of both arguments, decides whether to end the debate or run another round, and gives a tentative answer |

### 2.2 Per-Round Flow

```
Round r:
  1. Debater(Affirmative) speaks
     - Round 1: restates {AFF_ANS} and justifies it
     - Later rounds: reads {NEG_ANS}, either holds its position or switches to
       another option from OPTIONS
  2. Debater(Negative) speaks
     - Reads the latest {AFF_ANS}, either rebuts (picks a new answer from
       OPTIONS with justification) or explicitly agrees
  3. Judge produces structured JSON:
     {
       "Preference": "Yes" | "No",
       "Supported Side": "Affirmative" | "Negative",
       "Reason": "...",
       "Debate Answer": <one of OPTIONS>
     }
  4. Branching:
     - Preference = "Yes"                → terminate immediately, adopt this Debate Answer
     - Preference = "No" and round < 5   → proceed to the next round
     - Round 5 reached with no "Yes" ever → enter the Finalization stage
```

### 2.3 Finalization (a fallback that caps worst-case token cost)

The maximum number of rounds is hard-coded to **5**. If the Judge never returns "Yes" within 5 rounds:

- The candidate set is narrowed from the full `{OPTIONS}` down to `{OPTIONS2}` (the deduplicated set of options that were actually proposed by either side during the debate)
- The Judge outputs a simplified JSON containing only `{Reason, Debate Answer}`
- A final answer is forced, and the debate ends

This `OPTIONS2` mechanism is also why iMAD can handle open-ended tasks like GSM8K math problems that have no natural multiple-choice options: instead of relying on a dataset-provided option set, it collects the answers actually proposed during the debate as the candidate pool.

### 2.4 Memory Synchronization

Before each round, the system rebuilds a full transcript (question + single-agent CoT + self-critique + every prior round's messages from all agents) and inserts it into `{TRANSCRIPT}`, sent identically to all three agents so they share the same view of the history. **This is the main source of token cost**: the transcript grows linearly with rounds, and the full transcript is resent to all three agents every round.

### 2.5 Output Validation

The Judge's JSON output is validated: no extraneous text outside the JSON object, no missing fields, `Preference` restricted to Yes/No, and `Debate Answer` restricted to the option set. Invalid output is discarded and the Judge prompt is reissued until a valid response is obtained.

### 2.6 The VQA Extension

The roles and state machine are unchanged; each agent's prompt gains an `{IMAGE}` input, and instructions explicitly require arguments to be grounded in concrete visual evidence, preventing debate from drifting away from the actual image content.

---

## Part 3: Would Reproducing This on GSM8K / CommonsenseQA / MMLU-Pro / CS1QA and Comparing to MMAD-ToM Be Effective?

### 3.1 Bottom Line

**Yes, this is a reasonable and informative experiment — but three preconditions need to be settled first, or the conclusion won't hold up.**

### 3.2 Three Things to Confirm First

**(1) Pick the right level to compare at**

iMAD is not a debate architecture — it is a gate that decides whether to debate at all. Comparing "iMAD (mostly skips debate)" directly against "MMAD-ToM running full debate every time" will make iMAD win on tokens and latency by construction, but that says nothing about whether MMAD-ToM's debate architecture is good — it's an unfair comparison. Two more informative designs:

- **Same-tier comparison**: use iMAD's FocusCal classifier as a baseline gate and compare it against MMAD-ToM's own triggering/termination mechanism (if it has one), both running the same debate engine, to see which decides more accurately.
- **Orthogonal comparison (more interesting)**: attach iMAD's classifier in front of the MMAD-ToM debate engine ("selective MMAD-ToM"), and compare always-on MMAD-ToM vs. iMAD-gated MMAD-ToM vs. MMAD-ToM's own gating (if any). This answers two questions at once: whether the architecture itself is good, and whether the gating mechanism transfers.

**(2) The classifier's zero-shot assumption needs to be re-verified**

iMAD's classifier is trained only on PubMedQA + GQA. Your four target benchmarks (math reasoning, commonsense reasoning, professional-exam MCQ, intro-programming QA) may have very different "hesitation feature" distributions from PubMedQA/GQA, so directly reusing the released checkpoint may not generalize well. Recommendation: using the same FocusCal recipe (same 41 features + loss), retrain the classifier on 1-2 of your own four datasets and hold out the rest for testing — this matches the paper's own "generalization" experimental logic, rather than just reusing someone else's trained weights. Whether this transfer succeeds or fails is itself a reportable finding.

**(3) The four datasets differ in answer format, and each needs a separate compatibility check against the debate protocol**

| Dataset | Answer Format | Compatibility with the iMAD Protocol |
|---|---|---|
| GSM8K | Open-ended numeric answer, no fixed options | The paper already includes GSM8K; it handles open-ended answers via the `OPTIONS2` mechanism (collecting answers actually proposed during debate as the candidate set), which can be reused directly |
| CommonsenseQA | 5-choice MCQ | Natively compatible; the easiest to adopt as-is |
| MMLU-Pro | 10-choice MCQ (harder than MMLU, with stronger distractors) | Natively compatible, but with a larger option set the Debater/Judge candidate space grows and the "switch answer" space widens, potentially making debates harder to converge (more likely to hit the 5-round cap) — worth watching for a token/round-count increase |
| CS1QA | Programming-course QA, answers tend toward natural-language/code explanations rather than plain MCQ or a number | Highest compatibility risk; first confirm whether the actual task in this dataset is classification-style QA or open-ended code explanation. If open-ended, follow the GSM8K-style `OPTIONS2` approach to construct a custom candidate set rather than assuming native fit |

### 3.3 Suggested Experimental Design

1. Fix a single backbone LLM and run MMAD-ToM (always-on), iMAD-gated MMAD-ToM, and (if it exists) MMAD-ToM's own gating on all four datasets, comparing Acc / #Tokens / ApT.
2. Use GSM8K as a sanity check first, since it overlaps with the original paper's benchmarks — confirm your reproduced iMAD pipeline (self-critique prompt + 41 features + FocusCal) matches the paper's trends at least in order of magnitude before extending to the other three datasets.
3. Do a small-scale format-adaptation pilot on CS1QA alone (a few dozen examples) to confirm how the candidate set is constructed and how the Judge's JSON schema is defined, before running the full dataset — this avoids having one incompatible format invalidate all four results.
4. If the classifier needs retraining, clearly specify the train/test split (e.g., train on CommonsenseQA + MMLU-Pro, hold out GSM8K + CS1QA, or vice versa), preserving the paper's own "train on two, test on the rest" narrative so the results remain comparable to the original.

### 3.4 One-Sentence Summary

This experiment is worth running, but its value is not in "proving MMAD-ToM beats iMAD" (they operate at different tiers, so a direct comparison is meaningless) — it's in **testing whether iMAD's gating idea transfers to your architecture and to new reasoning-style benchmarks**. A successful transfer means selective triggering is a general technique you can borrow directly; a failed transfer (especially classifier generalization failure) is itself a boundary finding worth reporting.
