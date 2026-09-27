# MMAD-ToM Prompt Architecture and Usage Guide
# MMAD-ToM 提示词架构与使用说明

## 1. Purpose | 目的

**English.** This guide explains how the seven prompt templates in the revised MMAD-ToM design work together. Three student agents (M, P, Q) answer independently in Phase 0 and then complete four synchronous debate rounds. Rounds 1 and 3 emphasize peer-level Theory of Mind (ToM). Before students answer in Rounds 2 and 4, the teacher diagnoses each student separately and sends targeted guidance for that same round. Peers' stated confidence is visible in every debate round so its relationship with answer revision can be studied.

**中文。** 本说明介绍修订版 MMAD-ToM 中七个提示词模板的配合方式。三名学生智能体（M、P、Q）在 Phase 0 独立作答，随后进行四轮同步辩论。Round 1 和 3 侧重同伴层面的心智理论（ToM）分析。Round 2 和 4 的学生作答之前，教师分别诊断每名学生，并将定向指导送入当轮提示词。每一轮辩论都展示同伴报告的置信度，以研究置信度与答案修订的关系。

**Design versus current script | 设计与现有代码：** This document describes the **intended revised design**. The supplied older Python script calls the teacher *after* Rounds 2 and 4, so its control flow must be changed before it implements this guide. / 本文描述的是**目标设计**。此前的 Python 脚本在 Round 2、4 *结束后*调用教师，必须先修改调用位置，才能实现本文流程。

## 2. Prompt Files and Roles | 提示词文件与职责

| Template / 模板 | Sent to / 发送对象 | Purpose / 用途 |
|---|---|---|
| Phase 0 Student System Prompt | M, P, Q in Phase 0 | Defines independent reasoning and JSON output; no prior answer, peers, or teacher. / 规定独立推理及 JSON 输出，不涉及历史、同伴或教师。 |
| Phase 0 User Prompt | M, P, Q in Phase 0 | Supplies the current question and question ID. / 提供当前题目及题目 ID。 |
| Debate Student System Prompt | M, P, Q in Rounds 1–4 | Defines how to compare own and peer reasoning, treat confidence separately, and report revision. / 规定如何比较自身与同伴推理、单独看待置信度并报告改答。 |
| Peer Round User Prompt — Rounds 1 and 3 | M, P, Q in Rounds 1, 3 | Supplies the question and previous-phase student states, without tutor guidance. / 提供题目与上一阶段状态，不含教师指导。 |
| Teacher System Prompt — One Target Student per Call | Teacher before Rounds 2, 4 | Defines one-student diagnosis, uncertainty, and non-answer-revealing guidance. / 规定单学生诊断、处理不确定性及不泄露答案的指导。 |
| Teacher User Prompt — One Target Student | Teacher before Rounds 2, 4 | Supplies the question, target history, actual peer exposure, and prior tutor guidance if any. / 提供题目、目标学生历史、实际接触的同伴信息及已有教师指导。 |
| Teacher Round User Prompt — Rounds 2 and 4 | M, P, Q in Rounds 2, 4 | Supplies previous-phase states and the target student's newly generated guidance. / 提供上一阶段状态和刚生成的该学生专属指导。 |

**English.** “Teacher User Prompt” goes **to the teacher** to generate guidance. “Teacher Round User Prompt” goes **to a student** after guidance has been generated. A Markdown file in a repository is only a template: the experiment script must read it, replace its variables, and pass the resulting text to the model.

**中文。** “Teacher User Prompt” 是**发给教师**用于生成指导的；“Teacher Round User Prompt” 是指导生成后**发给学生**的。仓库中的 Markdown 文件只是模板：实验脚本必须读取文件、替换变量，再将生成的文本传给模型。

## 3. One-Question Execution Schedule | 单题执行顺序

| Stage / 阶段 | Call order / 调用顺序 | Available history / 可用历史 |
|---|---|---|
| Phase 0 | Call M, P, Q independently; append all initial outputs. / 三名学生独立作答并记录。 | Question only / 仅题目 |
| Round 1 | Build each peer prompt from Phase 0; generate all three outputs; append together. / 从 Phase 0 构造同伴提示，三人全部生成后统一追加。 | Phase 0 |
| Before Round 2 | Call teacher once for M, once for P, once for Q. / 教师分别诊断 M、P、Q。 | Through Round 1 / 截至 Round 1 |
| Round 2 | Give each student Round 1 states plus **its own** fresh guidance; append outputs together. / 每人读取 Round 1 状态及**自己**的指导，统一追加。 | Round 1 + current guidance / 当轮指导 |
| Round 3 | Use Round 2 states in peer prompts; do not inject Round 2 guidance again. / 使用 Round 2 状态，不重复注入旧指导。 | Round 2 |
| Before Round 4 | Call teacher separately for M, P, Q again. / 教师再次分别诊断三人。 | Through Round 3, including prior guidance record / 截至 Round 3，含此前教师指导记录 |
| Round 4 | Give each student Round 3 states plus **new** guidance; append final outputs together. / 使用 Round 3 状态及**新**指导，统一追加最终输出。 | Round 3 + current guidance / 当轮指导 |

**English.** Each student always reads a snapshot of the *previous phase*. A sequential implementation may call M, P, and Q one at a time, but it must hold their outputs in `round_outputs` until all three have finished. For one question, this schedule makes 15 student calls and 6 teacher calls, excluding any evaluation judge.

**中文。** 学生始终读取*上一阶段*的统一快照。程序可以依次调用 M、P、Q，但必须先将当轮输出暂存在 `round_outputs`，三人全部完成后才写入历史。每道题共需 15 次学生调用和 6 次教师调用；另行评估的 judge 不计在内。

## 4. Inputs, Outputs, and Routing | 输入、输出与路由

### Student call | 学生调用

**English.** Pass the matching system prompt and user prompt to the **same student model**. In a debate round, fill the user template with the question; that student's immediately previous answer, reasoning, and confidence; and the other two students' corresponding previous-phase answers, reasoning, and confidence. Teacher rounds additionally include only that student's own guidance. Parse `question_id`, `predicted_output`, `reasoning`, `confidence` (integer 0–100), and `revised` (boolean). The program should verify `revised` by comparing normalized current and immediately previous answers; record parsing failures separately.

**中文。** 将对应的 system prompt 和 user prompt 一起发给**同一名学生模型**。辩论轮的 user 模板要填入题目、该学生上一阶段的答案/推理/置信度，以及另外两名学生上一阶段的对应内容。教师轮额外加入且只加入该学生自己的指导。解析 `question_id`、`predicted_output`、`reasoning`、`confidence`（0–100 整数）和 `revised`（布尔值）。程序还应比较当前与紧邻上一阶段的规范化答案，核实 `revised`，并单独记录解析失败。

### Teacher call | 教师调用

**English.** Make **one call per target student**, not one combined call for all three. The teacher receives the question; the target student's history available before the current even round; the peer responses that this target actually saw before its earlier responses; and any earlier guidance delivered to this target. It receives no gold answer. Parse and save `INDEPENDENT_ANSWER`, `INDEPENDENT_REASONING`, `QUESTION_INTENT`, `MISCONCEPTION_HYPOTHESIS`, `AGENT_UNDERSTANDING`, and `GUIDANCE`. Route the last three fields as a separate Tutor Guidance section to the target student **in the current Round 2 or 4**. Store the full raw and parsed teacher output for audit; do not route another student's diagnosis.

**中文。** 教师**对每名目标学生调用一次**，而不是一次同时诊断三人。输入包括题目、偶数轮开始前该学生的历史、该学生在此前作答时实际看到的同伴内容，以及曾经收到的教师指导；不提供 gold answer。解析并保存 `INDEPENDENT_ANSWER`、`INDEPENDENT_REASONING`、`QUESTION_INTENT`、`MISCONCEPTION_HYPOTHESIS`、`AGENT_UNDERSTANDING` 和 `GUIDANCE`。将后三项组成独立的 Tutor Guidance 区块，发送给目标学生的**当前 Round 2 或 4**；保存教师原始及解析后的完整输出供审计，不传递其他学生的诊断。

**Interpretation | 解释边界：** The teacher's “independent” answer is its provisional assessment. If the question and student history are in the same model call, output order alone cannot guarantee that the answer was formed without exposure to student responses. Diagnoses of confidence-driven deference are hypotheses; observing an answer change after a high-confidence peer does not establish causation. / 教师的“独立答案”是暂定判断。若题目和学生历史放在同一次调用中，先输出答案并不能保证它未受到学生答案影响。“置信度驱动顺从”只是诊断假设，看到高置信度同伴后改答本身不能证明因果关系。

## 5. Template Variables | 模板变量

| Variable / 变量 | Meaning / 含义 |
|---|---|
| `{task_type_guidance}`, `{answer_format}` | Dataset-specific task description and required final-answer format. / 数据集对应的任务描述和答案格式。 |
| `{question_block}` | Question ID, problem text, and options or code where applicable. / 题目 ID、题干，以及适用时的选项或代码。 |
| `{round_num}` | Current round number (1–4). / 当前轮次（1–4）。 |
| `{own_previous_answer}`, `{own_previous_reasoning}`, `{own_previous_confidence}` | Target student's immediately previous state. / 目标学生紧邻上一阶段的状态。 |
| `{peer_1_*}`, `{peer_2_*}` | Other students' immediately previous labels, model names, answers, reasoning, and confidence. / 另两名学生上一阶段的标签、模型名、答案、推理和置信度。 |
| `{target_label}` | The one student being diagnosed in a teacher call. / 本次教师调用所诊断的一名学生。 |
| `{target_student_history_through_previous_round}` | Target history ending at Round 1 before Round 2, or Round 3 before Round 4. / Round 2 前截至 Round 1；Round 4 前截至 Round 3。 |
| `{peer_exposure_history_for_target}` | For each completed round, the peer information shown to the target *before* it answered. / 各已完成轮次中，该学生答题*之前*看到的同伴信息。 |
| `{prior_tutor_guidance_or_none}` | Round 2 guidance when preparing Round 4; otherwise none. / 准备 Round 4 时填入 Round 2 指导；否则填无。 |
| `{teacher_guidance_for_this_agent}` | Newly produced guidance for this target in the current teacher round. / 当前教师轮刚为该学生生成的指导。 |

**Implementation note | 实现提示：** If Python `str.format()` is used, literal JSON braces in system prompts must be escaped as `{{` and `}}`, or a template renderer that distinguishes literals from variables must be used. / 若用 Python 的 `str.format()`，system prompt 中 JSON 示例的字面花括号必须写为 `{{`、`}}`，也可以使用能区分字面量与变量的模板方法。

## 6. Generation and Evaluation | 生成与评估

**English.** The target shared generation settings are `temperature=0.3`, `top_p=0.9`, and `max_tokens=2048` for student and teacher calls. The older script defaults to temperature 0.7; check the effective run configuration. Store five student outputs per agent (Phase 0 plus four rounds), six teacher outputs per question, and the exact inputs or reconstructed exposure needed to audit answer changes. The evaluation judge is separate from the debate and should never feed gold labels or correctness judgments back to the agents. CS1QA needs its own correctness judge before drift and recovery can be computed for that dataset.

**中文。** 学生与教师调用的目标共享参数为 `temperature=0.3`、`top_p=0.9`、`max_tokens=2048`。旧脚本默认温度为 0.7，正式运行要核对实际配置。每名学生每题保存五条输出（Phase 0 加四轮），每题保存六条教师输出，并保存准确输入或可重建的接触记录，以便审查答案变化。评估 judge 独立于辩论，不能将 gold label 或正确性判断反馈给智能体。CS1QA 需要接入其正确性 judge 后，才能计算该数据集的 drift 与 recovery。
