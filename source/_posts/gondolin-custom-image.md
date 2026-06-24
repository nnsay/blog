---
title: Gondolin 沙盒自定义镜像实践
date: 2026-06-24 19:14:59
tags:
  - 研究
  - DevOps
  - TypeScript
excerpt: 在 macOS 上构建 Gondolin 自定义镜像，并通过 TypeScript SDK 创建和执行基于该镜像的虚拟机。
---

# 声明

本文由 🤖AI 协作完成, 内容已过实际测试.

[Gondolin](https://github.com/earendil-works/gondolin) 是一个轻量级的沙盒虚拟机方案，基于 libkrun 和 QEMU 提供快速启动的隔离执行环境。最近项目中有需要定制沙盒镜像的需求，调研了一下 Gondolin 的自定义镜像流程，记录如下。

<!-- more -->

# 1. 背景

Gondolin 默认提供了一些基础镜像，但实际使用中往往需要预装特定依赖。比如我们需要一个同时内置 Node.js 和 Python 3 的运行环境，以便在沙盒中直接执行脚本，而不每次都重新安装依赖。

Gondolin 的自定义镜像构建目前支持 Alpine 发行版，也可以通过 OCI 镜像（如 Debian）作为 rootfs 基础。构建产物包括内核、initramfs 和 rootfs，可以直接通过 CLI 或 SDK 使用。

# 2. 前置准备

在 macOS 上构建自定义镜像需要安装以下工具：

```bash
brew install lz4 e2fsprogs
```

确保 `mke2fs` 在 PATH 上：

```bash
export PATH="$(brew --prefix e2fsprogs)/sbin:$PATH"
```

全局安装 Gondolin CLI：

```bash
npm install -g @earendil-works/gondolin
```

> Docker 仅在配置了 `postBuild.commands` 或跨架构构建时才会自动使用。如果只是简单镜像，不需要安装 Docker。

# 3. 镜像构建

## 3.1 配置

创建 `build-config.json`：

```json
{
  "arch": "aarch64",
  "distro": "alpine",
  "alpine": {
    "version": "3.23.0",
    "kernelPackage": "linux-virt",
    "kernelImage": "vmlinuz-virt",
    "rootfsPackages": [
      "linux-virt",
      "rng-tools",
      "bash",
      "ca-certificates",
      "curl",
      "e2fsprogs",
      "nodejs",
      "npm",
      "uv",
      "python3",
      "py3-pip",
      "openssh"
    ],
    "initramfsPackages": [],
    "krunfwVersion": "v5.2.1"
  },
  "rootfs": {
    "label": "gondolin-root"
  }
}
```

关键字段说明：

| 字段 | 说明 |
|------|------|
| `arch` | 目标架构，`aarch64`（Apple Silicon）或 `x86_64`（Intel） |
| `distro` | 发行版，目前仅支持 `alpine` |
| `alpine.rootfsPackages` | 安装到 rootfs 的 Alpine 包 |
| `postBuild.commands` | 构建后执行的命令 |
| `postBuild.copy` | 将主机文件复制到 guest rootfs |
| `oci.image` | 使用 OCI 镜像作为 rootfs 基础 |

## 3.2 执行构建

```bash
gondolin build --config build-config.json --output ./assets
```

构建成功后输出类似：

```
Build successful!
  Build ID: e3dbcc39-fd6e-56a3-ba30-cfa90e106a38
  Output directory: ./assets
```

构建产物：

```
assets/
  manifest.json          # 构建元数据
  vmlinuz-virt           # Linux 内核
  initramfs.cpio.lz4     # 压缩 initramfs
  rootfs.ext4            # Root 文件系统
  krun-kernel            # libkrun 兼容内核
  krun-empty-initrd      # krun 空 initrd
```

## 3.3 快速验证

使用 build ID 直接启动：

```bash
GONDOLIN_GUEST_DIR=./assets gondolin bash -- node --version && python3 --version
```

# 4. SDK 使用

## 4.1 基础用法

安装依赖：

```bash
npm install @earendil-works/gondolin
```

创建并执行 VM：

```typescript
import { VM } from "@earendil-works/gondolin";

const CUSTOM_IMAGE_ID = "e3dbcc39-fd6e-56a3-ba30-cfa90e106a38";

async function main() {
  const vm = await VM.create({
    sandbox: {
      imagePath: CUSTOM_IMAGE_ID,
    },
    sessionLabel: "my-custom-vm",
  });
  console.log("VM created, session ID:", vm.id);

  const result = await vm.exec("node --version");
  console.log("stdout:", result.stdout);

  await vm.close();
}

main().catch(console.error);
```

## 4.2 执行命令

`vm.exec()` 支持两种形式：

```typescript
// 字符串形式：通过 /bin/sh -lc 执行（支持管道、变量等）
const r1 = await vm.exec("echo $HOME | wc -c");

// 数组形式：直接执行（不搜索 PATH，必须写绝对路径）
const r2 = await vm.exec(["/bin/echo", "hello"]);
```

## 4.3 imagePath 的三种方式

| 方式 | 示例 | 说明 |
|------|------|------|
| Build ID | `"e3dbcc39-fd6e-56a3-ba30-cfa90e106a38"` | 使用本地镜像仓库中的构建 |
| 路径 | `"./assets"` | 指向包含 `manifest.json` 的目录 |
| 显式对象 | `{ kernelPath, initrdPath, rootfsPath }` | 直接指定各文件路径 |

## 4.4 手动启动控制

```typescript
const vm = await VM.create({
  sandbox: { imagePath: CUSTOM_IMAGE_ID },
  autoStart: false,  // 先不启动
});

// 配置完成后手动启动
await vm.start();

await vm.close();
```

## 4.5 文件系统操作

SDK 提供了简化的 guest 文件系统 API：

```typescript
// 读写文件
const content = await vm.fs.readFile("/etc/os-release", { encoding: "utf-8" });
await vm.fs.writeFile("/tmp/hello.txt", "hello from host\n");

// 目录操作
await vm.fs.mkdir("/tmp/workspace", { recursive: true });
const entries = await vm.fs.listDir("/tmp/workspace");

// 删除
await vm.fs.deleteFile("/tmp/hello.txt", { force: true });
```

## 4.6 流式输出

对于长时间运行的命令，可以使用流式输出：

```typescript
const proc = vm.exec("for i in 1 2 3; do echo $i; sleep 1; done", {
  stdout: "pipe",
});

for await (const chunk of proc) {
  process.stdout.write(chunk);
}

const result = await proc;
console.log("exitCode:", result.exitCode);
```

## 4.7 交互式 Shell

```typescript
const result = await vm.shell();
console.log("exitCode:", result.exitCode);
```

## 4.8 SSH 与 Ingress

需要交互式调试或暴露服务时：

```typescript
// SSH
const ssh = await vm.enableSsh({ user: "root", listenPort: 2222 });
console.log("SSH command:", ssh.command);

// Ingress
const ingress = await vm.enableIngress({ listen: "127.0.0.1:3000" });
console.log("Ingress listening on:", ingress.url);
```

# 5. 常用配置示例

**预装 Python 包：**

```json
{
  "postBuild": {
    "commands": [
      "pip3 install --break-system-packages httpie"
    ]
  }
}
```

**使用 OCI 镜像（Debian）：**

```json
{
  "arch": "aarch64",
  "distro": "alpine",
  "oci": {
    "image": "docker.io/library/debian:bookworm-slim",
    "pullPolicy": "if-not-present"
  }
}
```

> OCI 构建时 `alpine.rootfsPackages` 被忽略，rootfs 来自 OCI 镜像，但 boot 层仍由 Alpine 提供。

**固定 rootfs 大小：**

```json
{
  "rootfs": {
    "sizeMb": 2048
  }
}
```

# 6. 踩坑记录

1. **`mke2fs: command not found`** —— `brew install e2fsprogs` 后需要手动加 PATH。
2. **容器构建失败** —— macOS 上 `postBuild.commands` 会自动触发 Docker。如果不需要 post-build 步骤，直接移除 `postBuild` 即可原生构建。
3. **架构不匹配** —— Apple Silicon 用 `aarch64`，Intel 用 `x86_64`。跨架构构建需要 Docker：
   ```bash
   gondolin build --arch x86_64 --config build-config.json --output ./x64-assets
   ```

# 7. 参考

- [Gondolin 自定义镜像文档](https://github.com/earendil-works/gondolin/blob/main/docs/custom-images.md)
- [SDK VM 控制文档](https://github.com/earendil-works/gondolin/blob/main/docs/sdk-vm.md)
- [SDK 概述](https://github.com/earendil-works/gondolin/blob/main/docs/sdk.md)
