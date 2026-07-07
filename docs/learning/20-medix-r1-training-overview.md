# MediX-R1 训练项目总览

## 学习目标

本文解释 `MediX-R1/training` 的整体训练架构。读完后，你应该知道训练脚本如何进入 veRL/EasyR1 风格训练框架，以及模型、数据、rollout 和 reward 如何连接。

## 项目定位

`MediX-R1` 是医学多模态推理模型训练与评测代码。`training/` 目录基于 veRL/EasyR1 风格实现强化学习训练。

核心目标：

```text
用医学 VQA 数据训练视觉语言模型，使模型学会医学图像问答、格式化推理和模态识别。
```

## 目录结构

```text
MediX-R1/training/
├── README.md
├── run_train.sh
├── merge_model.sh
├── requirements.txt
├── setup.py
├── pyproject.toml
├── examples/
│   ├── config.yaml
│   ├── medix-r1_2b_dapo.sh
│   ├── medix-r1_8b_gspo.sh
│   ├── format_prompt/
│   └── reward_function/
└── verl/
    ├── trainer/
    ├── workers/
    ├── utils/
    ├── models/
    └── single_controller/
```

## 训练入口

README 建议通过：

```bash
bash run_train.sh
```

当前 `run_train.sh` 内容是：

```bash
bash examples/medix-r1_8b_dapo.sh
```

但当前目录没有 `examples/medix-r1_8b_dapo.sh`。实际存在：

```text
examples/medix-r1_2b_dapo.sh
examples/medix-r1_8b_gspo.sh
```

因此训练前要先修正 `run_train.sh` 或补齐对应脚本。

## 训练脚本做了什么

以 `examples/medix-r1_2b_dapo.sh` 为例，它会设置：

- `CUDA_VISIBLE_DEVICES`
- `MODEL_PATH`
- 数据集路径。
- prompt 模板。
- reward function。
- rollout batch size。
- actor batch size。
- KL 设置。
- checkpoint 保存路径。
- GPU 数量。

然后调用：

```bash
python3 -m verl.trainer.main \
    config=examples/config.yaml \
    data.train_files=... \
    worker.actor.model.model_path=${MODEL_PATH} \
    worker.reward.reward_function=./examples/reward_function/medical.py:compute_score \
    ...
```

这说明训练配置由两部分合成：

1. `examples/config.yaml` 默认配置。
2. shell 脚本中的命令行覆盖项。

## 训练主链路

训练主链路：

```text
examples/*.sh
  -> python3 -m verl.trainer.main
  -> OmegaConf 读取 config.yaml 和 CLI 覆盖项
  -> PPOConfig.deep_post_init()
  -> ray.init()
  -> Runner.run()
  -> tokenizer / processor
  -> FSDPWorker
  -> AutoRewardManager
  -> create_dataloader()
  -> RayPPOTrainer
  -> trainer.init_workers()
  -> trainer.fit()
```

## Runner 的作用

`verl/trainer/main.py` 中的 `Runner` 是 Ray remote actor。

它负责：

1. 打印配置。
2. 加载 tokenizer。
3. 加载 processor。
4. 设置 worker class。
5. 创建 Ray 资源池。
6. 创建 reward manager。
7. 创建 dataloader。
8. 创建并启动 `RayPPOTrainer`。

## RayPPOTrainer 的作用

`RayPPOTrainer` 是训练控制器。

它负责：

- 初始化资源池。
- 创建 Actor/Rollout/Ref Worker。
- 创建 Critic Worker。
- 控制 KL。
- 计算训练步数。
- 协调 rollout、reward、advantage、actor update、checkpoint。

当前配置使用 hybrid engine：

```text
ActorRolloutRef 共享同一个 FSDPWorker
```

这意味着同一组模型权重在不同阶段扮演：

- Actor：训练更新。
- Rollout：生成样本。
- Ref：计算参考 log probability。

## Reward function

训练使用：

```text
examples/reward_function/medical.py:compute_score
```

这个函数返回每条样本的分项奖励：

- `overall`
- `format`
- `accuracy_llm`
- `accuracy_embed`
- `modality`

reward 结果会写入 token-level reward tensor 的最后一个 response token 位置。

## 当前训练脚本差异

README 中列出多个模型配置：

- 2B DAPO
- 7B GRPO
- 8B DAPO
- 8B GRPO
- 30B DAPO

但当前文件树中只看到：

- 2B DAPO
- 8B GSPO

因此学习时要以当前文件树为准。README 可以作为设计意图参考，但不能当作当前完整脚本清单。

## 学习重点

训练主线可以概括为：

```text
shell 脚本决定实验
config.yaml 提供默认参数
verl.trainer.main 组装 Ray 训练
FSDPWorker 负责模型训练和 rollout
medical.py 负责医学奖励
checkpoint 保存训练结果
```

先理解这条主线，再深入 Ray、FSDP、vLLM 和 reward 细节。
