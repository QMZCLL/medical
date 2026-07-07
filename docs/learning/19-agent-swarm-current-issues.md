# medix-agent-swarm 当前问题清单

## 学习目标

本文汇总当前 `medix-agent-swarm` 中影响学习、运行和验证的主要问题。它不是批评列表，而是帮助你知道应该先修哪里。

## 问题一：缺失 swarm/events.py

多处源码导入：

```python
from .events import Event, EventType
```

涉及文件：

```text
swarm/__init__.py
swarm/shared_context.py
swarm/lead_agent.py
swarm/swarm_coordinator.py
```

但当前目录里实际文件是：

```text
swarm/events_20260428_231035.py
```

没有：

```text
swarm/events.py
```

直接影响：

```bash
python -c "import swarm"
```

会报：

```text
ModuleNotFoundError: No module named 'swarm.events'
```

推荐修复：

- 将 `events_20260428_231035.py` 复制或重命名为 `events.py`。
- 或修改所有导入指向带时间戳的文件。

更推荐前者，因为 README 和源码都按 `events.py` 这个稳定模块名设计。

## 问题二：缺失 validation/auto_fixer.py

`validation/__init__.py` 中写：

```python
from .auto_fixer import AutoFixer
```

但当前目录里实际文件是：

```text
validation/auto_fixer_20260428_231043.py
```

没有：

```text
validation/auto_fixer.py
```

直接影响：

```bash
python -c "import validation"
```

会报：

```text
ModuleNotFoundError: No module named 'validation.auto_fixer'
```

这会导致 `AgentLoop` 中的约束自动修复不可用。

推荐修复：

- 将带时间戳的文件复制或重命名为 `auto_fixer.py`。
- 或修改 `validation/__init__.py` 导入。

## 问题三：LLM 配置路径硬编码

`core/llm_client.py` 中有：

```python
sys.path.insert(0, '/Users/saintgeo/Desktop/self-learn/swarm')
from config import LLM_CONFIG
```

当前 workspace 的 `config.py` 在：

```text
/home/qmz/projects/learning-agent/medical/config.py
```

直接影响：

- 在当前机器上，旧路径不存在或不是目标配置目录。
- LLMClient 可能导入失败或导入错误配置。

推荐修复：

- 根据 `__file__` 推导 workspace 根目录。
- 或通过环境变量指定配置路径。
- 或把配置读取改成显式模块路径。

## 问题四：README 与源码事实不完全一致

README 中说法包括：

- 所有测试通过。
- 自主任务认领。
- 多个工具和阶段完成。

但当前源码存在导入阻塞，所以不能把 README 的测试结论当作当前可运行事实。

另外，当前任务分配依赖 `assigned_agent`，并不是完全自由认领。

推荐处理：

- 文档中明确区分“设计意图”和“当前源码行为”。
- 测试结论以当前环境重新运行结果为准。

## 问题五：Skill 参数 schema 过于粗略

当前参数类型推断规则很简单：

```text
参数名含 count/limit/max/iterations -> number
其他 -> string
```

影响：

- 复杂参数无法准确表达。
- LLM 可能传错类型。
- 参数描述只是从参数名生成，信息量不足。

推荐修复：

- 在 `SKILL.md` 中定义更明确的 schema。
- 或给 Skill 函数增加类型注解解析。
- 或为每个 Skill 提供显式参数定义。

## 问题六：约束系统是软约束

`validate_tool_call()` 发现 Agent 使用非推荐工具时只返回 warning，不阻止执行。

这意味着：

- Agent 仍能调用全部注册 Skills。
- `allowed_tools` 目前不是权限系统。

推荐修复：

- 区分 warning 和 blocking。
- 对医疗高风险行为设置硬阻断。
- 在日志和最终结果中记录约束修复情况。

## 问题七：DeepResearch 依赖链较长

DeepResearch 依赖：

- LLM 查询规划。
- DDGS 网络搜索。
- Milvus 检索。
- LLM 证据综合。

任何一层失败都会影响结果。

推荐调试顺序：

1. 单独验证 LLMClient。
2. 单独验证 DDGS 搜索。
3. 单独验证 Milvus 检索。
4. 再验证 DeepResearchWorkflow。

## 问题八：知识库规模较小

当前只有 10 个 txt 文档和一个小型 Milvus Lite 文件。

影响：

- 对高血压、糖尿病等示例问题效果较好。
- 对广泛医学问题覆盖不足。
- DeepResearch 的本地知识库部分证据有限。

推荐扩展：

- 增加文档来源 metadata。
- 增加指南、药物、症状、疾病分类文档。
- 增加导入前去重和版本管理。

## 问题九：process_with_swarm 每次创建协调器

当前便捷函数：

```python
coordinator = SwarmCoordinator(enable_swarm=enable_swarm)
return await coordinator.process(...)
```

每次请求都会创建新的协调器。

影响：

- 可能重复初始化 Agent。
- 可能重复加载知识库或 embedding 模型。
- 性能不稳定。

不过短期记忆使用单例，能在同进程中部分共享状态。

推荐修复：

- 在服务化场景中复用 SwarmCoordinator。
- 把重型资源初始化做成显式单例或依赖注入。

## 推荐修复顺序

最小可运行路线：

```text
1. 补齐 swarm/events.py
2. 补齐 validation/auto_fixer.py
3. 修复 core/llm_client.py 配置路径
4. 安装 requirements
5. import validation / swarm / core.llm_client
6. 验证 Skill 自动发现
7. 验证 Milvus 知识库
8. 跑一个简单 AgentLoop 问题
9. 再验证 Swarm 和 DeepResearch
```

## 学习重点

这些问题不妨碍理解架构，但会影响实际运行。学习时要把两件事分开：

- 架构学习：理解模块如何设计。
- 工程修复：让当前源码在本环境中跑起来。

先读懂，再修复，会比直接跑测试更有效。
