# 网络研究工具

## 学习目标

本文解释项目中用于 DeepResearch 的网络搜索、网页抓取和证据综合工具。

## 使用位置

```text
medix-agent-swarm/research/web_search.py
medix-agent-swarm/research/deep_research_workflow.py
medix-agent-swarm/research/evidence_synthesizer.py
medix-agent-swarm/.claude/skills/deep-research/script/research.py
```

## 工具组成

DeepResearch 相关工具包括：

| 工具 | 作用 |
|---|---|
| DDGS / duckduckgo-search | 网络搜索 |
| httpx | 异步 HTTP 请求 |
| BeautifulSoup | 网页正文清洗 |
| WebSearchTool | 项目封装的搜索工具 |
| EvidenceSynthesizer | LLM 证据综合 |
| ResearchReport | 结构化报告对象 |

## DDGS

`web_search.py` 尝试导入：

```python
try:
    from ddgs import DDGS
except ImportError:
    from duckduckgo_search import DDGS
```

这兼容了新旧包名。

如果两者都没有安装：

```python
DDGS_AVAILABLE = False
```

网络搜索会返回空结果。

## WebSearchTool.search()

搜索函数参数：

```python
search(
    query: str,
    max_results: int = 10,
    region: str = "cn-zh",
    safesearch: str = "on",
    timelimit: Optional[str] = None,
    retry_count: int = 2
)
```

默认会把 query 增强为：

```text
<query> 医学
```

如果 query 中已经有“医学”，则不重复添加。

## 搜索 backend

搜索会依次尝试：

```text
bing
duckduckgo
auto
```

只要某个 backend 返回结果，就停止尝试。

这样能提高搜索成功率，但也意味着结果来源不完全固定。

## SearchResult

搜索结果结构：

```python
SearchResult(
    title: str,
    url: str,
    snippet: str,
    source: "web"
)
```

DeepResearch 默认使用 title、url、snippet 做证据综合。

## 医学域名白名单

`WebSearchTool` 定义了医学网站白名单：

```text
pubmed.ncbi.nlm.nih.gov
mayoclinic.org
webmd.com
who.int
cdc.gov
nih.gov
uptodate.com
medscape.com
healthline.com
medicalnewstoday.com
```

并提供：

```python
filter_by_domain()
```

但当前 `search()` 主流程没有强制调用白名单过滤。因此搜索结果不一定都来自白名单。

## httpx 与 BeautifulSoup

`fetch_content()` 用：

```python
httpx.AsyncClient(timeout=self.timeout)
```

请求网页，再用 BeautifulSoup：

1. 删除 `script` 和 `style`。
2. 提取文本。
3. 清理空白。
4. 截断到 `max_length`。

这个能力在 `search_with_content()` 中可选启用。

当前 DeepResearch 主流程没有默认抓全文，主要使用搜索摘要。

## EvidenceSynthesizer

证据综合器把 web_results 和 kb_results 拼成 prompt，让 LLM 输出：

- 关键发现。
- 证据等级。
- 信息来源。
- 置信度。
- 信息冲突。
- 综合总结。
- 建议。

然后解析为：

```python
ResearchReport
```

## ResearchReport

结构：

```python
ResearchReport(
    query: str,
    key_findings: List[str],
    evidence_level: str,
    sources: List[Dict[str, str]],
    confidence: float,
    conflicts: List[str],
    summary: str,
    recommendations: List[str],
)
```

这是 DeepResearch 返回给 Skill 的核心对象。

## 当前网络研究风险

1. DDGS 后端可能失败。
2. 搜索结果可能不是权威医学来源。
3. 默认使用 snippet，不抓全文。
4. LLM 综合结果不等于严格循证审查。
5. 置信度和证据等级来自模型输出解析。
6. 网络访问可能受限。

## 调试建议

分层验证：

```text
1. import ddgs 或 duckduckgo_search
2. WebSearchTool.search("高血压 最新指南")
3. filter_by_domain()
4. fetch_content(url)
5. EvidenceSynthesizer.synthesize()
6. DeepResearchWorkflow.run()
7. deep_research Skill
```

## 学习重点

网络研究工具不是“真自动科研系统”，而是：

```text
搜索摘要 + 本地知识库 + LLM 综合
```

它适合辅助回答，但医疗严肃场景仍需要明确来源、证据等级和人工审查。
