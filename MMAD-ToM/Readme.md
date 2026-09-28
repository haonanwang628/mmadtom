# MMAD-ToM Prompt Architecture and Usage Guide
## 1. Purpose 
 This guide explains how the seven prompt templates in the revised MMAD-ToM design work together. Three student agents (M, P, Q) answer independently in Phase 0 and then complete four synchronous debate rounds. Rounds 1 and 3 emphasize peer-level Theory of Mind (ToM). Before students answer in Rounds 2 and 4, the teacher diagnoses each student separately and sends targeted guidance for that same round. Peers' stated confidence is visible in every debate round so its relationship with answer revision can be studied.



**Design versus current script |** This document describes the **intended revised design**. The supplied older Python script calls the teacher *after* Rounds 2 and 4, so its control flow must be changed before it implements this guide. 

## 2. Prompt Files and Roles 

| Template | Sent to  | Purpose |
|---|---|---|
| Phase 0 Student System Prompt | M, P, Q in Phase 0 | Defines independent reasoning and JSON output; no prior answer, peers, or teacher.  |
| Phase 0 User Prompt | M, P, Q in Phase 0 | Supplies the current question and question ID.  |
| Debate Student System Prompt | M, P, Q in Rounds 1–4 | Defines how to compare own and peer reasoning, treat confidence separately, and report revision. / 
| Peer Round User Prompt — Rounds 1 and 3 | M, P, Q in Rounds 1, 3 | Supplies the question and previous-phase student states, without tutor guidance. 
| Teacher System Prompt — One Target Student per Call | Teacher before Rounds 2, 4 | Defines one-student diagnosis, uncertainty, and non-answer-revealing guidance. 
| Teacher User Prompt — One Target Student | Teacher before Rounds 2, 4 | Supplies the question, target history, actual peer exposure, and prior tutor guidance if any. 
| Teacher Round User Prompt — Rounds 2 and 4 | M, P, Q in Rounds 2, 4 | Supplies previous-phase states and the target student's newly generated guidance. 

**English.** “Teacher User Prompt” goes **to the teacher** to generate guidance. “Teacher Round User Prompt” goes **to a student** after guidance has been generated. A Markdown file in a repository is only a template: the experiment script must read it, replace its variables, and pass the resulting text to the model.



## 3. One-Question Execution Schedule 

| Stage | Call order  | Available history |
|---|---|---|
| Phase 0 | Call M, P, Q independently; append all initial outputs.  | Question only 
| Round 1 | Build each peer prompt from Phase 0; generate all three outputs; append together. | Phase 0 |
| Before Round 2 | Call teacher once for M, once for P, once for Q. / | Through Round 1 /  Round 1 |
| Round 2 | Give each student Round 1 states plus **its own** fresh guidance; append outputs together.  | Round 1 + current guidance /|
| Round 3 | Use Round 2 states in peer prompts; do not inject Round 2 guidance again. / | Round 2 |
| Before Round 4 | Call teacher separately for M, P, Q again. | Through Round 3, including prior guidance record  |
| Round 4 | Give each student Round 3 states plus **new** guidance; append final outputs together. | Round 3 + current guidance  |

**English.** Each student always reads a snapshot of the *previous phase*. A sequential implementation may call M, P, and Q one at a time, but it must hold their outputs in `round_outputs` until all three have finished. For one question, this schedule makes 15 student calls and 6 teacher calls, excluding any evaluation judge.



## 4. Inputs, Outputs, and Routing

### Student call 

**English.** Pass the matching system prompt and user prompt to the **same student model**. In a debate round, fill the user template with the question; that student's immediately previous answer, reasoning, and confidence; and the other two students' corresponding previous-phase answers, reasoning, and confidence. Teacher rounds additionally include only that student's own guidance. Parse `question_id`, `predicted_output`, `reasoning`, `confidence` (integer 0–100), and `revised` (boolean). The program should verify `revised` by comparing normalized current and immediately previous answers; record parsing failures separately.



### Teacher call

**English.** Make **one call per target student**, not one combined call for all three. The teacher receives the question; the target student's history available before the current even round; the peer responses that this target actually saw before its earlier responses; and any earlier guidance delivered to this target. It receives no gold answer. Parse and save `INDEPENDENT_ANSWER`, `INDEPENDENT_REASONING`, `QUESTION_INTENT`, `MISCONCEPTION_HYPOTHESIS`, `AGENT_UNDERSTANDING`, and `GUIDANCE`. Route the last three fields as a separate Tutor Guidance section to the target student **in the current Round 2 or 4**. Store the full raw and parsed teacher output for audit; do not route another student's diagnosis.



**Interpretation |：** The teacher's “independent” answer is its provisional assessment. If the question and student history are in the same model call, output order alone cannot guarantee that the answer was formed without exposure to student responses. Diagnoses of confidence-driven deference are hypotheses; observing an answer change after a high-confidence peer does not establish causation. 

## 5. Template Variables 

| Variable | Meaning |
|---|---|
| `{task_type_guidance}`, `{answer_format}` | Dataset-specific task description and required final-answer format. 
| `{question_block}` | Question ID, problem text, and options or code where applicable. 
| `{round_num}` | Current round number (1–4).
| `{own_previous_answer}`, `{own_previous_reasoning}`, `{own_previous_confidence}` | Target student's immediately previous state.
| `{peer_1_*}`, `{peer_2_*}` | Other students' immediately previous labels, model names, answers, reasoning, and confidence.
| `{target_label}` | The one student being diagnosed in a teacher call. 
| `{target_student_history_through_previous_round}` | Target history ending at Round 1 before Round 2, or Round 3 before Round 4. 
| `{peer_exposure_history_for_target}` | For each completed round, the peer information shown to the target *before* it answered. 
| `{prior_tutor_guidance_or_none}` | Round 2 guidance when preparing Round 4; otherwise none. 
| `{teacher_guidance_for_this_agent}` | Newly produced guidance for this target in the current teacher round. 

**Implementation note** If Python `str.format()` is used, literal JSON braces in system prompts must be escaped as `{{` and `}}`, or a template renderer that distinguishes literals from variables must be used.
## 6. Generation and Evaluation

The target shared generation settings are `temperature=0.3`, `top_p=0.9`, and `max_tokens=2048` for student and teacher calls. The older script defaults to temperature 0.7; check the effective run configuration. Store five student outputs per agent (Phase 0 plus four rounds), six teacher outputs per question, and the exact inputs or reconstructed exposure needed to audit answer changes. The evaluation judge is separate from the debate and should never feed gold labels or correctness judgments back to the agents. CS1QA needs its own correctness judge before drift and recovery can be computed for that dataset.

