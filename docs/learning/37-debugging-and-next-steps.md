# 调试路线与后续学习方向

## 学习目标

本文给出当前 workspace 的最小修复路线、验证顺序和后续扩展方向。适合在读完整套文档后使用。

## 总体原则

不要从“跑完整系统”开始。这个 workspace 同时包含：

- LLM API。
- function calling。
- Milvus Lite。
- Mem0。
- DDGS 网络搜索。
- Ray。
- FSDP。
- vLLM。
- HuggingFace datasets。
- PhysioNet。

任何一个环节失败都会让完整命令失败。正确方式是分层验证。

## 第一阶段：修复 medix-agent-swarm 导入

目标：让 Python 能导入核心模块。

需要处理：

```text
swarm/events.py
validation/auto_fixer.py
core/llm_client.py 配置路径
```

验证命令：

```bash
cd medix-agent-swarm
python -c "import validation; print('validation ok')"
python -c "import swarm; print('swarm ok')"
python -c "import core.llm_client; print('llm_client ok')"
```

预期：三个命令都能打印 ok。

## 第二阶段：验证 Skill 自动发现

目标：确认 `.claude/skills` 可被扫描并注册。

验证命令：

```bash
cd medix-agent-swarm
python -c "from pathlib import Path; from core.skill_loader import discover_skills; print([s['function_name'] for s in discover_skills(Path('.'))])"
```

预期能看到：

```text
search_knowledge
assess_risk
analyze_symptoms
recommend_lifestyle
disease_code
clinical_guideline
deep_research
search_history
search_similar_cases
```

如果少了某个 Skill，检查目录名、`SKILL.md`、`script/*.py` 和函数名是否一致。

## 第三阶段：验证知识库

目标：确认 Milvus Lite 和 embedding 可用。

验证命令：

```bash
cd medix-agent-swarm
python knowledge/scripts/import_hardcoded_data.py
```

风险：

- sentence-transformers 模型下载失败。
- Milvus Lite 文件权限问题。
- collection 已存在导致重复数据。

可以进一步写一个只调用 `MedicalKnowledgeBase.search()` 的小脚本做 smoke test。

## 第四阶段：验证 LLM API

目标：确认 `LLM_CONFIG` 可用。

最小验证：

```python
from core.llm_client import LLMClient
```

再验证普通 `chat()`。

最后再验证 `chat_with_tools()`。

注意：不要在文档或日志中打印真实 API key。

## 第五阶段：验证单 Agent

目标：在不启用复杂 Swarm 的情况下验证 AgentLoop。

建议先用简单问题：

```text
多喝水有什么好处？
```

再用需要知识库的问题：

```text
高血压患者饮食注意什么？
```

观察：

- 是否调用 Skill。
- 是否返回免责声明。
- 是否保存短期记忆。

## 第六阶段：验证 Swarm

目标：验证多 Agent 协作。

示例问题：

```text
胸闷气短冒冷汗，严重吗？需要看什么科？
```

观察：

- LeadAgent 是否分解多个 subtasks。
- 是否进入 Swarm 模式。
- DiagnosticAgent 和 ConsultationAgent 是否参与。
- 是否超时。
- 最终回答是否包含高危提醒。

## 第七阶段：验证 DeepResearch

目标：验证网络搜索和证据综合。

建议问题：

```text
高血压最新诊疗指南有哪些核心建议？
```

按层验证：

```text
DDGS 搜索
Milvus 检索
EvidenceSynthesizer
DeepResearchWorkflow
deep_research Skill
```

如果网络不可用，DeepResearch 不应作为基础功能验证项。

## 第八阶段：修复 MediX-R1 训练脚本

当前 `run_train.sh` 指向不存在的：

```text
examples/medix-r1_8b_dapo.sh
```

实际存在：

```text
examples/medix-r1_2b_dapo.sh
examples/medix-r1_8b_gspo.sh
```

先决定要跑哪个实验，再修改 `run_train.sh`。

不要直接按 README 中不存在的脚本运行。

## 第九阶段：训练 smoke test

目标：在小范围内验证训练链路。

建议：

1. 降低 batch size。
2. 降低 GPU 数。
3. 使用小数据或少量样本。
4. 先验证 dataloader。
5. 再验证 reward function。
6. 最后跑短训练。

需要重点关注：

- torch / CUDA / flash-attn 兼容性。
- vLLM 启动。
- Ray GPU 资源。
- reward function 的 LLM API。

## 第十阶段：评测 smoke test

目标：验证单任务评测。

建议先跑非 MIMIC 任务，例如 `medqa`。

流程：

```text
generate 一小部分
evaluate 一小部分
score
```

如果没有 GPU 跑 judge，可以使用 OpenRouter，但需要 API key。

## 后续扩展方向

### 文档方向

- 增加请求生命周期图。
- 增加训练生命周期图。
- 增加工具依赖图。
- 增加每个 Skill 的输入输出样例。
- 增加每个 eval task 的数据格式说明。

### 工程方向

- 修复缺失模块文件。
- 移除旧路径硬编码。
- 把 Skill schema 显式化。
- 强化约束系统为硬约束。
- 让 Swarm 约束真正参与路由。
- 把知识库文档来源和版本记录结构化。
- 增加小型离线测试。

### 研究方向

- 对比不同 reward 权重。
- 对比 LLM judge 与 embedding judge。
- 分析格式奖励饱和后的训练信号。
- 扩充医学知识库。
- 用本地 MediX-R1 模型接入 Agent 系统。

## 完成学习后的自测问题

你应该能回答：

1. `main.py` 到最终回答经过哪些对象。
2. Agent 如何把 Skill 转成 function calling tools。
3. Milvus 和 Mem0 的区别是什么。
4. DeepResearch 的四个阶段是什么。
5. `MediX-R1` reward function 的权重如何组成。
6. Ray、FSDP、vLLM 在训练中分别负责什么。
7. `eval.py` 的 generate、evaluate、score 各做什么。
8. 当前项目最先应该修复哪三个问题。

如果这些问题都能解释清楚，就已经具备继续维护和扩展该项目的基础。
