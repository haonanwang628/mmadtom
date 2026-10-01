# MMAD-ToM · GSM8K（Prompt V2，主流程）

本目录只包含 MMAD-ToM 主流程，即 S0 → R1 → T1 → R3 → T2 → S4，不含对照组和消融条件。
设计依据：`../prompt/v1/MMAD-ToM_experiment_design.md`（流程、决策 D-a 到 D-d、指标定义）和 `../prompt/v2/V1_to_V2_changes.md`（V2 的 prompt 升级）。

## 文件

| 文件 | 作用 |
|---|---|
| `mmadtom_prompts.py` | 全部 prompt 模板、构造函数、解析器和 guidance 泄露检查（GSM8K，`PROMPT_VERSION = "v2-gsm8k"`） |
| `MMAD-ToM_prompt_V2_GSM8K.md` | 给人阅读的 prompt 文档，由 `export_prompt_md.py` 自动导出，**不要手改** |
| `run_mmadtom.py` | 运行流程：通过 OpenAI 兼容 API（vLLM）按阶段批量生成；每题一个 JSON 记录，支持断点续跑 |
| `analyze.py` | 分析：读取一个或多个 run 目录，输出 `summary.json`、`report.md`、`states.csv`、`transitions.csv` |
| `mock_backend.py` | 离线的假模型后端，用于不接 GPU 时检查流程能否跑通（输出的数字没有实验意义） |
| `test_smoke.py` | 离线测试，共 34 项：与 prompt V2 的一致性、解析器、信息隔离、mock 端到端运行、断点续跑和分析结果的自洽性 |
| `endpoints.example.json` | 模型服务地址的配置模板 |

## 相对 `prompt/v2` 的唯一改动

S2 和 S4 额外显示学生**自己**在上一轮 peer 轮之前的状态：S2 显示 S0，S4 显示 S2。

原因：teacher 的 guidance 策略会要求学生"回到你之前的推理"（例如诊断为 PEER_CONFORMITY 时）。但学生模型在两次调用之间没有记忆，V2 原版的 S2 / S4 只给了当前状态，学生无法照做。新增的只是学生自己的内容，不违反 D-b（S2 / S4 不看同伴）。泄露检查中"学生能看到的数字"也相应包括了这部分内容。

除此之外，所有 prompt 都与 `prompt/v2/prompts_v2.py` 在 GSM8K 下的输出逐字相同，由 `test_smoke.py` 检查。

## 使用

### 0. 离线自检（不需要 GPU）

```bash
python test_smoke.py                                                   # 应输出 34/34 checks passed
python run_mmadtom.py --backend mock --limit 20 --out-dir runs/mock    # 跑一遍完整流程
python analyze.py runs/mock
```

### 1. 启动模型服务（vLLM，示例）

```bash
vllm serve mistralai/Mistral-7B-Instruct-v0.3 --port 8001 --max-model-len 16384
vllm serve microsoft/Phi-4-mini-instruct      --port 8002 --max-model-len 16384
vllm serve Qwen/Qwen2.5-7B-Instruct           --port 8003 --max-model-len 16384
vllm serve google/gemma-3-27b-it              --port 8004 --max-model-len 32768   # 需先在 HF 接受 license；约 55GB
vllm serve meta-llama/Llama-3.1-8B-Instruct   --port 8005 --max-model-len 8192    # judge，可选
```

把 `endpoints.example.json` 复制为 `endpoints.json`，填入实际地址。

- `judge` 可以省略：省略后，FINAL 无法解析的输出记为 `unscored`，并按答错计入准确率，但**不会**被算作漂移。
- 如果服务需要 API key，放在环境变量 `MODEL_API_KEY` 中。

### 2. 运行

```bash
# pilot：前 100 题，seed 0
python run_mmadtom.py --endpoints endpoints.json --limit 100 --seed 0 --out-dir runs/pilot_seed0

# 正式实验：全部 1319 题 × 3 个 seed（每个 seed 用单独的目录）
for s in 0 1 2; do
  python run_mmadtom.py --endpoints endpoints.json --seed $s --out-dir runs/full_seed$s
done
```

- 默认数据：`../../single-agent/data/gsm8k/test.json`，可用 `--data` 指定。题号为 `gsm8k-test-<index>`。
- 执行顺序：先对所有题跑完 S0，再统一跑 S1，以此类推，依次是 S0 → S1 → T1 → S2 → S3 → T2 → S4 → 评分。
- **断点续跑**：中断后用相同的参数重新运行即可，已完成的 (题目, 阶段, 学生) 会自动跳过；出错的单元会在下次运行时重试。
- 一个输出目录只能对应一个 prompt 版本和一个 seed，参数不一致时脚本会拒绝运行。
- `--stop-after S0` 可以只跑到某个阶段，例如先单独检查 S0 的解析率。
- 每个请求都带一个由 (seed, 题号, 阶段, 学生, 尝试次数) 确定的 `seed`，在 vLLM 下可以复现。

### 3. 分析

```bash
python analyze.py runs/full_seed0 runs/full_seed1 runs/full_seed2 --out analysis/full
```

## 生成与重试规则

| 环节 | 规则 |
|---|---|
| 学生 | T=0.3，top_p=0.9，max_tokens=1024。FINAL 无法解析时重新采样 1 次（`--format-retries`），失败的尝试会保留在记录中 |
| Teacher | Gemma-3-27B-IT，T=0.3，max_tokens=1400。guidance 未通过 `leakage_check()`（泄露答案、出现学生看不到的数字、评判对错、点名同伴、内容为空）时重新生成，最多 2 次；仍不通过就改用兜底 guidance |
| 评分 | 规则优先：数字规范化后与 `#### gold` 比较。FINAL 无法解析时调用 judge 抽取答案；另外 5% 规则评过的样本让 judge 复核（`--judge-audit-rate`）。评分结果不会回到辩论中 |

## 记录格式（`records/<qid>.json`）

```text
qid, index, question, gold, prompt_version, seed, models, complete
stages.S0..S4.{M,P,Q}: answer, confidence, reasoning, decision, sections{...}, parse_status, raw,
                       finish_reason, usage, format_retries, failed_attempts,
                       eval_answer, correct, score_source (rule | judge_extract | unscored), [judge, judge_audit]
peer_views.S1/S3.{M,P,Q}: [[显示标签, 真实学生], ...]   # 学生实际看到的同伴及其顺序
teacher.T1/T2.{M,P,Q}: teacher_answer, teacher_correct, sections{...}, guidance, guidance_source,
                       regenerations, attempts[{raw, leakage, ...}]
errors: [{stage, agent, error}]
```

## 分析输出（`report.md` / `summary.json`）

所有比例都附带 k/n 和 Wilson 95% 置信区间。

1. **准确率**：每个阶段、每个学生、三人合并，以及多数投票。多个 seed 时另给出各 seed 之间的均值和标准差。
2. **转变矩阵**：相邻阶段之间的 C→C / C→I / I→C / I→I（以及 I→I 中改过答案的数量）、无法评估的数量、改答案率。
3. **漂移与恢复**：
   - peer 轮（R1、R3）：
     - 谄媚漂移：C→I，且新答案是学生本轮看到的某个同伴答案；
     - 置信度驱动的谄媚漂移：属于谄媚漂移，且被采纳的同伴置信度高于自己；
     - 自发漂移：C→I，新答案没有同伴给过；
     - 健康说服：I→C，新答案来自同伴；
     - 独立恢复：I→C，新答案没有来源；
     - "暴露条件下的谄媚漂移率"：分母只算自己答对、且至少看到一个答错的同伴的学生。
   - teacher 轮（R2、R4）：恢复率、漂移率、对 teacher 的谄媚（C→I 且新答案等于 teacher 的答案），都按 teacher 答对 / 答错分组。
4. **持续性**：
   - R2 恢复后在 R3 再次漂移的比例；
   - R2 恢复后在 S4 仍然正确的比例；
   - S2 答对的学生在 R3 后仍然答对的比例；
   - 顽固率：S3、S4 都答错，并且 T2 标出了未解决的误解；
   - S0 → S4 的整体转变。
5. **学生 ToM 质量**：
   - 自相矛盾的修改：所有同伴都标为 `exposes_flaw_in_mine: NO`，却改了答案；
   - 标记为 YES 的 precision 和 recall；
   - 误报率：自己答对、同伴答错时，却标了 YES。
6. **置信度**：答对和答错时的平均置信度、ECE；答案没变但置信度下降 ≥20 的比例。
7. **Teacher**：
   - T1 / T2 的准确率；
   - guidance 第一次生成就未通过检查的比例，以及各类原因的分布；
   - 重新生成和兜底的比例；
   - guidance 超过 4 句的比例；
   - T1 诊断与规则标签的对照表，以及"是否检测到改动"的一致率；
   - T1 的 evidence 分布；
   - T2 方向标签与按 gold 计算的方向的一致率，按 teacher 答对 / 答错分组。
8. **质量控制**：各模型的解析状态和格式重试率、被截断（finish_reason=length）的比例、评分来源分布、DECISION 与实际改动不一致的比例、judge 复核的一致率。

`transitions.csv` 每行是一个学生在一对相邻阶段上的转变，`states.csv` 每行是一个学生在一个阶段的状态，可以直接用于回归或混合效应模型。
