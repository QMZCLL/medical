# 评测工具

## 学习目标

本文解释 `MediX-R1/eval` 使用的评测工具：vLLM、OpenAI client、OpenRouter、HuggingFace datasets、PhysioNet、MedEvalKit、filelock 和 psutil。

## 工具地图

| 工具 | 作用 |
|---|---|
| vLLM | 启动候选模型和 judge 模型服务 |
| OpenAI client | 访问本地 vLLM 或 OpenRouter |
| OpenRouter | 云端 judge 备用 |
| HuggingFace datasets | 加载 benchmark 数据 |
| PhysioNet | 下载 MIMIC-CXR 图像 |
| MedEvalKit | 单独评测 MMMU-Medical |
| filelock | 并发写 JSONL 时加锁 |
| psutil | 管理 server 进程 |
| Pillow | 图像读取和 base64 编码 |

## vLLM

评测中 vLLM 提供 OpenAI-compatible server。

用途：

1. 候选模型生成回答。
2. 本地 judge 模型评分。

评测代码默认访问：

```text
http://localhost:8005/v1/
```

如果本地没有 vLLM server，生成或评分会失败。

## OpenAI client

`utils.py` 使用：

```python
from openai import OpenAI
```

本地 vLLM：

```python
OpenAI(
    base_url="http://localhost:8005/v1/",
    api_key="DUMMY_KEY",
)
```

OpenRouter：

```python
OpenAI(
    base_url="https://openrouter.ai/api/v1",
    api_key=os.getenv("OPENROUTER_API_KEY"),
)
```

这两者使用同样的 client，但 base_url 不同。

## OpenRouter

OpenRouter 用于没有足够 GPU 跑本地 judge 时。

运行前需要：

```bash
export OPENROUTER_API_KEY=...
```

评测命令：

```bash
python eval.py \
  --model MBZUAI/MediX-R1-8B \
  --eval_model qwen/qwen3-14b \
  --judge_server openrouter \
  --tasks all \
  --evaluate true
```

不要把 API key 写入仓库或文档。

## HuggingFace datasets

`tasks.yaml` 中的每个任务都有：

```yaml
path:
name:
split:
question_column:
answer_column:
choices_column:
```

`load_dataset_with_params()` 根据这些字段加载数据。

支持：

- 多选题。
- 纯文本问答。
- 图像 VQA。
- 报告生成和总结。

## PhysioNet

MIMIC-CXR 任务需要从 PhysioNet 下载图像。

环境变量：

```text
PHYSIONET_UNAME
PHYSIONET_PWD
```

`eval.sh` 会：

```bash
source .env
export PHYSIONET_UNAME
export PHYSIONET_PWD
```

没有凭据时，不要运行 MIMIC-CXR 任务。

## MedEvalKit

MMMU-Medical 使用 MedEvalKit，不走主 `eval.py`。

原因：

- MedEvalKit 有独立模型适配器。
- 依赖版本可能与主 eval 环境冲突。
- Qwen3-VL 需要单独注册。

README 建议创建独立环境运行。

## filelock

评测支持多进程 worker。多个进程写同一个 JSONL 时需要避免冲突。

`filelock` 用于写文件加锁，确保并发写入不破坏 JSONL 格式。

## psutil

`psutil` 用于查找和杀掉 vLLM server 进程。

评测阶段结束后：

```python
kill_server_process(candidate_server)
kill_server_process(eval_server)
```

避免后台服务残留占用 GPU。

## Pillow

Pillow 用于：

- 读取图像。
- 转 RGB。
- 保存成 JPEG bytes。
- 编码成 base64。

VLM 任务中，图像会作为 data URL 传给 OpenAI-compatible API。

## 当前评测工具风险

1. 本地 vLLM 端口固定为 8005，端口冲突会失败。
2. `num_workers=128` 对普通机器过高。
3. OpenRouter 模式依赖网络和 API key。
4. MIMIC 数据依赖 PhysioNet 凭据。
5. MedEvalKit 需要单独环境。
6. HuggingFace 数据下载可能受网络影响。

## 调试建议

建议按这个顺序：

```text
1. 只加载 tasks.yaml
2. 只加载一个小任务数据集
3. 启动候选模型 vLLM
4. 对一条样本 generate
5. 启动 judge 或 OpenRouter
6. 对一条样本 evaluate
7. 聚合 score
8. 扩大到完整任务
```

## 学习重点

评测工具链的本质是：

```text
datasets 提供题目
vLLM 提供模型服务
OpenAI client 统一访问服务
judge model 生成分数
filelock 保护并发写入
psutil 清理服务进程
```

理解这条链路后，就能定位评测失败是数据、服务、judge、并发还是聚合问题。
