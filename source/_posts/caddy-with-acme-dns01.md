---
title: 本地 Caddy 接入 alidns 插件（ACME DNS-01 验证）
date: 2026-09-20 16:53:48
tags:
  - DevOps
excerpt: 让内网服务用上“浏览器开箱即信”的 HTTPS：xcaddy 编译 Caddy + alidns 插件，走 DNS-01 签发 Let's Encrypt 泛域名证书——客户端零导入，续期全自动
---

# 声明

本文由 🤖AI 协作完成.

> 2026-09-20 实测记录。目标：本地 macOS 得到一个编译了 `dns.providers.alidns` 的 Caddy，
> 用于验证泛域名证书的 ACME DNS-01 签发路线（与自产 CA 的 `tls internal` 互为备选）。

# 1. 背景

- Caddy 内置 ACME 客户端，但 **DNS provider 一律是第三方 Go module**，官方标准镜像/Homebrew 二进制里都没有。
- `/docs/modules/` 官网文档页只描述"如果装了它配置长什么样"，**不代表已内置**；判定标准是 `caddy list-modules`。
- 插件必须编译进二进制，两种方式：
  1. [caddyserver.com/download](https://caddyserver.com/download) 在线勾选插件定制下载；
  2. 本地 `xcaddy` 构建（本文采用，适合反复迭代）。

相关文档：

- 官方 Wiki（权威指南）：https://caddy.community/t/how-to-use-dns-provider-modules-in-caddy-2/8148
- alidns 插件仓库：https://github.com/caddy-dns/alidns
- Docker 定制构建：https://hub.docker.com/_/caddy 的 "Adding custom Caddy modules" 一节

# 2. 构建

```bash
# 1. 工具链（一次性）
brew install go
go install github.com/caddyserver/xcaddy/cmd/xcaddy@latest   # 装到 ~/go/bin，需在 PATH 中

# 2. 编译（xcaddy 自动拉 Caddy 源码 + 插件）
cd "$(mktemp -d)"
xcaddy build --with github.com/caddy-dns/alidns              # 可加版本：xcaddy build v2.11.4 --with ...
```

产物为当前目录下的 `caddy` 二进制。

# 3. 安装与验证

```bash
# 验证插件已编入
./caddy version
./caddy list-modules | grep alidns       # 期望：dns.providers.alidns

# 安装（本项目本地无 Homebrew caddy、无 brew services，~/.local/bin 即唯一实例）
mkdir -p ~/.local/bin
cp ./caddy ~/.local/bin/caddy
caddy version && caddy list-modules | grep alidns
```

注意：这是手动二进制，没有服务管理器托管，不会开机自启；临时验证够用，
需要常驻时自行配 LaunchAgent。

# 4. 本地测试配置

测试 Caddyfile 就一个文件：`~/.config/Caddyfile`（凭证走环境变量，跑之前 export 两个变量即可）：

```caddyfile
{
	acme_dns alidns {
		access_key_id {env.ALIYUN_ACCESS_KEY_ID}
		access_key_secret {env.ALIYUN_ACCESS_KEY_SECRET}
	}
	email jimmy.w@aliyun.com
}

# 泛域名测试（alidns 需托管 newtranx.com 的 DNS）
*.nnsay.cn {
	header Content-Type application/json
	respond `{"ok":true,"msg":"caddy alidns test"}` 200
}

```

> [!TIP]
>
> 可以通过本地 hosts 添加解析记录: `127.0.0.1   debug.nnsay.cn` 来测试

```bash
export ALIYUN_ACCESS_KEY_ID=... ALIYUN_ACCESS_KEY_SECRET=...
caddy run --config ~/.config/Caddyfile
```

启动后观察日志：Caddy 会调用 alidns API 写一条 `_acme-challenge` TXT 记录，
签发成功则证书落盘到本地存储目录，随后泛域名站点可用 HTTPS 访问。

alidns 凭证建议用只授 DNS 权限的 RAM 子账号 AK（详见 https://github.com/libdns/alidns ）。

> 注：`--envfile` 传环境文件也行（Caddy **不会**自动加载 `.env`）；凭证同样可以直接
> 写进 Caddyfile，本地测试图省事可接受，交付配置仍建议环境变量。

# 5. 验证结果（2026-09-20 实测通过）

## 5.1. 成功日志的关键节点

```
http.acme_client    trying to solve challenge  {"challenge_type": "dns-01", ...}
http.acme_client    authorization finalized   {"authz_status": "valid"}
http.acme_client    successfully downloaded available certificate chains  {"count": 3, ...}
tls.obtain          certificate obtained successfully  {"issuer": "acme-v02.api.letsencrypt.org-directory"}
```

`got renewal info` 日志会给出下次续期窗口（到期前 2/3 生命周期内随机挑时间），
DNS-01 续期同样全自动（重写 TXT → 验证 → 换证），无需人工干预。

## 5.2. 端到端验证

不动 `/etc/hosts` 的话，用 `--resolve` 模拟解析（效果等价）：

```bash
curl -s --resolve debug.nnsay.cn:443:127.0.0.1 https://debug.nnsay.cn/
# {"ok":true,"msg":"caddy alidns test"}

# 查看实际下发的证书
echo | openssl s_client -connect 127.0.0.1:443 -servername debug.nnsay.cn \
  | openssl x509 -noout -subject -issuer -dates
# subject=CN=*.nnsay.cn
# issuer=C=US, O=Let's Encrypt, CN=YE2     ← 公共 CA，浏览器/curl 零导入直接信任
# notAfter=Dec 19 ...（90 天有效期）
```

curl 未带 `-k`、未配置任何 CA——原生信任通过，验证了"客户端零导入"这条路线特征。

# 6. Docker 部署等价方案

交付/服务器侧用官方 builder 镜像多阶段构建（compose 从 `image:` 改 `build:`）：

```dockerfile
FROM docker.m.daocloud.io/library/caddy:2-builder AS builder
RUN xcaddy build --with github.com/caddy-dns/alidns

FROM docker.m.daocloud.io/library/caddy:2-alpine
COPY --from=builder /usr/bin/caddy /usr/bin/caddy
```

容器内凭证通过 compose `environment:` 注入。

# 7. 路线对比（何时选 alidns，何时选 tls internal）

|            | 自产 CA（`tls internal`） | ACME + alidns（DNS-01）       |
| ---------- | ------------------------- | ----------------------------- |
| 外网依赖   | 无（零依赖交付）          | 需要（DNS API 可达）          |
| 客户端     | 需导入 root 证书          | 零导入                        |
| 每客户隔离 | 重跑 `scripts/gen-ca.sh`  | 需各自域名                    |
| 适用       | 私有化交付兜底            | 有网客户，浏览器/工具开箱即信 |

两条路线可在同一 Caddyfile 内按站点共存。

# 8. QA（踩坑排查）

## 8.1. `fork/exec ./caddy: permission denied`

`go build -o` 指向**已存在的目录**时，二进制会写进该目录内部，而不是覆盖为该路径。
若在 `tmp/caddy/` 目录上执行 build，产物实际是 `tmp/caddy/caddy`，
shell 里 `./caddy` 解析到目录本身，就会报 `fork/exec: permission denied`。

**解决**：build 在一个不含同名文件/目录的空目录里进行；或执行 `tmp/caddy/caddy`。
可用 `file ./caddy` 判断——输出 `directory` 说明执行的是目录。

## 8.2. `cannot found specified zone:nnsay.cn`（AppendRecords 失败）

alidns API 报 `cannot found specified zone`：**AK 所属的阿里云账号在
「云解析 DNS」域名列表里看不到该域名**。与 Caddyfile 语法无关，LE/ZeroSSL
两个 CA 都会在同一步失败（瓶颈在 DNS API 这头）。

**解决**：换用解析所在账号的 AK；若账号一致，给 RAM 用户授 `AliyunDNSFullAccess`。
排查入口：阿里云控制台 → 云解析 DNS → 域名列表，确认域名在当前 AK 的账号名下。
