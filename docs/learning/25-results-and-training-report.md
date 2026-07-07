# 训练结果与指标报告解读

## 学习目标

本文解释 `MediX-R1/images/train/training_report.md` 中的训练指标，帮助你把 reward function 和训练曲线联系起来。

## 相关文件

```text
MediX-R1/images/train/training_report.md
MediX-R1/images/train/actor-loss.png
MediX-R1/images/train/actor-entropy-loss.png
MediX-R1/images/train/response-length-mean.png
MediX-R1/images/train/reward-accuracy_llm.png
MediX-R1/images/train/reward-accuracy_embed.png
MediX-R1/images/train/reward-format.png
MediX-R1/images/train/reward-modality.png
MediX-R1/images/train/critic-mean.png
```

报告对象：

```text
MediX-R1 2B DAPO
模型：Qwen3-VL-2B-Instruct
训练步数：约 600 steps
```

## actor/pg_loss

`actor/pg_loss` 是策略梯度损失。

报告中的趋势：

- 初期从约 0.10 快速下降。
- 中期在 0 到 0.03 之间波动。
- 后期接近 0。

理解方式：

```text
损失下降到接近 0，说明策略更新幅度变小，训练趋于稳定。
```

如果出现极端发散，才需要重点警惕。

## actor/entropy_loss

熵反映模型输出多样性。

报告中观察到：

- 初期缓慢下降。
- step 200-400 出现明显下降。
- step 400 后恢复。

解释：

```text
熵下降 + 响应长度缩短，可能说明模型一度倾向生成更短、更单一的回答。
后续恢复说明策略没有永久坍缩。
```

熵和 response length 要一起看。

## response_length/mean

平均响应长度反映输出行为。

报告中：

- 初期从约 220 降到 165。
- 中期最低到约 95 token。
- 后期反弹到约 305，再稳定到约 220。

这与熵曲线一致，说明模型在训练中探索过“短回答策略”，后来又恢复较完整输出。

## reward/accuracy_llm

这是 LLM judge 判断答案正确性的奖励。

报告中：

- 从约 0.03 上升到约 0.22。
- 后期稳定在 0.20-0.25。

含义：

```text
2B 模型医学回答准确性有提升，但绝对值仍低。
```

这也是总奖励中权重最高的分项。

## reward/accuracy_embed

这是答案与标准答案的医学 embedding 相似度奖励。

报告中：

- 从约 0.03 上升到约 0.45。
- 后期稳定在 0.43-0.50。

它高于 LLM judge 奖励，说明模型更容易学到语义接近标准答案，而不一定被 judge 判为完全正确。

## reward/format

格式奖励检查：

```text
<thinking>...</thinking><answer>...</answer>
```

报告中：

- 很快接近 1.0。
- 后续几乎满分。

含义：

```text
格式是最容易学习的奖励，早期快速饱和。
```

一旦饱和，它对后续训练提供的区分度就变弱。

## reward/modality

模态奖励检查模型是否正确识别医学图像模态。

报告中：

- step 70-130 从约 0.10 跃升到约 0.85。
- 后续维持高位。

这说明模型快速发现了模态识别策略，是训练中的明显能力跃迁。

## critic/score/mean

`critic/score/mean` 是 batch 中 `reward/overall` 的平均综合表现。

根据 reward function，默认权重：

| 分项 | 权重 |
|---|---|
| accuracy_llm | 51.75% |
| accuracy_embed | 33.75% |
| format | 10% |
| modality | 4.5% |

报告中：

- 前 100 步快速从约 0.13 上升到约 0.35。
- 后续在 0.30-0.38 区间波动。

说明模型很快学到基本策略，之后进入平台期。

## 如何关联 reward 与曲线

可以这样看：

```text
format 很快满分
modality 快速跃升
accuracy_embed 稳步提高
accuracy_llm 提升较慢且绝对值低
overall 受 accuracy_llm 和 accuracy_embed 主导
```

因此平台期主要不是格式问题，而是医学答案准确性进一步提升困难。

## 可能的改进方向

从报告推断，后续可以考虑：

1. 提高训练步数，观察平台期是否继续突破。
2. 使用更大模型。
3. 改进 LLM judge 稳定性。
4. 调整 embedding 阈值或引入更细粒度奖励。
5. 防止格式奖励过早饱和后失去训练信号。
6. 监控响应长度和熵，避免短回答策略。

## 学习重点

读训练报告时不要只看 overall。应按 reward function 拆开：

```text
格式是否学会
模态是否学会
语义是否接近
医学正确性是否被 judge 认可
输出长度和熵是否健康
```

这样才能判断模型到底在哪些能力上提升，哪些能力仍然不足。
