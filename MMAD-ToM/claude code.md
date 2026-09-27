# 把新版 pipeline Word 文档、双语 .md 使用说明、七个 prompt 文件，以及旧 Python 脚本放进 Claude Code 能访问的同一项目目录。然后发送下面这段。文件名若不同，让 Claude Code 按内容定位。
Please inspect the MMAD-ToM project and implement the revised pipeline, but **do not run the experiment**.

Read these materials before editing code:

1. `MMAD_ToM_Debate_Pipeline_Revised.docx` — the intended framework and round schedule.
2. `MMAD_ToM_Prompt_Guide_Bilingual.md` — how the prompt templates are paired, populated, and routed.
3. The seven Phase 0, debate student, peer round, teacher round, and teacher diagnosis prompt files in this repository — use their actual contents as the model prompts.
4. The existing Python debate script — use it as the implementation starting point.

If you cannot read the DOCX reliably, tell me before implementing the schedule. If the old script conflicts with the revised framework, follow the revised framework and report the discrepancy. Do not silently rewrite the research design or replace the supplied prompt text with your own version.

Implement this per-question sequence:

- Phase 0: M, P, and Q answer independently.
- Round 1: peer-level ToM using the Phase 0 snapshot.
- **Before Round 2 student generation:** call the teacher separately for M, P, and Q using history through Round 1. Each student receives only its own newly generated guidance in Round 2.
- Round 3: peer-level ToM using the Round 2 snapshot, without reinjecting the earlier guidance.
- **Before Round 4 student generation:** call the teacher separately for M, P, and Q using history through Round 3, including the earlier guidance in the target student's exposure record. Each student receives only its own new guidance in Round 4.

In every debate round, show each student its own preceding response and both peers' preceding answers, reasoning, and stated confidence. Build all three student prompts from the same previous-phase snapshot. Collect their current outputs before appending any of them to history.

For each teacher call, provide the question, the target student's available history, and the peer content that target actually saw before earlier responses. Do not provide the gold answer or the other students' unrestricted histories. Save the teacher's raw output, parsed fields, and exact guidance delivered to the target student. Keep gold answers and judge results outside all student and teacher prompts.

Use `temperature=0.3`, `top_p=0.9`, and `max_tokens=2048` for student and teacher generation. Keep the teacher model ID configurable and fail clearly if it is missing; do not choose a teacher model for me. Preserve the existing dataset adapters, per-item records, checkpointing, and resume behavior where possible. Keep parsing failures separate from genuine answer changes, and make CSV export robust to failed items. Do not invent a CS1QA correctness judge; leave an explicit integration point if the existing judge implementation is unavailable.

**Scope and execution limits:** Write the implementation and inspect it statically. A syntax check is allowed. Do not load or download models, run inference, process benchmark items, submit a GPU or Slurm job, or launch a full experiment. Do not report experimental accuracy or claim that model execution has been validated.

When finished, provide:

1. The files changed and the purpose of each change.
2. A concise walkthrough of one question from Phase 0 through Round 4, stating exactly when each teacher call occurs.
3. The prompt file used for each type of model call.
4. What you checked statically and what remains unverified until a model run.
5. Any unresolved inputs required before running the experiment.
