# 记忆系统

## 学习目标

本文解释 `medix-agent-swarm` 的短期记忆、长期记忆和会话总结机制。读完后，你应该知道多轮对话上下文从哪里来，历史案例如何检索，以及记忆系统在没有外部服务时如何降级。

## 关键源码

```text
memory/short_term.py
memory/long_term.py
memory/session_summary.py
memory/entropy_manager.py
memory/agent_identity.py
swarm/swarm_coordinator.py
.claude/skills/search-history/script/search.py
.claude/skills/search-similar-cases/script/search.py
```

## 记忆系统分层

当前项目有三类记忆：

| 类型 | 作用 | 存储 |
|---|---|---|
| 短期记忆 | 当前会话上下文 | 内存或 Redis |
| 长期记忆 | 跨会话相似案例 | Mem0 |
| 会话总结 | Swarm 执行后的结构化记录 | 本地文件管理器 |

另外还有 `AgentIdentityManager`，用于保存 Agent 身份和协作记录，但它不是主请求链路中最关键的部分。

## 短期记忆

短期记忆由 `ShortTermMemory` 管理。

主要数据结构：

```python
ConversationHistory(
    session_id: str,
    messages: List[Dict[str, str]],
    created_at: datetime,
    last_updated: datetime,
    metadata: Dict[str, Any]
)
```

消息格式：

```python
{
    "role": "user",
    "content": "...",
    "timestamp": "..."
}
```

支持的角色包括：

- `user`
- `assistant`
- `tool`

## 短期记忆的单例行为

`ShortTermMemory` 实现了单例模式：

```python
_instance = None
```

这意味着同一个 Python 进程中创建多个 `ShortTermMemory()`，会共享同一个实例。这样 `search_history` Skill 可以拿到 AgentLoop 写入的会话消息。

## 短期记忆如何进入 AgentLoop

`SwarmCoordinator.__init__()` 创建短期记忆：

```python
self.short_term_memory = ShortTermMemory(storage_type="memory")
```

然后注入到每个 Worker：

```python
worker.loop.short_term_memory = self.short_term_memory
```

`AgentLoop` 执行时会：

1. 从短期记忆读取历史消息。
2. 把当前用户消息写入短期记忆。
3. 把工具调用和工具结果摘要写入短期记忆。
4. 把最终回答写入短期记忆。

## Redis 后端

`ShortTermMemory` 支持 `storage_type="redis"`。

Redis 存储使用 key：

```text
session:<session_id>
```

过期时间：

```text
3600 秒
```

当前主流程默认使用内存：

```python
ShortTermMemory(storage_type="memory")
```

因此没有 Redis 也能运行短期记忆。

## 熵管理

短期记忆和长期记忆都尝试使用：

```python
MemoryEntropyManager
```

作用：

- 去重重复消息。
- 压缩较长历史。
- 估算消息熵和重复率。

短期记忆在 `get_recent_messages()` 中自动调用 `auto_clean()`。

这能避免会话历史无限膨胀，但也意味着传给 LLM 的历史上下文可能不是原始完整消息，而是经过清理后的版本。

## 长期记忆

长期记忆由 `LongTermMemory` 管理，底层依赖 Mem0。

初始化顺序：

1. 检查是否安装 `mem0`。
2. 从参数、`config.py` 或环境变量读取 API key。
3. 初始化 `MemoryClient`。
4. 如果失败，则设置 `enabled=False`。

没有 Mem0 API key 时，系统应该降级为：

```text
长期记忆禁用，短期记忆继续工作。
```

## 长期记忆保存

请求结束后，`SwarmCoordinator` 调用：

```python
self.long_term_memory.add_session_summary(
    session_id=session_id,
    question=question,
    answer=final_answer,
    metadata={...}
)
```

存储文本大致是：

```text
问题：...
回答：回答前 500 字...
```

用户 ID 当前固定为：

```text
medix_user
```

这意味着它还不是多用户隔离的产品级设计。

## 长期记忆检索

请求开始时，系统调用：

```python
self.long_term_memory.search_similar_sessions(query=question, limit=3)
```

结果会放入 `enhanced_context["historical_cases"]`，供 LeadAgent 和 Worker Agent 参考。

`search_similar_cases` Skill 也可以主动查询 Mem0。

## search-history Skill

`search_history(session_id, limit=10)` 从短期记忆读取当前会话历史。

它适合回答追问，例如：

```text
用户：我有高血压
用户：那我饮食上怎么办？
```

Agent 可以通过会话历史知道“那”指的是高血压。

## search-similar-cases Skill

`search_similar_cases(query, max_results=3)` 从长期记忆搜索相似案例。

如果长期记忆没有启用，会返回：

```text
长期记忆功能未启用。无法搜索历史案例。
```

这是合理降级，不应视为系统崩溃。

## 当前限制

1. 长期记忆 user_id 固定为 `medix_user`。
2. Mem0 是外部服务，需要 API key。
3. 短期记忆默认只在进程内有效，重启丢失。
4. Redis 配置在 README 中提到，但主流程默认没有启用。
5. 工具结果摘要只截取前 200 字，可能丢失细节。
6. 多轮对话效果高度依赖模型是否正确利用历史上下文。

## 学习重点

记忆系统不是一个单独的聊天记录功能，而是贯穿请求生命周期：

```text
请求开始：读取短期和长期记忆
Agent 执行：持续写入短期记忆
请求结束：保存长期会话摘要
后续请求：检索历史上下文和相似案例
```

理解这个闭环，才能理解项目为什么宣称支持多轮对话和相似案例。
