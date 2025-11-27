---
title: embedding 模型选型测试
date: 2025-11-27 16:45:03
tags:
  - AI
  - DevOps
excerpt: 使用语句相似度(STS)测试 embedding 模型
---

# 声明

本文由 🤖AI 改写润色而成

# 1. 业务背景介绍

在构建 RAG 应用程序时，选择一款合适的 embedding 模型是一项重要决策。因为 RAG 应用中最关键的环节就是通过用户输入的语句来检索相关文档，基于此背景，我选择使用语句相似度(STS)来评估 embedding 模型的质量。

# 2. 模型测试

在 embedding 模型的评估中，我采用了当前主流的工具 [mteb](https://github.com/embeddings-benchmark/mteb) 进行测试。[mteb](https://github.com/embeddings-benchmark/mteb) 是一个专业的 embedding 模型评估工具包，提供了丰富的任务和指标，能够帮助我们快速、全面地评估模型质量。
在本次测试中，我选择了中文 STSB 任务作为评估标准。中文 STSB 任务是一项专门测量中文句子相似度的任务，可有效评估模型是否能够将语义相似的中文句子准确映射到向量空间中的相似位置。

## 2.1 测试脚本

尽管市面上的 embedding 模型种类繁多，但我选取了 BAAI/bge-m3 和 Qwen/Qwen3-Embedding-0.6B 这两款较为知名的模型进行对比测试。

```python
import os
import pandas as pd
import matplotlib.pyplot as plt
import seaborn as sns
import mteb
from mteb import MTEB
from sentence_transformers import SentenceTransformer

# ================= 配置区域 =================
MODEL_LIST = [
    "BAAI/bge-m3",
    "Qwen/Qwen3-Embedding-0.6B",
]
OUTPUT_DIR = "./output/embedding-eval_results"

# ================= 核心逻辑 =================


def get_chinese_stsb_task():
    print("🔍 正在搜索中文 STSB 任务...")
    # 关键修改：不要同时传 tasks=["STSB"] 和 languages，否则 languages 会被忽略
    # 我们先拉取所有中文 STS 任务，再从中筛选
    try:
        # ISO 639-3 代码：cmn 代表 Mandarin
        tasks = mteb.get_tasks(languages=["cmn"], task_types=["STS"])
        # 过滤出名字里带 STSB 的任务
        target_tasks = [t for t in tasks if "STSB" == t.metadata.name]

        print(f"✅ 锁定任务: {[t.metadata.name for t in target_tasks]}")
        return target_tasks
    except Exception as e:
        print(f"❌ 任务搜索失败: {e}")
        return []


def run_evaluation():
    if not os.path.exists(OUTPUT_DIR):
        os.makedirs(OUTPUT_DIR)

    all_results = []

    # 1. 获取任务
    tasks = get_chinese_stsb_task()
    if not tasks:
        return pd.DataFrame()

    print(f"📋 待测模型列表: {MODEL_LIST}\n")

    for model_name in MODEL_LIST:
        print(f"🔹 正在加载并评估模型: {model_name} ...")

        try:
            model = SentenceTransformer(model_name, trust_remote_code=True)

            # 实例化 MTEB
            evaluation = MTEB(tasks=tasks)

            # 运行测试
            # 注意: 如果任务是多语言的(Multilingual)，这里需要强制指定 eval_langs
            # 但如果我们上面已经加载了特定语言的任务，这里就不需要了
            results = evaluation.run(
                model,
                output_folder=os.path.join(OUTPUT_DIR, "raw_logs"),
                encode_kwargs={"batch_size": 32},
            )

            # --- 结果调试与提取区域 ---
            # 这里的 results 是一个列表，包含每个 Task 的 TaskResult
            for result in results:
                print(f"   🔎 [调试] 正在解析任务: {result.task_name}")

                # 获取 test 集结果
                test_scores = result.scores.get("test")
                metrics = test_scores[0]
                spearman = metrics.get("cosine_spearman", 0) * 100
                pearson = metrics.get("cosine_pearson", 0) * 100

                print(
                    f"✅ {model_name} 得分: Spearman={spearman:.2f}, Pearson={pearson:.2f}"
                )
                all_results.append(
                    {"Model": model_name, "Spearman": spearman, "Pearson": pearson}
                )

        except Exception as e:
            import traceback

            print(f"❌ 模型 {model_name} 测试失败: {str(e)}")
            traceback.print_exc()

    return pd.DataFrame(all_results)


def generate_report(df):
    if df.empty:
        print("没有结果可以生成报告。")
        return

    csv_path = os.path.join(OUTPUT_DIR, "benchmark_summary.csv")
    df.to_csv(csv_path, index=False)
    print(f"\n📄 详细数据报表已保存至: {csv_path}")
    print(df)

    plt.figure(figsize=(10, 6))
    sns.set_theme(style="whitegrid")

    ax = sns.barplot(
        x="Model", y="Spearman", data=df, palette="viridis", hue="Model", legend=False
    )

    plt.title("Embedding Models Performance on STSB", fontsize=16)
    plt.ylabel("Spearman Correlation (%)", fontsize=12)
    plt.xlabel("Model Name", fontsize=12)
    plt.ylim(0, 100)
    plt.xticks(rotation=15)

    for i in ax.containers:
        ax.bar_label(i, fmt="%.2f", padding=3)

    plot_path = os.path.join(OUTPUT_DIR, "benchmark_chart.png")
    plt.tight_layout()
    plt.savefig(plot_path, dpi=300)
    print(f"📊 可视化对比图已保存至: {plot_path}")


if __name__ == "__main__":
    df_results = run_evaluation()
    generate_report(df_results)

```

> [!NOTE]
> 脚本中`MODEL_LIST`模型列表可根据实际需求进行修改。

## 2.2 测试结果

| Model                     | Spearman | Pearson |
| ------------------------- | -------- | ------- |
| BAAI/bge-m3               | 80.60    | 80.01   |
| Qwen/Qwen3-Embedding-0.6B | 78.18    | 78.40   |

### 测试指标解读

- **Spearman Correlation (斯皮尔曼等级相关系数)** —— 最重要指标

  - **核心含义**：评估排名（Rank）的一致性
  - **评估重点**：不关注具体数值，仅关注排序的准确性
  - **判断标准**：如果人类认为"句子 A"比"句子 B"更相似，模型是否也认为"句子 A"的得分 > "句子 B"的得分？
  - **示例说明**：
    - 人类打分：Pair A (5 分), Pair B (3 分), Pair C (1 分)
    - 模型打分：Pair A (0.9), Pair B (0.4), Pair C (0.1)
    - 结果：虽然具体数值不同，但大小顺序完全一致 (A>B>C)，Spearman 相关系数接近 100
  - **重要性**：在 RAG（检索增强生成）或搜索引擎中，我们最关注的是将最相关的文档排在首位，具体分数差异（如 0.9 还是 0.8）相对次要。因此，在 STS 任务中，Spearman 通常被视为主指标 (Main Score)。

- **Pearson Correlation (皮尔逊线性相关系数)**
  - **核心含义**：评估线性关系（Linearity）的强度
  - **评估重点**：要求模型分数不仅排序正确，且变化幅度需与人类分数成比例（数据点应大致呈直线分布）
  - **示例说明**：如果人类分数从 1 变为 2（翻倍），模型分数也应呈现相应比例的增长
  - **局限性**：容易受离群点（Outlier）影响。例如，即使大部分数据预测准确，但若存在少量偏差极大的数据点，Pearson 相关系数可能显著下降。

## 2.3 测试(选型)总结

测试结果显示，BAAI/bge-m3 的得分比 Qwen/Qwen3-Embedding-0.6B 高出约 2 分。然而，从模型资源消耗来看，前者文件大小达 4.3G，后者仅为 1.1G。这一差异很可能源于 BAAI/bge-m3 拥有更多的模型参数，使其在 STSB 任务中表现出略优的性能。

此外，两款模型均支持 1024 维向量输出，但在上下文处理能力上存在显著差异：BAAI/bge-m3 支持 8192 的上下文长度，而 Qwen3-Embedding-0.6B 则达到 32K。考虑到总体性能差异不大，Qwen3-Embedding-0.6B 凭借其小巧的尺寸、较低的资源占用以及出色的质量，成为一个极具竞争力的选择。

# 3. 总结

本文提供了一个简洁实用的测试框架，用于对比不同 Embedding 模型在 STSB 任务中的性能表现。希望这个测试脚本能够为大家在选择适合的 embedding 模型时提供有价值的参考。

需要说明的是，本次测试仅针对 STSB 任务进行评估，可能存在一定的片面性。mteb 工具包还支持多种其他任务的测试，如自然语言推理（NLI）、文本分类等，建议大家根据实际业务场景选择相应的测试任务。

如果您希望快速了解模型性能而不愿亲自进行测试，也可以参考 [mteb 排行榜](https://huggingface.co/spaces/mteb/leaderboard)，这也是一个可靠的模型选型依据。
