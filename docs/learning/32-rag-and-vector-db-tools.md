# RAG 与向量数据库工具

## 学习目标

本文讲解项目中使用的 RAG、Milvus Lite、SentenceTransformers 和 embedding 检索工具。

## 使用位置

```text
medix-agent-swarm/knowledge/milvus_kb.py
medix-agent-swarm/knowledge/scripts/import_hardcoded_data.py
medix-agent-swarm/.claude/skills/*/script/*.py
medix-agent-swarm/research/deep_research_workflow.py
```

## RAG 是什么

RAG 是 Retrieval-Augmented Generation，检索增强生成。

在本项目中，它的意思是：

```text
先从医学知识库检索相关文档，
再把检索结果交给 LLM，
让 LLM 生成更有依据的回答。
```

## SentenceTransformers

SentenceTransformers 用来把文本转成向量。

当前默认模型：

```text
BAAI/bge-small-zh-v1.5
```

它适合中文语义检索。

加载方式：

```python
SentenceTransformer(model_path_or_name, device="cpu")
```

输入：

```text
高血压生活方式建议
```

输出：

```text
一个固定维度的浮点向量
```

## embedding 的作用

embedding 把文本变成向量后，语义相近的文本在向量空间中距离更近。

例如：

```text
血压高怎么办
高血压生活方式建议
```

字面不同，但语义接近，向量检索可以找到相关文档。

## Milvus Lite

Milvus 是向量数据库。Milvus Lite 是本地文件版本，不需要单独启动服务器。

当前数据库文件：

```text
medix-agent-swarm/knowledge/data/milvus_lite.db
```

连接方式：

```python
MilvusClient(db_path)
```

默认 collection：

```text
medical_knowledge
```

## collection schema

源码创建 collection 时指定：

```python
dimension = self.embedding_dim
metric_type = "COSINE"
auto_id = True
```

插入数据包含：

```python
{
    "vector": [...],
    "content": "...",
    "metadata": "{...json...}"
}
```

## 文档导入

导入脚本读取：

```text
knowledge/data/documents/*.txt
```

把每个文档转换为：

```python
{
    "id": "...",
    "content": "...",
    "metadata": {
        "type": "lifestyle",
        "disease": "hypertension",
        "source": "...",
        "filename": "..."
    }
}
```

然后调用：

```python
kb.add_documents(all_docs)
```

## 检索

检索函数：

```python
kb.search(query, top_k=5, filter_type=None)
```

流程：

```text
query -> embedding -> Milvus search -> hits -> content + metadata + score
```

如果指定 `filter_type`，会限制文档类型。

## 哪些工具依赖 RAG

| Skill | 依赖方式 |
|---|---|
| `search_knowledge` | 通用 Milvus 检索 |
| `recommend_lifestyle` | 检索 lifestyle 文档 |
| `disease_code` | 检索 disease_classification 文档 |
| `clinical_guideline` | 检索 clinical_guideline 文档 |
| `assess_risk` | 规则评估后查询补充建议 |
| `analyze_symptoms` | 症状分类后查询疾病信息 |
| `deep_research` | 与网络搜索并行使用 Milvus |

## score 的含义

代码中：

```python
"score": 1 - hit["distance"]
```

因为使用 COSINE 距离，score 越高表示越相似。

不同库和 metric 对 distance 的定义可能不同，所以 score 更适合相对排序，不宜当作绝对医学可信度。

## RAG 与医学可信度

检索到相关文档不等于答案一定正确。

原因：

- 文档可能过时。
- 文档可能覆盖范围有限。
- LLM 可能误读工具结果。
- 检索相关不等于医学证据等级高。

因此 ResearchAgent 还需要 EvidenceSynthesizer 和来源说明。

## 当前项目的坑

1. 首次加载 embedding 模型可能下载失败。
2. 本地 Milvus Lite 文件可能与当前文档不同步。
3. 删除或重新导入 collection 需要谨慎。
4. `metadata` 作为 JSON 字符串保存，filter 不够稳健。
5. 当前知识库只有 10 个 txt 文档。
6. 所有依赖 Milvus 的 Skills 都会受知识库初始化影响。

## 调试建议

按顺序验证：

```text
1. sentence_transformers 是否可导入
2. embedding 模型是否能加载
3. MilvusClient 是否能打开 db 文件
4. collection 是否存在
5. kb.search("高血压", top_k=1) 是否返回结果
6. Skill 是否能格式化结果
```

## 学习重点

RAG 工具链可以简化为：

```text
文本 -> 向量 -> 向量库检索 -> 文档片段 -> LLM 生成
```

本项目的 RAG 是 Agent Skills 的底层能力，不是独立的检索系统。
