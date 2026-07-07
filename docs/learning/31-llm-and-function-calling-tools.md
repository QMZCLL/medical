# LLM 与 function calling 工具

## 学习目标

本文解释项目如何使用 OpenAI-compatible API、`LLMClient` 和 function calling，把本地 Skills 暴露给 LLM。

## 使用位置

```text
medix-agent-swarm/core/llm_client.py
medix-agent-swarm/core/agent_loop.py
medix-agent-swarm/core/skill_registry.py
medix-agent-swarm/agents/*.py
MediX-R1/eval/utils.py
MediX-R1/training/examples/reward_function/medical.py
```

## OpenAI-compatible API

OpenAI-compatible API 指接口格式兼容 OpenAI Chat Completions，但后端模型可以不是 OpenAI 官方模型。

当前根目录 `config.py` 中：

```python
LLM_CONFIG = {
    "api_key": "",
    "model_name": "doubao-seed-1-6-flash-250828",
    "base_url": "https://ark.cn-beijing.volces.com/api/v3",
    "temperature": 0.7,
    "max_tokens": 8192,
}
```

只要服务支持 `/chat/completions` 和相应参数，就可以接入。

## LLMClient

`LLMClient` 封装在：

```text
medix-agent-swarm/core/llm_client.py
```

它使用：

```python
from openai import AsyncOpenAI
```

初始化：

```python
self.client = AsyncOpenAI(
    api_key=self.config["api_key"],
    base_url=self.config["base_url"]
)
```

主要方法：

- `chat()`：普通文本对话。
- `chat_with_retry()`：带重试的普通对话。
- `chat_with_tools()`：带工具 schema 的 function calling。
- `create_tool_message()`：把工具执行结果转成 OpenAI tool message。

## chat()

`chat()` 输入：

```python
messages = [
    {"role": "user", "content": "..."}
]
```

输出：

```python
content: str
```

它用于：

- LeadAgent 任务分解。
- LeadAgent 汇总结果。
- DeepResearch 查询规划。
- EvidenceSynthesizer 证据综合。

## chat_with_tools()

`chat_with_tools()` 输入：

```python
messages: List[Dict[str, Any]]
tools: Optional[List[Dict[str, Any]]]
tool_choice: "auto"
```

输出：

```python
LLMResponse(
    content: Optional[str],
    tool_calls: List[ToolCall],
    finish_reason: str
)
```

如果模型返回工具调用，`tool_calls` 中包含：

```python
ToolCall(
    id=...,
    name=tc.function.name,
    arguments=json.loads(tc.function.arguments)
)
```

## function calling 的本质

function calling 不是 LLM 直接执行函数。它只是让模型输出一个结构化请求：

```json
{
  "name": "search_knowledge",
  "arguments": {
    "query": "高血压治疗"
  }
}
```

真正执行函数的是 `AgentLoop` 和 `SkillRegistry`。

流程：

```text
SkillRegistry.to_openai_format()
  -> LLM 看到 tools schema
  -> LLM 返回 tool_calls
  -> AgentLoop 调 SkillRegistry.execute()
  -> 本地 Python 函数执行
  -> 工具结果作为 tool message 返回给 LLM
```

## tool message

工具结果消息格式：

```python
{
    "role": "tool",
    "tool_call_id": tool_call_id,
    "name": tool_name,
    "content": json.dumps(result, ensure_ascii=False)
}
```

LLM 看到这条消息后，可以基于工具结果生成最终回答。

## 与 Agent system prompt 的关系

每个 Agent 的 system prompt 会告诉模型：

- 有哪些 Skills。
- 什么时候使用哪些 Skills。
- 最多调用几个 Skills。
- 输出格式是什么。
- 医疗安全边界是什么。

function calling schema 只告诉模型“工具长什么样”，system prompt 决定“工具应该怎么用”。

## MediX-R1 中的 OpenAI client

`MediX-R1/eval/utils.py` 直接使用同步 `OpenAI` client：

```python
OpenAI(
    base_url="http://localhost:8005/v1/",
    api_key="DUMMY_KEY",
)
```

它访问本地 vLLM server。

OpenRouter judge 使用：

```python
OpenAI(
    base_url="https://openrouter.ai/api/v1",
    api_key=os.getenv("OPENROUTER_API_KEY"),
)
```

训练 reward function 中也用 HTTP 请求调用 LLM judge。

## 当前项目中的关键风险

1. `core/llm_client.py` 配置路径硬编码旧目录。
2. `api_key` 当前为空，实际调用会失败。
3. 不同 OpenAI-compatible 服务对 tools 支持程度不同。
4. tool arguments 必须是合法 JSON。
5. 模型可能过度调用工具，所以 AgentLoop 设置了工具调用上限。
6. 评测和训练 reward 使用的 OpenAI client 与 Agent 项目的 `LLMClient` 不是同一封装。

## 调试建议

按顺序验证：

```text
1. from config import LLM_CONFIG
2. AsyncOpenAI 初始化
3. chat() 普通对话
4. chat_with_tools() 空工具或简单工具
5. SkillRegistry.to_openai_format()
6. AgentLoop 完整工具调用
```

不要一开始就跑完整 Swarm，否则很难判断是 LLM、function calling、Skill、Milvus 还是记忆系统出错。

## 学习重点

function calling 的核心是“模型决策，程序执行”：

```text
LLM 选择工具名和参数
AgentLoop 执行本地函数
LLM 读取执行结果再生成回答
```

这也是 Agent 系统区别于普通聊天机器人的关键。
