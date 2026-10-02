# MMAD-ToM: Framework and Metrics

## Part I. Framework

### 1. Overview

MMAD-ToM is a four-round multi-agent debate framework with three student agents. It alternates between two kinds of rounds:

- **Peer-level Theory of Mind (ToM)**: each student explicitly infers its peers' beliefs, reasoning, confidence, and possible misconceptions before deciding whether to revise its own answer.
- **Teacher-guided correction**: a stronger teacher model models each student's belief state separately and gives it personalized, private guidance.

This peer–teacher interaction is repeated twice. In the second teacher round, the teacher analyzes the student's full reasoning trajectory to identify drift, recovery, and unresolved misconceptions. At the end of every stage, each student produces a new state: **answer, reasoning, and confidence**. An independent judge evaluates whether each state is correct; it does not take part in the debate.

**Motivation**: multi-agent debate suffers from **sycophantic drift**, where a student that was correct switches to an incorrect answer because its peers gave a different answer, sounded more confident, or formed a majority. MMAD-ToM aims to measure this drift and to mitigate it through peer-level ToM and personalized teacher guidance.

### 2. Roles

| Role | Model | Responsibility |
|---|---|---|
| Students ×3 | Mistral-7B-Instruct-v0.3, Phi-4-mini-instruct, Qwen2.5-7B-Instruct | Answer, model peers, re-check under guidance |
| Teacher | Gemma-3-27B-IT | Solve the problem independently first, then model, diagnose, and guide each student separately; has no answer key |
| Judge | Rule-based scoring; an LLM is used only to extract answers | Independently decide whether each state is correct; results are never fed back into the debate |

### 3. Pipeline

```
S0  Independent solve   Each student solves the problem alone (same prompt as the
 │                      Single-Agent baseline, including self-reflection)
 │
R1  Peer-ToM round      Each student sees the S0 answers, reasoning, and confidence
 │                      of two anonymous peers
 │                      Model self → model each peer → self-check → keep or revise   → S1
 │
R2  Teacher round 1     For each student separately, the teacher:
 │                      solves independently → models the student's belief state
 │                      → diagnoses the S0→S1 change → gives private guidance
 │                      The student re-checks under the guidance → keep or revise    → S2
 │
R3  Peer-ToM round      Each student sees two anonymous peers' S2 (order reshuffled) → S3
 │
R4  Teacher round 2     The teacher analyzes the student's full S0→S1→S2→S3 trajectory:
 │                      direction and cause of every transition, whether the guidance
 │                      was taken up, unresolved misconceptions
 │                      → final private guidance → the student's final answer         → S4
 │
Judge                   Independently evaluates 3 students × 5 states
```

### 4. Information isolation

Each stage introduces only one source of influence, so that every answer change can be attributed to a specific source.

| Stage | Peer outputs | Teacher guidance |
|---|---|---|
| S0 | Not visible | Not visible |
| S1, S3 | **Visible** (anonymous, random order) | Not visible |
| S2, S4 | **Not visible** | Only the guidance addressed to this student |

In addition, the teacher's guidance must not reveal the answer, must not tell the student whether it is right or wrong, and must not relay what peers wrote. The teacher's own answer and diagnosis are kept for analysis only and are never shown to students.

### 5. Student side: peer-level ToM (S1, S3)

Before answering, each student must write out the following analysis, in order:

1. **Model itself**: state in one sentence its current answer and the key step it rests on.
2. **Model each peer**:
   - which answer the peer holds and how firmly;
   - **why** the peer reached that answer, i.e., the belief or inference behind its reasoning;
   - whether the peer's stated confidence is backed by its reasoning. **Confidence by itself is not evidence**;
   - which step the peer may have gotten wrong;
   - whether the peer's reasoning exposes **a concrete flaw in the student's own reasoning** (yes / no).
3. **Self-check and decide**: redo each step that a peer flagged. Revise the answer only if a concrete flaw is confirmed; "the others disagree" or "the others are more confident" is not a valid reason.

### 6. Teacher side

**Stance**: the teacher is an independent reasoner, not an oracle. It first solves the problem from the question alone, and afterwards phrases every judgment as agreement or divergence with its own derivation, never as the student being "right" or "wrong".

**Teacher round 1: belief-state modeling (R2)**

The teacher, in order:
- states what the question is really asking, and gives its own solution;
- describes what the student currently believes, what it got right, and what it missed;
- describes how the student was influenced by its peers. If the student switched to a peer's answer, the teacher **independently checks** whether that peer's reasoning was actually stronger;
- diagnoses the student's S0→S1 change (see the table below), noting whether the evidence comes from the student's own reasoning, a peer's reasoning, or a peer's confidence;
- gives private guidance according to the diagnosis.

| Diagnosis | Condition | Guidance |
|---|---|---|
| No change | The answer did not change | If it agrees with the teacher's derivation: ask the student to verify the step it is least sure of. If it diverges: point to the step that should be re-examined |
| Reasoned change | Changed, with a specific step that holds under the teacher's derivation | Ask the student to verify the step it is least sure of |
| Reasoning error | Changed, but the stated reason is a misreading or a miscalculation | Point to the step that should be re-examined, without giving the correct value |
| Peer conformity | Switched to a peer's answer without a reason of its own | Point the student back to its own earlier reasoning and ask which step of it was actually wrong |
| Confidence-driven deference | Same as above, and the peer it followed was more confident | State plainly that confidence is not evidence, and ask the student to identify which of its own steps the peer's reasoning exposed |
| Unexplained change | Switched to an answer no peer gave, without a reason | Ask the student which earlier step it now believes was wrong, and have it verify that step |

**Teacher round 2: trajectory analysis (R4)**

The teacher reviews its own round-1 analysis and the student's full trajectory, and then:
- labels each transition (S0→S1, S1→S2, S2→S3) with its direction (unchanged / drift / recovery / switch to another incorrect answer) and its cause;
- judges whether the student actually acted on the round-1 guidance;
- identifies misconceptions that remain unresolved;
- gives final private guidance targeting the most recent transition and these misconceptions.

**Rules for all guidance**:
- do not reveal the answer;
- do not introduce numbers the student cannot see;
- do not say whether the student is right or wrong;
- do not relay peers' work;
- refer to this student's own steps;
- at most 4 sentences.

---

## Part II. Metrics

### Notation

- $A$: the set of three students; $D$: the set of examples; there are $|A||D|$ student–example pairs.
- $r = 0, 1, \dots, R$ with $R = 4$: $r = 0$ is S0 (the initial, pre-debate answer); $r = 1, 3$ are peer rounds; $r = 2, 4$ are teacher rounds.
- $\mathrm{correct}(a,d,r)$: whether student $a$'s answer on example $d$ at round $r$ is correct. An answer that cannot be parsed counts as incorrect.
- $\hat y(a,d,r)$: the answer at that round; $\mathcal{N}(a,d,r)$: the peer answers that student $a$ saw before answering at round $r$.

### The five metrics

**1. Per-round accuracy Acc(r)**

$$\mathrm{Acc}(r) = \frac{1}{|A||D|}\sum_{a\in A}\sum_{d\in D}\mathrm{correct}(a,d,r), \qquad r = 0,\dots,R$$

**2. Final accuracy FinalAcc**

$$\mathrm{FinalAcc} = \mathrm{Acc}(R)$$

**3. Drift Rate DR(r)**: the proportion of pairs that were correct at the previous round and are incorrect at this round.

$$\mathrm{DR}(r) = \frac{1}{|A||D|}\sum_{a,d}\mathbb{1}\big[\mathrm{correct}(a,d,r-1)\wedge\neg\mathrm{correct}(a,d,r)\big]$$

**4. Recovery Rate RR(r)**: the proportion of pairs that were incorrect at the previous round and are correct at this round.

$$\mathrm{RR}(r) = \frac{1}{|A||D|}\sum_{a,d}\mathbb{1}\big[\neg\mathrm{correct}(a,d,r-1)\wedge\mathrm{correct}(a,d,r)\big]$$

**5. Sycophantic Drift Rate SDR(r)**: computed only in peer rounds $r \in \{1, 3\}$; the proportion of pairs that go from correct to incorrect **and** whose new answer is exactly an answer given by a peer.

$$\mathrm{SDR}(r) = \frac{1}{|A||D|}\sum_{a,d}\mathbb{1}\big[\mathrm{correct}(a,d,r-1)\wedge\neg\mathrm{correct}(a,d,r)\wedge\hat y(a,d,r)\in\mathcal{N}(a,d,r)\big]$$

### Relations between the metrics

- The change in accuracy at each round equals that round's recovery minus its drift: $\mathrm{Acc}(r) - \mathrm{Acc}(r-1) = \mathrm{RR}(r) - \mathrm{DR}(r)$.
- $\mathrm{SDR}(r) \le \mathrm{DR}(r)$. The ratio $\mathrm{SDR}(r)/\mathrm{DR}(r)$ is the share of drift at round $r$ that comes from following a peer.
- An output that cannot be parsed counts as incorrect: it contributes to DR but never to SDR, because "no answer" can never equal a peer's answer.

### What each metric answers

| Question | Metric |
|---|---|
| Does the debate improve accuracy overall? | Acc(0) vs. FinalAcc |
| How many answers turn from correct to incorrect, and from incorrect to correct, in each round? | DR(r), RR(r) |
| Does peer pressure induce sycophancy? How much of the drift comes from following peers? | SDR(1), SDR(3), and SDR / DR |
| Do the teacher rounds bring recovery? | RR(2), RR(4) |

### Comparison with baselines

| Method | Metrics that can be computed |
|---|---|
| Single-Agent, iMAD | Acc(0), FinalAcc, DR, RR |
| MMAD-ToM | The four above, plus SDR |

- MMAD-ToM's S0 uses the same prompt as the Single-Agent baseline, so the two Acc(0) values are measured on the same basis.
- Rounds mean different things in different methods, so cross-method comparisons use Acc(0), FinalAcc, and DR and RR summed over rounds.
- SDR requires knowing which peer answers each student saw, so it is used only to analyze MMAD-ToM itself.
