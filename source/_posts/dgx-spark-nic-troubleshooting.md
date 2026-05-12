---
title: DGX Spark ConnectX-7 网卡丢失排查
date: 2026-05-12 14:42:17
tags:
  - DevOps
excerpt: 两台相同硬件的 DGX Spark，一台能看到 ConnectX-7 网卡，另一台却凭空消失
---

# 声明

本文由 🤖AI 协作完成, 内容已过实际测试.

# 1. 背景

DGX Spark 高速网卡"消失"之谜：一次 PCIe 热插拔的排查实录

我有两台 NVIDIA DGX Spark（GB10 GPU，128GB 统一内存），计划用高速线缆直连组成双机集群，部署一个单机显存放不下的大模型。

两台机器基本配置相同：

- **spark-2336**：已 OTA 升级到 DGX 7.5.0
- **spark-264d**：出厂版本 DGX 7.2.3

在准备组网时，我发现了一个奇怪的现象。

# 2. 发现问题

在 spark-264d 上执行 `ip -br link show`，能看到 4 个高速网卡接口：

```
enp1s0f0np0      DOWN    4c:bb:47:2b:26:4e
enp1s0f1np1      DOWN    4c:bb:47:2b:26:4f
enP2p1s0f0np0    DOWN    4c:bb:47:2b:26:52
enP2p1s0f1np1    DOWN    4c:bb:47:2b:26:53
```

但在 spark-2336 上，这些接口**完全不存在**，只有板载有线网卡和 WiFi：

```
enP7s7           DOWN    4c:bb:47:30:23:36
wlP9s9           UP      f8:3d:c6:6d:15:6c
```

同样的硬件平台，为什么一台有高速网卡，另一台没有？

# 3. 定位问题

## 第一层：PCIe 设备是否存在？

首先用 `lspci` 确认硬件层面的差异：

```bash
# spark-264d — 有 4 个 ConnectX-7 端口
$ lspci | grep -i 'mellanox\|connectx'
0000:01:00.0 Ethernet controller: Mellanox Technologies MT2910 Family [ConnectX-7]
0000:01:00.1 Ethernet controller: Mellanox Technologies MT2910 Family [ConnectX-7]
0002:01:00.0 Ethernet controller: Mellanox Technologies MT2910 Family [ConnectX-7]
0002:01:00.1 Ethernet controller: Mellanox Technologies MT2910 Family [ConnectX-7]

# spark-2336 — 一个都没有
$ lspci | grep -i 'mellanox\|connectx'
（无输出）
```

ConnectX-7 在 spark-2336 的 PCIe 总线上完全不可见。这看起来像是硬件问题——但真的是吗？

## 第二层：PCIe Bridge 下游设备去哪了？

进一步检查 PCIe bridge 的 sysfs 目录：

```bash
# spark-264d — bridge 下游有设备
$ ls /sys/bus/pci/devices/0000:00:00.0/ | grep '0000:'
0000:01:00.0
0000:01:00.1

# spark-2336 — bridge 下游为空
$ ls /sys/bus/pci/devices/0000:00:00.0/ | grep '0000:'
（无输出）
```

PCIe bridge 存在，但下游设备消失了。这不像是硬件未安装的表现——如果网卡物理不存在，连 bridge 都不应该分配资源。

## 第三层：内核启动日志揭示真相

关键一步——查看 `journalctl -b -k` 内核启动日志：

```bash
$ journalctl -b -k | grep -i 'mlx\|pci.*01:00\|cx7'
```

日志揭示了完整的故事：

```
11:33:11 pci 0000:01:00.0: [15b3:1021] type 00 class 0x020000 PCIe Endpoint   ← 启动时检测到了！
11:33:11 mlx5_core 0000:01:00.0: firmware version: 28.45.4028                  ← 固件正常
11:33:11 mlx5_core 0000:01:00.0 enp1s0f0np0: renamed from eth0                ← 网络接口创建成功
11:33:12 cx7-pcie-hotplug MTKP0001:00: PCIe hotplug driver initialized
11:33:12 cx7-pcie-hotplug MTKP0001:00: Hotplug enabled                        ← 热插拔启用！
...
11:33:27 cx7-pcie-hotplug MTKP0001:00: Cable removal                          ← 线缆未连接！
11:33:27 pcieport 0000:00:00.0: PCIe Bus Error: severity=Correctable
11:33:27 pcieport 0000:00:00.0:    [ 0] RxErr (First)                         ← 设备被移除
```

**真相大白**：ConnectX-7 网卡在启动时被正常识别、驱动加载成功、网络接口也创建了。但 15 秒后，`cx7-pcie-hotplug` 驱动检测到端口没有插入线缆，主动将整个 PCIe 设备热移除了。

# 4. 确认问题

为什么 spark-264d 没有这个行为？对比两台机器的热插拔状态：

```bash
# spark-2336（7.5.0）
$ cat /sys/bus/platform/drivers/cx7-pcie-hotplug/MTKP0001:00/pcie_hotplug/hotplug_enabled
1

# spark-264d（7.2.3）
$ cat /sys/bus/platform/drivers/cx7-pcie-hotplug/MTKP0001:00/pcie_hotplug/hotplug_enabled
0
```

进一步确认两台机器的 `mtk_pcie_hotplug` 内核模块完全相同（srcversion 一致），内核版本也相同（6.17.0-1014-nvidia）。差异在于：**OTA 7.5.0 升级后，系统固件/ACPI 层面启用了热插拔功能**。

完整的因果链：

```
OTA 升级到 7.5.0
  → ACPI/固件配置变更
  → cx7-pcie-hotplug 驱动的 hotplug_enabled = 1
  → 驱动开始监控 ConnectX-7 线缆状态
  → 检测到端口无线缆连接
  → 触发 "Cable removal" 事件
  → 将 ConnectX-7 从 PCIe 总线热移除
  → lspci / ip link 均不可见
```

# 5. 解决思路

理解了原因后，解决方案就很清晰：

**这不是 bug，是 feature。** DGX Spark 的 ConnectX-7 通过外部线缆连接，热插拔功能的设计目的是：

- 未插线时自动移除设备（节省功耗和系统资源）
- 插线后自动恢复设备（即插即用）

因此：

1. **不需要修复** spark-2336，只需插上高速线缆，网卡会自动恢复
2. **升级** spark-264d 到 7.5.0，保持两台机器版本一致
3. 用高速线缆直连两台设备，即可组网

## 升级 spark-264d

```bash
sudo apt update
sudo apt upgrade dgx-spark-ota-update-meta
sudo apt full-upgrade
sudo reboot
```

升级后验证：`hotplug_enabled = 1`，且同样因未插线而看不到高速网卡——行为与 spark-2336 一致。

# 6. 结论

| 项目     | 说明                                             |
| -------- | ------------------------------------------------ |
| 表面现象 | 高速网卡在 lspci 和 ip link 中完全不可见         |
| 误导方向 | 硬件故障、驱动缺失、PCIe 插槽问题                |
| 根本原因 | OTA 升级启用了 PCIe 热插拔，未插线时自动移除设备 |
| 解决方案 | 插上线缆即可，无需任何修复操作                   |

# 7. 所得所感

## 不要被表象欺骗

`lspci` 看不到设备，第一反应是硬件问题。但 Linux 的 PCIe 热插拔机制可以在运行时动态添加/移除设备。**设备不可见 ≠ 设备不存在**。

## 内核日志是最可靠的证据

`dmesg` 可能被清空，但 `journalctl -b -k` 保留了完整的启动日志。通过它我们看到了设备从"被检测到"到"被移除"的完整生命周期，这是破案的关键。

## 对比法是排查利器

两台相同硬件不同表现，最高效的排查方式就是逐层对比：

- PCIe 设备列表 → 确认差异范围
- sysfs 拓扑 → 确认是动态移除而非物理缺失
- 内核日志 → 找到移除的触发点
- 驱动参数 → 定位配置差异

## OTA 升级可能改变系统行为

同一个内核模块，升级前后行为不同。差异不在驱动代码本身，而在固件/ACPI 层面的配置。升级后要关注的不仅是"新增了什么"，还有"原有行为是否改变"。

# 8. 技术沉淀：热插拔日志速查

当怀疑网卡被热插拔机制关闭时，用以下命令快速确认：

```bash
# 查看 hotplug 驱动事件（最直接）
journalctl -b -k | grep 'cx7-pcie-hotplug'

# 查看网卡断开过程
journalctl -b -k | grep 'mlx5_core.*cleanup\|mlx5_core.*Link down\|Cable removal'

# 查看 PCIe 物理层断开
journalctl -b -k | grep 'RxErr'

# 实时监控插拔事件
journalctl -kf | grep -i 'cx7\|mlx5\|Cable'

# 检查当前 hotplug 状态
cat /sys/bus/platform/drivers/cx7-pcie-hotplug/MTKP0001:00/pcie_hotplug/hotplug_enabled
```

事件时间线模式：

```
Hotplug enabled → mlx5_core Link down → E-Switch cleanup → Cable removal → RxErr
```

看到这个序列，就是热插拔在工作。插上线缆后应看到反向序列（Cable insertion → 设备枚举 → 驱动加载 → Link up）。

---

_排查环境：NVIDIA DGX Spark / Ubuntu 24.04 / Kernel 6.17.0-1014-nvidia / NVIDIA Driver 580.142 / ConnectX-7 (MT2910)_
_排查时间：2026-05-12_
