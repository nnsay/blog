---
title: EMQX 开源版本的集群搭建和配置指南
date: 2025-02-07 17:07:04
tags:
  - DevOps
  - 研究
excerpt: 学习和应用开源版本EMQX
---

# 1. [EMQX 开源版本](https://docs.emqx.com/zh/emqx/latest/)

从 v5.7 开始 EMQX 的文档就不分别为企业版和开源版单独提供文档地址了, 而在文档内种中通过提示和菜单加以区分, 最新的文档地是: [EMQX 文档](https://docs.emqx.com/zh/emqx/latest). 源码地址: https://github.com/emqx/emqx

本文将探究以下功能:

- 管理员 Dashboard
- 集成
  - Webhook
  - MQTT/HTTP 连接器
- 访问控制
  - 客户端认证
  - 客户端授权
- MQTT over TCP

> [!NOTE]
> 本文主要是研究 EQMX 开源版本, 试图自己搭建 Borker 服务, 之前的博客[EMQX 应用实践](https://nnsay.cn/2024/10/29/aws-iot-mqtt-alternative)则主要介绍其企业版本中的 Serverless 版本服务的使用; 开源版和企业版在 UI 和操作上很相似, 本文会适当滤过类似操作, 但是会提供详细操作的文档地址.

# 2. 集群搭建

## 2.1 [下载安装](https://www.emqx.com/zh/downloads-and-install/broker)

虽然官方文档提供下载安装方式, 但是经过测试 macos14-arm64 在本地 macOS M2 Pro 环境跑步起来, 报错详情如下:

```bash
➜  ~ export DYLD_LIBRARY_PATH=/opt/homebrew/Cellar/openssl@3/3.4.0/lib/:$DYLD_LIBRARY_PATH
➜  ~ emqx foreground                                                                      
WARNING: ulimit -n is 256; 1024 is the recommended minimum.
WARNING: Default (insecure) Erlang cookie is in use.
WARNING: Configure node.cookie in /Users/wangjian/.local/share/emqx/etc/emqx.conf or override from environment variable EMQX_NODE__COOKIE
WARNING: NOTE: Use the same cookie for all nodes in the cluster.
Kernel pid terminated (application_controller) ("{application_start_failure,kernel,{{shutdown,{failed_to_start_child,on_load,{on_load_function_failed,quicer_nif,{error,{load_failed,\"Failed to load NIF library: 'dlopen(/Users/wangjian/.local/share/emqx/lib/quicer-0.1.10/priv/libquicer_nif.so, 0x0002): Library not loaded: /opt/homebrew/opt/openssl@3/lib/libcrypto.3.dylib\n  Referenced from: <FA249546-AFC9-3252-A8E2-8527D09738FA> /Users/wangjian/.local/share/emqx/lib/quicer-0.1.10/priv/libquicer_nif.so\n  Reason: tried: '/opt/homebrew/opt/openssl@3/lib/libcrypto.3.dylib' (code signature in <0EF13B00-8944-3A1E-AC18-21753DE7EEDF> '/opt/homebrew/Cellar/openssl@3/3.4.0/lib/libcrypto.3.dylib' not valid for use in process: mapping process and mapped file (non-platform) have different Team IDs), '/System/Volumes/Preboot/Cryptexes/OS/opt/homebrew/opt/openssl@3/lib/libcrypto.3.dylib' (no such file), '/opt/homebrew/opt/openssl@3/lib/libcrypto.3.dylib' (code signature in <0EF13B00-8944-3A1E-AC18-21753DE7EEDF> '/opt/homeb


Crash dump is being written to: /Users/wangjian/.local/share/emqx/log/erl_crash.dump...done
```

以上问题尝试多种方式未能解决, 所以建议使用容器化方式安装部署.

## 2.2 [Docker 部署](https://docs.emqx.com/zh/emqx/latest/deploy/install-docker-ce.html)

使用 docker compose 本地创建一个小集群, 创建`docker-compose.emqx.yml`文件并添加如下内容:

```yaml
version: "3"

services:
  ### emqx (https://hub.docker.com/r/emqx/emqx)
  emqx1:
    image: emqx/emqx:latest
    container_name: emqx1
    environment:
      - "EMQX_NAME=emqx"
      - "EMQX_HOST=node1.emqx.io"
      - "EMQX_CLUSTER__DISCOVERY_STRATEGY=static"
      - "EMQX_CLUSTER__STATIC__SEEDS=[emqx@node1.emqx.io, emqx@node2.emqx.io]"
    ports:
      - "1883:1883" # 暴露 MQTT 的端口
      - "18083:18083" # 暴露管理面板端口（可选）
    networks:
      emqx-bridge:
        aliases:
          - node1.emqx.io

  emqx2:
    image: emqx/emqx:latest
    container_name: emqx2
    environment:
      - "EMQX_NAME=emqx"
      - "EMQX_HOST=node2.emqx.io"
      - "EMQX_CLUSTER__DISCOVERY_STRATEGY=static"
      - "EMQX_CLUSTER__STATIC__SEEDS=[emqx@node1.emqx.io, emqx@node2.emqx.io]"
    ports:
      - "2883:1883"
      - "28083:18083"
    networks:
      emqx-bridge:
        aliases:
          - node2.emqx.io

networks:
  emqx-bridge:
    driver: bridge
```

使用上面的 docker compose 配置文件部署完毕后可以进行一些常规的检查:

- 查看集群状态

  ```bash
  docker exec -it emqx1 sh -c "emqx ctl cluster status"
  ```

  集群状态输出结果如下:

  ```log
  Cluster status: #{running_nodes => ['emqx@node1.emqx.io','emqx@node2.emqx.io'],
                  stopped_nodes => []}
  ```

- 部署服务

  ```bash
  docker compose -f docker-compose.emqx.yml up -d
  ```

- 查看日志

  ```bash
  docker compose -f docker-compose.emqx.yml logs -f
  ```

- 访问 EMQX Dashboard

  在浏览器中访问 dashboard 地址: http://localhost:18083. 使用默认的管理员账登录, 用户名: `admin`, 密码: `public`

  ![](https://img.picgo.net/2025/02/07/26b1073e05f1b252d5e84158fc122dba376873d947b1db98.png)

# 3. 测试和验证

在 2.2 中在两个实例的集群中我们仅让一个服务暴露了 TCP 端口, 在本地通过 MQTT over TCP 链接测试即可.

## 3.1 [访问控制](https://docs.emqx.com/zh/emqx/latest/dashboard/acloverview.html)

- [客户端认证](https://docs.emqx.com/zh/emqx/latest/dashboard/authn.html)

  在 Dashboard 中选择`访问控制`下的`客户端认证`, 新建`内置数据库`数据源作为认证方式, 其支持两种账号类型的管理: username 和 clientid, 为了和 EQMX Serverless 版本一样, 这里选择 username.

  > [!NOTE]
  >
  > 每个类型的数据库源只能被添加一次

- [客户端授权](https://docs.emqx.com/zh/emqx/latest/dashboard/authz.html)

  在 Dashboard 中选择`访问控制`下的`客户端授权`, 新建`内置数据库`数据源作为授权方式, 内置数据库授权方式支持针对 username 和 clientid

  > [!NOTE]
  >
  > 默认客户端授权中有一个`File`类型数据源, 在其中可以通过特殊语法编辑客户端的授权规则, 还有一些高级功能例如条件组合和正则表达式

测试过程比较简单且和 EMQ 的云产品操作类似, 即在客户端认证中创建若干用户, 在客户端授权中创建针对 username 和 clientid 的授权规则, 详细说明参考文档: [客户端认证](https://docs.emqx.com/zh/emqx/latest/dashboard/authn.html)和[客户端授权](https://docs.emqx.com/zh/emqx/latest/dashboard/authz.html), 最后使用 MQTT GUI 客户端进行测试即可.

> [!NOTE]
> MQTT GUI 客户端下载地址: https://mqttx.app/zh/downloads

## 3.2 [HTTP 转发](https://docs.emqx.com/zh/emqx/latest/data-integration/data-bridge-webhook.html)

HTTP 转发也和 EMQ 的云产品类似, 是通过`HTTP 连接器`实现的, 该操作也比较简单, 新建一个`HTTP服务`, 然后按照引导创建`规则`和`动作`, 目的是把所有 Broker 接受到的消息通过 HTTP 转发到第三方, 详细说明参考文档: [HTTP Server](https://docs.emqx.com/zh/emqx/latest/data-integration/data-bridge-webhook.html)

> [!NOTE]
>
> - HTTP 转发需要提前开发一个接收服务, 可以参考之前博客(EMQX 应用实践)中的[3.2.3 HTTP 服务实现](https://nnsay.cn/2024/10/29/aws-iot-mqtt-alternative/#3-2-4-HTTP-%E6%9C%8D%E5%8A%A1%E5%AE%9E%E7%8E%B0)
> - 如果 HTTP 服务没有跑在和 EQMX 一样网络环境中而是在本地, 则需要注意 Host 地址是: ==host.docker.internal==

# 4. 高级话题

## 4.1 [自动集群](https://docs.emqx.com/zh/emqx/latest/deploy/cluster/create-cluster.html#%E8%87%AA%E5%8A%A8%E9%9B%86%E7%BE%A4)

在 2.2 中使用的是集群发现策略中的静态模式, 这种模式主要是服务自动组成集群, 而自动集群组建的方式有以下几种:

- [基于 static 节点列表自动集群](https://docs.emqx.com/zh/emqx/latest/deploy/cluster/create-cluster.html#%E5%9F%BA%E4%BA%8E-static-%E8%8A%82%E7%82%B9%E5%88%97%E8%A1%A8%E8%87%AA%E5%8A%A8%E9%9B%86%E7%BE%A4)

- [基于 DNS 自动集群](https://docs.emqx.com/zh/emqx/latest/deploy/cluster/create-cluster.html#%E5%9F%BA%E4%BA%8E-dns-%E8%87%AA%E5%8A%A8%E9%9B%86%E7%BE%A4)

- [基于 etcd 自动集群](https://docs.emqx.com/zh/emqx/latest/deploy/cluster/create-cluster.html#%E5%9F%BA%E4%BA%8E-etcd-%E8%87%AA%E5%8A%A8%E9%9B%86%E7%BE%A4)

- [基于 kubernetes 自动集群](https://docs.emqx.com/zh/emqx/latest/deploy/cluster/create-cluster.html#%E5%9F%BA%E4%BA%8E-kubernetes-%E8%87%AA%E5%8A%A8%E9%9B%86%E7%BE%A4)

## 4.2 [集群负载均衡](https://docs.emqx.com/zh/emqx/latest/deploy/cluster/lb.html)

根据[官方文档](https://docs.emqx.com/zh/emqx/latest/deploy/cluster/mria-introduction.html)描述:

- EMQX 5.0 采用 [Mria](https://github.com/emqx/mria) 架构
- 只有集群规模超过 3 个节点，才建议使用 Core + Replicant 复制模式
- 仅 EMQX 企业版支持 Core + Replicant 节点集群。开源版只支持 Core 节点集群

综上可知如果是开源版本我们应该选择传动的负载均衡集群模式, 即和微服务多副本高可用方式一样, 部署多个服务实例, 实例之间自动集群组网, 为了访问高可用可以在多个实例之上添加负载均衡. 官方文档中的 [TLS 终结与负载均衡](https://docs.emqx.com/zh/emqx/latest/deploy/cluster/lb.html#tls-%E7%BB%88%E7%BB%93%E4%B8%8E%E8%B4%9F%E8%BD%BD%E5%9D%87%E8%A1%A1) 是一个简单且有效的集群搭建方式.

## 4.3 [管理员管理](https://docs.emqx.com/zh/emqx/latest/admin/admin-guide.html)

管理员管理有很多管理项目这里不一一介绍, 仅重点介绍一下通过[命令行](https://docs.emqx.com/zh/emqx/latest/admin/cli.html)管理集群:

- 登录容器(或者部署服务的主机)

  ```bash
  docker exec -it emqx1 bash
  ```

- 使用命令管理

  ```bash
  emqx --help

  # 重置默认管理员密码
  emqx ctl admins passwd admin admin@pwd
  ```

## 4.3 [密码认证](https://docs.emqx.com/zh/emqx/latest/access-control/authn/pwoverview.html)

EMQX 支持通过密码进行身份验证，这种最简单，也是使用最多的认证方式。此时，客户端需要提供能够表明身份的凭据，例如用户名、客户端 ID 以及对应的密码并将其存储在特定数据源（数据库）中。除简单便捷的内置数据库外，EMQX 还支持通过与多类后端数据库的集成提供密码认证，包括 MySQL、PostgreSQL、MongoDB 和 Redis。

EMQX 的内置数据库是: [Mnesia](https://elixirschool.com/zh-hans/lessons/storage/mnesia). Mnesia 是 Erlang 运行时中自带的一个数据库管理系统（DBMS）, Mnesia 的数据库模型可以混合了关系型和对象型数据模型的特征，让它可以用来开发任何规模的分布式应用程序。

> [!NOTE]
> 如果使用内置数据库, 则该数据库也将随着 EMQX 集群形成一个数据库集群, 数据会自动复制到多个节点保证高可用.

## 4.4 [REST API](https://docs.emqx.com/zh/emqx/v5.8/admin/api.html)

EMQX 提供了管理监控 REST API，这些 API 遵循 OpenAPI (Swagger) 3.0 规范。EMQX 服务启动后，您可以访问 http://localhost:18083/api-docs/index.html 来查看 API 的文档。

在使用 REST API 前在 Dashboard `系统设置`中添加`API密码`, API 密码是一组 API Key + Secret Key, 可以作为调用 API 时的`basicAuth`来认证.

# 5. 总结

EMQX 开源版本可以对标其企业版本中 Serverless 版本的服务, Serverless 版本服务不支持 MQTT over TCP, 所以如果使用 EMQX 开源版本搭建自己的服务则可以解决这个问题更好的让 IoT 设备简单的接入数据; EMQX 开源版本和其 NanoMQ 相比有一些优势, 例如前者提供了 Dashboard 管理端, 大大方便了配置和操作; 在集群搭建方面仅 EMQX 企业版支持 Core + Replicant 节点集群, 所以 EQMX 开源版和 NanoMQ 最终的集群方式差不多, 都是部署多个服务实例通过负载均衡实现高可用, 但是前者每个服务实例还可以组成静态集群数据可互通. 总结: 如果你业务量规模不大, 但实例 NanoMQ 就可以支持, 如果还想要增加一些高可用性, 那么 EMQX 开源版是个不错的选择.
