---
title: ds4 本地推理实战
date: 2026-05-29 11:49:23
tags:
  - 研究
  - AI
excerpt: 使用 ds4 在 DGX Spark 单机实现 DeepSeek V4 Flash 高性能推理
---

# 1. [ds4](https://github.com/antirez/ds4) 介绍

DwarfStar 是一个专为 DeepSeek V4 Flash 优化的小型原生推理引擎，在内存非常大的机器上支持 DeepSeek V4 PRO。它有意保持狭窄：不是通用的 GGUF 运行器，也不是其他运行时的包装器：它是完全自包含的。除了以正确和快速的方式运行模型外，项目的目标是提供 DS4 特定的加载、提示渲染、工具调用、KV 状态处理（RAM 和磁盘）、服务器 API 以及集成编码代理，所有这些都可以与编码代理或提供的 CLI 接口一起工作。还有用于 GGUF 和 imatrix 生成的工具，以及用于质量和速度测试的工具。

# 2. 工程验证

## 2.1 模型权重下载

源码中使用 ./download_model.sh 文件下载权重, 但是这个脚本是从 [HuggingFace](https://huggingface.co/antirez/deepseek-v4-gguf) 上下载, 这里我们通过 [modelscope](https://modelscope.cn/models/hf/antirez-deepseek-v4-gguf/summary) 下载, 本文选择 q2-imatrix 对应的模型, 大小为 86.72GB.

```bash
git clone https://github.com/antirez/ds4
cd ds4
mkdir -p gguf

# 下载模型权重
modelscope download --model hf/antirez-deepseek-v4-gguf DeepSeek-V4-Flash-IQ2XXS-w2Q2K-AProjQ8-SExpQ8-OutQ8-chat-v2-imatrix.gguf --local_dir ./gguf/
# 创建 ds4flash.gguf 软链接, 之后运行命令可以省去 -m 参数
ln -sfn ./gguf/DeepSeek-V4-Flash-IQ2XXS-w2Q2K-AProjQ8-SExpQ8-OutQ8-chat-v2-imatrix.gguf ds4flash.gguf
# 可选, 下载 MTP 模型权重
modelscope download --model hf/antirez-deepseek-v4-gguf DeepSeek-V4-Flash-MTP-Q4K-Q8_0-F32.gguf --local_dir ./gguf/
```

## 2.2 构建

ds4 有四种构建, 本文在 Spark 上执行, 所以选择: make cuda-spark

```bash
make                  # macOS Metal
make cuda-spark       # Linux CUDA, DGX Spark / GB10
make cuda-generic     # Linux CUDA, other local CUDA GPUs
make cpu              # CPU-only diagnostics build
```

## 2.3 运行

- CLI

  ```bash
  $ ./ds4 -p "介绍一下中国航天史"
  ds4: CUDA backend initialized on NVIDIA GB10 (sm_121)
  ds4: CUDA host registration skipped: operation not supported
  ds4: CUDA loading model tensors into device cache: 80.04 GiB
  ds4: CUDA startup model cache prepared 80.76 GiB of tensor spans in 11.188s
  ds4: cuda backend initialized for graph diagnostics
  ds4: context buffers 1053.75 MiB (ctx=32768, backend=cuda, prefill_chunk=4096, raw_kv_rows=4352, compressed_kv_rows=8194)
  processing 13 input tokens: 13/13 (100.0%)
  ... ...
  ```

- Server
  - 启动服务

    ```bash
    $ ./ds4-server --ctx 100000 --kv-disk-dir /tmp/ds4-kv --kv-disk-space-mb 8192 --host 0.0.0.0 --mtp ./gguf/DeepSeek-V4-Flash-MTP-Q4K-Q8_0-F32.gguf
    ```

    > [!TIP]
    >
    > - ds4-server 可以通过 -m 指定模型位置, 如果没有指定, 默认是 ./ds4flash.gguf
    > - ds4-server 可以通过 --mtp 来使用多token预测特性, 但是需要事先下载好 MTP 模型权重
    - 调用

      ```bash
      curl --location 'http://spark-264d.local:8000/v1/chat/completions' \
      --header 'Content-Type: application/json' \
      --header 'Authorization: ••••••' \
      --data '{
          "model": "deepseek-v4-flash",
          "messages": [
              {
                  "role": "user",
                  "content": [
                      {
                          "type": "text",
                          "text": "介绍一下中国航天史"
                      }
                  ]
              }
          ],
          "reasoning_effort": "none"
      }'
      ```

  ## 2.4 运行总结
  - 统一内存占用: 101GB
  - 平均速率: 15.10 t/s

# 3. 总结

ds4 是一个令人印象深刻的推理工具。虽然是 DeepSeek V4 专一定制引擎，但它在 DGX Spark 上真正发挥了统一内存架构的优势 — 128GB 内存跑 284B 参数模型, Server 模式生成速率 15.10 t/s, 约为 vLLM + Qwen3.6-27B-FP8 + 双 4090 (TP=2) 方案的 3 倍。部署极其简单, 一条 `make cuda-spark` 即可完成构建。在高性能单机上本地部署这类工具, 意味着无需访问外网 API, 没有 token 用量焦虑, 数据完全留在本地, 是本地推理的理想方向。

实践与测试中的一些观察:

- **内存占用高**: 101GB 统一内存 (模型 ~80GB + KV cache + 运行时开销), 接近 128GB 上限, 长上下文场景需关注 OOM 风险
- **速率数据来源**: 15.10 t/s 来自 Server 模式日志, 非 CLI 模式; 官方基准为 13.8 t/s, 差异可能因测试条件不同
- **量化精度未验证**: 本文使用 q2-imatrix (非对称 2-bit), 未对输出质量做系统评估, 实际使用需自行判断精度损失是否可接受
- **MTP 未实测**: 已下载 MTP 权重, 但未做开启前后的速率/质量对比
- **KV 磁盘持久化未验证**: 配置了 `--kv-disk-dir`, 但未测试重启后 KV cache 恢复效果, 这是 ds4 的核心差异化特性
- **工具调用/Agent 未涉及**: ds4-agent、DSML 原生工具调用、ds4-eval 等高级功能均未在本文验证
