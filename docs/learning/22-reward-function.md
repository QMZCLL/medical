# 医学 Reward Function

## 学习目标

本文解释 `examples/reward_function/medical.py` 如何计算医学训练奖励，以及各分项奖励代表什么。

## 关键源码

```text
MediX-R1/training/examples/reward_function/medical.py
MediX-R1/training/verl/workers/reward/function.py
MediX-R1/training/images/train/training_report.md
```

## reward 在训练中的位置

训练时模型生成 response 后，`AutoRewardManager` 会调用自定义 reward function。

当前医学 reward：

```text
examples/reward_function/medical.py:compute_score
```

它的输入是一批样本：

```python
reward_inputs: list[dict[str, str]]
```

每个元素包含：

- `response`
- `response_length`
- `ground_truth`

输出是一批分数：

```python
{
    "overall": ...,
    "format": ...,
    "accuracy_llm": ...,
    "accuracy_embed": ...,
    "modality": ...
}
```

## 总分权重

代码中定义：

```python
ACCURACY_LLM_WEIGHT = 0.575
ACCURACY_EMBED_WEIGHT = 0.375
MODALITY_WEIGHT = 0.05
```

`compute_score()` 默认：

```python
format_weight = 0.1
content_weight = 1 - format_weight
```

总分：

```text
overall =
  content_weight * 0.575 * accuracy_llm
+ content_weight * 0.375 * accuracy_embed
+ content_weight * 0.05  * modality
+ format_weight * format
```

默认权重展开：

| 分项 | 实际权重 |
|---|---|
| LLM 准确性 | 51.75% |
| embedding 相似度 | 33.75% |
| 模态匹配 | 4.5% |
| 格式 | 10% |

## 格式奖励

`format_reward()` 检查输出是否符合：

```text
<thinking>...</thinking><answer>...</answer>
```

逻辑：

1. 找 `<thinking>`。
2. 从这个位置开始裁剪。
3. 用正则 fullmatch。
4. 匹配则 1.0，否则 0.0。

这鼓励模型输出固定推理格式。

## LLM 准确性奖励

`accuracy_reward_llm()` 会：

1. 从 `<answer>` 标签中提取预测答案。
2. 从 ground truth 中提取标准答案。
3. 检查无效答案。
4. 如果完全相同，返回 1.0。
5. 否则调用远程 LLM judge 判断 YES/NO。

judge prompt 要求模型只回答：

```text
YES
NO
```

如果包含 YES，则得 1.0，否则 0.0。

## embedding 准确性奖励

`accuracy_reward_embed()` 使用：

```python
SentenceTransformer("abhinand/MedEmbed-large-v0.1", trust_remote_code=True)
```

它会计算预测答案和标准答案的 embedding cosine similarity。

默认阈值：

```text
0.8
```

相似度达到阈值则得 1.0，否则 0.0。

这个奖励比 LLM judge 更稳定，但可能把语义相近但医学上关键差异的答案误判为正确。

## 模态奖励

`modality_reward()` 检查模型在 `<thinking>` 前预测的图像模态是否与 ground truth 前缀一致。

逻辑：

```text
predict_modality = predict 中 <thinking> 前的内容
ground_truth_modality = ground_truth 中第一个 > 之前的内容加 >
```

两者大写后完全相同则得 1.0。

这鼓励模型识别医学图像模态，例如 X-ray、CT、MRI 等。

## 无效答案过滤

`INVALID_SYMBOLS`：

```python
["__", ":", "?"]
```

如果答案为空、长度为 1，或包含这些符号，则视为无效答案。

无效答案在准确性奖励中得 0。

## 并行计算

`compute_score()` 用：

```python
ThreadPoolExecutor(max_workers=16)
```

并行计算每条样本奖励。

这对 LLM judge API 调用有帮助，但也可能带来并发请求压力。

## 与 AutoRewardManager 的关系

`AutoRewardManager` 会加载 reward function 文件。

如果 reward type 是 batch，调用：

```python
compute_reward_batch()
```

它会把 reward 的 `overall` 写入 response 最后一个有效 token：

```python
reward_tensor[i, cur_response_length - 1] = score["overall"]
```

同时把分项写入 metrics，供日志和训练报告使用。

## 当前 reward 风险

1. LLM judge 依赖外部 API 和 `config.py`。
2. `ground_truth.split(">", maxsplit=1)[1]` 假设 ground truth 一定含 `>`。
3. MedEmbed 模型首次加载需要下载。
4. embedding 阈值是硬编码。
5. 格式奖励容易饱和，对后期训练贡献有限。
6. ThreadPool 并发请求可能触发 API 限流。

## 与训练报告的关系

`images/train/training_report.md` 中解释了：

- `reward/accuracy_llm`
- `reward/accuracy_embed`
- `reward/format`
- `reward/modality`
- `critic/score/mean`

这些指标都来自 reward function 的分项输出。

## 学习重点

这个 reward function 是 composite reward：

```text
格式是否正确
答案是否被 LLM judge 判为正确
答案 embedding 是否接近标准答案
图像模态是否识别正确
```

理解 reward，才能理解训练曲线为什么会出现格式奖励快速饱和、embedding 奖励高于 LLM 奖励、响应长度和熵波动等现象。
