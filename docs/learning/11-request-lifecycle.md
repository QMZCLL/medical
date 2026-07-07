# 请求生命周期

## 学习目标

本文解释 `medix-agent-swarm` 中一次用户咨询从输入到输出的完整链路。重点看数据如何流动，而不是每个类的全部实现细节。

## 生命周期总览

一次请求大致经历 9 步：

```text
1. main.py 获取用户输入
2. 生成或复用 session_id
3. process_with_swarm 创建 SwarmCoordinator
4. 读取短期记忆和长期记忆
5. LeadAgent 调 LLM 分解任务
6. SwarmCoordinator 选择单 Agent 或 Swarm
7. AgentLoop 驱动 LLM 和 Skills
8. 结果后处理、汇总、保存记忆
9. main.py 打印回答、建议和免责声明
```

## 第一步：CLI 获取用户输入

入口在：

```text
medix-agent-swarm/main.py
```

`interactive_mode()` 会循环读取：

```python
user_input = input("💬 您的问题：").strip()
```

特殊命令：

- `exit`、`quit`、`q`：退出。
- `clear`：清屏。
- `help`：打印帮助。

普通文本会作为健康问题处理。

## 第二步：会话 ID

交互式会话启动时生成一个 `session_id`：

```python
session_id = f"{datetime.now().strftime('%Y%m%d-%H%M%S')}-{str(uuid.uuid4())[:8]}"
```

这个 `session_id` 会贯穿整个会话，用于短期记忆和长期记忆记录。

## 第三步：调用 process_with_swarm

`main.py` 调用：

```python
result = await process_with_swarm(user_input, session_id=session_id)
```

`process_with_swarm()` 在 `swarm/swarm_coordinator.py` 末尾定义，它会创建：

```python
coordinator = SwarmCoordinator(enable_swarm=enable_swarm)
return await coordinator.process(question, context, session_id=session_id)
```

这意味着每次请求都会初始化一个新的 `SwarmCoordinator` 实例。

## 第四步：初始化协调器

`SwarmCoordinator.__init__()` 创建：

- `LLMClient`
- `LeadAgent`
- `ConsultationAgent`
- `DiagnosticAgent`
- `ResearchAgent`
- `SessionSummaryManager`
- `ShortTermMemory`
- `LongTermMemory`

然后把同一个短期记忆实例注入到所有 Worker Agent 的 `loop.short_term_memory`。

## 第五步：读取记忆

`SwarmCoordinator.process()` 先读取当前会话短期记忆：

```python
recent_history = self.short_term_memory.get_recent_messages(
    session_id=session_id,
    limit=10
)
```

再检索长期记忆：

```python
similar_memories = self.long_term_memory.search_similar_sessions(
    query=question,
    limit=3
)
```

如果有结果，会构造 `enhanced_context`：

```text
recent_history     当前会话历史
historical_cases   相似历史案例
```

这个上下文会传给 `LeadAgent` 和 Worker Agent。

## 第六步：LeadAgent 分解任务

`LeadAgent.assess_and_decompose()` 构造 system prompt，让 LLM 返回 JSON：

```json
{
  "subtasks": [
    {
      "description": "回答用户问题",
      "assigned_agent": "consultation_agent"
    }
  ]
}
```

源码通过正则提取第一个 JSON 块：

```python
json_match = re.search(r'\{.*\}', content, re.DOTALL)
```

如果解析失败，会回退到 ConsultationAgent。

如果 LLM 调用异常，则返回空 `subtasks`，后续会走 fallback。

## 第七步：路由到单 Agent 或 Swarm

`SwarmCoordinator.process()` 根据 `subtasks` 数量判断：

| 条件 | 路由 |
|---|---|
| `len(subtasks) == 1` | 单 Agent |
| `len(subtasks) >= 2 and enable_swarm` | Swarm |
| `len(subtasks) == 0` 或 Swarm 关闭 | fallback 到 ConsultationAgent |

单 Agent 模式直接调用：

```python
result = await agent.process({
    'question': question,
    'context': enhanced_context,
    'session_id': session_id
})
```

Swarm 模式会进入 `_process_with_swarm()`。

## 第八步：AgentLoop 执行

无论哪个 Worker Agent，默认都会进入：

```python
BaseAgent.process()
  -> BaseAgent.run_loop()
  -> AgentLoop.run()
```

`AgentLoop` 做三件核心事情：

1. 初始化消息。
2. 调用 LLM。
3. 根据 LLM 是否返回 tool calls 决定执行 Skill 或结束。

工具调用循环：

```text
LLM 返回 tool_calls
  -> SkillRegistry.execute()
  -> 工具结果写回 messages
  -> 下一轮 LLM
```

最终文本回答会被写入短期记忆。

## 第九步：结果后处理和保存

单 Agent 模式：

- 从 Agent 返回结果中提取 `answer`。
- 确保结果有 `disclaimer` 和 `suggestions`。
- 保存长期记忆会话总结。

Swarm 模式：

- Worker 结果写入 `SharedContext`。
- `LeadAgent.synthesize_results()` 汇总多个 Agent 贡献。
- 保存 `SessionSummary`。
- 保存 Mem0 长期记忆。
- 返回 `agents_involved`、`subtasks_completed`、`swarm_metadata` 等元数据。

## 第十步：CLI 打印结果

`main.py` 根据结果打印：

- 单 Agent 或群体智能模式。
- 执行时间。
- 回答正文。
- 核心建议。
- 免责声明。

输出依赖返回字典字段：

```text
answer
swarm_enabled
agents_involved
suggestions
disclaimer
timeout_occurred
```

## 关键数据结构

### 输入

```python
{
    "question": "...",
    "context": {...},
    "session_id": "..."
}
```

### 单 Agent 输出

```python
{
    "answer": "...",
    "iterations": 2,
    "agent_id": "consultation_agent",
    "suggestions": [...],
    "disclaimer": "...",
    "swarm_enabled": False,
    "session_id": "..."
}
```

### Swarm 输出

```python
{
    "answer": "...",
    "swarm_enabled": True,
    "session_id": "...",
    "agents_involved": [...],
    "subtasks_completed": 2,
    "total_time": 12.3,
    "swarm_metadata": {...},
    "timeout_occurred": False,
    "suggestions": [...],
    "disclaimer": "..."
}
```

## 当前链路的脆弱点

1. `process_with_swarm()` 每次都创建新协调器，可能导致重复初始化模型、知识库和 Agent。
2. `LeadAgent` 依赖 LLM 输出可解析 JSON。
3. `AgentLoop` 依赖模型支持 OpenAI-compatible function calling。
4. Skill 参数由函数签名粗略推断，复杂类型支持有限。
5. 多数 Skill 初始化会加载 Milvus 和 embedding 模型，首次响应可能较慢。
6. 缺失 `events.py` 和 `auto_fixer.py` 会在导入阶段阻塞。

理解这些脆弱点后，再看测试和修复路线会更清楚。
