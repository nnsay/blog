---
title: 机器学习--数据整理
date: 2024-08-14 18:21:54
tags:
  - Code
  - 研究
excerpt: 使用 AWS 和开源的 Data Wrangler 进行数据整理
---

# 1. 实践: [使用 Amazon Data Wrangler 准备机器学习 SageMaker 数据](https://docs.amazonaws.cn/sagemaker/latest/dg/data-wrangler.html)

Amazon SageMaker Data Wrangler（Data Wrangler）是 Amazon SageMaker Studio Classic 的一项功能，它提供了导入、准备、转换、特征化和分析数据的 end-to-end 解决方案。您可以将 Data Wrangler 数据准备流集成到机器学习 (ML) 工作流中，以简化和精简数据预处理和特征工程，只需少量甚至无需编码。您还可以添加自己的 Python 脚本和转换，以自定义工作流。

Data Wrangler 可提供以下核心功能，帮助您分析和准备用于机器学习应用程序的数据。

## **导入**

连接亚马逊简单存储服务 (Amazon S3)、（Athena）、亚马逊 Redshift、Snow Amazon Athena flake 和 Databricks 并从中导入数据。

![image-20240814120236299](https://raw.githubusercontent.com/nnsay/gist/main/imgimage-20240814120236299.png)

## **数据流**

创建数据流以定义一系列机器学习数据准备步骤。您可以使用流合并来自不同数据源的数据集，确定要应用于数据集的转换数量和类型，并定义可集成到机器学习管线中的数据准备工作流。

![image-20240814121037018](https://raw.githubusercontent.com/nnsay/gist/main/imgimage-20240814121037018.png)

## **转换**

使用标准*转换*（如字符串、矢量和数字数据格式化工具）清理和转换数据集。使用转换（如文本和日期/时间嵌入以及分类编码）特征化数据。

![image-20240814121322172](https://raw.githubusercontent.com/nnsay/gist/main/imgimage-20240814121322172.png)

## **生成数据见解**

使用 Data Wrangler 数据见解和质量报告，自动验证数据质量并检测数据中的异常。

![image-20240814121608515](https://raw.githubusercontent.com/nnsay/gist/main/imgimage-20240814121608515.png)

## **分析**

在流中的任意点分析数据集中的特征。Data Wrangler 包括内置的数据可视化工具，如散点图和直方图，以及目标泄漏分析和快速建模等数据分析工具，以了解特征相关性。

![image-20240814121455258](https://raw.githubusercontent.com/nnsay/gist/main/imgimage-20240814121455258.png)
![data-wrangler-insights-reportb55ae18a7f977502.png](https://img.picgo.net/2024/08/14/data-wrangler-insights-reportb55ae18a7f977502.png)

## **导出**

将数据准备工作流导出至其他位置。以下是一些示例位置：

- Amazon Simple Storage Service (Amazon S3)桶
- Amazon SageMaker 模型构建管道 — 使用 SageMaker 管道自动部署模型。您可以将转换后的数据直接导出至管线。
- Amazon F SageMaker eature Store — 将功能及其数据存储在中央存储中。
- Python 脚本 – 将数据及其转换存储在 Python 脚本中，用于您的自定义工作流

# 2. 总结和实践

1. Data Wrangle 集成在 Studio 中, 对开发背景的数据科学家来说不太友好, 因为开发者背景工程师和喜欢使用自己的工具尽量在本地调试

2. [awesome-data-wrangling](https://github.com/peerside/awesome-data-wrangling): A curated list of data wrangling resources with a bias towards command line tools without steep learning curves.

3. 利用[Data Wrangler Extension for Visual Studio Code](https://github.com/microsoft/vscode-data-wrangler)进行数据质量分析

   - 查看模式

   ![](https://img.picgo.net/2024/08/14/f8fa6c17b75ec98656c34834b7e9addf4a91a3a0f208d1eb.png)

- 编辑模式

  - 选择列填充默认值

    ![](https://img.picgo.net/2024/08/14/5722dc380dec149a82ab16b460c4c39989e633a0374d9eab.png)

  - 多个步骤进行数据处理并导出![](https://img.picgo.net/2024/08/14/c0fd9f828ff4a34a4d415a641d5f86fcb92d2edc88d0cf7f.png)

  - 导出到笔记

    ![](https://img.picgo.net/2024/08/14/90ebd3e18def0cbfd04dbbab47b4dbbd26867ee3b4739727.png)

4. 结合[altair](https://altair-viz.github.io/index.html)进行数据质量观察

   ```python
   import altair as alt

   alt.data_transformers.disable_max_rows()
   # sampled_data = df_clean.sample(n=5000, random_state=1)
   # sampled_data['timestamp'] = pd.to_datetime(sampled_data['timestamp'])
   ```

   ```python
     line_chart = alt.Chart(df_clean).mark_line().encode(
         x='timestamp:T',
         y='unit_sales:Q'
     ).properties(
         title='Unit Sales Over Time'
     )
     line_chart.show()
   ```

   ![](https://img.picgo.net/2024/08/14/data-wrangle-linec418e913de4bd087.png)

   ```python
     scatter_plot = alt.Chart(df_clean).mark_circle(size=60).encode(
         x='scaled_price:Q',
         y='unit_sales:Q',
         color='product_subcategory:N'
     ).properties(
         title='Unit Sales vs Scaled Price'
     )
     scatter_plot.show()
   ```

   ![](https://img.picgo.net/2024/08/14/data-wrangle-plot23cd9d5ce92aaae9.png)
