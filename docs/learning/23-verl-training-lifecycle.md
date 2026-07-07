# veRL 训练生命周期

## 学习目标

本文解释 `MediX-R1/training/verl` 中训练生命周期的核心流程，帮助你从框架角度理解一次 RL 训练如何运转。

## 关键源码

```text
verl/trainer/main.py
verl/trainer/ray_trainer.py
verl/trainer/data_loader.py
verl/workers/fsdp_workers.py
verl/workers/reward/function.py
verl/utils/dataset.py
verl/protocol.py
```

## 生命周期总览

```text
main()
  -> 读取配置
  -> 初始化 Ray
  -> Runner.run()
  -> 加载 tokenizer / processor
  -> 创建资源池
  -> 创建 reward actor
  -> 创建 train / val dataloader
  -> RayPPOTrainer.init_workers()
  -> RayPPOTrainer.fit()
```

## 配置加载

`verl/trainer/main.py` 使用 OmegaConf：

```python
cli_args = OmegaConf.from_cli()
default_config = OmegaConf.structured(PPOConfig())
file_config = OmegaConf.load(config_path)
ppo_config = OmegaConf.merge(default_config, file_config, cli_args)
```

然后：

```python
ppo_config.deep_post_init()
```

这一步会把字符串形式的配置转换为内部结构，并处理 reward function 路径等派生字段。

## Ray 初始化

如果 Ray 未初始化：

```python
ray.init(runtime_env=runtime_env)
```

runtime env 设置了：

- tokenizer 并行。
- NCCL 日志级别。
- vLLM 日志级别。
- PyTorch CUDA 内存相关配置。

这些环境变量会影响分布式训练稳定性。

## Runner

`Runner` 是：

```python
@ray.remote(num_cpus=1)
class Runner:
```

它在 Ray 中运行训练主逻辑。

主要动作：

1. 打印配置。
2. `get_tokenizer()`。
3. `get_processor()`。
4. 设置 worker mapping。
5. 创建资源池。
6. 创建 reward manager。
7. 创建 dataloader。
8. 创建 trainer。
9. 初始化 workers。
10. 开始训练。

## 角色映射

`Role` 中定义了多个角色：

- Actor
- Rollout
- Critic
- RefPolicy
- RewardModel
- ActorRolloutRef

当前主流程使用：

```python
role_worker_mapping = {
    Role.ActorRolloutRef: ray.remote(FSDPWorker),
    Role.Critic: ray.remote(FSDPWorker),
}
```

即 Actor、Rollout、Ref 共享一个 Worker 类型。

## 资源池

`ResourcePoolManager` 根据配置创建 GPU 资源池：

```python
resource_pool_spec = {
    "global_pool": [config.trainer.n_gpus_per_node] * config.trainer.nnodes
}
```

它会检查 Ray 可用 GPU 数：

```python
if gpus_available < gpus_required:
    raise ValueError(...)
```

所以如果配置写 8 卡，但机器只有 1 卡，会在这里失败。

## Dataloader

`create_dataloader()` 创建：

- `train_dataloader`
- `val_dataloader`

底层数据集是 `RLHFDataset`。

训练集使用：

- `RandomSampler`，带 seed。
- `StatefulDataLoader`，方便 checkpoint 恢复。
- `drop_last=True`。

验证集使用：

- 顺序加载。
- `drop_last=False`。

## RewardManager

`AutoRewardManager` 被包装成 Ray remote actor：

```python
RemoteRewardManager = ray.remote(AutoRewardManager).options(num_cpus=config.worker.reward.num_cpus)
reward_fn = RemoteRewardManager.remote(config.worker.reward, tokenizer)
```

训练和验证各有一个 reward actor。

这样 reward 计算可以和模型计算异步并行。

## RayPPOTrainer 初始化检查

`RayPPOTrainer.__init__()` 做多项检查：

- rollout batch size 是否能整除 actor global batch size。
- rollout batch size 乘以 rollout.n 是否能整除 experience micro batch。
- 如果使用 critic，也检查 critic batch。
- GRPO/RLOO 要求 `rollout.n > 1`。
- 根据 dataloader 和 epoch 计算训练步数。

这些检查能在训练前暴露配置错误。

## KL 与 advantage

如果 `disable_kl=True`：

```python
self.use_reference_policy = False
self.kl_ctrl = FixedKLController(init_kl_coef=0.0)
```

否则使用 reference policy 和 KL controller。

advantage 由：

```python
compute_advantage_return()
```

根据不同 estimator 计算。

## Worker 初始化

`init_workers()` 创建 resource pool 和 colocated worker。

hybrid engine 下，Actor/Rollout/Ref 放在同一个资源池中。

这可以减少模型重复加载，但需要更复杂的状态切换。

## 训练循环

`fit()` 的完整细节较多，可以先理解抽象流程：

```text
采样 prompt batch
  -> rollout 生成 response
  -> reward function 打分
  -> 计算 old log probs / ref log probs
  -> 计算 KL 和 advantage
  -> actor 更新
  -> 记录 metrics
  -> 保存 checkpoint
  -> 定期验证
```

医学训练中，reward function 是理解训练行为的关键。

## checkpoint

checkpoint 相关代码在：

```text
verl/utils/checkpoint/
```

训练脚本指定：

```bash
trainer.save_checkpoint_path=...
```

保存后可用：

```bash
bash merge_model.sh
```

把分片 checkpoint 合并成 HuggingFace 模型。

## 学习重点

veRL 训练生命周期可以记成四组对象：

```text
配置：PPOConfig + OmegaConf
数据：RLHFDataset + StatefulDataLoader
执行：Ray + ResourcePool + FSDPWorker
反馈：AutoRewardManager + medical.py
```

掌握这四组对象之间的关系，就能读懂大部分训练流程。
