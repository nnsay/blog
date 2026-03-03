---
title: Keycloak OIDC 实践
date: 2026-03-03 14:18:58
tags:
  - DevOps
  - Tips
excerpt: 私有化部署避坑指南：如何用 Docker 快速搭建支持 HTTP 的 Keycloak OIDC 服务
---

# 声明

本文由 🤖AI 协作完成, 内容已过实际测试.

# 前言：为什么我们需要一个“不安全”的 Keycloak？

在做企业级软件的私有化交付、内网 POC（概念验证）或本地开发时，我们经常面临一个尴尬的处境：**环境受限**。

- **没有域名**：只有裸 IP（如 `192.168.x.x`）。
- **没有 SSL 证书**：无法配置 HTTPS。
- **网络隔离**：服务跑在 Docker 容器里，宿主机防火墙策略严格。

Keycloak 作为业界最强大的开源 IAM（身份访问管理）系统，默认的安全策略非常严格（强制 HTTPS、强制 Localhost 访问），这导致在上述“不完美”的内网环境中部署时，经常遇到**管理控制台进不去**、**OIDC 登录无限重定向**、**Issuer 校验失败**等“神坑”。

本文将总结一套经过实战验证的**最佳实践**，教你如何通过 Docker 快速搭建一个**开箱即用、支持 HTTP、配置固化**的 Keycloak 服务。

---

# 1. 核心架构：容器化配置的“三大纪律”

为了让 Keycloak 在纯 HTTP 和 Docker 网络中正常工作，我们需要打破它的默认规则。以下是一个用于 POC 的 `docker-compose.yml` 配置范本：

```yaml
services:
  keycloak:
    build:
      context: .
      dockerfile: ./infra/keycloak/Dockerfile
    command: ['start-dev', '--import-realm']
    image: dip-keycloak:latest # 我们稍后构建的自定义镜像
    container_name: dip-keycloak
    command: ["start-dev", "--import-realm"] # start-dev 模式使用内置 H2 数据库，零依赖
    environment:
      # 1. 基础管理员账号（仅用于首次初始化，推荐通过 Realm 文件配置）
      KC_BOOTSTRAP_ADMIN_USERNAME: admin
      KC_BOOTSTRAP_ADMIN_PASSWORD: admin

      # 2. 开启 HTTP 支持
      KC_HTTP_ENABLED: "true"
      KC_HTTP_PORT: "8083" # 避开常用的 8080

      # 3. 【核心避坑点】明确指定 Hostname
      # 如果留空，Keycloak 可能会生成基于 localhost 的 Issuer，导致后端服务校验失败
      KC_HOSTNAME: "http://192.168.2.35:8083"

      # 4. 放宽安全限制
      KC_HOSTNAME_STRICT: "false"
      KC_HOSTNAME_STRICT_HTTPS: "false"
      KC_PROXY: "none" # 纯 IP 直连模式，不要设为 edge

    ports:
      - 8083:8083
    networks:
      - dip-network

  auth-bff:
    # 你的后端服务
    image: my-service:latest
    environment:
      # Issuer 必须与 Keycloak 的 KC_HOSTNAME 严格一致
      OIDC_ISSUER: "http://192.168.2.35:8083/realms/dip"
    extra_hosts:
      # 5. 【网络避坑点】容器访问宿主机 IP
      # 解决容器内无法解析或路由到宿主机 IP 的问题
      - "192.168.2.35:host-gateway"
    networks:
      - dip-network

networks:
  dip-network:
    driver: bridge
```

## 关键配置解读：

1.  **`KC_HOSTNAME`**：这是最关键的一行。OIDC 协议要求 Token 中的 `iss` (Issuer) 字段必须与客户端请求的地址一致。如果这里不写死为 IP，Keycloak 可能会签发 `localhost` 的 Token，导致你的服务报错 `Issuer Mismatch`。
2.  **`start-dev`**：对于 POC 或临时演示，使用 H2 内存/文件数据库足够，无需额外部署 PostgreSQL。
3.  **`extra_hosts`**：在 Docker 容器内部，直接访问宿主机 IP 往往会被防火墙拦截。通过 `host-gateway` 映射，让容器流量走 Docker 网关，确保网络互通。

# 2. 初始化工作流：从“手动点击”到“自动化镜像”

在私有化交付中，我们不能指望现场实施人员去 Keycloak 控制台手动创建 Realm 和 Client。我们需要实现 **Configuration as Code (配置即代码)** 。

## 1. 准备阶段：手动配置与导出

首先，启动官方 Keycloak 镜像，在浏览器中手动完成以下配置：

- 创建一个新的 Realm（例如 `dip`）。
- 创建 Client（设置 `Access Type: confidential`）。
- **技巧**：在 `Valid Redirect URIs` 中配置 `*`（通配符），因为私有化部署的 IP 是未知的，这样可以适配任何客户环境。
- **导出配置**：进入 `Realm Settings` -> `Action` -> `Partial Export`，导出为 `dip-realm.json`。

## 2. 构建阶段：自定义镜像

编写 `Dockerfile`，利用 Keycloak 的 `--import-realm` 机制，将配置固化到镜像中：

```dockerfile
FROM quay.io/keycloak/keycloak:26.5.4

# 将导出的业务 Realm 配置复制到 import 目录
# Keycloak 启动时会自动扫描该目录并导入配置
COPY ./infra/keycloak/realm/dip-realm.json /opt/keycloak/data/import/dip-realm.json

# 建议：在这里也可以放入 master-realm.json（见下文详解）
```

这样，客户拿到的镜像启动后，就已经有了配置好的 Realm 和 Client，真正的**开箱即用**。

# 3. 常见问题修复：解决“HTTPS Required”死锁

这是内网部署最大的痛点：当你通过 `http://192.168.x.x:8083` 访问 Keycloak **管理控制台**时，界面会报错 **"HTTPS Required"**，导致无法登录管理员账号。

这是因为 Keycloak 的 **Master Realm** 默认安全策略是 `sslRequired=external`（非 Localhost 必须 HTTPS）。

## 方法 A：热修复（适用于调试）

如果容器已经跑起来了，可以通过命令行工具 `kcadm.sh` 实时修改配置：

```bash
# 1. 进入容器
docker exec -it dip-keycloak bash

# 2. 登录内置工具 (操作本地服务)
/opt/keycloak/bin/kcadm.sh config credentials \
  --server http://localhost:8083 --realm master --user admin --password admin

# 3. 强制关闭 Master Realm 的 SSL 限制
/opt/keycloak/bin/kcadm.sh update realms/master -s sslRequired=NONE
```

_刷新浏览器，你会发现控制台可以登录了。_

## 方法 B：永久修复（推荐方案）

为了防止重启容器后配置丢失，或者为了交付给客户，我们需要使用 **Partial Import（增量导入）** 机制来永久覆盖这个设置。

1.  创建一个精简版的 `master-realm.json`：

    ```json
    {
      "realm": "master",
      "enabled": true,
      "sslRequired": "none"
    }
    ```

    _注意：不要包含 `users` 字段，以免覆盖管理员账号；不要包含 `id` 字段，以免冲突。_

2.  更新 `Dockerfile`，将此文件也复制进去：
    ```dockerfile
    COPY ./infra/keycloak/realm/master-realm.json /opt/keycloak/data/import/master-realm.json
    ```

Keycloak 启动时会检测到 `master` 域的导入文件，并自动将 SSL 策略更新为 `none`。

# 4. 客户端集成的小贴士

在解决了 Keycloak 端的问题后，你的应用（Client）可能会遇到 **“登录后无限重定向”** 的问题。

**原因**：现代浏览器对 Cookie 的安全限制。
如果你的应用在设置 Session Cookie 时标记了 `Secure`（仅限 HTTPS），但你在内网使用的是 HTTP，浏览器会**拒绝保存 Cookie**。
结果就是：Keycloak 认证成功 -> 跳回应用 -> 应用发现没 Cookie (未登录) -> 跳回 Keycloak -> Keycloak 发现已登录 -> 跳回应用 -> **死循环**。

**解决方案**：
在你的代码中判断 `FRONTEND_URL`：

- 如果是 `https://` 开头：设置 `Secure=True`, `SameSite=Lax`。
- 如果是 `http://` 开头：**强制设置 `Secure=False`**, `SameSite=Lax`。

# 5. 总结

要在私有化、纯 HTTP 的 Docker 环境中驯服 Keycloak，核心在于：

1.  **网络层**：使用 `KC_HOSTNAME` 绑定 IP，使用 `extra_hosts` 打通容器内外。
2.  **配置层**：使用 `import` 机制固化 Realm 配置，实现自动化交付。
3.  **安全层**：通过导入精简版 `master-realm.json` 彻底解除 HTTPS 限制。

掌握了这一套打法，无论是 POC 演示还是内网交付，你都能在 5 分钟内拉起一套标准的 OIDC 认证中心。
