---
title: 老款MacBook拯救指南
date: 2025-07-27 22:08:16
tags:
  - 日常
excerpt: 让老伙计在 Coding 的道路上继续前行
---

# 1. 老款 MacBook 要退休了吗?

我有一台 2015 早期款的 MacBook Pro, 它的 CPU 是 Intel 芯片,这款电脑配不高只有 4 核 8G, 且随着 Apple M 芯片的推出时间也越来越久, 很多编程的工具和类库都不支持这种平台了, 例如 Homebrew 中一些包依赖最新的 openssl@3, 但是最新的 openssl@3 在这种老平台上已经无法安装; 再例如一些 Python 的包如 torch, 也不支持 macosx x86_64 平台了. 如何让这样老伙计继续做一些轻量的开发或学习而不退休是本文想要讨论的内容.

# 2. 延寿方案

虽然很多软件工具不支持 macosx x86_64, 但是主流的 linux x86_64 肯定是支持, 所以首先想到可以通过虚拟化技术来延寿, 常见的虚拟化方案有两个:

- dev containner
- vm

开发容器方案社区中比较推崇, 且 VSCode 对 dev ontainner 也支持的比较好, 但是这种方案需要安装 Docker, Docker 本身在 Mac 平台就得带个虚拟机, 资源占用太多, 所以不太合适; 虚拟机的话肯定不能选择 GUI(例如 VM Fusion), 因为资源占用太多, 最好运行无 UI 的 Linux 系统且本身消耗较少, 经过查询我找到了 [Multipass](https://github.com/canonical/multipass).

Multipass 是 Ubuntu 的一个产品, 支持编排 Ubuntu 系统, Multipass 本身很轻量级, 启动的 Ubuntu 镜像也都是没有 GUI 的系统, 基本可以理解为是一个轻量级且只能编排特定 Linux 镜像的 WSL(Windows Subsystem for Linux).

# 3. 搭建 Mac 平台的 Linux 子系

Multipass 没有 Docker 那么重, 至少我启动实例后风扇不那么响, 而我如果启动 Docker 就基本干啥都很慢了, 使用它创建和配置 Linux 环境也很简单.

> [!IMPORTANT]
> 在安装 Multipass 的时候在其 Github Release 页面选择 1.14.1 的版本, 因为更新的版本对于 MacOS 有要求, 而我是 MacOS 12.7.6 的系统所以选择一个老版本.

## 3.1 查看镜像

```bash
➜  ~ multipass find
Image                       Aliases           Version          Description
core                        core16            20200818         Ubuntu Core 16
core18                                        20211124         Ubuntu Core 18
core20                                        20230119         Ubuntu Core 20
core22                                        20230717         Ubuntu Core 22
core24                                        20240603         Ubuntu Core 24
22.04                       jammy             20250702         Ubuntu 22.04 LTS
24.04                       noble,lts         20250704         Ubuntu 24.04 LTS
appliance:adguard-home                        20200812         Ubuntu AdGuard Home Appliance
appliance:mosquitto                           20200812         Ubuntu Mosquitto Appliance
appliance:nextcloud                           20200812         Ubuntu Nextcloud Appliance
appliance:openhab                             20200812         Ubuntu openHAB Home Appliance
appliance:plexmediaserver                     20200812         Ubuntu Plex Media Server Appliance

Blueprint                   Aliases           Version          Description
anbox-cloud-appliance                         latest           Anbox Cloud Appliance
charm-dev                                     latest           A development and testing environment for charmers
docker                                        0.4              A Docker environment with Portainer and related tools
jellyfin                                      latest           Jellyfin is a Free Software Media System that puts you in control of managing and streaming your media.
minikube                                      latest           minikube is local Kubernetes
ros2-humble                                   0.1              A development and testing environment for ROS 2 Humble.
ros2-jazzy
```

## 3.2 启动实例

选择一个合适的镜像启动实例, 作为以后第二系统.

```bash
multipass launch --cpus=4 --disk=40G --memory=4G --name=code lts
```

启动后即可以获得一个 Linux 实例, **可以视为子系统来使用**. 当然如果计算机资源较多可以创建多个.

## 3.3 配置子系统

首先进入子系统:

```bash
multipass shell code
```

进入后子系统后可以配置自己自定义的需求, 例如

- 安装[oh my bash](https://github.com/ohmybash/oh-my-bash) 让 SHELL 好看点
- 新建一个访问 Github 的 SSH 证书

最重要的一个是事情是添加宿主机 的 SSH 公钥到子系统, 这样方便在宿主机上使用 VSCode 的 Remote SSH 扩展来打开子系统的文件进行代码开发.

# 4. VSCode 和子系统集成

首先, 保证宿主机的公钥已经配置到了子系统.ssh/authorized_keys 中; 然后在宿主机的.ssh/config 中增加一个访问子系统的配置, 例如:

```toml
Host code
  HostName code
  User ubuntu
  IdentityFile ~/.ssh/id_rsa
```

通过以下命令可以测试是否可以连通:

```bash
ssh code
```

最后在宿主机上打开 VSCode, 搜索并安装 `Remote - SSH` 扩展, 安装完后可以在 Remote Exporer 中看到子系统的 SSH 连接, 点击连接, 等待连接成功后, 就可以继续选择打开子系统的文件系统, 进而进行代码开发, 所以老伙计又可以继续奋斗了.

# 5. 总结

通过使用轻量级的虚机工具 Multipass 实现类似 Windows 中的 WSL 一样的效果, 给老旧的 MacBook 创建一个 Linux 子系统, 进而避免编程工具和类库在宿主机不可用的尴尬状态, 缝缝补补又三年, 让老伙计在 Coding 的道路上继续前行.
