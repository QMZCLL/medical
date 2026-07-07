# 训练数据与配置

## 学习目标

本文解释 `MediX-R1/training` 如何配置训练数据、prompt、模型和 batch 参数。

## 关键源码

```text
training/examples/config.yaml
training/examples/medix-r1_2b_dapo.sh
training/examples/medix-r1_8b_gspo.sh
training/examples/format_prompt/medical_format.jinja
training/verl/trainer/data_loader.py
training/verl/utils/dataset.py
```

## 配置来源

训练配置来自两层：

1. `examples/config.yaml`：默认配置。
2. shell 脚本 CLI 参数：覆盖默认配置。

`verl/trainer/main.py` 中：

```python
cli_args = OmegaConf.from_cli()
default_config = OmegaConf.structured(PPOConfig())

if hasattr(cli_args, "config"):
    config_path = cli_args.pop("config", None)
    file_config = OmegaConf.load(config_path)
    default_config = OmegaConf.merge(default_config, file_config)

ppo_config = OmegaConf.merge(default_config, cli_args)
```

因此命令行覆盖优先级最高。

## 数据配置

`examples/config.yaml` 的 `data` 段定义：

```yaml
data:
  train_files: hiyouga/math12k@train
  val_files: hiyouga/math12k@test
  prompt_key: problem
  answer_key: answer
  image_key: images
  video_key: videos
  max_prompt_length: 4096
  max_response_length: 2048
  rollout_batch_size: 512
  val_batch_size: 1024
  format_prompt: ./examples/format_prompt/math.jinja
```

医学训练脚本会覆盖这些值，例如：

```bash
data.train_files=../data/medix-rl-data/data@train
data.val_files=../data/medix-rl-data/data@test
data.format_prompt=./examples/format_prompt/medical_format.jinja
data.answer_key=solution
data.image_key=image
```

## 数据路径格式

`RLHFDataset` 支持：

### HuggingFace dataset

```text
MBZUAI/medix-rl-data@train
```

`@` 后面是 split。

### 本地目录

如果路径是目录，会优先找 parquet 文件：

```text
train*.parquet
test*.parquet
```

否则尝试 `load_from_disk()`。

### 本地 parquet 文件

如果路径是文件，直接按 parquet 加载。

## 样本字段

医学数据预期字段：

| 字段 | 含义 |
|---|---|
| `image` | 医学图像 |
| `problem` | 医学问题 |
| `solution` | 标准答案 |

训练脚本通过：

```bash
data.answer_key=solution
data.image_key=image
```

告诉 dataloader 如何读取字段。

## Prompt 模板

`format_prompt` 指向 Jinja 模板。

医学模板：

```text
examples/format_prompt/medical_format.jinja
```

`RLHFDataset._build_messages()` 会先拿到原始 prompt：

```python
prompt_str = example[self.prompt_key]
```

如果存在模板，则：

```python
prompt_str = Template(self.format_prompt.strip()).render(content=prompt_str)
```

最终构造 chat messages。

## 多模态处理

如果样本包含 `image_key`：

1. prompt 中按 `<image>` 切分。
2. 构造包含 `{"type": "image"}` 的 content list。
3. 使用 processor 的 chat template。
4. 用 processor 处理图像和文本。

如果样本包含 `video_key`，则使用 video 处理流程。

如果没有图像和视频，则按纯文本 tokenizer 处理。

## 长度过滤

`filter_overlong_prompts=True` 时，数据集会过滤超长 prompt：

```python
len(processing_class.apply_chat_template(...)) <= max_prompt_length
```

这一步会使用多进程：

```python
num_proc=filter_overlong_prompts_workers
```

默认配置中可以避免过长样本导致训练阶段 OOM。

## batch 配置

几个容易混淆的 batch 参数：

| 配置 | 含义 |
|---|---|
| `data.rollout_batch_size` | 每次 rollout 的样本组规模 |
| `data.mini_rollout_batch_size` | 生成时的小批次，可为空 |
| `worker.actor.global_batch_size` | actor 更新全局 batch |
| `worker.rollout.n` | 每个 prompt 生成几个 response |
| `micro_batch_size_per_device_for_update` | 每张卡更新微批 |
| `micro_batch_size_per_device_for_experience` | 经验计算微批 |

`RayPPOTrainer` 会检查整除关系，例如：

```text
rollout_batch_size 必须能被 actor global batch size 整除
rollout_batch_size * rollout.n 必须能被 experience micro batch 整除
```

## 模型配置

`worker.actor.model.model_path` 指定模型。

示例：

```bash
MODEL_PATH=Qwen/Qwen3-VL-2B-Instruct
worker.actor.model.model_path=${MODEL_PATH}
```

与模型相关的配置：

```yaml
enable_gradient_checkpointing: true
trust_remote_code: false
freeze_vision_tower: false
```

视觉语言模型通常还需要 processor，而不只是 tokenizer。

## 算法配置

`algorithm` 段包括：

```yaml
adv_estimator: grpo
disable_kl: false
use_kl_loss: true
kl_penalty: low_var_kl
kl_coef: 1.0e-2
online_filtering: false
```

训练脚本可能覆盖：

```bash
algorithm.disable_kl=True
algorithm.online_filtering=True
```

这些会显著改变训练行为。

## 输出与 checkpoint

训练脚本设置：

```bash
trainer.experiment_name=...
trainer.save_checkpoint_path=...
trainer.total_epochs=2
trainer.n_gpus_per_node=...
```

checkpoint 保存后，可以用：

```bash
bash merge_model.sh
```

把 FSDP sharded checkpoint 合并成 HuggingFace 格式。

## 当前配置风险

1. `run_train.sh` 指向不存在脚本。
2. README 列出的脚本多于当前实际文件。
3. 部分脚本使用本地相对数据路径，部分使用 HuggingFace dataset 名。
4. reward function 导入根目录 `config.py`，路径依赖当前工作目录。
5. 大 batch 配置需要多 GPU，不适合普通机器直接运行。

## 学习重点

理解训练配置时记住：

```text
config.yaml 是默认模板
examples/*.sh 是具体实验
CLI 覆盖决定最终配置
RLHFDataset 根据配置加载和格式化样本
RayPPOTrainer 根据配置检查资源和 batch 合法性
```
