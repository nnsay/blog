---
title: 使用 LlamaIndex 构建 RAG 应用
date: 2025-09-15 11:08:35
tags:
- DevOps
- AI
  excerpt: 如何使用 LlamaIndex 框架，结合阿里云百炼模型服务，从零开始构建一个完整的 RAG 应用的实践指南。
---

# 声明

本文由 🤖AI 改写润色而成

# 1. LlamaIndex 简介

[LlamaIndex](https://docs.llamaindex.ai/en/stable/) 是一个面向大语言模型（LLM）应用开发的强大框架，专注于 Agent 和复杂工作流的构建，尤其擅长于开发检索增强生成（RAG）应用。

选择 LlamaIndex 的主要原因如下：

- **文档丰富**：官方文档分类清晰，提供了大量 Jupyter Notebook 示例，几乎覆盖所有核心模块，极大地降低了学习门槛。
- **生态支持广泛**：LlamaIndex 对接了众多生态服务，例如对国内阿里云百炼（DashScope）的支持非常充分，无论是 LLM、Embedding 还是 Rerank 模型服务，都能轻松集成。

本文将通过一个完整的代码实践，展示如何使用 LlamaIndex 构建一个简单的 RAG 应用。

# 2. 代码实践

本节所有代码均源自 Jupyter Notebook 项目，您可以直接复制并按顺序在自己的环境中执行。

## 2.1 安装依赖

首先，在您的 `pyproject.toml` 文件中添加以下依赖，并使用 `uv` 或 `poetry` 进行安装。

```toml
"llama-index-core>=0.12.49",
"llama-index-embeddings-dashscope>=0.3.0",
"llama-index-llms-dashscope>=0.3.1",
"llama-index-node-parser-topic>=0.2.0",
"llama-index-readers-json>=0.3.0",
"llama-index-vector-stores-postgres>=0.5.4",
```

## 2.2 全局配置

LlamaIndex 的全局配置是一大亮点。通过一次性设置，即可在项目各处直接调用模型服务，无需重复配置。这使得切换模型或服务时，仅需修改全局配置，而无需改动业务代码。

### 2.2.1 配置环境变量

创建一个 `.env` 文件来管理敏感信息，主要包括阿里云百炼的 API Key 和用于向量存储的 PostgreSQL 数据库地址。

```
DASHSCOPE_API_KEY=sk-xxx
DATABASE_URL=postgresql://root:1qaz2wsx@192.168.110.241:5432/vector_db?schema=public
```

使用 `dotenv` 库加载环境变量：

```python
import os
from dotenv import load_dotenv

load_dotenv(override=True)
```

### 2.2.2 配置 LlamaIndex 全局设置

在这里，我们主要配置全局的 LLM 和 Embedding 模型。

```python
import os
from llama_index.llms.dashscope import DashScope
from llama_index.embeddings.dashscope import (
    DashScopeEmbedding,
    DashScopeTextEmbeddingType,
)
from llama_index.core import Settings

# 配置 DashScope LLM，例如 qwen-max-latest
# 模型               最大输入   最大输出
# qwen-max-latest    129,024    8,192
# qwen2.5-14b-instruct 129,024    8,192
dashscope_llm = DashScope(
    model_name="qwen-max-latest",
    api_key=os.environ["DASHSCOPE_API_KEY"],
    max_tokens=8000,
)

# 配置 DashScope Embedding 模型
embedder = DashScopeEmbedding(
    model_name="text-embedding-v4",
    text_type=DashScopeTextEmbeddingType.TEXT_TYPE_DOCUMENT,
    api_key=os.environ["DASHSCOPE_API_KEY"],
    embed_batch_size=10,
)

# 应用全局设置
Settings.embed_model = embedder
Settings.llm = dashscope_llm
Settings.context_window = 131072  # LLM 的最大输入窗口
Settings.num_output = 8192         # 为文本生成保留的 Token 数量
Settings.chunk_overlap = 50        # 文本块重叠大小
Settings.chunk_size = 1024         # 文本块大小
```

# 3. 加载文档

LlamaIndex 支持从多种数据源加载文档。本文将演示两种常见方式：从 JSONL 文件和本地文件系统加载。

## 3.1 从 JSONL 文件加载

```python
from llama_index.readers.json import JSONReader

reader = JSONReader()
documents = reader.load_data(
    input_file="knowledge/solarsketch/knowledge.jsonl", extra_info={}
)
```

## 3.2 从本地文件加载

```python
from llama_index.core import SimpleDirectoryReader

reader = SimpleDirectoryReader(
    input_files=["knowledge/books/太阳能制造 光伏组件的环境设计概念.md"]
)
documents = reader.load_data(show_progress=True)
```

# 4. 拆分文档 (Node Parsing)

在 LlamaIndex 中，`Node` 是处理文档的基本单元，它代表了原始文档的一个片段。向量数据库中存储和检索的正是这些 `Node`。科学合理的文档拆分策略对 RAG 的性能和效果至关重要。

## 4.1 简单拆分 (Simple Spitting)

这是 LlamaIndex 的默认拆分方式，底层依赖 `nltk` 库进行语句拆分。

```python
from llama_index.core.node_parser import SimpleNodeParser

semantic_node_parser = SimpleNodeParser()
nodes = semantic_node_parser.get_nodes_from_documents(documents, show_progress=True)
```

## 4.2 语义拆分 (Semantic Splitting)

语义拆分是我个人推荐的方式，兼具效率与效果。它首先按句子拆分，然后通过计算语义相关性将关联紧密的句子合并成一个语义块。

```python
from llama_index.core.node_parser import (
    SemanticSplitterNodeParser,
)

semantic_node_parser = SemanticSplitterNodeParser(embed_model=Settings.embed_model)
nodes = semantic_node_parser.get_nodes_from_documents(documents, show_progress=True)
```

## 4.3 话题拆分 (Topic Splitting)

这是一种依赖 LLM 进行拆分的高级方式。它通过特定的 Prompt 指导大模型将文档内容分解为一系列独立的命题。以下是其默认的系统 Prompt：

```
PROPOSITION_SYSTEM_PROMPT = """Decompose the given content into clear and simple propositions, ensuring they are interpretable out of context. Follow these rules:
1. Split compound sentences into simple sentences. Maintain the original phrasing from the input whenever possible.
2. For any named entity that is accompanied by additional descriptive information, separate this information into its own distinct proposition.
3. Decontextualize the proposition by adding necessary modifiers to nouns or entire sentences and replacing pronouns (e.g., 'it', 'he', 'she', 'they', 'this', 'that') with the full name of the entities they refer to.
4. Present the results as a list of strings, formatted in JSON.
Here's an example:
Input: Title: Éostre. Section: Theories and interpretations, Connection to Easter Hares. Content:
The earliest evidence for the Easter Hare (Osterhase) was recorded in south-west Germany in 1678 by the professor of medicine Georg Franck von Franckenau, but it remained unknown in other parts of Germany until the 18th century. Scholar Richard Sermon writes that "hares were frequently seen in gardens in spring, and thus may have served as a convenient explanation for the origin of the colored eggs hidden there for children. Alternatively, there is a European tradition that hares laid eggs, since a hare's scratch or form and a lapwing's nest look very similar, and both occur on grassland and are first seen in the spring. In the nineteenth century the influence of Easter cards, toys, and books was to make the Easter Hare/Rabbit popular throughout Europe. German immigrants then exported the custom to Britain and America where it evolved into the Easter Bunny."
Output: [ "The earliest evidence for the Easter Hare was recorded in south-west Germany in 1678 by Georg Franck von Franckenau.", "Georg Franck von Franckenau was a professor of medicine.", "The evidence for the Easter Hare remained unknown in other parts of Germany until the 18th century.", "Richard Sermon was a scholar.", "Richard Sermon writes a hypothesis about the possible explanation for the connection between hares and the tradition during Easter", "Hares were frequently seen in gardens in spring.", "Hares may have served as a convenient explanation for the origin of the colored eggs hidden in gardens for children.", "There is a European tradition that hares laid eggs.", "A hare's scratch or form and a lapwing's nest look very similar.", "Both hares and lapwing's nests occur on grassland and are first seen in the spring.", "In the nineteenth century the influence of Easter cards, toys, and books was to make the Easter Hare/Rabbit popular throughout Europe.", "German immigrants exported the custom of the Easter Hare/Rabbit to Britain and America.", "The custom of the Easter Hare/Rabbit evolved into the Easter Bunny in Britain and America." ]"""
```

虽然理论上这种方式的检索效果更好，但它依赖 LLM 调用，因此**处理速度较慢，成本也更高**。

```python
from llama_index.node_parser.topic import TopicNodeParser

topic_node_parser = TopicNodeParser.from_defaults(similarity_method="llm")
nodes = topic_node_parser.get_nodes_from_documents(documents, show_progress=True)
```

# 5. 创建索引 (Indexing)

索引是将 `Node` 向量化并存入向量数据库的过程，为后续的快速检索做准备。

## 5.1 本地存储

对于开发和测试阶段，将索引存储在本地文件系统非常方便，但不建议在生产环境中使用。

```python
from llama_index.core import VectorStoreIndex

index = VectorStoreIndex(nodes)
index.storage_context.persist(persist_dir="./dist/llama_index")
```

## 5.2 使用 PGVector 数据库

在生产环境中，推荐使用专业的向量数据库，例如带有 `pg_vector` 扩展的 PostgreSQL。

```python
from llama_index.core import VectorStoreIndex, StorageContext
from llama_index.vector_stores.postgres import PGVectorStore
import os
from sqlalchemy import make_url

connection_string = os.environ["DATABASE_URL"]
url = make_url(connection_string)

vector_store = PGVectorStore.from_params(
    database=url.database,
    host=url.host,
    password=url.password,
    port=str(url.port),
    user=url.username,
    table_name="knowledge",
    embed_dim=1536,  # 必须与 Embedding 模型的维度一致
    hnsw_kwargs={
        "hnsw_m": 16,
        "hnsw_ef_construction": 64,
        "hnsw_ef_search": 40,
        "hnsw_dist_method": "vector_cosine_ops",
    },
)

storage_context = StorageContext.from_defaults(vector_store=vector_store)
index = VectorStoreIndex.from_documents(
    documents, storage_context=storage_context, show_progress=True
)
```

# 6. 查询 (Querying)

创建索引后，即可执行查询。通常有两种模式：

1.  **Retrieve + LLM**：将用户查询向量化，在向量数据库中检索最相似的 `Node`，将这些 `Node` 作为上下文信息提供给 LLM，最终由 LLM 生成答案。
2.  **Retrieve Only**：仅从向量数据库中检索并返回与查询相似的 `Node`，不经过 LLM 处理。

## 6.1 基于本地索引查询

```python
from llama_index.core import StorageContext, load_index_from_storage, QueryBundle
from llama_index.core.response.notebook_utils import display_source_node
from IPython.display import Markdown, display

# 从本地加载索引
storage_context = StorageContext.from_defaults(persist_dir="./dist/llama_index")
index = load_index_from_storage(storage_context)

# 创建查询引擎
query_engine = index.as_query_engine(similarity_top_k=3)

question = QueryBundle("什么是光场电气模块中的组件智能排布功能?")
response = query_engine.query(question)

print("=" * 20 + "Answer" + "=" * 20)
display(Markdown(response.response))

# 创建检索器，仅检索不生成
retriever = index.as_retriever(similarity_top_k=10)
retrieved_nodes = retriever.retrieve(question)

for i, node in enumerate(retrieved_nodes):
    print("=" * 20 + f"Retrived Node({i + 1})" + "=" * 20)
    display_source_node(node, source_length=10000)
```

## 6.2 基于 PGVector 索引查询

```python
from llama_index.core import VectorStoreIndex
from llama_index.vector_stores.postgres import PGVectorStore
import os
from sqlalchemy import make_url
from IPython.display import Markdown, display

connection_string = os.environ["DATABASE_URL"]
url = make_url(connection_string)

vector_store = PGVectorStore.from_params(
    database=url.database,
    host=url.host,
    password=url.password,
    port=str(url.port),
    user=url.username,
    table_name="knowledge",
    embed_dim=1536,
)

# 从向量数据库加载索引
index = VectorStoreIndex.from_vector_store(vector_store=vector_store)
query_engine = index.as_query_engine(similarity_top_k=3)

question = "SolarSketch平台的阴影实时分析功能如何实现8760小时逐时阴影模拟"
response = query_engine.query(question)

print("=" * 20 + "Answer:" + "=" * 20)
display(Markdown(response.response))  # type: ignore

# 仅检索
retriever = index.as_retriever(similarity_top_k=3)
retrieved_nodes = retriever.retrieve(question)

for i, node in enumerate(retrieved_nodes):
    print(print("=" * 20 + f"Retrived Knowledge({i + 1})" + "=" * 20))
    print(f"Score: {node.score:.4f}")
    print(f"Text: {node.get_content()}")
```

# 7. 高级技巧: Rerank

在 RAG 流程中，初步检索（Retrieve）是基于向量相似度进行的，得分高的 `Node` 不一定包含最准确的答案。此时，**Rerank（重排序）** 就可以发挥作用。它利用一个更精细的模型（通常是 LLM）对初步检索到的 `Node` 列表进行重新排序，将最相关的内容排在前面，从而提升最终生成答案的质量。

```python
from llama_index.core.postprocessor.llm_rerank import LLMRerank
from llama_index.core import QueryBundle
from llama_index.core.response.notebook_utils import display_source_node

question = QueryBundle("什么是光场电气模块中的组件智能排布功能?")

# 1. 初步检索
retriever = index.as_retriever(similarity_top_k=10)
retrieved_nodes = retriever.retrieve(question)

# 2. 使用 LLM Reranker 对结果进行重排序
ranker = LLMRerank(top_n=5) # 假设我们最终只需要最相关的5个
ranked_nodes = ranker.postprocess_nodes(retrieved_nodes, question)

# 打印重排序前的节点
print("--- Retrieved Nodes (Before Rerank) ---")
for i, node in enumerate(retrieved_nodes):
    print("=" * 20 + f"Retrived Node({i + 1}) | Score: {node.score:.4f}" + "=" * 20)
    display_source_node(node, source_length=1000)

# 打印重排序后的节点
print("\n--- Ranked Nodes (After Rerank) ---")
for i, node in enumerate(ranked_nodes):
    print("=" * 20 + f"Ranked Node({i + 1}) | Score: {node.score:.4f}" + "=" * 20)
    display_source_node(node, source_length=1000)
```

将重排序后的 `ranked_nodes` 作为上下文提供给 LLM，可以显著提高回答的准确性。

# 8. 总结

本文详细介绍了使用 LlamaIndex 框架构建一个完整 RAG 应用的核心步骤。我们从环境配置开始，依次经历了**加载文档、拆分文档、创建索引、执行查询**，并介绍了使用 **Rerank** 优化检索结果的高级技巧。

**核心要点回顾**:

- **配置先行**: LlamaIndex 的全局 `Settings` 极大地方便了模型的管理和切换。
- **拆分是关键**: `Node` 是 RAG 的基石，选择合适的拆分策略（如语义拆分）对最终效果有重要影响。
- **索引存储**: 根据场景选择合适的存储方式，开发时使用本地存储，生产环境推荐使用 PGVector 等专业向量数据库。
- **查询与优化**: 标准的查询引擎可以满足大部分需求，而引入 Rerank 可以在此基础上进一步提升检索结果的精准度，是 RAG 应用优化的重要一环。

通过遵循这些步骤，您可以快速地利用 LlamaIndex 和阿里云百炼等服务，构建出功能强大且高效的 RAG 应用。
