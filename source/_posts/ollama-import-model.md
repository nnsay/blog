---
title: Ollama 导入模型
date: 2025-02-20 18:43:34
tags:
  - 工具
  - AI
excerpt: Ollama 导入和使用第三方模型
---

# 1. [Ollama 导入模型](https://github.com/ollama/ollama/blob/main/docs/import.md)

为什么要导入模型? 当然是 Ollama 官方模型仓库找不到需要的模型. 导入到 Ollama 模型可以被 Ollama 和其他模型一样进行统一话的运行和管理; Ollama 支持 三种方式导入模型:

- [Importing a Safetensors adapter](https://github.com/ollama/ollama/blob/main/docs/import.md#Importing-a-fine-tuned-adapter-from-Safetensors-weights)
- [Importing a Safetensors model](https://github.com/ollama/ollama/blob/main/docs/import.md#Importing-a-model-from-Safetensors-weights)
- [Importing a GGUF file](https://github.com/ollama/ollama/blob/main/docs/import.md#Importing-a-GGUF-based-model-or-adapter)

本文要介绍的第三种, 即导入一个 GGUF 架构的模型.

虽然 Ollama 支持导入 Safetensors 架构的模型但是有限制, 目前仅支持基于以下架构的模型:

- Llama (including Llama 2, Llama 3, Llama 3.1, and Llama 3.2);
- Mistral (including Mistral 1, Mistral 2, and Mixtral);
- Gemma (including Gemma 1 and Gemma 2); and
- Phi3

> [!WARNING]
> 尽管 Ollama 也建议可以利用 Llama.cpp 的 `convert_hf_to_gguf.py` 将 Safetensors 架构的模型转化为 GGUF 架构然后进行导入, 但是`convert_hf_to_gguf.py`本身也有限制并不是所有 huggingface 的 Safetensors 模型都可以转换.

# 2. 导入模型

## 2.1 下载模型

下载模型常用的模型仓库是 HuggingFace, 假设需要需要导入的模型是: nomic-embed-text-v1.5.f32.gguf, 这个模型是做文本的 embedding.

```bash
curl -L -O https://huggingface.co/nomic-ai/nomic-embed-text-v1.5-GGUF/resolve/main/nomic-embed-text-v1.5.f32.gguf
```

## 2.3 创建模型文件

进入上一步下载的模型目录, 创建一个符合 Ollama 规则的模型文件(Modelfile)

```
FROM ./nomic-embed-text-v1.5.f32.gguf
```

> [!NOTE]
> Ollama 模型文件支持更多的指令, 详情可以参考: [Ollama Model File](https://github.com/ollama/ollama/blob/main/docs/modelfile.md)

## 2.4 导入

执行 Ollama 创建命令导入模型:

```bash
➜  nomic-embed-text-v1.5-GGUF ollama create nomic-embed-text-v1.5
gathering model components
copying file sha256:ed3a84b570c5513bfd6bfe0ed4cdc8d5a5de5c6b5029fbbc2822d59fc893c1f8 100%
parsing GGUF
using existing layer sha256:ed3a84b570c5513bfd6bfe0ed4cdc8d5a5de5c6b5029fbbc2822d59fc893c1f8
writing manifest
success

➜  nomic-embed-text-v1.5-GGUF ollama list
NAME                            ID              SIZE      MODIFIED
nomic-embed-text-v1.5:latest    dcc702a536e7    547 MB    22 seconds ago
```

## 2.5 测试验证

```bash
➜  curl http://localhost:11434/api/embeddings -d '{
  "model": "nomic-embed-text-v1.5:latest",
  "prompt": "The sky is blue because of Rayleigh scattering"
}'

{"embedding":[0.5890257954597473,0.40048784017562866,-3.302999258041382,-0.5260277986526489,0.748874843120575,1.5186493396759033,-0.1251959204673767,0.3963647484779358,0.06800717115402222,-1.1089140176773071,0.6924994587898254,1.2778451442718506,1.145833969116211,1.0888454914093018,0.2503916621208191,0.2929406464099884,0.15178453922271729,-0.6345349550247192,-0.21010783314704895,-0.19611655175685883,-1.7958444356918335,-0.6290745735168457,0.0385594442486763,-0.6691150665283203,1.261095404624939,1.2772819995880127,-0.1595984399318695,-0.002360038459300995,-0.29747554659843445,-0.48057258129119873,1.2051044702529907,-0.6382184028625488,-0.5399197936058044 ... ...
```

# 3. 总结

_:warning: AI 生成_

通过本文，我们详细介绍了如何在 Ollama 中导入 GGUF 架构的模型，并通过实际操作展示了从下载模型到导入及验证的完整流程。Ollama 的灵活性使得我们能够轻松扩展其模型库，当官方仓库中找不到所需模型时，我们可以通过导入其他架构的模型来满足需求。

在实际应用中，推荐大家优先从 [HuggingFace](https://huggingface.co/models) 和 [ModelScope](https://modelscope.cn/models) 等平台寻找需要的模型，这两大平台提供了丰富的开源模型资源。如果你对 Ollama 的模型导入和使用 vit 有更多的问题，欢迎随时交流与探讨！

希望这篇博客能帮助你更好地了解 Ollama 的模型导入功能，同时也为你的 AI 应用开辟更多可能性！感谢你的阅读！
