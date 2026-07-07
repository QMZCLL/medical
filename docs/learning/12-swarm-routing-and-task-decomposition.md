# Swarm 路由与任务分解

## 学习目标

本文解释 `SwarmCoordinator` 和 `LeadAgent` 如何决定使用单 Agent 还是多 Agent，以及任务如何被分配给 Worker Agent。

## 关键源码

```text
medix-agent-swarm/swarm/swarm_coordinator.py
medix-agent-swarm/swarm/lead_agent.py
medix-agent-swarm/swarm/shared_context.py
medix-agent-swarm/constraints/swarm_constraints.yaml
```

## SwarmCoordinator 的职责

`SwarmCoordinator` 是用户请求进入 Swarm 系统后的统一入口。

它负责：

1. 初始化三个 Worker Agent。
2. 初始化短期记忆和长期记忆。
3. 调用 `LeadAgent` 分解问题。
4. 根据子任务数量决定路由。
5. 在 Swarm 模式下创建 `SharedContext`。
6. 收集结果、保存会话总结、返回最终响应。

它不负责：

- 直接决定每个 Skill 怎么调用。
- 亲自生成医疗回答。
- 训练或更新模型。

## LeadAgent 的职责

`LeadAgent` 有两个核心职责：

1. `assess_and_decompose()`：让 LLM 判断问题应该分给哪些 Agent。
2. `synthesize_results()`：把多个 Agent 的贡献汇总成最终回答。

它的 system prompt 明确告诉 LLM：

- 简单问题尽量只用一个 Agent。
- 常见健康咨询优先用 ConsultationAgent。
- 复杂症状可用 DiagnosticAgent + ConsultationAgent。
- 需要指南和证据时使用 ResearchAgent。
- 输出 JSON，包含 `description` 和 `assigned_agent`。

## 路由规则

实际路由逻辑在 `SwarmCoordinator.process()` 中：

```text
subtasks = assessment.get("subtasks", [])

if len(subtasks) == 1:
    单 Agent
elif len(subtasks) >= 2 and enable_swarm:
    Swarm
else:
    fallback 到 ConsultationAgent
```

也就是说，系统是否进入 Swarm 主要取决于 `LeadAgent` 输出了几个子任务。

## 单 Agent 模式

当 `LeadAgent` 返回一个子任务：

```json
{
  "subtasks": [
    {
      "description": "回答用户关于感冒的咨询",
      "assigned_agent": "consultation_agent"
    }
  ]
}
```

系统会取出 `assigned_agent`，找到对应 Agent，然后调用：

```python
await agent.process({
    "question": question,
    "context": enhanced_context,
    "session_id": session_id
})
```

如果 `assigned_agent` 不存在，会降级到 `ConsultationAgent`。

## Swarm 模式

当子任务数量大于等于 2，且 `enable_swarm=True`，系统进入 `_process_with_swarm()`。

流程：

```text
创建 SharedContext
  -> attach 给所有 Worker
  -> 发布 SWARM_STARTED 事件
  -> LeadAgent.create_subtasks()
  -> 每个 Worker 获取分配给自己的任务
  -> Worker 并行执行
  -> SharedContext 记录贡献
  -> LeadAgent 汇总结果
```

## SharedContext 中的任务

`SubTask` 结构包含：

```python
id: str
type: str
description: str
assigned_agent: str
status: TaskStatus
result: Optional[Dict[str, Any]]
```

关键字段是 `assigned_agent`。Worker 通过：

```python
shared_context.get_subtasks_for_agent(worker.agent_id)
```

只获取分配给自己的待执行任务。

## 当前实现不是完全自主认领

README 中有“自主任务认领”的描述，但当前源码更准确的行为是：

```text
LeadAgent 生成 assigned_agent
SharedContext 保存 assigned_agent
Worker 只执行 assigned_agent 等于自己 agent_id 的任务
```

因此它不是自由市场式的任务竞标，而是中心化分配后的并行执行。

这种设计的优点：

- 行为更可控。
- 容易保证高危症状进入 DiagnosticAgent。
- 汇总时知道每个任务应该由谁负责。

缺点：

- 对 LeadAgent 的 LLM 输出质量依赖较强。
- Worker 的能力标签目前没有真正参与动态竞争。
- `constraints/swarm_constraints.yaml` 的规则没有完整嵌入路由逻辑。

## 约束文件的作用

`constraints/swarm_constraints.yaml` 定义了：

- 最大 Agent 数。
- 超时时间。
- 任务分解规则。
- Agent 选择规则。
- 质量控制。
- 冲突解决策略。

例如：

```yaml
agent_selection_rules:
  - if_symptoms: ["胸痛", "呼吸困难", "心悸", "昏厥", "剧烈头痛", "突然视力模糊"]
    must_include: ["diagnostic_agent"]
```

但当前 `LeadAgent.assess_and_decompose()` 主要靠 prompt 指导 LLM，`ConstraintValidator.get_required_agents()` 没有在主路由里强制调用。因此这些 YAML 更像约束能力的基础设施，而不是完整强制路由器。

## 超时处理

Swarm 执行时使用：

```python
asyncio.wait_for(..., timeout=90.0)
```

如果超时：

- 记录已完成 Agent。
- 记录未完成任务。
- 仍尝试汇总已有贡献。
- 返回结果中设置 `timeout_occurred=True`。

如果没有任何 Agent 完成，最终回答会提示系统超时并建议简化问题或紧急就医。

## 一个例子

用户问题：

```text
胸闷气短冒冷汗，严重吗？
```

理想分解：

```json
{
  "subtasks": [
    {
      "description": "评估胸闷气短冒冷汗的风险等级和紧急程度",
      "assigned_agent": "diagnostic_agent"
    },
    {
      "description": "提供紧急处理建议和就医提醒",
      "assigned_agent": "consultation_agent"
    }
  ]
}
```

路由结果：

```text
2 个 subtasks -> Swarm 模式
diagnostic_agent 执行风险分析
consultation_agent 执行处理建议
LeadAgent 汇总
```

## 阅读重点

读这部分源码时，关注三个问题：

1. `LeadAgent` 的 prompt 如何影响任务数量和 Agent 分配。
2. `SwarmCoordinator` 如何根据任务数量切换模式。
3. `SharedContext` 如何保存任务和贡献。

理解这三个点，就能看懂这个项目的“多 Agent 协作”核心。
