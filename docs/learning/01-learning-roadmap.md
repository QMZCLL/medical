# 学习路线

## 学习目标

本文给出一条从零理解当前 workspace 的路线。目标不是背目录，而是逐步建立三层认识：

1. 项目整体在做什么。
2. 请求、训练、评测分别如何流动。
3. 每个工具在系统中承担什么职责。

## 第一阶段：建立全局地图

先读：

1. `00-project-overview.md`
2. `02-source-vs-readme-facts.md`
3. `03-run-and-environment.md`

这一阶段只需要掌握：

- workspace 有 `medix-agent-swarm` 和 `MediX-R1` 两条主线。
- README 有些表述与当前源码不完全一致。
- 当前代码有若干启动阻塞，学习时要以源码为准。

不要急着运行全部测试。这个项目依赖 LLM、Milvus、Mem0、vLLM、HuggingFace、GPU 和外部网络，盲目运行会把问题混在一起。

## 第二阶段：读懂 medix-agent-swarm 运行链路

推荐顺序：

1. `10-agent-swarm-architecture.md`
2. `11-request-lifecycle.md`
3. `12-swarm-routing-and-task-decomposition.md`
4. `13-agent-loop-and-function-calling.md`
5. `14-skills-system.md`

这一阶段的核心问题是：

```text
用户问一句话，系统如何决定调用哪个 Agent，Agent 又如何决定调用哪个 Skill？
```

你需要抓住下面的主线：

```text
main.py
  -> process_with_swarm
  -> SwarmCoordinator.process
  -> LeadAgent.assess_and_decompose
  -> Agent.process
  -> AgentLoop.run
  -> SkillRegistry.execute
```

读源码时可以重点看这些文件：

```text
medix-agent-swarm/main.py
medix-agent-swarm/swarm/swarm_coordinator.py
medix-agent-swarm/swarm/lead_agent.py
medix-agent-swarm/core/agent_loop.py
medix-agent-swarm/agents/base_agent.py
medix-agent-swarm/agents/skill_registry_mixin.py
medix-agent-swarm/core/skill_registry.py
```

## 第三阶段：读懂 Agent 的支撑系统

推荐顺序：

1. `15-memory-system.md`
2. `16-knowledge-base-rag.md`
3. `17-deep-research-workflow.md`
4. `18-constraints-and-safety.md`
5. `19-agent-swarm-current-issues.md`

这一阶段的核心问题是：

```text
Agent 的回答信息从哪里来？系统怎样尽量让医疗回答更安全？
```

对应四类支撑系统：

- 短期记忆：当前会话上下文。
- 长期记忆：Mem0 相似历史案例。
- 知识库：Milvus Lite 语义检索。
- 安全约束：ConstraintValidator 和 AutoFixer。

这部分不只看“能不能跑”，更要理解每个系统的降级路径。比如 Mem0 没有 API key 时应禁用长期记忆，但短期记忆仍可工作。

## 第四阶段：读懂 MediX-R1 训练主线

推荐顺序：

1. `20-medix-r1-training-overview.md`
2. `21-training-data-and-config.md`
3. `22-reward-function.md`
4. `23-verl-training-lifecycle.md`

这一阶段的核心问题是：

```text
训练脚本如何把数据、模型、rollout 和 reward 连接起来？
```

你需要抓住下面的主线：

```text
run_train.sh
  -> examples/*.sh
  -> examples/config.yaml
  -> verl.trainer.main
  -> RayPPOTrainer
  -> FSDPWorker
  -> AutoRewardManager
  -> examples/reward_function/medical.py
```

读源码时先不要深挖所有 `verl/` 细节。先理解 `main.py` 如何组装 tokenizer、processor、worker、reward function 和 dataloader，再进入训练循环。

## 第五阶段：读懂评测与结果

推荐顺序：

1. `24-evaluation-pipeline.md`
2. `25-results-and-training-report.md`
3. `36-evaluation-tools.md`

这一阶段的核心问题是：

```text
模型回答如何生成，如何被 judge 模型评分，最终分数如何聚合？
```

评测三阶段：

```text
generate  -> 生成模型回答
evaluate  -> LLM-as-judge 打分
score     -> 聚合任务准确率
```

注意 MIMIC-CXR 相关任务需要 PhysioNet 凭据；OpenRouter judge 需要 `OPENROUTER_API_KEY`；本地 judge 需要 GPU 运行 vLLM。

## 第六阶段：系统学习工具链

推荐顺序：

1. `30-tools-overview.md`
2. `31-llm-and-function-calling-tools.md`
3. `32-rag-and-vector-db-tools.md`
4. `33-memory-tools.md`
5. `34-web-research-tools.md`
6. `35-training-tools.md`
7. `36-evaluation-tools.md`

每篇工具文档都按同一个问题展开：

- 这个工具解决什么问题。
- 项目里哪里用了它。
- 输入和输出是什么。
- 常见配置项是什么。
- 在当前项目里容易踩什么坑。

这部分适合在你已经看过运行链路后阅读，否则会像孤立工具介绍。

## 最小验证路线

如果你想把项目从“能读懂”推进到“能运行”，建议按这个顺序验证：

1. 修复缺失模块文件名：`events.py` 和 `auto_fixer.py`。
2. 修复 `core/llm_client.py` 的配置导入路径。
3. 安装 `medix-agent-swarm/requirements.txt`。
4. 先做 import 级验证，不直接跑完整 LLM 流程。
5. 初始化或验证 Milvus Lite 小知识库。
6. 使用一个无需 DeepResearch 的简单咨询问题验证 Agent Loop。
7. 最后再验证 Mem0、DDGS、DeepResearch。

训练侧建议单独验证：

1. 先确认 `run_train.sh` 指向存在的脚本。
2. 确认 GPU、vLLM、Ray、torch、transformers 版本。
3. 用小 batch 或单任务 smoke test 验证 dataloader 和 reward。
4. 再进入完整训练。

## 学习成果检查

学完后，你应该能画出三张图：

1. `medix-agent-swarm` 请求生命周期图。
2. `MediX-R1` 训练生命周期图。
3. 工具依赖图：LLM、function calling、Milvus、Mem0、DDGS、Ray、FSDP、vLLM、datasets 之间的关系。

如果你能用自己的话解释这三张图，就基本掌握了这个 workspace。
