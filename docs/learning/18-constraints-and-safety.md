# 约束系统与医疗安全边界

## 学习目标

本文解释项目如何通过约束文件、运行时验证和自动修复来控制医疗回答风险。

## 关键源码

```text
constraints/agent_constraints.yaml
constraints/swarm_constraints.yaml
constraints/validator.py
validation/auto_fixer_20260428_231043.py
validation/__init__.py
core/agent_loop.py
```

## 为什么需要约束

医疗问答有高风险。Agent 不能：

- 给出明确诊断。
- 开具体处方。
- 延误高危症状就医。
- 给出保证治愈的说法。
- 把在线建议包装成医生诊断。

项目用 Harness Engineering 的思路把这些规则显式写成约束。

## Agent 约束文件

`constraints/agent_constraints.yaml` 定义每个 Agent 的：

- 能力标签。
- 推荐工具。
- 禁止行为。
- 输出约束。

例如 ConsultationAgent：

```yaml
forbidden_actions:
  - diagnose_disease
  - prescribe_medication
  - guarantee_cure

output_constraints:
  - must_include_disclaimer
  - max_response_length: 2000
```

DiagnosticAgent 要求高危症状建议就医。

ResearchAgent 要求引用来源或证据。

## Swarm 约束文件

`constraints/swarm_constraints.yaml` 定义：

- 最大 Agent 数。
- 最大并行任务数。
- 超时时间。
- 任务分解规则。
- Agent 选择规则。
- 质量控制。
- 冲突解决策略。

例如高危症状：

```yaml
if_symptoms:
  - 胸痛
  - 呼吸困难
  - 心悸
  - 昏厥
must_include:
  - diagnostic_agent
```

当前主流程对这些规则的使用还不完整，更多是约束基础设施。

## ConstraintValidator

`constraints/validator.py` 提供三个主要能力：

### validate_tool_call()

检查某个 Agent 调用某个工具是否在推荐列表中。

如果不在列表中，当前返回 warning，不阻断执行。

这意味着 allowed tools 不是硬权限，而是软约束。

### validate_output()

检查最终回答是否违反输出约束。

当前检查包括：

- 是否缺少免责声明。
- 是否超过最大长度。
- 高危症状是否建议就医。
- ResearchAgent 是否引用来源。
- 是否出现明确诊断语句。
- 是否包含具体药物处方模式。

### get_required_agents()

根据问题关键词推荐必须包含的 Agent。

例如检测到“胸痛”时推荐 `diagnostic_agent`。

但当前 `SwarmCoordinator` 主路由没有强制调用这个方法。

## AutoFixer

自动修复器定义在：

```text
validation/auto_fixer_20260428_231043.py
```

它可以：

- 添加免责声明。
- 给高危症状添加紧急就医提醒。
- 截断过长输出。
- 简单替换明确诊断语句。

例如缺少免责声明时追加：

```text
【免责声明】
以上信息仅供参考，不能替代专业医生的诊断和治疗。如有疑虑，请及时就医。
```

## 当前导入问题

`validation/__init__.py` 当前写的是：

```python
from .auto_fixer import AutoFixer
```

但目录中没有 `auto_fixer.py`，只有：

```text
auto_fixer_20260428_231043.py
```

因此 `from validation import AutoFixer` 会失败。

这会导致 `AgentLoop` 中的约束修复模块不可用。

## AgentLoop 如何使用约束

`core/agent_loop.py` 顶部尝试导入：

```python
from constraints import ConstraintValidator
from validation import AutoFixer
```

如果成功：

- 初始化 `self.validator`。
- 初始化 `self.auto_fixer`。

工具调用前：

```python
validation_result = self.validator.validate_tool_call(agent.agent_id, tool_call.name)
```

最终回答后：

```python
validation_result = self.validator.validate_output(agent.agent_id, final_answer)
```

如果可自动修复：

```python
fixed_answer = self.auto_fixer.fix_output(final_answer, auto_fixable)
```

## 医疗安全边界

这个项目的正确安全边界应该是：

- 可以提供健康科普。
- 可以提示风险等级。
- 可以建议就医。
- 可以说明可能相关疾病或鉴别思路。
- 可以引用指南或证据。
- 不能确诊。
- 不能替代医生。
- 不能开处方。
- 高危症状必须建议立即就医。

## 当前限制

1. 约束验证对工具调用主要是 warning，不是硬阻断。
2. AutoFixer 文件名错误导致导入失败。
3. 输出安全主要靠文本匹配，可能误报或漏报。
4. 高危症状判断依赖关键词。
5. ResearchAgent 的引用来源检查比较粗略。
6. `swarm_constraints.yaml` 的 Agent 选择规则没有完整接入 LeadAgent 路由。

## 学习重点

安全系统要按三层理解：

```text
Prompt 层：告诉 Agent 不能做什么
约束层：运行时检查工具和输出
修复层：对可修复问题自动补充免责声明或就医提醒
```

当前项目已经有这三层的雏形，但运行前需要修复导入问题，并进一步强化硬约束。
