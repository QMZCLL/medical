# 项目总览

## 学习目标

读完本文后，你应该能回答三个问题：

1. 这个 workspace 里有哪些子项目。
2. 每个子项目解决什么问题。
3. 两条主线之间有什么关系，哪些内容只是并列放在同一 workspace 中。

## 当前目录结构

当前 workspace 根目录是：

```text
/home/qmz/projects/learning-agent/medical
```

核心内容分为两条主线：

```text
medical/
├── config.py
├── medix-agent-swarm/
└── MediX-R1/
```

`config.py` 是共享配置文件，主要提供：

- `LLM_CONFIG`：OpenAI-compatible LLM 接口配置。
- `MEM0_CONFIG`：Mem0 长期记忆接口配置。

当前文件中没有真实密钥，`api_key` 为空。

## 子项目一：medix-agent-swarm

`medix-agent-swarm` 是一个多智能体医疗助手原型。它的目标是把用户的医疗咨询问题交给多个 Agent 协作处理。

主要源码目录：

```text
medix-agent-swarm/
├── main.py
├── agents/
├── core/
├── swarm/
├── memory/
├── knowledge/
├── research/
├── constraints/
├── validation/
└── .claude/skills/
```

从源码看，它的核心链路是：

```text
用户输入
  -> main.py
  -> process_with_swarm()
  -> SwarmCoordinator
  -> LeadAgent 任务分解
  -> ConsultationAgent / DiagnosticAgent / ResearchAgent
  -> AgentLoop
  -> Skills
  -> Milvus / Mem0 / WebSearch / LLM
  -> 最终回答
```

这个子项目适合学习：

- Agent Loop 如何驱动工具调用。
- function calling 格式如何从本地 Skill 函数生成。
- 多 Agent 如何通过 SharedContext 汇总贡献。
- 医疗场景下如何加入免责声明、高危提醒和约束验证。
- RAG、长期记忆、短期记忆、网络搜索如何接入 Agent。

## 子项目二：MediX-R1

`MediX-R1` 是医学多模态推理模型的训练与评测代码。它更接近模型训练工程，而不是在线咨询应用。

主要源码目录：

```text
MediX-R1/
├── training/
│   ├── examples/
│   ├── verl/
│   ├── run_train.sh
│   └── merge_model.sh
├── eval/
├── eval2/
└── images/train/
```

训练主线：

```text
训练脚本
  -> examples/config.yaml
  -> python3 -m verl.trainer.main
  -> Ray Runner
  -> RayPPOTrainer
  -> FSDPWorker
  -> vLLM rollout
  -> reward_function/medical.py
  -> checkpoint
```

评测主线：

```text
eval.py
  -> 加载 tasks.yaml
  -> 启动候选模型 vLLM server
  -> 生成回答
  -> 启动 judge 模型或 OpenRouter judge
  -> LLM-as-judge 评分
  -> 聚合 score.txt
```

这个子项目适合学习：

- veRL/EasyR1 风格的 RL 训练框架。
- DAPO/GRPO/GSPO 类训练脚本如何配置。
- reward function 如何组合格式、LLM judge、embedding 和模态奖励。
- Ray、FSDP、vLLM rollout 如何协同。
- 医学 LLM/VLM benchmark 如何生成、评测和聚合结果。

## 两条主线的关系

这两个子项目在主题上都围绕医疗 AI，但源码层面不是同一个运行系统。

| 维度 | medix-agent-swarm | MediX-R1 |
|---|---|---|
| 目标 | 在线医疗助手原型 | 医学模型训练与评测 |
| 输入 | 用户自然语言问题 | 训练数据、评测数据、医学图像和问题 |
| 核心机制 | Agent、Skill、记忆、RAG、Swarm | Ray、FSDP、vLLM、reward、benchmark |
| 输出 | 医疗咨询回答 | checkpoint、评测结果、训练指标 |
| 运行依赖 | LLM API、Milvus Lite、Mem0、DDGS | GPU、vLLM、Ray、HuggingFace datasets |

可以把它们理解为：

- `MediX-R1` 负责“训练和评测医学推理模型”。
- `medix-agent-swarm` 负责“把 LLM 和工具组织成医疗助手系统”。

当前源码没有显示 `medix-agent-swarm` 直接加载 `MediX-R1` 本地模型 checkpoint。两者更像同一主题下的两个工程方向。

## 当前源码状态

需要特别注意：当前 workspace 更像一个源码快照，不是开箱即用的完整产品。

已确认事实：

- 根目录不是 git 仓库。
- 现有 Markdown 主要是三个 README。
- `medix-agent-swarm` 存在导入阻塞：缺少 `swarm/events.py` 和 `validation/auto_fixer.py`。
- `core/llm_client.py` 中有旧的 `/Users/.../swarm` 硬编码路径。
- `MediX-R1/training/run_train.sh` 指向当前不存在的 `examples/medix-r1_8b_dapo.sh`。

因此学习时不要只按 README 跑命令，而应先理解源码链路，再逐步修复运行问题。

## 推荐阅读方式

建议按下面顺序阅读：

1. 先读 `00-project-overview.md` 和 `01-learning-roadmap.md`。
2. 再读 `11-request-lifecycle.md`，掌握 Agent 系统的端到端流程。
3. 再读 `20-medix-r1-training-overview.md`，掌握训练主线。
4. 根据兴趣深入工具手册，如 Milvus、Mem0、vLLM、Ray、FSDP。
5. 最后读 `37-debugging-and-next-steps.md`，了解当前项目修复路线。
