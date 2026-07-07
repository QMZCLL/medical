# README 与源码事实对照

## 为什么需要这篇文档

当前项目 README 中包含不少“已完成”“路线图”“能力描述”。这些描述对理解项目意图有帮助，但学习和运行项目时必须以当前源码为准。

本文把已确认的源码事实和 README 说法分开，避免把愿景当成已经稳定可用的功能。

## 已确认的源码事实

### workspace 不是 git 仓库

在根目录执行：

```bash
git status --short
```

结果是：

```text
fatal: not a git repository (or any of the parent directories): .git
```

因此当前 workspace 无法在根目录直接做常规 git status、commit、branch 操作。

### 现有文档很少

当前已有 Markdown 文档主要是：

```text
MediX-R1/eval/README.md
MediX-R1/training/README.md
medix-agent-swarm/README.md
```

系统性的学习文档此前不存在。

### medix-agent-swarm 有导入阻塞

源码中多处导入：

```python
from .events import Event, EventType
```

但当前目录实际文件是：

```text
medix-agent-swarm/swarm/events_20260428_231035.py
```

没有：

```text
medix-agent-swarm/swarm/events.py
```

因此导入 `swarm` 会失败。

类似地，`validation/__init__.py` 导入：

```python
from .auto_fixer import AutoFixer
```

但当前目录实际文件是：

```text
medix-agent-swarm/validation/auto_fixer_20260428_231043.py
```

没有：

```text
medix-agent-swarm/validation/auto_fixer.py
```

这会导致 `import validation` 失败。

### LLM 配置导入路径存在旧路径

`medix-agent-swarm/core/llm_client.py` 中有：

```python
sys.path.insert(0, '/Users/saintgeo/Desktop/self-learn/swarm')
from config import LLM_CONFIG
```

当前 workspace 的配置文件位于：

```text
/home/qmz/projects/learning-agent/medical/config.py
```

这说明源码里还保留了旧机器或旧目录路径。学习时要注意区分“代码意图”和“当前路径可运行性”。

### Swarm 不是完全去中心化认领

README 强调 Agent Swarm 的去中心化和自主认领，但当前源码中 `LeadAgent` 要求输出 `assigned_agent`，`SharedContext` 的 `SubTask` 也包含 `assigned_agent` 字段。

当前更准确的说法是：

```text
LeadAgent 基于 LLM 做任务分解和 Agent 指派；
Worker Agent 并行执行分配给自己的任务；
结果通过 SharedContext 汇总。
```

这仍然是多 Agent 协作，但不是完全自由竞争式任务认领。

### medix-agent-swarm 的知识库是小型演示库

当前知识库文档数量是 10 个 txt 文件：

```text
knowledge/data/documents/
```

同时存在一个本地 Milvus Lite 文件：

```text
knowledge/data/milvus_lite.db
```

这说明它更像演示知识库，不是大规模医学知识库。

### MediX-R1 训练脚本与 README 不完全一致

`MediX-R1/training/README.md` 列出多个训练脚本，例如 7B、8B、30B 的 DAPO/GRPO 脚本。

当前实际看到的训练脚本包括：

```text
examples/medix-r1_2b_dapo.sh
examples/medix-r1_8b_gspo.sh
```

`run_train.sh` 当前调用：

```bash
bash examples/medix-r1_8b_dapo.sh
```

但该文件当前不存在。因此训练前需要先修正脚本路径或补齐脚本。

## README 中有价值但需谨慎理解的内容

README 的价值主要在于告诉你项目设计意图：

- `medix-agent-swarm` 想做 Skills-Agent 两层架构。
- 它希望通过 LLM function calling 自主调用医疗 Skills。
- 它希望整合短期记忆、长期记忆、Milvus 知识库、DeepResearch 和 Harness 约束。
- `MediX-R1` 想提供医学 VLM 的训练和评测流程。

这些方向与源码大体一致，但实现状态需要逐项核对。

## 阅读源码时的判断方法

遇到 README 与源码不一致时，建议按下面优先级判断：

1. 当前存在的源码文件和导入路径。
2. 当前脚本实际调用的文件。
3. 当前 requirements 和配置文件。
4. README 中的运行命令。
5. README 中的路线图和能力声明。

例如：

- README 说有 `validation/auto_fixer.py`，但文件不存在，则以文件树为准。
- README 说有多个训练脚本，但目录里不存在，则以当前目录为准。
- README 说“所有测试通过”，但当前导入都失败，则不能把测试通过当成当前事实。

## 对学习文档的写作影响

本套学习文档会采取下面规则：

- 讲架构意图时会说明 README 的设计目标。
- 讲运行链路时只引用当前源码。
- 讲可运行性时明确列出阻塞点。
- 讲工具链时说明哪些工具是核心依赖，哪些工具当前可能因为配置、依赖或网络无法直接运行。

这样做的目的不是否定 README，而是让学习者知道项目现在处于什么状态，以及下一步应该怎么修复。
