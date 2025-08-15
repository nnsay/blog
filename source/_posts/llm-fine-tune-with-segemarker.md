---
title: 使用 AWS SageMaker 微调模型
date: 2025-08-15 17:02:36
tags:
  - DevOps
  - AWS
  - AI
excerpt: 学习大模型微调各个步骤
---

# 声明

本文由 🤖AI 改写润色而成

# **引言：为何我们需要微调大模型？**

通用大语言模型（如 GPT、Llama）已具备惊人的知识广度和语言能力，但当面临高度垂直或专业的领域时，它们可能会显得“博而不精”。大型语言模型微调（Fine-Tuning）技术应运而生，它通过在特定的小规模数据集上对预训练模型进行“二次训练”，将一个“通才”模型雕琢成特定领域的“专家”。

本教程将以一个真实场景为例：**将 Meta 的 Llama 3.2 1B 指令模型微调为一个光伏行业政策解读专家**。我们将走通从数据准备到模型部署的完整链路，为您提供一套可复现、可扩展的实战工作流。

**项目流程概览:**
`[数据爬取] -> [对话生成] -> [格式转换] -> [上传S3] -> [SageMaker微调] -> [SageMaker部署] -> [API调用]`

# **1. 核心概念：大模型微调入门**

在开始实战之前，我们先快速了解几个核心概念。

- **什么是大模型微调？**

  简单来说，微调就是在预训练好的通用大模型基础上，使用我们自己准备的、针对特定任务的数据集，对模型进行小范围的参数更新。这个过程弥合了通用知识与特定需求之间的鸿沟，让模型能够更精准地理解专业术语、遵循特定指令、或模仿某种说话风格。

  例如，一个通用的聊天模型可能无法准确回答“2025 年分布式光伏并网补贴的具体政策依据是什么？”，但经过光伏政策数据微调后，它就能给出专业且精准的回答。

- **主流微调方法简介**

  - **持续预训练 (Continual Pre-Training, CPT):** 使用大量无监督的行业语料（如海量行业文档），让模型学习特定领域的语言范式和背景知识。此方法能提升模型的行业表现，但需要巨大的数据量（通常千万级 Token 以上）。
  - **监督式微调 (Supervised Fine-Tuning, SFT):** 使用高质量的“指令-回答”或对话式数据集，教模型如何响应特定类型的提问。此方法能高效提升模型在特定业务场景下的表现，所需数据量相对较小（数百至数千条即可）。
  - **直接偏好优化 (Direct Preference Optimization, DPO):** 通过“优选回答”和“劣质回答”的对比数据，让模型学习人类的偏好，进一步提升回答的质量和安全性。

  考虑到数据获取成本和任务目标，**本文将采用 SFT 方法**，它是在性能和资源投入之间取得平衡的最佳选择。

# **2. 准备数据：从原始新闻到对话语料**

高质量的数据是微调成功的基石。我们将分步完成从原始数据采集到初始对话数据集生成的全过程。

- **工作流程：**

  1.  **数据采集：** 使用爬虫工具 `crawl4ai` 从“北极星光伏网”抓取近期的政策新闻 URL。
  2.  **内容提取：** 遍历 URL 列表，抓取每篇新闻的详细内容，并保存为 Markdown 格式。
  3.  **对话生成：** **这是关键一步**。我们巧妙地利用第三方强力大模型（如 GPT-4o）作为“数据标注专家”，为每一篇新闻文章自动生成高质量、多角度的问答对话。
  4.  **格式化存储：** 将所有生成的对话数据存储为结构化的 `JSONL` 文件，为下一步处理做好准备。

- **代码实现：**

- main.py

```python
import asyncio
import glob
import os
from pathlib import Path
from typing import Dict, Tuple
import pandas as pd
from datasets import Dataset
from src.llm import generate_conversation
from src.crawl import get_policy, get_policy_urls


async def main():
    policy_csv_file = "output/policy_urls.csv"
    policy_urls = await get_policy_urls()
    policy_df = pd.DataFrame(policy_urls)
    if os.path.exists(policy_csv_file):
        existing_df = pd.read_csv(policy_csv_file)
        policy_df = pd.concat([existing_df, policy_df], ignore_index=True)
        policy_df.drop_duplicates(subset=["link"], keep="last", inplace=True)
    policy_df.set_index("date", inplace=True)
    policy_df = policy_df.sort_index(ascending=False)
    policy_df.to_csv(policy_csv_file)

    total_count = len(policy_urls)
    finished_count = 0
    for url in policy_urls:
        policy_conversation_csv = f"output/conversations/{url.file_id}.csv"
        if os.path.exists(policy_conversation_csv):
            finished_count += 1
            print(
                f"policy <{url.title}> conversation generation done(1) ({finished_count}/{total_count})"
            )
            continue

        policy_news_md = f"output/policy-news/{url.file_id}.md"
        if os.path.exists(policy_conversation_csv):
            with open(policy_news_md, "r", encoding="utf-8") as md:
                content = md.read()
        else:
            content = await get_policy(url.link)
            with open(policy_news_md, "w", encoding="utf-8") as md:
                md.write(content)

        conversations = generate_conversation(content)
        finished_count += 1
        print(
            f"policy <{url.title}> conversation generation done(2) ({finished_count}/{total_count})"
        )
        df = pd.DataFrame(conversations)
        df.set_index("role", inplace=True)
        df.to_csv(policy_conversation_csv)

    generate_dataset()


def generate_dataset():
    policy_info: Dict[str, Tuple[str, str]] = {}
    df = pd.read_csv("output/policy_urls.csv")
    for _, row in df.iterrows():
        policy_info[str(row["file_id"])] = (row["date"], row["title"])

    all_csv_files = glob.glob(os.path.join("output/conversations", "*.csv"))
    filed_name = "conversations"
    all_messages = []
    for csv_file in all_csv_files:
        file_id = Path(csv_file).stem
        _, title = policy_info[file_id]
        df = pd.read_csv(csv_file)
        messages = []
        for _, row in df.iterrows():
            role, content = row["role"], row["content"]
            if role == "user":
                content = f"关于光伏政策<{title}>, 用户提问: {content}"
            messages.append({"role": role, "content": content})
        all_messages.append({filed_name: messages})

    pretty_df = pd.DataFrame(all_messages)
    # generate jsonl format dataset
    pretty_df.to_json(
        "output/dataset/special.jsonl",
        orient="records",
        lines=True,
        force_ascii=False,
    )
    print("generate jsonl format dataset done")

    # generate arrow format dataset
    ds = Dataset.from_pandas(pretty_df)
    ds.save_to_disk("output/dataset/special")
    print("generate arrow format dataset done")


if __name__ == "__main__":
    asyncio.run(main())
```

`````

- src/crawl.py

```python
import json
from typing import List
from crawl4ai import (
    AsyncWebCrawler,
    BrowserConfig,
    CacheMode,
    CrawlerRunConfig,
    JsonCssExtractionStrategy,
)
from urllib.parse import urlparse
from pathlib import Path


class PolicyUrl(dict):
    title: str
    link: str
    date: str
    file_id: str

    def __init__(self, title, link, date, file_id):
        super().__init__(title=title, link=link, date=date, file_id=file_id)
        self.title = title
        self.link = link
        self.date = date
        self.file_id = file_id


async def get_policy_urls(max_pages: int = 6):
    policy_url_schema = {
        "name": "policy news",
        "baseSelector": "div.cc-layout-3 .box .center li",
        "fields": [
            {
                "name": "title",
                "selector": "a",
                "type": "text",
            },
            {"name": "link", "selector": "a", "type": "attribute", "attribute": "href"},
            {"name": "date", "selector": "span", "type": "text"},
        ],
    }
    urls = [f"https://guangfu.bjx.com.cn/zc/{page}" for page in range(1, max_pages + 1)]
    browser_conf = BrowserConfig(headless=True, java_script_enabled=True)
    run_conf = CrawlerRunConfig(
        cache_mode=CacheMode.BYPASS,
        extraction_strategy=JsonCssExtractionStrategy(policy_url_schema),
        wait_for="css:.cc-layout-3 .box .center",
    )
    result: List[PolicyUrl] = []
    async with AsyncWebCrawler(config=browser_conf) as crawler:  # type: ignore
        all_res = await crawler.arun_many(urls, config=run_conf)
        for res in all_res:  # type: ignore
            if not res.success:
                print(f"[ERROR] {res.url} => {res.error_message}")
                continue
            policyUrls = json.loads(res.extracted_content)
            for url in policyUrls:
                path_str = urlparse(url["link"]).path
                file_id = Path(path_str).stem
                result.append(PolicyUrl(**url, file_id=file_id))
    return result


async def get_policy(policy_url: str) -> str:
    browser_conf = BrowserConfig(headless=True, java_script_enabled=True)
    run_conf = CrawlerRunConfig(
        cache_mode=CacheMode.BYPASS,
        css_selector="div#article_cont",
        wait_for="css:.cc-article",
    )
    async with AsyncWebCrawler(config=browser_conf) as crawler:
        result = await crawler.arun(url=policy_url, config=run_conf)
        return result.markdown  # type: ignore
```

- src/llm.py

````python
import json
import os
import re
from typing import List
from openai import OpenAI
from dotenv import load_dotenv

load_dotenv(override=True)


def build_prompt(markdown_text: str) -> str:
    return f"""
# 角色

你是一位专业的AI数据标注专家, 擅长从复杂的行业文档中提取信息, 并将其转化为高质量、多角度的对话式数据集, 用于大模型微调。你的专长领域是新能源和光伏行业政策分析。

# 背景

我将提供一篇关于中国新能源光伏行业政策的 Markdown 格式新闻文章。这篇文章包含了政策细节、行业影响、专家观点等信息。

# 任务

你的核心任务是根据我提供的文章内容, 生成尽可能多的高质量的对话(Conversation)。每一对对话都应包含一个“用户(user)”的提问和一个“助手(assistant)”的回答。这些对话需要满足以下所有要求。

# 关键要求

1.  **绝对忠实原文**:
    *   **所有**“助手(assistant)”的回答都**必须**完全基于所提供文章的内容, 不得捏造、歪曲或引入文章之外的任何信息。
    *   如果文章中没有相关信息来回答某个问题, 你应该构思一个用户问题, 然后让助手回答“根据所提供的文章, 其中并未提及关于[问题主题]的具体信息。”

2.  **模拟多样化用户角色 (Persona)**:
    *   构思“用户(user)”提问时, 请模拟来自不同背景的人。这会让问题更多样、更有深度。请至少覆盖以下角色中的 **4种**:
        *   **行业投资者**: 关心投资回报、市场机会、风险点。
        *   **光伏企业主/高管**: 关心政策对企业运营、成本、供应链、技术路线的影响。
        *   **一线工程师/技术人员**: 关心技术标准、具体参数、实施细节。
        *   **政策研究员/学者**: 关心政策设计的初衷、长远影响、与其他政策的关联。
        *   **普通民众/潜在消费者**: 关心政策如何影响电价、如何申请补贴、对自己生活的好处。
        *   **新闻记者**: 寻求对政策核心要点的总结、关键数据的提炼或有冲击力的观点。

3.  **问题类型多样化**:
    *   不要只问“是什么”。请设计多样化的问题, 涵盖不同认知层面：
        *   **信息提取**: “政策中提到的具体补贴金额是多少？”
        *   **核心摘要**: “能否用三句话总结一下这项新政策的核心内容？”
        *   **影响分析**: “这项政策对分布式光伏的发展会有什么具体影响？”
        *   **原因探究**: “文章有没有提到为什么选择在这个时间点出台这项政策？”
        *   **对比分析**: “新政策和旧政策相比, 主要有哪些不同点？” (如果文章有提及)
        *   **操作指南**: “如果一家企业想申请这个项目, 根据文章描述, 需要满足哪些条件？”

4.  **回答的结构化与专业性**:
    *   “助手(assistant)”的回答应该清晰、准确、专业。
    *   如果原文信息适合, 可以使用列表（bullet points）或分点阐述, 使回答更具条理性。

# 输出格式

请严格按照下面的 JSON 格式输出:
\```json
{{"conversations": [ {{ "role": "user", "content": "问题1" }} , {{ "role": "assistant", "content": "回答1" }} ]}}
\```

这是一个包含多个对话对象的JSON对象。`coversations`key 包含多个对话, 每个对话对象都包含一个 `user` 和一个 `assistant` 的角色和内容。

基于上述要求请处理以下文章:

{markdown_text}
"""


def generate_conversation(markdown_text: str) -> List[str]:
    client = OpenAI(api_key=os.environ["OPENAI_API_KEY"])
    prompt = build_prompt(markdown_text)
    try:
        completion = client.chat.completions.create(
            model="gpt-4o",
            messages=[{"role": "user", "content": prompt}],
            response_format={"type": "json_object"},
        )
        output = completion.choices.message.content
        if output is None:
            return []
        match = re.search(r"```json\s*([\s\S]*?)\s*```", output)
        if match:
            output = match.group(1).strip()
        parsed_output = json.loads(output)
        return parsed_output.get("conversations", [])
    except Exception as e:
        print(f"catch error: {e} ")
        return []
`````

# **3. 生成 SFT 训练数据：适配与优化**

上一步我们已经生成了通用的对话数据，为什么还需要一个专门的步骤来“生成训练数据”呢？这主要基于两个重要原因：

1.  **适配训练框架的数据格式：**
    不同的微调框架对数据格式有不同的要求。我们最初生成的对话格式（`[{'role': 'user', 'content': '...'}, {'role': 'assistant', 'content': '...'}]`）是一种通用标准，但本文采用的 **AWS SageMaker JumpStart Llama 3.2 微调脚本**，要求的是一种更简洁的**指令格式**（Instruction-based）。因此，我们需要将对话数据转换为包含 `instruction` (问题) 和 `answer` (答案) 键的 JSONL 文件，并辅以一个 `template.json` 文件来告诉 SageMaker 如何将这两部分组合成一个完整的对话。

2.  **解决能力退化（最佳实践说明）：**
    仅使用高度专业化的数据进行微调，可能会导致模型在其他通用能力上出现“遗忘”或退化。业界推荐的最佳实践是将专业数据与通用对话数据按一定比例（如 8:2）混合训练。
    > **注意：** 本教程为了简化流程，并未执行数据融合步骤，但在生产环境中，强烈建议引入此环节以保证模型的泛化能力。

- **代码实现：**

```python
#
# 准备微调数据
#
import glob
import os
from pathlib import Path
from typing import Dict, Tuple
import pandas as pd


policy_info: Dict[str, Tuple[str, str]] = {}
df = pd.read_csv("output/policy_urls.csv")
for _, row in df.iterrows():
    policy_info[str(row["file_id"])] = (row["date"], row["title"])

all_csv_files = glob.glob(os.path.join("output/conversations", "*.csv"))
all_instructions = []
for csv_file in all_csv_files:
    file_id = Path(csv_file).stem
    _, title = policy_info[file_id]
    df = pd.read_csv(csv_file)
    instruction = {}
    for _, row in df.iterrows():
        role, content = row["role"], row["content"]
        if role == "user":
            content = f"关于光伏政策<{title}>, 用户提问:{content}"
        else:
            instruction["answer"] = content
            all_instructions.append(instruction)
            instruction = {}

pretty_df = pd.DataFrame(all_instructions)
pretty_df.to_json(
    "output/aws/train-instrucion-data/train.jsonl",
    orient="records",
    lines=True,
    force_ascii=False,
)
print(f"generate {len(pretty_df)} instructions")
```

数据准备好后还需要准备模板，模板内容如下 (`template.json`)：

```json
{
  "prompt": "关于光伏政策<{context}>, 提问: {question}",
  "completion": "{answer}"
}
```

# **4. 执行微调任务：启动 SageMaker JumpStart**

数据准备就绪后，我们就可以利用 SageMaker JumpStart 来执行微调。JumpStart 极大地简化了流程，它提供了预置的训练脚本和模型镜像，我们只需配置好环境并提交数据即可。

- **核心步骤：**

  1.  **配置环境：** 设置好 AWS 角色（Role）、区域（Region）和 S3 存储桶（Bucket）。
  2.  **上传数据：** 将上一步生成的 `train.jsonl` 和 `template.json` 文件上传到 S3 指定位置。
  3.  **定义 Estimator：** 创建一个 `JumpStartEstimator` 实例，在其中指定基础模型 ID (`meta-textgeneration-llama-3-2-1b-instruct`)、训练实例类型 (`ml.g5.xlarge`) 和输出路径。
  4.  **设置超参数：** 配置训练参数，如 `epoch`（训练轮数）、`max_input_length`（最大输入长度）等。
  5.  **启动训练：** 调用 `.fit()` 方法，传入训练数据在 S3 的路径，正式开始微调任务。

- **代码实现：**

```python
#
# 上传数据并微调
#
from time import strftime
import sagemaker
from sagemaker.s3 import S3Uploader
from sagemaker.jumpstart.estimator import JumpStartEstimator
import boto3

# 配置 SageMaker 角色和 S3 路径
role = "arn:aws-cn:iam::xxxx:role/sandbox-SageMakerExecutorRole"
aws_region = "cn-northwest-1"
bucket = "sandbox-sagemaker"

boto_session = boto3.Session(region_name=aws_region)
sagemaker_session = sagemaker.Session(boto_session=boto_session)  # type: ignore


local_data_dir = "output/aws/train-instrucion-data" # 数据+模板
train_data_location = f"s3://{bucket}/llm-fine-tuning/train-data"
train_output_location = f"s3://{bucket}/llm-fine-tuning/output"
S3Uploader.upload(local_data_dir, train_data_location)
print(f"Training data: {train_data_location}")

# 1. find the model ID for the model of your choice in the Built-in Algorithms with pre-trained Model Table.
# model_id = "huggingface-llm-gemma-7b-instruct"
# model_version = "1.3.10"

model_id = "meta-textgeneration-llama-3-2-1b-instruct"
model_version = "1.2.5"

# 2. using the model ID, define your training job as a JumpStart estimator.
estimator = JumpStartEstimator(
    model_id=model_id,
    model_version=model_version,
    role=role,
    region=aws_region,
    sagemaker_session=sagemaker_session,
    environment={"accept_eula": "true"},
    instance_type="ml.g5.xlarge",
    output_path=train_output_location,
)
estimator.set_hyperparameters(
    chat_dataset=True, instruction_tuned=False, epoch="5", max_input_length="1024"
)

# 3. run estimator.fit() on your model, pointing to the training data to use for fine-tuning.
job_name = f"llama3-2-1b-finetune-{strftime('%Y-%m-%d-%H-%M-%S')}"
print(f"Starting training job: {job_name}")
estimator.fit(
    inputs={
        "training": train_data_location,
    },
)
```

# **5. 模型部署：从训练产物到在线服务**

训练任务完成后，SageMaker 会将微调后的模型文件保存在 S3 中。下一步是将其部署为一个可以实时调用的 API 端点（Endpoint）。

本文提供了一种非常稳健的部署方法：**先将训练任务输出的模型文件下载、重新打包，再上传至 S3，最后使用该打包文件进行部署**。这个过程确保了模型制品的完整性和格式的规范性。

> **备选方案说明：** 在许多标准场景下，可以直接使用训练任务输出的 `S3ModelArtifacts` 路径来创建 `JumpStartModel` 并进行部署，无需手动下载和重新上传。本文提供的脚本为您展示了一个更明确、更可控的流程，尤其在需要对模型文件进行检查或修改时非常有用。

- **代码实现：**

```python
from urllib.parse import urlparse
import sagemaker
from sagemaker.jumpstart.model import JumpStartModel
import boto3
import subprocess
import os

# 和训练时保持一致
role = "arn:aws-cn:iam::xxxx:role/sandbox-SageMakerExecutorRole"
aws_region = "cn-northwest-1"
bucket = "sandbox-sagemaker"

boto_session = boto3.Session(region_name=aws_region)
sagemaker_session = sagemaker.Session(boto_session=boto_session)  # type: ignore
sagemaker_client = boto_session.client("sagemaker")
s3_client = boto_session.client("s3")

model_id = "meta-textgeneration-llama-3-2-1b-instruct"
model_version = "1.2.5"
training_job_name = "llama-3-2-1b-instruct-2025-07-04-04-58-52-402"  # MUST SETED

# ----- 1. 使用 JumpStartModel 从训练任务创建模型 -----
job_description = sagemaker_client.describe_training_job(
    TrainingJobName=training_job_name
)
model_data_s3_path = job_description["ModelArtifacts"]["S3ModelArtifacts"]
# image_uri = job_description["AlgorithmSpecification"]["TrainingImage"]
print(f"train job output: {model_data_s3_path}")

tmpdir = "./dist"
tmp_model_dir = os.path.join(tmpdir, "model_files")
local_packaged_model = os.path.join(tmpdir, "model.tar.gz")
os.makedirs(tmp_model_dir, exist_ok=True)
sync_command = ["aws", "s3", "sync", model_data_s3_path, tmp_model_dir]
subprocess.run(sync_command, capture_output=True, text=True)
print("sync model file done")
tar_command = ["tar", "-czvf", local_packaged_model, "-C", tmp_model_dir, "."]
subprocess.run(tar_command, capture_output=True, text=True)
print("tar model file done")
s3_url = urlparse(model_data_s3_path)
bucket_name = s3_url.netloc
target_s3_prefix = os.path.dirname(os.path.dirname(s3_url.path.lstrip("/")))
target_s3_key = f"{target_s3_prefix}/model.tar.gz"
s3_client.upload_file(local_packaged_model, bucket_name, target_s3_key)
final_model_data_path = f"s3://{bucket_name}/{target_s3_key}"
print(f"uploaded model tar.gz to {final_model_data_path}")
trained_model = JumpStartModel(
    model_id=model_id,
    model_version=model_version,
    model_data=final_model_data_path,
    role=role,
    sagemaker_session=sagemaker_session,
    instance_type="ml.g5.xlarge",
)

# ----- 2. 部署模型到端点 -----
predictor = trained_model.deploy(
    instance_type="ml.g5.xlarge", endpoint_name=training_job_name, accept_eula=True
)

print(f"部署成功端点: {training_job_name}")
```

# **6. 调用模型：与你的专属专家对话**

模型部署成功后，我们就可以像调用任何 API 一样与它交互了。需要注意的是，即便是微调后的模型，我们也需要遵循其基础模型（Llama 3.2 Instruct）的对话模板（Chat Template）来构建输入，以确保模型能正确理解角色和对话轮次。

- **调用流程：**

  1.  **获取对话模板：** 使用 `transformers` 库的 `AutoTokenizer` 下载或加载 Llama 3.2 的分词器和对话模板。
  2.  **构建请求：** 定义一个对话列表（包含 system prompt 和 user a message），然后使用 `tokenizer.apply_chat_template()` 方法将其转换为模型能够识别的字符串格式。
  3.  **发送请求：** 使用 `boto3` 客户端，将格式化后的输入和推理参数（如 `max_new_tokens`, `temperature`）封装成 JSON `payload`，调用 SageMaker Runtime 的 `invoke_endpoint` 方法。
  4.  **解析响应：** 解码返回的结果，即可看到微调后模型的回答。

- **代码实现：**

```python
# download_assets.py
from transformers import AutoTokenizer
import os

# The model ID from Hugging Face Hub
model_id = "meta-llama/Llama-3.2-1B-Instruct"
# The local directory where you want to save the files
tokenizer_directory = "./output/tokenizers/llama-3.2-1b-instruct"

os.makedirs(tokenizer_directory, exist_ok=True)

print(f"Downloading tokenizer for '{model_id}' to '{tokenizer_directory}'...")

# Download and save the tokenizer files
tokenizer = AutoTokenizer.from_pretrained(model_id)
tokenizer.save_pretrained(tokenizer_directory)

print("Tokenizer downloaded and saved successfully.")

```

调用模型测试脚本如下：

```python
import boto3
import json
from transformers import AutoTokenizer

# 您需要部署成功的端点的名称
endpoint_name = "llama-3-2-1b-instruct-2025-07-04-04-58-52-402"
aws_region = "cn-northwest-1"

boto_session = boto3.Session(region_name=aws_region)
runtime_client = boto_session.client(
    "sagemaker-runtime",
    region_name=aws_region,
)

# promt style 1: generate chat template from plain text
# user_prompt = (
#     "能否用三句话总结一下洛阳市新出台的减污降碳协同创新试点建设实施方案的核心内容？"
# )
# formatted_prompt = (
#     f"<|begin_of_text|><|start_header_id|>user<|end_header_id|>\n\n"
#     f"{user_prompt}<|eot_id|><|start_header_id|>assistant<|end_header_id|>\n\n"
# )
# promt style 2: generate chat template from tokenizer
conversation = [
    {
        "role": "system",
        "content": "你是一个新能源光伏领域的专家, 你可以帮助回答各类光伏领域的问题, 请用中文回答问题。",
    },
    {
        "role": "user",
        "content": "介绍《关于进一步加快杭州市光伏发电项目建设的实施意见》",
    },
]
# use the online/local model(meta-llama/Llama-3.2-1B-Instruct) tokenizer
tokenizer = AutoTokenizer.from_pretrained("./output/tokenizers/llama-3.2-1b-instruct")
formatted_prompt = tokenizer.apply_chat_template(
    conversation, tokenize=False, add_generation_prompt=True
)
print(f"{formatted_prompt}\n")
payload = {
    "inputs": formatted_prompt,
    "parameters": {
        "max_new_tokens": 1024,
        "top_p": 0.9,
        "temperature": 0.6,
        "repetition_penalty": 2.0,
        "stop": ["<|eot_id|>", "<|end_of_text|>"],
    },
}

encoded_payload = json.dumps(payload).encode("utf-8")
response = runtime_client.invoke_endpoint(
    EndpointName=endpoint_name,
    ContentType="application/json",
    Body=encoded_payload,
)
result_bytes = response["Body"].read()
result = result_bytes.decode("utf-8")
print(result)
```

# **7. 总结与展望**

本文详细地走过了从数据采集到模型部署的全过程，成功地使用 AWS SageMaker 将一个通用大模型微调为了一个光伏政策领域的专属模型。这个端到端的实战案例为您提供了在自有数据上进行模型微调的完整蓝图。

当然，这只是一个起点。在生产环境中，要获得最佳效果，还需要在更多方面进行探索和优化，例如：

- **构建验证集：** 在训练过程中监控模型在未见过数据上的表现，以防止过拟合。
- **超参数调优：** 系统性地寻找最佳的学习率、epoch 等参数组合。
- **数据融合：** 如前文所述，融合通用数据以保持模型的泛化能力。

我本人也处在不断学习和探索的阶段，文中的实践如有不足之处，欢迎大家交流指正。希望这篇教程能为您开启大模型微调之旅提供有力的帮助！
