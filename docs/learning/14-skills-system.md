# Skills 系统

## 学习目标

本文解释 `medix-agent-swarm` 的 Skills 如何被发现、注册、转换为 function calling 格式并执行。

## Skills 在项目中的位置

Skills 位于：

```text
medix-agent-swarm/.claude/skills/
```

当前主要 Skill：

```text
search-knowledge
assess-risk
analyze-symptoms
recommend-lifestyle
disease-code
clinical-guideline
deep-research
search-history
search-similar-cases
```

每个 Skill 一般包含：

```text
SKILL.md
script/
  *.py
```

`SKILL.md` 提供元数据和描述，`script/*.py` 提供真正可执行的 Python 函数。

## 关键源码

```text
agents/skill_registry_mixin.py
core/skill_loader.py
core/skill_registry.py
.claude/skills/*/script/*.py
```

## 自动发现流程

三个 Worker Agent 都继承：

```python
SkillRegistryMixin
```

它们的 `register_tools()` 调用：

```python
self.register_all_skills()
```

自动发现流程：

```text
SkillRegistryMixin.register_all_skills()
  -> discover_skills(project_root)
  -> 扫描 .claude/skills/
  -> 读取 SKILL.md frontmatter
  -> 找 script/*.py
  -> 根据目录名推断函数名
  -> load_skill_function()
  -> 注册到 SkillRegistry
```

## 函数名推断规则

Skill 目录名是 kebab-case，例如：

```text
search-knowledge
```

函数名推断为 snake_case：

```text
search_knowledge
```

因此脚本中必须存在同名函数：

```python
async def search_knowledge(query: str, max_results: int = 5):
    ...
```

如果函数名不匹配，Skill 会被跳过。

## 参数推断规则

`SkillRegistryMixin._infer_skill_parameters()` 使用 `inspect.signature()` 读取函数签名。

规则很简单：

- 没有默认值的参数是 required。
- 参数名包含 `count`、`limit`、`max`、`iterations` 时类型标为 number。
- 其他参数类型默认是 string。

例如：

```python
async def clinical_guideline(query: str, max_results: int = 1):
    ...
```

会推断：

```text
query       string required
max_results number optional
```

这种方式简单，但不适合复杂嵌套参数。

## SkillRegistry 的职责

`SkillRegistry` 保存每个 Skill：

```python
{
    "function": function,
    "description": description,
    "parameters": parameters,
    "is_async": True or False
}
```

它提供两个关键方法：

### to_openai_format()

把 Skill 转为 OpenAI tools 格式，交给 LLM function calling。

### execute()

根据工具名执行 Skill：

```python
result = await self.skill_registry.execute(tool_name, **arguments)
```

如果 Skill 是 async，直接 await。

如果 Skill 是 sync，放到 executor 中运行。

## 9 个 Skill 的功能分类

| Skill | 类型 | 主要依赖 |
|---|---|---|
| `search_knowledge` | 知识库检索 | Milvus, SentenceTransformers |
| `recommend_lifestyle` | 生活方式建议 | Milvus |
| `disease_code` | ICD-10 查询 | Milvus |
| `clinical_guideline` | 临床指南检索 | Milvus |
| `assess_risk` | 风险评估 | 规则引擎, Milvus 补充 |
| `analyze_symptoms` | 症状分析 | 规则引擎, Milvus 补充 |
| `deep_research` | 深度研究 | LLM, DDGS, Milvus, EvidenceSynthesizer |
| `search_history` | 短期记忆查询 | ShortTermMemory |
| `search_similar_cases` | 长期记忆查询 | Mem0 |

## Skill 与 Agent 的关系

当前三个 Worker Agent 都注册全部 Skills。

区别不在工具列表，而在：

- system prompt。
- 推荐使用策略。
- 结果后处理。
- Swarm 中的任务分配。

例如：

- ConsultationAgent 更常用 `search_knowledge`、`recommend_lifestyle`。
- DiagnosticAgent 更常用 `assess_risk`、`analyze_symptoms`。
- ResearchAgent 更常用 `clinical_guideline`、`deep_research`。

但从源码注册角度，它们都能调用全部 Skills。约束文件中定义的 allowed tools 当前主要用于日志警告，不会硬阻断执行。

## Skill 执行错误处理

如果 Skill 不存在：

```python
{
    "success": False,
    "error": "Skill not found: ..."
}
```

如果执行异常：

```python
{
    "success": False,
    "error": "Skill execution failed: ...",
    "skill": "..."
}
```

这些错误会作为 tool 结果返回给 LLM。最终用户是否看到清晰错误，取决于 LLM 生成的最终回答。

## 添加新 Skill 的步骤

按当前机制，新增一个 Skill 需要：

1. 在 `.claude/skills/` 下创建目录，例如 `drug-interaction`。
2. 写 `SKILL.md`，包含 YAML frontmatter。
3. 在 `script/` 下写 Python 文件。
4. 定义与目录名对应的函数：

```python
async def drug_interaction(...):
    ...
```

5. 函数参数尽量使用简单类型。
6. 让 Agent 启动时自动发现。

不需要手动修改三个 Agent 的注册代码，因为它们都会调用 `register_all_skills()`。

## 当前系统的限制

1. 依赖文件名和函数名约定，缺少显式 manifest。
2. 参数类型推断过于粗略。
3. 缺少参数描述的精细控制。
4. 全部 Agent 注册全部 Skills，权限边界主要靠 prompt 和约束日志。
5. Skill 初始化可能触发外部模型或数据库加载。
6. 没有对 Skill 执行结果做统一 schema 校验。

## 学习重点

理解 Skills 系统时，把它看成三层：

```text
本地 Python 函数
  -> SkillRegistry 元数据和执行器
  -> OpenAI function calling tools
```

LLM 看到的是工具 schema；Python 执行的是本地函数；AgentLoop 负责在两者之间做转换。
