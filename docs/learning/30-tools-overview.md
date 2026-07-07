# 工具链总览

## 学习目标

本文给出当前 workspace 使用工具的全景图。后续工具手册会逐类展开。

## 工具分类

当前项目工具链可以分为 6 类：

| 类别 | 代表工具 | 使用位置 |
|---|---|---|
| LLM 与 function calling | OpenAI-compatible API, AsyncOpenAI, LLMClient | `medix-agent-swarm/core/llm_client.py` |
| RAG 与向量库 | Milvus Lite, SentenceTransformers | `knowledge/milvus_kb.py` |
| 记忆系统 | ShortTermMemory, Redis, Mem0 | `memory/` |
| 网络研究 | DDGS, httpx, BeautifulSoup, EvidenceSynthesizer | `research/` |
| 训练工具 | veRL/EasyR1, Ray, FSDP, vLLM, torch, transformers, wandb | `MediX-R1/training/` |
| 评测工具 | vLLM, OpenRouter, HuggingFace datasets, PhysioNet, MedEvalKit | `MediX-R1/eval/` |

## medix-agent-swarm 工具链

`medix-agent-swarm` 的工具链服务于在线问答：

```text
LLMClient
  -> Chat Completions
  -> function calling
  -> AgentLoop
  -> Skills
  -> Milvus / Mem0 / DDGS / 规则引擎
```

其中：

- OpenAI-compatible API 提供模型推理。
- function calling 让模型选择 Skill。
- Milvus Lite 提供医学知识检索。
- Mem0 提供长期记忆。
- ShortTermMemory 提供当前会话上下文。
- DDGS 和 httpx 支持 DeepResearch。
- YAML 约束和 AutoFixer 支持医疗安全边界。

## MediX-R1 工具链

`MediX-R1` 的工具链服务于模型训练与评测：

```text
HuggingFace datasets
  -> RLHFDataset
  -> RayPPOTrainer
  -> FSDPWorker
  -> vLLM rollout
  -> medical reward function
  -> checkpoint
  -> eval.py
  -> vLLM generation
  -> LLM judge
  -> score aggregation
```

其中：

- Ray 负责任务调度和资源池。
- FSDP 负责大模型分布式训练。
- vLLM 负责高吞吐生成。
- torch 和 transformers 提供模型基础。
- reward function 提供医学训练信号。
- datasets 加载评测数据。
- OpenRouter 可作为云端 judge 备用。

## 工具之间的关系

可以把工具分成两层：

### 应用层工具

应用层服务于用户问答：

- LLM API
- function calling
- Milvus
- Mem0
- DDGS
- BeautifulSoup

它们共同支撑 `medix-agent-swarm`。

### 训练评测层工具

训练评测层服务于模型能力构建：

- Ray
- FSDP
- vLLM
- datasets
- reward function
- MedEvalKit

它们共同支撑 `MediX-R1`。

## 容易混淆的工具

### Milvus 与 Mem0

Milvus 是本地医学知识库，用来检索医学文档。

Mem0 是长期记忆服务，用来检索历史会话和相似案例。

两者都和向量检索有关，但业务对象不同。

### vLLM 在训练和评测中的角色

训练中，vLLM 用于 rollout 生成。

评测中，vLLM 用于启动候选模型和 judge 模型的 OpenAI-compatible server。

### LLMClient 与 OpenAI client

`medix-agent-swarm` 封装了自己的 `LLMClient`。

`MediX-R1/eval` 直接使用 `openai.OpenAI` 访问本地 vLLM 或 OpenRouter。

两者都是 OpenAI-compatible 思路，但封装层不同。

### Qdrant 与 Milvus

README 和 research 模块中提到 Qdrant，但当前 DeepResearch 主流程使用的是 Milvus 知识库和 WebSearch。`research/knowledge_base.py` 可视为扩展方向，不是当前最核心路径。

## 工具学习顺序

建议按下面顺序学习：

1. OpenAI-compatible API 和 function calling。
2. AgentLoop 如何调用 Skills。
3. Milvus 和 SentenceTransformers。
4. ShortTermMemory 与 Mem0。
5. DDGS、httpx、BeautifulSoup 和 EvidenceSynthesizer。
6. Ray、FSDP、vLLM。
7. datasets、OpenRouter、MedEvalKit。

这个顺序先应用，再训练，符合当前 workspace 的两条主线。

## 工具调试原则

不要一次性验证所有工具。按依赖层拆开：

```text
纯 Python import
  -> 本地配置读取
  -> LLM API
  -> Skill 自动发现
  -> Milvus 检索
  -> Mem0 检索
  -> 网络搜索
  -> AgentLoop
  -> Swarm
```

训练评测同理：

```text
依赖安装
  -> dataset 加载
  -> tokenizer / processor
  -> reward function
  -> Ray 资源
  -> vLLM
  -> 小 batch smoke test
  -> 完整训练或评测
```

## 学习重点

工具不是孤立堆砌。每个工具都要回答：

```text
它解决什么问题？
它在哪条链路上？
它的输入输出是什么？
它失败时会影响哪个模块？
```

后续工具手册都按这个思路展开。
