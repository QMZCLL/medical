# DeepResearch 工作流

## 学习目标

本文解释 `deep_research` Skill 如何把 LLM 查询规划、网络搜索、Milvus 检索和证据综合组合起来。

## 关键源码

```text
.claude/skills/deep-research/script/research.py
research/deep_research_workflow.py
research/web_search.py
research/evidence_synthesizer.py
research/knowledge_base.py
knowledge/milvus_kb.py
core/llm_client.py
```

## DeepResearch 解决什么问题

普通知识库检索只能查本地 10 个演示文档。DeepResearch 想解决更复杂的问题：

```text
当用户询问最新进展、复杂治疗方案、循证医学证据时，系统需要同时查本地知识库和网络信息，再综合成报告。
```

## 调用入口

Skill 函数：

```python
async def deep_research(query: str, max_iterations: int = 2)
```

它会创建：

```python
workflow = DeepResearchWorkflow()
```

然后调用：

```python
report = await workflow.run(
    question=query,
    max_web_results=max_iterations * 5,
    max_kb_results=max_iterations * 3
)
```

## 总体流程

`DeepResearchWorkflow.run()` 的主流程：

```text
1. 查询规划
2. 并行网络搜索和 Milvus 检索
3. 收集 web_results 和 kb_results
4. EvidenceSynthesizer 调 LLM 综合证据
5. 返回 ResearchReport
6. Skill 格式化成 answer
```

## 查询规划

`_plan_queries()` 会让 LLM 把复杂问题拆成 2-3 个子查询。

例如原问题：

```text
2型糖尿病如何治疗？
```

可能拆成：

```text
2型糖尿病的药物治疗方案
2型糖尿病的生活方式管理
2型糖尿病的并发症预防
```

如果 LLM 调用失败，则降级为只使用原始问题。

## 网络搜索

网络搜索工具：

```python
WebSearchTool
```

底层尝试导入：

```python
from ddgs import DDGS
```

如果失败，再尝试：

```python
from duckduckgo_search import DDGS
```

搜索时会尝试多个 backend：

```text
bing -> duckduckgo -> auto
```

搜索 query 会追加“医学”关键词，以提高医学相关性。

## 网络结果结构

搜索结果使用：

```python
SearchResult(
    title: str,
    url: str,
    snippet: str,
    source: "web"
)
```

DeepResearch 默认只把标题、URL 和摘要交给证据综合器，没有默认抓取全文。

## Milvus 检索

DeepResearch 同时调用：

```python
self.knowledge_base.search(query=query, top_k=top_k, filter_type=None)
```

这会搜索本地医学知识库，不限制文档类型。

返回结果是字典列表，包含：

- `content`
- `metadata`
- `score`

## 并行执行

网络搜索和知识库检索通过：

```python
asyncio.gather(*search_tasks, return_exceptions=True)
```

并行执行。单个搜索任务失败不会直接让整个工作流失败，而是记录 warning 后跳过。

## EvidenceSynthesizer

证据综合器负责把多来源信息交给 LLM，总结成结构化报告。

输入：

- 原始 query。
- web_results。
- kb_results。

输出：

```python
ResearchReport(
    query=...,
    key_findings=[...],
    evidence_level="A/B/C",
    sources=[...],
    confidence=0.0-1.0,
    conflicts=[...],
    summary="...",
    recommendations=[...]
)
```

## 证据等级

prompt 中定义：

- A 级：高质量随机对照试验或系统评价。
- B 级：队列研究或病例对照研究。
- C 级：专家共识或观察性研究。

当前证据等级由 LLM 输出文本解析而来，不是严格的文献质量自动评估。

## Skill 输出格式

`deep_research` Skill 返回：

```python
{
    "answer": "...格式化研究报告...",
    "findings": report.key_findings,
    "confidence": "high/medium/low",
    "sources": len(report.sources),
    "evidence_level": report.evidence_level,
    "status": "completed",
    "data_sources": "Web Search + Milvus RAG + Evidence Synthesis"
}
```

如果失败：

```python
{
    "answer": "深度研究失败：...",
    "findings": [],
    "confidence": "low",
    "sources": 0,
    "status": "error"
}
```

## 与 ResearchAgent 的关系

`ResearchAgent` 的 system prompt 建议：

- 优先使用 `clinical_guideline`。
- 需要最新信息或复杂问题时使用 `deep_research`。
- 综合多个信息来源。
- 给出证据等级。

因此 DeepResearch 是 ResearchAgent 的高级工具，不是每个问题都应该调用。

## 当前限制

1. 网络搜索依赖外部网络和 DDGS 后端稳定性。
2. 搜索结果默认只用 snippet，不保证信息完整。
3. 证据等级依赖 LLM 自评，不是严格循证流程。
4. `DeepResearchWorkflow` 初始化会创建 LLMClient 和知识库，依赖较多。
5. 如果 `core/llm_client.py` 配置路径未修复，查询规划和证据综合都会失败。
6. 医学来源白名单存在，但 `search()` 主流程当前没有强制过滤白名单。

## 学习重点

DeepResearch 是多工具编排：

```text
LLM 负责拆问题和综合
DDGS 负责找网络信息
Milvus 负责找本地医学知识
EvidenceSynthesizer 负责生成结构化报告
```

调试时要分层判断失败点：查询规划、网络搜索、知识库检索、证据综合，任何一层失败都会影响最终报告。
