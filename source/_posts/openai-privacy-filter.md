---
title: OpenAI 隐私过滤器
date: 2026-05-28 16:38:23
tags:
  - DevOps
  - 研究
excerpt: 实验和测试 OpenAI 隐私过滤器
---

# 1. [privacy-filter](https://github.com/openai/privacy-filter) 介绍

## 基本介绍

OpenAI 隐私过滤器是一种双向 token 分类模型，用于检测和屏蔽文本中的个人身份信息（PII）。该模型适用于高吞吐量的数据脱敏工作流，团队可在本地运行此模型，其具备快速、上下文感知及可调优的特点。

OpenAI 隐私过滤器首先通过自回归方式预训练，获得一个与 gpt-oss 架构相似但规模更小的检查点。随后，我们将该检查点转换为针对隐私标签分类体系的双向 token 分类器，并通过监督分类损失进行后训练。（有关 gpt-oss 的架构细节，请参阅 gpt-oss 模型卡。）与逐个 token 生成文本不同，该模型在单次前向传播中对输入序列进行标记，然后通过约束 Viterbi 算法解码出连贯的片段。对于每个输入 token，模型会预测一个覆盖以下所述 8 个输出类别的标签分类概率分布。

亮点：

- 宽松的 Apache 2.0 许可证：非常适合实验、定制和商业部署。
- 小型化：可在网页浏览器或笔记本电脑上运行——总计 15 亿参数，其中活跃参数为 5,000 万。
- 可微调：通过简单且数据高效的微调，使模型适应特定的数据分布。
- 长上下文：128,000 个 token 的上下文窗口支持高吞吐量处理长文本，无需分块。
- 运行时控制：可通过预设操作点配置精确率/召回率权衡以及检测到的片段长度。

## 输出形状

隐私过滤器可检测 8 种隐私片段类别：

1. `account_number`
2. `private_address`
3. `private_email`
4. `private_person`
5. `private_phone`
6. `private_url`
7. `private_date`
8. `secret`

在执行 token 分类时，每个非背景片段类别都会扩展为带边界标记的 token 类别：`B-<label>`、`I-<label>`、`E-<label>`、`S-<label>`，再加上背景类别 `O`。因此，token 级输出类别的总数为 33：1 个背景类别 + 8 个片段标签 \* 4 个边界标记 = 33 个类别。这意味着输出头为每个 token 生成 33 个 logits。对于长度为 T 的序列，输出形状为 `[T, 33]`；对于批次大小为 B 的情况，输出形状为 `[B, T, 33]`。

token 标签词汇表包括背景标签 `O` 以及每个隐私类别的 BIOES 标记变体：`account_number`、`private_address`、`private_email`、`private_person`、`private_phone`、`private_url`、`private_date` 和 `secret`。换言之，对于每个类别，模型会预测 `B-`、`I-`、`E-` 和 `S-` 形式，分别对应起始、内部、结束和单 token 片段。在推理阶段，这些 per-token logits 会通过约束序列解码转换为连贯的 BIOES 片段标签。

# 2. 工程验证

在验证之前提前下载好模型:

```bash
mkdir -p openai-mirror/privacy-filter
modelscope download --model openai-mirror/privacy-filter --local_dir openai-mirror/privacy-filter
```

## 2.1 命令方式

命令属于原生模式, 只需要 original 模型

- 安装命令行

  ```bash
  git clone https://github.com/openai/privacy-filter.git
  cd privacy-filter

  uv tool install . --with "httpx[socks]"
  ```

- 执行测试

  ```bash
  $ OPF_CHECKPOINT=$(pwd)/openai-mirror/privacy-filter/original

  $ opf "Alice was born on 1990-01-02."
  Alice was born on <PRIVATE_DATE>.

  $ opf "My name is Harry Potter and my email is harry.potter@hogwarts.edu."
  My name is <PRIVATE_PERSON> and my email is <PRIVATE_EMAIL>.

  ```

- CPU vs GPU

  ```bash
  $ time opf --device cpu "My name is Harry Potter and my email is harry.potter@hogwarts.edu."
  My name is <PRIVATE_PERSON> and my email is <PRIVATE_EMAIL>.

  real    0m2.274s
  user    0m7.706s
  sys     0m0.721s

  $ time opf "My name is Harry Potter and my email is harry.potter@hogwarts.edu."
  My name is <PRIVATE_PERSON> and my email is <PRIVATE_EMAIL>.

  real    0m17.723s
  user    0m17.766s
  sys     0m1.743s

  $ time opf --device cpu "我的名字是哈利·波特，我的电子邮件是 harry.potter@hogwarts.edu。"
  我的名字是哈利·波特，我的电子邮件是 <PRIVATE_EMAIL>。

  real    0m2.326s
  user    0m8.286s
  sys     0m0.720s

  $ time opf "我的名字是哈利·波特，我的电子邮件是 harry.potter@hogwarts.edu。"
  我的名字是哈利·波特，我的电子邮件是 <PRIVATE_EMAIL>。

  real    0m18.929s
  user    0m18.930s
  sys     0m1.809s
  ```

## 2.2 代码方式

- 安装依赖

  ```bash
  uv init
  uv venv
  uv add transformers torch
  ```

- 准备脚本

```python
from transformers import pipeline
import json

classifier = pipeline(
    task="token-classification",
    model="/home/dgx/.newtranx/llm/openai-mirror/privacy-filter",
  	trust_remote_code=True
)

input = "My name is Harry Potter and my email is harry.potter@hogwarts.edu."
output = classifier(input)
print(json.dumps(output, ensure_ascii=False, indent=2, default=float))
```

- 执行

  ```bash
  $ uv run python main.py
  Uninstalled 1 package in 0.62ms
  Installed 1 package in 2ms
  Loading weights: 100%|██████████████████████████████████████████████████████████████████████████████████| 140/140 [00:00<00:00, 10061.56it/s]
  [
    {
      "entity": "B-private_person",
      "score": 0.9999985694885254,
      "index": 3,
      "word": "ĠHarry",
      "start": 10,
      "end": 16
    },
    {
      "entity": "E-private_person",
      "score": 0.9999990463256836,
      "index": 4,
      "word": "ĠPotter",
      "start": 16,
      "end": 23
    },
    {
      "entity": "B-private_email",
      "score": 0.9999983310699463,
      "index": 9,
      "word": "Ġhar",
      "start": 39,
      "end": 43
    },
    {
      "entity": "I-private_email",
      "score": 0.9999984502792358,
      "index": 10,
      "word": "ry",
      "start": 43,
      "end": 45
    },
    {
      "entity": "I-private_email",
      "score": 1.0,
      "index": 11,
      "word": ".p",
      "start": 45,
      "end": 47
    },
    {
      "entity": "I-private_email",
      "score": 0.9999998807907104,
      "index": 12,
      "word": "ot",
      "start": 47,
      "end": 49
    },
    {
      "entity": "I-private_email",
      "score": 1.0,
      "index": 13,
      "word": "ter",
      "start": 49,
      "end": 52
    },
    {
      "entity": "I-private_email",
      "score": 1.0,
      "index": 14,
      "word": "@",
      "start": 52,
      "end": 53
    },
    {
      "entity": "I-private_email",
      "score": 1.0,
      "index": 15,
      "word": "hog",
      "start": 53,
      "end": 56
    },
    {
      "entity": "I-private_email",
      "score": 1.0,
      "index": 16,
      "word": "warts",
      "start": 56,
      "end": 61
    },
    {
      "entity": "E-private_email",
      "score": 0.9999971389770508,
      "index": 17,
      "word": ".edu",
      "start": 61,
      "end": 65
    }
  ]
  ```

# 3. 总结

隐私过滤器（Privacy Filter）是一个本地 PII 检测/脱敏工具，适合在日志、训练数据、RAG 入库、客服记录、Prompt 入模前识别姓名、邮箱、电话、地址等个人敏感信息，并作为隐私保护的预处理层使用。但它不是合规保证，也不能承诺 100% 检出，复杂格式、多语言、上下文隐含信息仍建议配合规则兜底和人工抽检, 但可以做为隐私设计多层防护层之一; 除此之外本文对照测试发现两个关键结果:

1. 中文效果没有英文好;
2. CPU效率更高;

综上所述在适合的场景中谨慎使用.
