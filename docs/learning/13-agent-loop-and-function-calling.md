# AgentLoop 与 function calling

## 学习目标

本文解释 `AgentLoop` 如何把 LLM、工具调用和最终回答串起来。它是 `medix-agent-swarm` 中最重要的执行引擎。

## 关键源码

```text
medix-agent-swarm/core/agent_loop.py
medix-agent-swarm/core/llm_client.py
medix-agent-swarm/core/skill_registry.py
medix-agent-swarm/agents/base_agent.py
```

## AgentLoop 解决什么问题

普通 LLM 只能根据 prompt 直接生成文本。AgentLoop 要解决的是：

```text
当模型需要外部信息时，如何让它调用本地函数，并把函数结果再交给模型继续推理？
```

在这个项目中，本地函数就是 Skills。

## 核心循环

`AgentLoop.run()` 的主流程：

```text
创建任务状态
  -> 初始化 messages
  -> 获取 OpenAI function calling 格式 tools
  -> 调用 LLM
  -> 如果 LLM 返回 tool_calls
       执行 Skills
       把工具结果追加到 messages
       继续下一轮
     否则
       把 LLM 文本作为最终回答
       做约束验证和自动修复
       保存短期记忆
       返回结果
```

## 初始化消息

消息由 `_initialize_messages()` 构建：

1. Agent 的 system prompt。
2. 当前会话短期记忆。
3. 本轮用户输入。

示意：

```python
messages = [
    {"role": "system", "content": agent.get_system_prompt()},
    {"role": "user", "content": "...历史消息..."},
    {"role": "assistant", "content": "...历史回答..."},
    {"role": "user", "content": agent.format_user_input(input_data)}
]
```

不同 Agent 的 system prompt 决定它偏向使用哪些 Skills 和输出什么格式。

## 获取 tools

Agent 调用：

```python
tools_openai_format = agent.get_tools_for_llm()
```

底层来自：

```python
SkillRegistry.to_openai_format()
```

输出类似 OpenAI tools 格式：

```json
{
  "type": "function",
  "function": {
    "name": "search_knowledge",
    "description": "搜索医学知识库",
    "parameters": {
      "type": "object",
      "properties": {
        "query": {
          "type": "string",
          "description": "Query"
        }
      },
      "required": ["query"]
    }
  }
}
```

## LLM 返回两种结果

### 情况一：返回 tool_calls

如果模型认为需要调用工具，会返回 `tool_calls`。

`AgentLoop` 会：

1. 记录 assistant tool call 消息。
2. 遍历每个 tool call。
3. 用 `agent.execute_tool()` 执行本地 Skill。
4. 把结果包装成 `role=tool` 消息。
5. 进入下一轮 LLM 调用。

### 情况二：返回普通文本

如果模型直接给出回答，或工具结果已经足够，`AgentLoop` 认为任务完成。

然后做：

- 输出约束验证。
- 自动修复免责声明或高危提醒。
- 写入短期记忆。
- 调用 Agent 的 `post_process_result()`。

## Skill 调用次数限制

`AgentLoop` 初始化参数：

```python
max_iterations: int = 10
max_tool_calls: int = 2
```

如果工具调用次数达到上限，它会追加提示：

```text
已完成 2 次信息检索。请基于已获取的信息提供最终答复。
```

这样可以避免模型反复调用工具导致无限循环。

## 约束验证

`AgentLoop` 尝试导入：

```python
from constraints import ConstraintValidator
from validation import AutoFixer
```

如果导入成功：

- tool call 前调用 `validate_tool_call()`。
- 最终输出后调用 `validate_output()`。
- 对可修复问题调用 `AutoFixer.fix_output()`。

当前注意点：`validation/auto_fixer.py` 缺失会导致 `validation` 导入失败，进而约束修复功能不可用。

## 短期记忆写入

AgentLoop 会写入三类消息：

| 时机 | role | 内容 |
|---|---|---|
| 开始执行 | `user` | 本轮用户输入 |
| 请求工具 | `assistant` | 调用工具名称 |
| 工具返回 | `tool` | 工具结果摘要 |
| 最终回答 | `assistant` | 最终回答 |

短期记忆用于下一轮对话上下文。

## 与 BaseAgent 的关系

`BaseAgent.process()` 默认调用：

```python
return await self.run_loop(input_data)
```

`run_loop()` 再调用：

```python
return await self.loop.run(self, input_data, session_id=session_id)
```

因此只要 Agent 继承 `BaseAgent` 并实现：

- `get_system_prompt()`
- `register_tools()`

就可以接入同一套 AgentLoop。

## 常见失败点

### 模型不支持 function calling

如果 OpenAI-compatible endpoint 不支持 tools 参数，`chat_with_tools()` 会失败。

### tool arguments 不是合法 JSON

`LLMClient.chat_with_tools()` 使用：

```python
json.loads(tc.function.arguments)
```

如果模型返回非法 JSON，会抛异常。

### Skill 参数推断太粗

参数类型主要靠参数名猜：

- 包含 `count`、`limit`、`max`、`iterations` 的参数标为 number。
- 其他默认为 string。

这对复杂参数不够精确。

### 工具执行失败

`SkillRegistry.execute()` 会捕获异常并返回：

```python
{
    "success": False,
    "error": "...",
    "skill": "..."
}
```

但最终回答是否正确处理这个错误，取决于 LLM 如何理解工具结果。

## 一个简化例子

用户问：

```text
高血压患者饮食注意什么？
```

可能流程：

```text
ConsultationAgent system prompt 告诉模型可用 recommend_lifestyle
  -> LLM 返回 tool_call: recommend_lifestyle(diagnosis="高血压")
  -> Skill 搜索 Milvus 中 lifestyle 类型文档
  -> 工具返回高血压饮食、运动、用药建议
  -> LLM 基于工具结果生成最终回答
  -> AgentLoop 添加免责声明并保存短期记忆
```

## 学习重点

理解 AgentLoop 时抓住一个核心：

```text
LLM 不是直接拥有工具能力；
AgentLoop 把 LLM 的 tool_calls 翻译成本地函数执行，
再把执行结果返回给 LLM。
```

这就是本项目 Agent 能调用医学知识库、记忆和 DeepResearch 的基础。
