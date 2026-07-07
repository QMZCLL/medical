# 训练工具

## 学习目标

本文解释 `MediX-R1/training` 使用的主要训练工具：veRL/EasyR1、Ray、FSDP、vLLM、torch、transformers、wandb 和相关数据处理工具。

## 工具地图

| 工具 | 作用 |
|---|---|
| veRL/EasyR1 | RL 训练框架基础 |
| Ray | 分布式任务调度 |
| FSDP | 分布式模型参数切分训练 |
| vLLM | 高吞吐 rollout 生成 |
| torch | 张量计算和模型训练 |
| transformers | 模型、tokenizer、processor |
| datasets | 加载训练和评测数据 |
| qwen-vl-utils | 处理 Qwen VL 视觉输入 |
| wandb | 训练日志和实验追踪 |
| flash-attn | 高效 attention |

## veRL/EasyR1

`training/verl/` 是训练框架代码，结构接近 veRL/EasyR1。

它提供：

- 配置结构。
- 数据加载。
- Ray worker 管理。
- PPO/GRPO/RLOO 类训练流程。
- reward function 接口。
- checkpoint 管理。

在本项目中，训练入口是：

```bash
python3 -m verl.trainer.main
```

## Ray

Ray 负责分布式执行。

使用位置：

```text
verl/trainer/main.py
verl/trainer/ray_trainer.py
verl/single_controller/
```

核心概念：

- remote actor：远程执行对象。
- resource pool：GPU 资源池。
- worker group：一组分布式 worker。

训练开始时会：

```python
ray.init(runtime_env=runtime_env)
```

`Runner` 和 `AutoRewardManager` 都是 Ray remote actor。

## FSDP

FSDP 是 Fully Sharded Data Parallel。

作用：

```text
把大模型参数、梯度、优化器状态切分到多张 GPU 上，降低单卡显存压力。
```

本项目使用：

```text
verl/workers/fsdp_workers.py
```

配置项包括：

```yaml
worker.actor.fsdp.enable_full_shard: true
worker.actor.fsdp.enable_cpu_offload: false
worker.actor.fsdp.enable_rank0_init: true
```

## vLLM

vLLM 在训练中用于 rollout 生成。

配置位于：

```yaml
worker.rollout:
  n: 5
  temperature: 1.0
  top_p: 1.0
  gpu_memory_utilization: 0.3
  tensor_parallel_size: 1
```

rollout 的意思是：

```text
给定 prompt，让当前策略模型生成 response，用于后续 reward 和策略更新。
```

vLLM 的优势是生成吞吐高，但它和 FSDP、模型权重同步的配合复杂。

## torch

torch 提供：

- tensor。
- autograd。
- distributed。
- optimizer。
- FSDP 底层能力。

训练配置中 batch、mask、reward tensor、advantage、KL 都最终落到 torch tensor 上。

## transformers

transformers 用于：

- 加载 tokenizer。
- 加载 processor。
- 支持 Qwen VL 类模型。
- 使用 chat template。

`get_tokenizer()` 和 `get_processor()` 会根据模型路径加载对应组件。

视觉语言模型必须同时关注 tokenizer 和 processor。

## datasets

`RLHFDataset` 使用 HuggingFace `datasets`：

- `load_dataset()`
- `load_from_disk()`
- parquet 加载。

支持 HuggingFace 名称、本地目录和本地 parquet 文件。

## wandb

配置中：

```yaml
trainer.logger: ["file", "wandb"]
```

说明训练日志可以写入本地文件和 Weights & Biases。

如果没有配置 wandb 环境，可能需要关闭或登录。

## flash-attn

flash-attn 提供更高效 attention。

它对环境敏感：

- CUDA 版本。
- PyTorch 版本。
- Python 版本。
- GPU 架构。

安装问题通常不是业务代码问题，而是环境兼容问题。

## 常见训练工具问题

| 问题 | 可能原因 |
|---|---|
| Ray 报 GPU 不足 | `n_gpus_per_node` 大于实际 GPU |
| flash-attn 安装失败 | CUDA/PyTorch 不匹配 |
| vLLM 启动失败 | 显存不足或版本不兼容 |
| dataset 加载失败 | 网络、路径、split 错误 |
| reward 很慢 | LLM judge API 延迟或限流 |
| wandb 报错 | 未登录或网络不可用 |

## 学习重点

训练工具链的核心关系：

```text
datasets 提供样本
transformers 处理模型输入
Ray 调度 worker
FSDP 承载训练
vLLM 生成 rollout
reward function 反馈分数
wandb/file 记录指标
```

每个工具负责一段链路，调试时要定位问题属于哪一段。
