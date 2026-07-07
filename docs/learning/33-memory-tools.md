# 记忆工具

## 学习目标

本文详细解释项目中的 ShortTermMemory、Redis、Mem0 和 MemoryEntropyManager。

## 使用位置

```text
medix-agent-swarm/memory/short_term.py
medix-agent-swarm/memory/long_term.py
medix-agent-swarm/memory/entropy_manager.py
medix-agent-swarm/swarm/swarm_coordinator.py
medix-agent-swarm/core/agent_loop.py
```

## 记忆工具分类

| 工具 | 类型 | 当前作用 |
|---|---|---|
| ShortTermMemory | 本地会话记忆 | 保存当前会话消息 |
| Redis | 可选持久化后端 | 保存短期记忆 |
| Mem0 | 长期记忆服务 | 保存和检索跨会话摘要 |
| MemoryEntropyManager | 清理工具 | 去重、压缩、估算熵 |

## ShortTermMemory

ShortTermMemory 是当前会话记忆管理器。

主要方法：

- `create_session()`
- `add_message()`
- `get_session()`
- `get_recent_messages()`
- `get_history()`
- `clear_session()`

默认存储：

```python
storage_type="memory"
```

这意味着消息保存在 Python 进程内。

## ConversationHistory

短期记忆中的会话对象：

```python
ConversationHistory(
    session_id,
    messages,
    created_at,
    last_updated,
    metadata
)
```

每条消息：

```python
{
    "role": role,
    "content": content,
    "timestamp": datetime.now().isoformat()
}
```

## OpenAI 格式历史

`get_history()` 会把内部消息转换为 OpenAI messages 格式：

```python
[
    {"role": "user", "content": "..."},
    {"role": "assistant", "content": "..."}
]
```

它只保留 `user` 和 `assistant`，过滤 `tool`。

这份历史会被 AgentLoop 加到下一轮 LLM prompt 中。

## Redis 后端

如果使用：

```python
ShortTermMemory(storage_type="redis")
```

它会连接 Redis：

```python
redis.Redis(host=..., port=..., db=..., password=...)
```

并使用：

```python
setex(key, 3600, value)
```

保存一小时。

当前主流程默认没有启用 Redis。

## Mem0

Mem0 用于长期记忆。

初始化：

```python
MemoryClient(api_key=mem0_api_key)
```

API key 来源优先级：

1. 构造参数。
2. `config.py` 中的 `MEM0_CONFIG`。
3. 环境变量 `MEM0_API_KEY`。

如果没有 key 或初始化失败：

```python
self.enabled = False
```

系统继续运行，只是没有长期记忆。

## 长期记忆保存

保存会话摘要：

```python
self.mem0.add(
    messages=[{"role": "user", "content": memory_text}],
    user_id="medix_user",
    metadata={...}
)
```

当前 user_id 固定为 `medix_user`。

这适合单用户 demo，不适合多用户产品。

## 长期记忆检索

检索：

```python
self.mem0.search(
    query=query,
    user_id="medix_user",
    limit=limit * 2
)
```

结果格式化为：

```python
{
    "memory_id": "...",
    "content": "...",
    "score": 0.0,
    "metadata": {...},
    "timestamp": "..."
}
```

再通过 entropy manager 去重。

## MemoryEntropyManager

熵管理器用于控制记忆膨胀。

主要能力：

- 检测重复消息。
- 压缩较早消息。
- 估算重复率和熵水平。
- 去重相似会话。

短期记忆在读取 recent messages 时自动清理。

长期记忆在搜索结果返回后自动去重。

## search-history 与 search-similar-cases

两个 Skills 把记忆能力暴露给 Agent：

```text
search_history(session_id, limit)
search_similar_cases(query, max_results)
```

前者查当前会话。

后者查跨会话相似历史。

## 常见问题

### 为什么追问没有理解上下文

可能原因：

- `session_id` 没有复用。
- AgentLoop 没注入 short_term_memory。
- 历史消息被过滤或压缩。
- 模型没有利用上下文。

### 为什么相似案例为空

可能原因：

- Mem0 未安装。
- Mem0 API key 为空。
- `LongTermMemory.enabled=False`。
- 之前没有保存会话摘要。

### 为什么重启后短期记忆没了

默认是内存存储。要跨进程保留，需要 Redis 或其他持久化后端。

## 学习重点

记忆工具要按时间尺度理解：

```text
当前请求内：messages
当前会话内：ShortTermMemory
跨会话：Mem0
长期维护：SessionSummary 和 AgentIdentity
```

不要把 Milvus 知识库和 Mem0 长期记忆混淆。前者保存医学知识，后者保存历史会话经验。
