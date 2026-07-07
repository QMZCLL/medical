# medix-agent-swarm 架构总览

## 学习目标

本文帮助你理解 `medix-agent-swarm` 的整体结构。读完后，你应该能说清楚：

- 用户问题进入系统后由哪些模块处理。
- 三个 Worker Agent 各自负责什么。
- Skills、记忆、知识库、约束系统分别处在什么位置。
- 当前源码与 README 对“Swarm”的描述有什么差异。

## 项目定位

`medix-agent-swarm` 是一个医疗咨询 Agent 系统原型。它不是医学模型训练框架，而是把 LLM、function calling、医学知识库、记忆系统和多个 Agent 组合成一个问答助手。

入口文件：

```text
medix-agent-swarm/main.py
```

核心模块：

```text
agents/       三个 Worker Agent
core/         LLM 客户端、AgentLoop、SkillRegistry
swarm/        SwarmCoordinator、LeadAgent、SharedContext
memory/       短期记忆、长期记忆、会话总结
knowledge/    Milvus Lite 医学知识库
research/     DeepResearch 工作流
constraints/  约束验证规则
validation/   输出自动修复
.claude/skills/  可被 Agent 调用的 Skill 函数
```

## 总体架构图

```text
用户问题
  |
  v
main.py
  |
  v
process_with_swarm()
  |
  v
SwarmCoordinator
  |
  +-- 读取短期记忆 ShortTermMemory
  +-- 读取长期记忆 LongTermMemory / Mem0
  |
  v
LeadAgent
  |
  +-- 调 LLM 判断问题复杂度
  +-- 输出 subtasks 和 assigned_agent
  |
  v
单 Agent 或多 Agent 执行
  |
  +-- ConsultationAgent
  +-- DiagnosticAgent
  +-- ResearchAgent
  |
  v
AgentLoop
  |
  +-- LLM 生成回答或 tool_calls
  +-- SkillRegistry 执行 Skills
  |
  v
Milvus / Mem0 / DDGS / LLM / 规则引擎
  |
  v
最终回答
```

## 五层结构

### 第一层：用户入口

`main.py` 提供交互式命令行：

- 输入健康问题。
- 输入 `exit` 或 `quit` 退出。
- 输入 `clear` 清屏。
- 输入 `help` 查看帮助。

用户问题最终调用：

```python
result = await process_with_swarm(user_input, session_id=session_id)
```

### 第二层：协调层

协调层主要在 `swarm/` 目录：

- `SwarmCoordinator`：统一入口，决定单 Agent 还是多 Agent。
- `LeadAgent`：调用 LLM 做任务分解和结果汇总。
- `SharedContext`：保存子任务、事件和各 Agent 的贡献。

注意：当前源码中 `LeadAgent` 会直接指定 `assigned_agent`。因此它不是完全自由抢任务的 Swarm，而是“LLM 分解任务 + 指派 Agent + 并行执行 + 汇总结果”。

### 第三层：Agent 层

三个 Worker Agent：

| Agent | 主要角色 | 源码文件 |
|---|---|---|
| ConsultationAgent | 健康咨询、生活方式建议、普通科普 | `agents/consultation_agent.py` |
| DiagnosticAgent | 症状分析、风险评估、鉴别诊断思路 | `agents/diagnostic_agent.py` |
| ResearchAgent | 指南检索、循证证据、深度研究 | `agents/research_agent.py` |

三者都继承 `BaseAgent`，并通过 `SkillRegistryMixin` 注册同一组 Skills。

### 第四层：Agent Loop 和 Skill 层

`AgentLoop` 是执行核心。它做的事情是：

1. 准备 system prompt、历史上下文和用户输入。
2. 把 Skills 转成 OpenAI function calling 格式。
3. 调用 LLM。
4. 如果 LLM 返回 `tool_calls`，执行对应 Skill。
5. 把工具结果追加回消息列表。
6. 再次调用 LLM，直到得到最终文本回答或达到迭代限制。

Skill 注册和执行由 `SkillRegistry` 管理。

### 第五层：外部能力层

Agent 可调用的能力来自：

- Milvus Lite：医学知识库语义检索。
- SentenceTransformers：生成 embedding。
- Mem0：长期记忆。
- DDGS、httpx、BeautifulSoup：网络搜索和网页抓取。
- OpenAI-compatible API：LLM 对话和 function calling。
- YAML 约束文件：Agent 使用边界和输出安全约束。

## 当前已知架构问题

当前源码有几个影响运行的架构问题：

1. `swarm/events.py` 缺失，但多处导入 `.events`。
2. `validation/auto_fixer.py` 缺失，但 `validation/__init__.py` 导入它。
3. `core/llm_client.py` 中硬编码旧路径。
4. README 中“自主认领”和源码中 `assigned_agent` 中心化分配不完全一致。
5. 多数 Skill 初始化会加载 embedding 模型和 Milvus，启动成本较高。

## 如何阅读这个子项目

建议按下面顺序看源码：

```text
main.py
swarm/swarm_coordinator.py
swarm/lead_agent.py
agents/base_agent.py
core/agent_loop.py
agents/skill_registry_mixin.py
core/skill_loader.py
core/skill_registry.py
.claude/skills/*/script/*.py
memory/*.py
knowledge/milvus_kb.py
research/*.py
constraints/*.yaml
```

先掌握请求流，再看具体工具。否则很容易陷入某个 Skill 或某个外部库细节，忘了它在整体系统里的位置。
