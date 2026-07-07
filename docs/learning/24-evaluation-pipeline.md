# 评测 Pipeline

## 学习目标

本文解释 `MediX-R1/eval` 如何评测医学 LLM/VLM 模型，包括生成回答、LLM judge 打分和最终聚合。

## 关键源码

```text
MediX-R1/eval/eval.py
MediX-R1/eval/utils.py
MediX-R1/eval/tasks.yaml
MediX-R1/eval/eval.sh
MediX-R1/eval/submit.sh
```

## 三阶段流程

评测分三阶段：

```text
generate  -> 生成模型回答
evaluate  -> judge 模型评分
score     -> 聚合分数
```

可以一起运行：

```bash
bash eval.sh
```

也可以用 `eval.py` 分阶段运行。

## 任务配置

任务定义在：

```text
tasks.yaml
```

分为：

- `llm`：文本任务。
- `vlm`：视觉语言任务。

LLM 任务包括：

- MMLU medical 系列。
- MedMCQA。
- MedQA。
- USMLE self assessment。
- PubMedQA。
- MIMIC-CXR report summarization。

VLM 任务包括：

- SLAKE VQA。
- VQA-RAD。
- PathVQA。
- PMC-VQA。
- MIMIC-CXR report generation。

## eval.py 参数

主要参数：

| 参数 | 作用 |
|---|---|
| `--tasks` | 选择任务，支持 `all`、`llm`、`vlm` 或具体任务名 |
| `--model` | 候选模型 |
| `--eval_model` | judge 模型 |
| `--generate` | 是否生成回答 |
| `--evaluate` | 是否评分 |
| `--score` | 是否聚合 |
| `--num_workers` | 并发 worker 数 |
| `--tensor_parallel_size` | vLLM tensor parallel |
| `--judge_server` | `local` 或 `openrouter` |

## 阶段一：生成回答

当 `--generate true`：

1. 根据 `tasks.yaml` 加载所有样本。
2. 检查输出 JSONL 中已经完成的 sample id。
3. 启动候选模型 vLLM server。
4. 对剩余样本调用 `process_sample()`。
5. 保存到：

```text
results/<model_name>/<model_name>.jsonl
```

支持断点续跑，已经存在的 sample id 会跳过。

## vLLM candidate server

`utils.py` 中的生成函数使用 OpenAI client：

```python
OpenAI(
    base_url="http://localhost:8005/v1/",
    api_key="DUMMY_KEY",
)
```

这说明候选模型通过本地 vLLM OpenAI-compatible server 提供服务。

## 图像输入

如果样本包含图像，会把 PIL image 转成 base64 JPEG：

```text
data:image/jpeg;base64,...
```

然后用 OpenAI-compatible multimodal message 发送。

如果没有图像，则发送纯文本。

## 阶段二：LLM judge 评分

当 `--evaluate true`：

1. 读取生成阶段 JSONL。
2. 启动 judge 模型。
3. 对每条 response 调用 `process_evaluation()`。
4. 保存到：

```text
results/<model_name>/<model_name>_eval.jsonl
```

judge 有两种模式：

| 模式 | 说明 |
|---|---|
| `local` | 本地 vLLM 跑 judge 模型 |
| `openrouter` | 调 OpenRouter API |

OpenRouter 模式需要：

```bash
export OPENROUTER_API_KEY=...
```

文档中不要记录真实 key。

## 评分逻辑

非 MIMIC 任务通常是二值评分：

```text
1 = 回答正确
0 = 回答错误
```

MIMIC 任务是 0-5 分。

judge prompt 要求模型返回 JSON，`eval.py` 再解析 `score`。

为提高稳定性，`utils.py` 中会让 judge 多轮重新评估，然后取结果。

## 阶段三：分数聚合

当 `--score true`：

1. 读取 eval JSONL。
2. 从 sample id 中解析任务名。
3. 对 MIMIC 任务按总分除以满分。
4. 对其他任务按 correct / total。
5. 写入：

```text
results/<model_name>/<model_name>_score.txt
```

如果多个任务，会计算 Overall。

## MIMIC-CXR 特殊要求

MIMIC-CXR 任务需要 PhysioNet 凭据：

```text
PHYSIONET_UNAME
PHYSIONET_PWD
```

`eval.sh` 会 source `.env`。

如果没有凭据，不应运行 MIMIC 相关任务。

## MMMU-Medical

MMMU-Medical 不走主 `eval.py` 流程，而是通过 MedEvalKit 单独跑。

README 建议：

1. 克隆 MedEvalKit。
2. 创建单独环境。
3. 复制 Qwen3-VL model adapter。
4. 注册 Qwen3-VL。
5. 运行 `eval_mmmu_med.sh`。

原因是 MedEvalKit 有自己的依赖版本，可能与 vLLM 环境冲突。

## 当前评测风险

1. `argparse` 中 `type=bool` 解析命令行布尔值容易误用，字符串 `"false"` 也可能被视为 True。
2. `--num_workers 128` 对普通机器过高。
3. 本地 judge 需要足够 GPU。
4. HuggingFace dataset 下载可能失败。
5. MIMIC 图像下载依赖 PhysioNet 凭据。
6. score 阶段依赖已有 eval JSONL，不能单独凭空运行。

## 学习重点

评测 pipeline 的核心是：

```text
候选模型生成回答
judge 模型判断回答是否正确
脚本按任务聚合分数
```

生成、评分、聚合最好分开理解和调试，不要一开始就运行 `--tasks all --generate true --evaluate true --score true`。
