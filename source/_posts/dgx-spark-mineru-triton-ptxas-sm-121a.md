---
title: MinerU sm_121a 编译错误排查
date: 2026-05-19 16:43:18
tags:
  - DevOps
  - Troubleshooting
excerpt: MinerU 在 NVIDIA DGX Spark / GB10 上触发 sm_121a / ptxas 编译错误
---

# 声明

本文由 🤖AI 协作完成, 内容已过实际测试.

# 1. 问题背景

本次问题发生在 NVIDIA DGX Spark 机器上，核心环境如下：

```text
OS: Ubuntu 24.04.4 LTS
GPU: NVIDIA GB10
Driver: 580.142
nvidia-smi CUDA Version: 13.0
容器内 PyTorch: 2.9.0+cu129
容器内 CUDA Toolkit: 12.9
vLLM: 0.11.2
MinerU backend: vllm-async-engine
```

MinerU 使用 Docker Compose 部署，并开启 VLM 模式解析 PDF。调用 `/file_parse` 接口时，MinerU 会初始化 VLM 推理引擎，内部链路大致为：

```text
MinerU API
→ /file_parse
→ MinerU VLM pipeline
→ vllm-async-engine
→ Qwen2VLForConditionalGeneration
→ vLLM / FlashAttention / Triton kernel
→ CUDA PTX 编译
```

本次问题的核心是：**DGX Spark / GB10 的 `sm_121a` 架构在容器内 Triton 默认使用的 `ptxas` 中没有被识别，导致 vLLM EngineCore 初始化失败。**

# 2. 初始错误现象

MinerU API 服务本身可以正常启动：

```text
Start MinerU FastAPI Service: http://0.0.0.0:8000
Application startup complete
Uvicorn running on http://0.0.0.0:8000
```

但在第一次请求 `/file_parse` 时，MinerU 开始初始化 VLM 引擎：

```text
Using vllm-async-engine as the inference engine for VLM.
```

随后 vLLM 加载模型：

```text
Resolved architecture: Qwen2VLForConditionalGeneration
Using max model len 4096
Starting to load model ...
Model loading took 2.1612 GiB memory
```

模型加载之后，vLLM 进入 profile / warmup / kernel 编译流程，然后报错：

```text
Internal Triton PTX codegen error

ptxas fatal : Value 'sm_121a' is not defined for option 'gpu-name'

Repro command:
/usr/local/lib/python3.12/dist-packages/triton/backends/nvidia/bin/ptxas
-lineinfo -v --gpu-name=sm_121a /tmp/xxx.ptx -o /tmp/xxx.ptx.o
```

最终 MinerU 请求失败：

```text
RuntimeError: Engine core initialization failed.
```

# 3. 关键错误定位

这次错误最重要的几行是：

```text
ptxas fatal : Value 'sm_121a' is not defined for option 'gpu-name'
```

以及：

```text
/usr/local/lib/python3.12/dist-packages/triton/backends/nvidia/bin/ptxas
```

这说明：

1. Triton 生成了面向 `sm_121a` 的 PTX。
2. Triton 调用 `ptxas` 编译该 PTX。
3. 被调用的 `ptxas` 不认识 `sm_121a`。
4. 因此 Triton kernel 编译失败。
5. vLLM EngineCore 初始化失败。
6. MinerU `/file_parse` 返回错误。

这里的重点不是 MinerU 业务逻辑，而是 **vLLM 初始化过程中触发的 Triton JIT 编译失败**。

# 4. 错误触发链路

从日志堆栈看，失败链路大致如下：

```text
MinerU /file_parse
→ aio_do_parse
→ _async_process_vlm
→ aio_vlm_doc_analyze
→ ModelSingleton().get_model(...)
→ AsyncLLM.from_engine_args(...)
→ EngineCoreClient.make_async_mp_client(...)
→ launch_core_engines(...)
→ EngineCore 初始化
→ _initialize_kv_caches
→ determine_available_memory
→ profile_run
→ self.model.embed_multimodal(...)
→ Qwen2VL visual encoder
→ apply_rotary_pos_emb_vision
→ vllm_flash_attn/layers/rotary.py
→ vllm_flash_attn/ops/triton/rotary.py
→ Triton JIT compile
→ ptxas --gpu-name=sm_121a
→ ptxas fatal
```

关键失败位置是：

```text
vllm/vllm_flash_attn/ops/triton/rotary.py
```

即 vLLM 内部 FlashAttention / rotary embedding 相关 Triton kernel 编译失败。

# 5. 容器环境验证

进入容器后执行：

```bash
which ptxas || true
/usr/local/cuda/bin/ptxas --version || true

python3 - <<'PY'
import triton, torch, os
print("torch:", torch.__version__)
print("cuda available:", torch.cuda.is_available())
print("device:", torch.cuda.get_device_name(0))
print("capability:", torch.cuda.get_device_capability(0))
print("TRITON_PTXAS_PATH:", os.environ.get("TRITON_PTXAS_PATH"))
PY
```

输出显示：

```text
/usr/local/cuda/bin/ptxas

ptxas: NVIDIA (R) Ptx optimizing assembler
Cuda compilation tools, release 12.9, V12.9.86

torch: 2.9.0+cu129
cuda available: True
device: NVIDIA GB10
capability: (12, 1)
TRITON_PTXAS_PATH: None
```

这组信息说明：

1. 容器内确实存在 `/usr/local/cuda/bin/ptxas`。
2. PyTorch 可以识别 NVIDIA GB10。
3. GB10 的 capability 是 `(12, 1)`。
4. 但 `TRITON_PTXAS_PATH` 没有设置。
5. 因此 Triton 默认没有使用 `/usr/local/cuda/bin/ptxas`。
6. 从报错路径看，Triton 实际使用的是 Python 包内置的 `ptxas`：

```text
/usr/local/lib/python3.12/dist-packages/triton/backends/nvidia/bin/ptxas
```

这就是问题的关键。

# 6. 根因结论

根因可以明确归纳为：

```text
DGX Spark / GB10 的计算能力是 12.1；
vLLM 在初始化 Qwen2-VL VLM 模型时触发 Triton JIT 编译；
Triton 为 GB10 生成了 sm_121a 目标架构；
但 Triton 默认调用的 package 内置 ptxas 不支持 sm_121a；
因此 ptxas 编译失败；
最终导致 vLLM EngineCore 初始化失败，MinerU 解析请求失败。
```

本质上是一个 **新 GPU 架构 + 容器内 Triton / ptxas 版本路径不匹配** 的问题。

# 7. 解决方案

## 7.1 显式指定 Triton 使用系统 CUDA ptxas

在 MinerU 的 Docker Compose 服务中增加环境变量：

```yaml
environment:
  TRITON_PTXAS_PATH: /usr/local/cuda/bin/ptxas
  TORCH_CUDA_ARCH_LIST: "12.1a"
  VLLM_WORKER_MULTIPROC_METHOD: spawn
  PYTORCH_ALLOC_CONF: expandable_segments:True
```

其中最关键的是：

```yaml
TRITON_PTXAS_PATH: /usr/local/cuda/bin/ptxas
```

它的作用是让 Triton 不再默认使用：

```text
/usr/local/lib/python3.12/dist-packages/triton/backends/nvidia/bin/ptxas
```

而是改用：

```text
/usr/local/cuda/bin/ptxas
```

## 7.2 推荐 Compose 配置片段

```yaml
services:
  mineru-api:
    image: your-mineru-image
    container_name: mineru-api
    environment:
      TRITON_PTXAS_PATH: /usr/local/cuda/bin/ptxas
      TORCH_CUDA_ARCH_LIST: "12.1a"
      VLLM_WORKER_MULTIPROC_METHOD: spawn
      PYTORCH_ALLOC_CONF: expandable_segments:True
    ipc: host
    ulimits:
      memlock: -1
      stack: 67108864
```

如果容器中没有 `/usr/local/cuda/bin/ptxas`，可以把宿主机 CUDA 挂载进去：

```yaml
volumes:
  - /usr/local/cuda:/usr/local/cuda:ro
```

## 7.3 清理编译缓存

修改环境变量后，建议清理 Triton / torch compile 缓存：

```bash
docker exec -it mineru-api-1 bash -lc '
rm -rf ~/.triton/cache ~/.cache/triton /tmp/torchinductor_* /tmp/triton_* || true
'
```

然后重启服务：

```bash
docker compose down
docker compose up -d
docker compose logs -f mineru-api
```

# 8. 修复后的验证结果

增加环境变量后，再次请求 MinerU，日志显示 vLLM 引擎已经成功初始化：

```text
Using vllm-async-engine as the inference engine for VLM
Initializing a V1 LLM engine
Starting to load model
Using FLASH_ATTN backend
Model loading took 2.1612 GiB memory
Encoder cache will be initialized
Using cache directory ... for vLLM's torch.compile
Compiling a graph for dynamic shape takes 10.12 s
Available KV cache memory: 5.49 GiB
GPU KV cache size: 480,032 tokens
Graph capturing finished
init engine ... took 46.84 seconds
get vllm-async-engine predictor cost: 84.0s
Two Step Extraction: 100%
```

最关键的是，修复后的日志中没有再出现：

```text
ptxas fatal : Value 'sm_121a' is not defined for option 'gpu-name'
```

最终请求返回：

```text
POST /file_parse HTTP/1.1" 200 OK
```

这说明：

```text
vLLM EngineCore 初始化成功
Triton kernel 编译不再触发 sm_121a ptxas fatal
MinerU VLM 主解析流程成功完成
/file_parse 返回 200 OK
```

# 9. 修复后出现的非相关错误

修复后日志里仍然出现了一个错误：

```text
mineru.utils.llm_aided:llm_aided_title - Connection error
httpx.ConnectError: [Errno -2] Name or service not known
Request:
http://vllm-qwen:8000/v1/chat/completions
```

该错误发生在 MinerU 的 AI 辅助标题优化阶段：

```text
llm_aided_title
```

配置中可以看到：

```text
base_url: http://vllm-qwen:8000/v1
model: qwen
enable: True
```

该问题的原因是 MinerU 容器无法解析 `vllm-qwen` 这个服务名，属于 Docker 网络 / DNS / 外部 LLM 服务配置问题。

它与本次 `sm_121a` / `ptxas` 问题无关，并且没有阻断 MinerU 主流程，因为最终请求已经返回：

```text
200 OK
```

如果不需要 AI title，可以关闭该功能；如果需要，则需要单独修复 `vllm-qwen` 的网络访问配置。

# 10. 最终结论

本次问题的最终结论是：

```text
MinerU 在 DGX Spark / GB10 上使用 vllm-async-engine 初始化 VLM 模型时，
vLLM 触发 Triton JIT 编译；
Triton 为 GB10 生成 sm_121a 目标架构；
但默认调用的 Triton package 内置 ptxas 不支持 sm_121a；
导致 ptxas fatal，进而导致 vLLM EngineCore 初始化失败。
```

有效修复方式是：

```text
显式设置 TRITON_PTXAS_PATH=/usr/local/cuda/bin/ptxas，
让 Triton 使用容器内系统 CUDA Toolkit 的 ptxas，
同时设置 TORCH_CUDA_ARCH_LIST=12.1a。
```

修复后验证结果：

```text
ptxas sm_121a fatal 不再出现
vLLM EngineCore 初始化成功
torch.compile 成功
CUDA graph capture 成功
MinerU Two Step Extraction 成功
/file_parse 返回 200 OK
```

因此，本次 `sm_121a` 问题本质是：

```text
DGX Spark / GB10 新架构适配问题
+
Triton 默认 ptxas 路径选择问题
+
vLLM 在 Qwen2-VL rotary kernel 编译阶段触发

```
