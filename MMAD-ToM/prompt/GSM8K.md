"""Export the V2 prompt templates for one task to Markdown.

Task slots (answer format, solution hint, leakage rule, ...) are filled from TASKS;
per-item runtime variables stay as {placeholders}. Regenerate after any prompt edit:

    python export_prompt_md.py            # -> MMAD-ToM_prompt_V2_GSM8K.md
"""

import sys
from pathlib import Path

import prompts_v2 as P


class _Keep(dict):
    def __missing__(self, key):
        return "{" + key + "}"


def fill(template: str, **slots) -> str:
    return template.format_map(_Keep(**slots))


def block(text: str) -> str:
    return "```text\n" + text.strip("\n") + "\n```\n"


def main(task: str = "gsm8k") -> Path:
    cfg = P.TASKS[task]
    common = {
        "question_block": "Question ID: {qid}\nQuestion: {question}",
        "final_line": P.final_line(task),
        "answer_rules": P.answer_rules(task),
        "solution_hint": cfg["solution_hint"],
        "answer_placeholder": cfg["answer_placeholder"],
    }
    skeleton = "".join(P.PEER_MODEL_FIELDS.format(label=l) for l in P.PEER_LABELS)

    md = []
    w = md.append
    w(f"# MMAD-ToM Prompt V2 — {task.upper()}\n")
    w(f"> 由 `export_prompt_md.py` 从 `prompts_v2.py` 自动导出（`PROMPT_VERSION = \"{P.PROMPT_VERSION}\"`），"
      "请勿手改；改 prompt 后重新运行导出脚本。\n"
      "> 任务相关的部分已经填入 GSM8K 的取值；`{花括号}` 里的是运行时按题目和学生状态填入的变量。"
      "填好后的完整示例见 `rendered_example_v2.md`。\n")

    w("## 0. 总览\n")
    w("| 阶段 | 调用者 | System | User 模板 | 学生能看到的内容 | 输出段落 |\n|---|---|---|---|---|---|\n"
      "| S0 | 学生 ×3 | Student system | S0 | 题目 | SOLUTION → FINAL |\n"
      "| R1 → S1 | 学生 ×3 | Student system | Peer-ToM（第 1 轮） | 题目、自己的 S0、两个匿名同伴的 S0 | SELF_MODEL → PEER_MODELS → SELF_CHECK → DECISION → SOLUTION → FINAL |\n"
      "| R2 | Teacher ×3（每个学生单独一次） | Teacher system | T1 | — | QUESTION_INTENT → TEACHER_SOLUTION → TEACHER_ANSWER → STUDENT_BELIEF_STATE → PEER_INFLUENCE → CHANGE_DIAGNOSIS → MISCONCEPTION → GUIDANCE |\n"
      "| R2 → S2 | 学生 ×3 | Student system | Tutor-guided（S2） | 题目、自己的 S1、T1 的 GUIDANCE（**看不到同伴**） | SELF_MODEL → GUIDANCE_READING → RECHECK → DECISION → SOLUTION → FINAL |\n"
      "| R3 → S3 | 学生 ×3 | Student system | Peer-ToM（第 2 轮） | 题目、自己的 S2、两个匿名同伴的 S2 | 同 S1 |\n"
      "| R4 | Teacher ×3 | Teacher system | T2 | — | TEACHER_SOLUTION → TEACHER_ANSWER → TRAJECTORY → TRANSITIONS → GUIDANCE_UPTAKE → UNRESOLVED_MISCONCEPTIONS → FINAL_GUIDANCE |\n"
      "| R4 → S4 | 学生 ×3 | Student system | Tutor-guided（S4） | 题目、自己 S0–S3 的答案轨迹、自己的 S3、T2 的 FINAL_GUIDANCE（**看不到同伴**） | 同 S2 |\n"
      "| 评分 | Judge | Judge system | Judge | 单条输出，不知道是哪个模型、哪个阶段 | JSON |\n")
    w("\n只有 teacher 的 `GUIDANCE` / `FINAL_GUIDANCE` 会转给学生，而且要先通过 `leakage_check()`："
      "检查是否泄露答案、是否出现新数字、是否评判对错、是否转述同伴。"
      "不通过就重新生成，最多 2 次；仍不通过就改用通用 guidance。\n")
    w(f"\n生成参数：学生 `{P.STUDENT_GEN}`；teacher `{P.TEACHER_MODEL_ID}` `{P.TEACHER_GEN}`；"
      f"judge `{P.JUDGE_MODEL_ID}` `{P.JUDGE_GEN}`。"
      "对于 Gemma-3 和 Mistral，system prompt 会拼接到 user turn 前面。\n")

    w("\n## 1. Student system（所有学生阶段共用）\n")
    w(block(P.student_system(task)))

    w("\n## 2. S0 — 独立作答\n")
    w(block(fill(P.STUDENT_S0_USER, **common)))

    w("\n## 3. R1 / R3 — Peer-ToM 轮（生成 S1 / S3）\n")
    w("`{peer_round}` 为 1 或 2。`{peer_block}` 是匿名同伴的内容，每个同伴按以下格式给出，"
      "顺序按 (seed, 题目, 阶段, 学生) 随机打乱：\n")
    w(block("--- Student A ---\nAnswer: {answer}\nConfidence: {confidence}\nReasoning:\n{reasoning}"))
    w("`{since_note}`：第 1 轮为空；第 2 轮填入：\n")
    w(block(P.SINCE_TUTOR_NOTE))
    w("User 模板：\n")
    w(block(fill(P.STUDENT_PEER_TOM_USER, peer_model_skeleton=skeleton, **common)))

    w("\n## 4. R2 / R4 — Tutor-guided 轮（生成 S2 / S4，看不到同伴）\n")
    w(f"- `{{stage_title}}`：S2 为 `{P.STAGE_TITLE_S2}`，S4 为 `{P.STAGE_TITLE_S4}`\n"
      "- `{trajectory_block}`：S2 为空；S4 填入 `=== Your answers so far ===` 加上 "
      "`S0: a (conf x) -> S1: … -> S3: …`\n"
      "- `{guidance_instruction}`：主条件填入下面这句（D-c）：\n")
    w(block(P.GUIDANCE_INSTRUCTION_MAIN))
    w("User 模板（主条件）：\n")
    w(block(fill(P.STUDENT_TEACHER_GUIDED_USER, guidance_instruction=P.GUIDANCE_INSTRUCTION_MAIN, **common)))

    w("\n## 5. Teacher system（T1 和 T2 共用）\n")
    w(block(P.teacher_system(task)))

    w("\n## 6. R2 — Teacher T1：建模学生的信念状态\n")
    w("`{peer_block}` 是这个学生在 R1 中看到的同伴内容，标签和顺序与学生看到的一致。"
      "带 `s1_` 前缀的字段取自学生 S1 输出中对应的段落。\n")
    w(block(fill(P.TEACHER_T1_USER, **common)))

    w("\n## 7. R4 — Teacher T2：分析完整轨迹\n")
    w("带 `t1_` 前缀的字段取自 teacher 自己在 T1 的私有输出。"
      "`{peers_before_s1}` 和 `{peers_before_s3}` 是这个学生在两轮 peer 轮中分别看到的同伴内容。\n")
    w(block(fill(P.TEACHER_T2_USER, **common)))

    w("\n## 8. Judge（独立评分，不参与辩论）\n")
    w("主路径是规则评分：解析 FINAL 行，把数字规范化后和 `#### gold` 比较。"
      "只有在 `parse_status == \"unparsed\"` 时，或在 5% 的抽样复核中，才调用 judge 模型。\n")
    w("System：\n")
    w(block(P.JUDGE_SYSTEM.format(judge_answer_desc=cfg["judge_answer_desc"])))
    w("User：\n")
    w(block(fill(P.JUDGE_USER, question_block="Question: {question}",
                 judge_answer_desc=cfg["judge_answer_desc"],
                 judge_equivalence=cfg["judge_equivalence"]).replace("{{", "{").replace("}}", "}")))

    w("\n## 9. 对照组与消融条件中被替换的部分\n")
    w("所有条件的 system prompt 都与主条件相同，只替换下面列出的部分。\n")
    w("\n### C0 Self-only：所有 4 轮都换成自我复查（不看同伴，也没有 teacher）\n")
    w(block(fill(P.STUDENT_SELF_REFLECT_USER, **common)))
    w("\n### C1 Peer-neutral：R1 / R3 换成不带 ToM 结构的 peer 轮\n")
    w(block(fill(P.STUDENT_PEER_NEUTRAL_USER, **common)))
    w("\n### C2 Peer-ToM：R1 / R3 与主条件相同，R2 / R4 换成 C0 的自我复查\n")
    w("\n### C4 Generic teacher：R2 / R4 的 guidance 换成下面这段固定文本\n")
    w(block(P.generic_guidance(task)))
    w("\n### C5 No-confidence：R1 / R3 的 `{peer_block}` 去掉 Confidence 行\n")
    w("\n### C6 Fallible-tutor（D-c 消融）：`{guidance_instruction}` 换成\n")
    w(block(P.GUIDANCE_INSTRUCTION_FALLIBLE))

    w("\n## 10. 诊断与转变标签（供分析使用）\n")
    w(f"- CHANGE_DIAGNOSIS（T1）：{' / '.join(P.CHANGE_DIAGNOSIS_LABELS)}\n"
      f"- TRANSITIONS 中的 cause（T2）：{' / '.join(P.TRANSITION_CAUSE_LABELS)}\n"
      "- TRANSITIONS 中的方向（T2）：STABLE / DRIFT / RECOVERY / LATERAL\n"
      f"- evidence：{' / '.join(P.EVIDENCE_SOURCES)}（T2 中还可以是 NONE）\n"
      "- 解析函数：`parse_student`、`parse_teacher`、`parse_peer_models`、`parse_change_diagnosis`、`parse_transitions`\n")

    out = Path(__file__).with_name(f"MMAD-ToM_prompt_V2_{task.upper()}.md")
    out.write_text("\n".join(md))
    return out


if __name__ == "__main__":
    print(f"wrote {main(*(sys.argv[1:2] or ['gsm8k']))}")
