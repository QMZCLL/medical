# 运行环境与配置

## 学习目标

本文说明当前 workspace 的运行前提、依赖边界和最小验证顺序。重点不是“一条命令跑通所有内容”，而是把不同依赖拆开理解。

## Python 环境

`medix-agent-swarm` README 建议：

```bash
conda create -n medix-swarm python=3.12 -y
conda activate medix-swarm
cd medix-agent-swarm
pip install -r requirements.txt
```

`MediX-R1/eval` README 建议：

```bash
conda create -n medixr1_eval python=3.11 -y
conda activate medixr1_eval
pip install uv
uv pip install -r requirements.txt
uv pip install vllm==0.16.0
```

`MediX-R1/training` README 写的是 Python 3.9+，但训练依赖很重，包含 torch、flash-attn、vLLM、Ray、transformers 等。

建议把三个环境分开：

| 环境 | 用途 | 原因 |
|---|---|---|
| `medix-swarm` | 多 Agent 医疗助手 | LLM API、Milvus、Mem0、DDGS |
| `medixr1_eval` | 模型评测 | vLLM、datasets、judge model |
| `medixr1_train` | 模型训练 | torch、flash-attn、Ray、FSDP、vLLM |

不要把全部依赖装进一个环境，否则 vLLM、torch、flash-attn、MedEvalKit 之间容易冲突。

## 共享配置文件

根目录有：

```text
config.py
```

它包含：

```python
LLM_CONFIG = {
    "api_key": "",
    "model_name": "doubao-seed-1-6-flash-250828",
    "base_url": "https://ark.cn-beijing.volces.com/api/v3",
    "temperature": 0.7,
    "max_tokens": 8192,
}

MEM0_CONFIG = {
    "api_key": "",
}
```

这里没有真实密钥。运行需要用户自己提供可用的 API key。

注意：`medix-agent-swarm/core/llm_client.py` 当前硬编码了旧路径再导入 `config`，需要修复后才能可靠使用当前根目录的 `config.py`。

## medix-agent-swarm 主要依赖

来自 `medix-agent-swarm/requirements.txt`：

- `openai`：调用 OpenAI-compatible Chat Completions 和 function calling。
- `sentence-transformers`：加载 embedding 模型。
- `pymilvus`：连接 Milvus Lite。
- `mem0ai`：长期记忆。
- `redis`：可选短期记忆持久化。
- `duckduckgo-search` 或 `ddgs`：网络搜索。
- `httpx`、`beautifulsoup4`：网页抓取和清洗。
- `loguru`：日志。
- `pyyaml`：读取约束 YAML。
- `pandas`、`numpy`、`torch`、`transformers`：数据和模型基础依赖。

这些依赖里，真正运行时最容易出问题的是：

- `sentence-transformers` 首次下载模型。
- `pymilvus` 和 Milvus Lite 文件路径。
- `mem0ai` 没有 API key。
- 网络搜索在受限网络中失败。
- LLM API key 或 base_url 不可用。

## MediX-R1 训练主要依赖

`MediX-R1/training/requirements.txt` 包含：

- `torch`
- `flash-attn`
- `vllm`
- `ray[default]`
- `transformers`
- `datasets`
- `qwen-vl-utils`
- `sentence-transformers`
- `wandb`
- `openai`

训练侧依赖 GPU 和 CUDA 环境。`flash-attn` 和 `vllm` 对 CUDA、PyTorch、Python 版本非常敏感。

训练脚本通常需要：

- 多 GPU。
- 可访问 HuggingFace 模型和数据集。
- 可用的 LLM API 或本地 judge 服务，用于 reward function。
- 足够磁盘保存 checkpoint。

## MediX-R1 评测主要依赖

`MediX-R1/eval/requirements.txt` 包含：

- `vllm`
- `openai`
- `datasets`
- `pyyaml`
- `pandas`
- `filelock`
- `psutil`
- `Pillow`
- `google-genai`
- `pydash`

评测涉及三类外部资源：

1. HuggingFace datasets。
2. 本地 vLLM server。
3. judge model，可以是本地 Qwen，也可以是 OpenRouter。

MIMIC-CXR 任务还需要 PhysioNet 凭据。

## 最小运行前置修复

在 `medix-agent-swarm` 中，运行前至少要处理：

1. 补齐或重命名 `swarm/events.py`。
2. 补齐或重命名 `validation/auto_fixer.py`。
3. 修复 `core/llm_client.py` 的配置路径。
4. 安装 requirements。
5. 确认 `config.py` 可被正确导入。

在 `MediX-R1/training` 中，运行前至少要处理：

1. 修复 `run_train.sh` 指向不存在脚本的问题。
2. 确认使用的是实际存在的训练脚本。
3. 确认模型名、数据集路径、GPU 数量与机器一致。
4. 确认 reward function 使用的 `config.py` 可导入且 LLM API 可用。

## 推荐验证顺序

### medix-agent-swarm

先做 import 验证：

```bash
cd medix-agent-swarm
python -c "import validation; print('validation ok')"
python -c "import swarm; print('swarm ok')"
python -c "import core.llm_client; print('llm_client ok')"
```

再验证 Skill 发现：

```bash
python -c "from core.skill_loader import discover_skills; from pathlib import Path; print(len(discover_skills(Path('.'))))"
```

再验证知识库：

```bash
python knowledge/scripts/import_hardcoded_data.py
```

最后再运行：

```bash
python main.py
```

### MediX-R1

先验证脚本存在：

```bash
cd MediX-R1/training
ls examples/medix-r1_*.sh
```

再做配置和 import 验证：

```bash
python -m verl.trainer.main config=examples/config.yaml trainer.val_only=True
```

这个命令仍可能因为 GPU、模型下载或依赖问题失败，但失败位置会比直接训练更清楚。

评测侧先跑单任务、低 worker 数：

```bash
cd MediX-R1/eval
python eval.py --model MBZUAI/MediX-R1-8B --tasks medqa --num_workers 1 --score true
```

如果没有生成文件，单独 `--score true` 会失败，因为它需要已有 eval JSONL。评测应按 generate、evaluate、score 顺序运行。

## 环境风险总结

| 风险 | 影响 | 建议 |
|---|---|---|
| 根目录不是 git 仓库 | 不能提交和看 diff | 用文件验证替代提交证据 |
| 缺失模块文件 | Agent 项目无法导入 | 先修复文件名或导入 |
| 旧路径硬编码 | 配置加载不稳定 | 改为相对项目根目录导入 |
| LLM API key 为空 | LLM 调用失败 | 在本地安全配置，不写入文档 |
| embedding 模型下载 | RAG 初始化慢或失败 | 预先缓存模型 |
| vLLM/Ray/flash-attn 版本 | 训练评测失败 | 独立环境安装 |
| MIMIC 凭据缺失 | MIMIC 任务不可跑 | 只跑非 MIMIC 任务或配置凭据 |
