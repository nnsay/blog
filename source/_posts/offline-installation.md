---
title: 服务器离线环境软件安装实践指南
date: 2025-10-31 10:29:53
tags:
  - DevOps
  - AI
excerpt: 涵盖了从 Caddy、Docker 到 Python、GPU 驱动及 AI 大模型的多种常用工具的离线部署方法与经验总结
---

# 声明

本文由 🤖AI 改写润色而成

# 1. 前言：为何需要离线安装

在客户服务器因涉密或特殊安全需求而无法连接互联网的背景下，离线安装软件成为一项必须掌握的技能。本文整理了若干常见工具的离线安装方法以供参考。

模拟无网环境的方法 (适用于主流 Linux 发行版):

```bash
# 1. 允许所有已经建立的连接和相关连接的出站流量 (这是保持 SSH 不断的关键)
sudo iptables -A OUTPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# 2. 允许本地回环地址的流量 (很多程序依赖它)
sudo iptables -A OUTPUT -o lo -j ACCEPT

# 3. 将出站链 (OUTPUT chain) 的默认策略设置为 DROP (丢弃)
# 这会阻止所有不符合以上规则的新连接
sudo iptables -P OUTPUT DROP

ping www.baidu.com

# 4. 将默认策略改回 ACCEPT (允许所有)
sudo iptables -P OUTPUT ACCEPT

# 5. 清空 OUTPUT 链中的所有规则
sudo iptables -F OUTPUT
```

> [!NOTE]
> 本文操作环境为 Ubuntu 22.04。

# 2. Caddy

在私有化部署场景中，Caddy 能便捷地用作反向代理及静态资源服务器。

## 2.1 下载二进制包

其离线安装采用静态二进制文件的方式，可从以下任一地址下载对应平台的二进制包：

- [GitHub Releases](https://github.com/caddyserver/caddy/releases)
- [Caddy Download](https://caddyserver.com/download)

## 2.2 配置服务

```bash
# Move the caddy binary into your $PATH
sudo mv caddy /usr/bin/
# Test that it worked
caddy version
# Create a group named caddy
sudo groupadd --system caddy
# Create a user named caddy with a writeable home directory
sudo useradd --system \
    --gid caddy \
    --create-home \
    --home-dir /var/lib/caddy \
    --shell /usr/sbin/nologin \
    --comment "Caddy web server" \
    caddy

# Next, choose a systemd unit file based on your use case
# The usual place to save the service file is: /etc/systemd/system/caddy.service
...

# start the service for the first time with the usual systemctl dance
sudo systemctl daemon-reload
sudo systemctl enable --now caddy
```

服务配置文件 (`/etc/systemd/system/caddy.service`):

```toml
[Unit]
Description=Caddy
Documentation=https://caddyserver.com/docs/
After=network.target network-online.target
Requires=network-online.target

[Service]
Type=notify
User=caddy
Group=caddy
ExecStart=/usr/bin/caddy run --environ --config /etc/caddy/Caddyfile
ExecReload=/usr/bin/caddy reload --config /etc/caddy/Caddyfile --force
TimeoutStopSec=5s
LimitNOFILE=1048576
PrivateTmp=true
ProtectSystem=full
AmbientCapabilities=CAP_NET_ADMIN CAP_NET_BIND_SERVICE

[Install]
WantedBy=multi-user.target
```

## 2.3 验证

```bash
# Verify that it is running:
systemctl status caddy
```

## 2.4 使用服务

```bash
# edit your configuration
sudo nano /etc/caddy/Caddyfile

# To read your full logs and to avoid lines being truncated:
journalctl -u caddy --no-pager | less +G

# If using a config file, you can gracefully reload Caddy after making any change
sudo systemctl reload caddy

# You can stop the service with
sudo systemctl stop caddy
```

# 3. Docker

以 Ubuntu 22.04 LTS (Jammy) 为例，离线安装主要分为下载 DEB 包和本地安装两个步骤。

## 3.1 下载 DEB 包

从官方仓库 [https://download.docker.com/linux/ubuntu/dists/pool/stable/](https://download.docker.com/linux/ubuntu/dists/pool/stable/) 下载所需文件。

```bash
# 定义要下载的软件包名称数组
package_names=(
  containerd.io_1.7.28-1~ubuntu.22.04~jammy_amd64.deb
  docker-ce_28.5.1-1~ubuntu.22.04~jammy_amd64.deb
  docker-ce-cli_28.5.1-1~ubuntu.22.04~jammy_amd64.deb
  docker-buildx-plugin_0.29.1-1~ubuntu.22.04~jammy_amd64.deb
  docker-compose-plugin_2.6.0~ubuntu-jammy_amd64.deb
)

# 定义基础 URL
base_url="https://download.docker.com/linux/ubuntu/dists/jammy/pool/stable/amd64/"

# 定义下载目录
download_dir="./tmp/docker-package"

# 创建目录（如果不存在）
mkdir -p "${download_dir}"

# 循环遍历软件包名称数组并下载每个包
for package_name in "${package_names[@]}"
do
  # 构造完整的输出文件路径
  output_file="${download_dir}/${package_name}"
  echo "正在下载 ${package_name} 到 ${output_file}..."
  # 使用 -o 指定输出文件的路径和名称
  curl -o "${output_file}" "${base_url}${package_name}"
done

echo "所有软件包下载完成！"
```

## 3.2 安装 DEB 包

```bash
sudo dpkg -i $download_dir/*.deb
```

## 3.3 验证

- **服务**
  ```bash
  sudo systemctl status docker
  sudo systemctl start docker
  sudo usermod -aG docker ${USER}
  newgrp docker
  ```
- **命令**
  ```bash
  docker version
  docker compose version
  ```

# 4. Python

不同于 Windows 和 macOS，Linux 系统常规的源码编译方式需要处理复杂的系统依赖。因此，本文推荐一种更便捷的方式：使用预编译的独立二进制包进行安装，思路类似于 `pyenv`。

## 4.1 下载二进制包

从 [python-build-standalone/releases](https://github.com/astral-sh/python-build-standalone/releases) 下载合适的版本，例如:
`cpython-3.12.12+20251014-aarch64-unknown-linux-gnu-install_only.tar.gz`

## 4.2 安装

```bash
tar -xzvf cpython-3.12.12+20251014-aarch64-unknown-linux-gnu-install_only.tar.gz
mv python /usr/local/python-3.12.12
```

## 4.3 验证

```bash
echo 'export PATH="/usr/local/python-3.12.12/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
python3 --version
pip3 --version
```

# 5. LLM 模型

首先在有网络的本地环境中下载所需模型，再将其同步到目标服务器上即可。

```bash
# 使用 huggingface-cli 下载
hf download opendatalab/MinerU2.5-2509-1.2B --local-dir=./llm-model/mineru2.5
# 使用 modelscope-cli 下载
modelscope download --model Qwen/Qwen3-32B --local_dir ./llm-model/qwen3-32b
modelscope download --model Qwen/Qwen3-Embedding-0.6B --local_dir ./llm-model/qwen3-embedding-0.6b
```

下载 [MinerU](https://github.com/opendatalab/MinerU) 模型:

```bash
uv pip install "mineru[core]"
uv run mineru-models-download --help

uv run mineru-models-download --source modelscope --model_type all
```

# 6. GPU + CUDA

> [!WARNING]
> 注意：本章节内容仅为资料整理，尚未经过实际操作验证。

## 6.1 驱动和 CUDA

- **视频教程**: [Ubuntu 22.04 企业级 AI 生产环境 nvidia 显卡驱动 cuda 安装](https://www.bilibili.com/video/BV1JH4y137sy)
- **文档教程**:
  - [Ubuntu 22.04 安装 NVIDIA 显卡驱动完整步骤](https://blog.csdn.net/a772304419/article/details/146601092)
  - [ubuntu22.04 安装 nvidia 显卡驱动——超详细、最简单](https://zhuanlan.zhihu.com/p/620214740)

## 6.2 Nvidia Container Toolkit

官方文档未直接提供离线安装包，但我们可以从社区发布的版本（如 GitHub Releases）中下载 DEB 软件包到本地，再进行安装。

首先从 [GitHub](https://github.com/NVIDIA/nvidia-container-toolkit/releases) 下载合适的包，然后拷贝到目标机器进行安装。

```bash
# 下载
toolkit="https://github.com/NVIDIA/nvidia-container-toolkit/releases/download/v1.18.0-rc.6/nvidia-container-toolkit_1.18.0-rc.6_deb_amd64.tar.gz"
wget -O ./nvidia-container-toolkit.tar.gz $toolkit

# 解压并拷贝到目标机器
mkdir -p nvidia-container-toolkit
tar zxvf nvidia-container-toolkit.tar.gz
ls -l release-v1.18.0-rc.6-experimental/packages/ubuntu18.04/amd64/

# 在目标机器安装
cd release-v1.18.0-rc.6-experimental/packages/ubuntu18.04/amd64/
sudo dpkg -i *.deb
# 如果出现依赖问题，需要提前下载好依赖包一并安装，或使用 apt --fix-broken install 联网修复

# 配置容器运行时
sudo nvidia-ctk runtime configure --runtime=docker

# 重启 Docker 服务
sudo systemctl restart docker

# 验证安装
sudo docker run --rm --gpus all nvidia/cuda:11.6.2-base-ubuntu20.04 nvidia-smi

dpkg -l | grep "nvidia"
```

# 7. Ollama

## 7.1 安装

```bash
# 下载并解压
curl -LO https://ollama.com/download/ollama-linux-amd64.tgz
sudo rm -rf /usr/lib/ollama
sudo tar -C /usr -xzf ollama-linux-amd64.tgz

# 启动 Ollama (前台运行)
ollama serve

# 在另一个终端验证
ollama -v
```

## 7.2 配置为服务

安装完成后，强烈建议在生产环境中将其配置为系统服务（Service）以确保稳定运行。

- **创建用户和组**:
  ```bash
  sudo useradd -r -s /bin/false -U -m -d /usr/share/ollama ollama
  sudo usermod -a -G ollama $(whoami)
  newgrp ollama
  groups
  ```
- **创建服务文件** (`/etc/systemd/system/ollama.service`):

  ```toml
  [Unit]
  Description=Ollama Service
  After=network-online.target

  [Service]
  ExecStart=/usr/bin/ollama serve
  User=ollama
  Group=ollama
  Restart=always
  RestartSec=3
  Environment="PATH=$PATH"
  Environment="OLLAMA_KEEP_ALIVE=-1"
  Environment="OLLAMA_HOST=0.0.0.0"
  Environment="OLLAMA_CONTEXT_LENGTH=256000"
  Environment="CUDA_VISIBLE_DEVICES=0,1,2,3"

  [Install]
  WantedBy=multi-user.target
  ```

  > [!NOTE] > `Environment` 部分按需修改。

- **启动服务**:
  ```bash
  sudo systemctl daemon-reload
  sudo systemctl enable --now ollama
  # 查看服务日志
  journalctl -u ollama -f
  ```

## 7.3 验证与使用

```bash
# 测试 embedding
host=127.0.0.1
curl -X POST http://$host:11434/api/embed \
  -H "Content-Type: application/json" \
  -d '{
    "model": "qwen3-embedding:0.6b",
    "input": "The quick brown fox jumps over the lazy dog."
  }'
# 测试 chat
curl http://$host:11434/api/chat -d '{
  "model": "qwen3:30b",
  "messages": [{
    "role": "user",
    "content": "How many letter r are in strawberry?"
  }],
  "think": false,
  "stream": false
}'
```

## 7.4 离线加载模型

离线加载模型的方法很简单：Ollama 的模型文件默认存储在 `~/.ollama/models/` 目录中。我们只需在有网络的机器上执行 `ollama pull <model_name>` 下载所需模型，然后将整个 `models` 目录同步到目标服务器的相应位置即可。

# 8. 总结：离线安装的核心经验

离线环境下的软件安装虽然比在线安装繁琐，但遵循一定的规律便能轻松应对。核心经验可以总结为以下几点：

1.  **优先选择二进制包**：尽可能寻找软件官方提供的静态编译二进制文件（如 Caddy、Python Standalone）或适用于目标系统的标准软件包（如 `.deb`、`.rpm`）。这是最简单、最不容易出错的方式，能最大限度地避免处理复杂的依赖关系。
2.  **“打包”思维处理依赖**：对于像 Docker 或 Nvidia 工具链这样有多个依赖包的软件，关键在于提前在有网环境中识别并下载所有必需的软件包。将这些包（程序本身 + 所有依赖）一同复制到目标服务器上，再使用 `dpkg -i *.deb` 等命令进行批量安装。
3.  **分离程序与数据**：对于 Ollama 这类应用，除了安装程序本身，还需要离线部署其使用的数据（大模型文件）。通用的方法是在线下载好所需数据，然后将存储数据的整个目录（如 `~/.ollama/models`）迁移到离线服务器的对应位置。
4.  **万变不离其宗**：无论软件多么复杂，离线安装的本质都是“资源搬运”。提前规划，理清程序、依赖、配置和数据这四个要素，就能从容应对大多数离线部署场景。
