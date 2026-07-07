# 知识库与 RAG

## 学习目标

本文解释 `medix-agent-swarm` 如何用 Milvus Lite 和 SentenceTransformers 做医学知识检索，以及这些检索结果如何被 Skills 和 Agent 使用。

## 关键源码

```text
knowledge/milvus_kb.py
knowledge/scripts/import_hardcoded_data.py
knowledge/data/documents/*.txt
knowledge/data/milvus_lite.db
.claude/skills/search-knowledge/script/search.py
.claude/skills/recommend-lifestyle/script/lifestyle.py
.claude/skills/disease-code/script/code.py
.claude/skills/clinical-guideline/script/guideline.py
```

## RAG 在本项目中的角色

RAG 的作用是让 Agent 不只依赖 LLM 内部知识，而是可以检索本地医学文档。

本项目的 RAG 流程：

```text
txt 医学文档
  -> import_hardcoded_data.py
  -> SentenceTransformer 生成向量
  -> Milvus Lite 保存 content + metadata + vector
  -> Skill 查询
  -> 返回相关文档片段
  -> LLM 基于结果生成回答
```

## 当前知识库内容

当前 `knowledge/data/documents/` 下有 10 个 txt 文件，大致分为：

- 生活方式建议。
- ICD-10 疾病编码。
- 临床指南。
- 高危症状文档。

示例文件：

```text
01_lifestyle_hypertension.txt
02_lifestyle_diabetes.txt
10_icd10_cardiovascular.txt
20_guideline_hypertension.txt
21_guideline_diabetes.txt
```

当前知识库规模较小，应理解为演示知识库，不是完整医疗知识库。

## MedicalKnowledgeBase

核心类：

```python
class MedicalKnowledgeBase:
    ...
```

默认配置：

```python
db_path = "./knowledge/data/milvus_lite.db"
collection_name = "medical_knowledge"
embedding_model = "BAAI/bge-small-zh-v1.5"
```

它做四件事：

1. 加载 embedding 模型。
2. 连接 Milvus Lite。
3. 创建 collection。
4. 提供 `add_documents()` 和 `search()`。

## Embedding 模型加载

源码会先查本地 HuggingFace cache：

```text
~/.cache/huggingface/hub/models--BAAI--bge-small-zh-v1.5/snapshots
```

如果本地有 snapshot，就从本地加载；否则从模型名加载，可能触发下载。

这意味着首次运行 RAG 可能较慢，也可能因为网络受限失败。

## 文档导入流程

导入脚本：

```text
knowledge/scripts/import_hardcoded_data.py
```

它会：

1. 读取 `knowledge/data/documents/*.txt`。
2. 根据文件名前缀推断类型。
3. 构造 metadata。
4. 调用 `MedicalKnowledgeBase.add_documents()`。
5. 测试检索。

文件编号规则：

| 编号范围 | 类型 |
|---|---|
| 01-09 | lifestyle |
| 10-19 | disease_classification |
| 20-29 | clinical_guideline |
| 其他 | general |

## 文档分块

`_chunk_text()` 默认：

```python
chunk_size = 1024
overlap = 100
```

长文档会被切成重叠块，避免单条向量太长。

分块 metadata 会包含：

```text
doc_id
chunk_id
total_chunks
```

## 检索流程

`search(query, top_k=5, filter_type=None)` 做：

1. 用 embedding 模型把 query 转成向量。
2. 构造 Milvus filter。
3. 调用 `milvus_client.search()`。
4. 解析 `content` 和 `metadata`。
5. 返回带 score 的文档列表。

返回格式：

```python
{
    "id": hit["id"],
    "content": "...",
    "metadata": {...},
    "score": 1 - hit["distance"]
}
```

## 类型过滤

部分 Skill 会传入 `filter_type`：

```python
filter_type="lifestyle"
filter_type="disease_classification"
filter_type="clinical_guideline"
```

这使得同一个 Milvus collection 可以服务不同类型的工具。

注意：当前 filter 表达式是对 JSON 字符串做 `like` 匹配：

```python
metadata like "%\"type\": \"lifestyle\"%"
```

这种方式简单，但不如结构化字段过滤稳健。

## 哪些 Skill 使用知识库

| Skill | 查询方式 |
|---|---|
| `search_knowledge` | 不限制类型，通用搜索 |
| `recommend_lifestyle` | 过滤 `lifestyle` |
| `disease_code` | 过滤 `disease_classification` |
| `clinical_guideline` | 过滤 `clinical_guideline` |
| `assess_risk` | 规则判断后补充知识库建议 |
| `analyze_symptoms` | 规则分析后补充疾病信息 |
| `deep_research` | 并行使用 Milvus 和网络搜索 |

## RAG 的输入输出例子

输入：

```text
高血压 生活方式建议 饮食 运动 用药
```

过滤：

```text
filter_type = lifestyle
```

输出：

```text
高血压生活方式建议
一、饮食建议
1. 低盐饮食
...
```

Agent 再把这段作为工具结果交给 LLM，生成更自然的回答。

## 当前限制

1. 知识库规模小。
2. 文档来源没有统一引用元数据。
3. filter 使用字符串匹配，结构化程度不高。
4. 首次加载 embedding 模型成本高。
5. Milvus Lite 不适合高并发写入。
6. 检索结果是否被正确引用，取决于 LLM 最终回答。

## 学习重点

RAG 在本项目中不是独立问答引擎，而是 Skill 的底层能力：

```text
Agent 选择 Skill
  -> Skill 搜索 Milvus
  -> 返回文本证据
  -> LLM 综合生成回答
```

因此调试 RAG 时要分清：

- 是 Milvus 没检索到。
- 是 Skill 格式化有问题。
- 是 LLM 没正确使用工具结果。
