---
title: NanoMQ
date: 2025-01-09 18:36:05
tags:
  - DevOps
excerpt: 学习和应用轻量高性能 MQTT Broker -- NanoMQ
---

# 1. [NanoMQ](https://nanomq.io/)

[NanoMQ](https://nanomq.io/) 是一个用 C 语言写的轻量级 MQTT Broker, 从产品上看似乎属于 EMQX 公司, EMQX 公司有开源版本的另一款产品-- [emqx](https://github.com/emqx/emqx), 与 NanoMQ 不同的是 emqx 是用 erlang 开发的.

NanoMQ MQTT 代理 （NanoMQ） 是一个全方位的边缘消息传递平台，包括用于 IoT/IIoT 的超快 MQTT 代理和用于 SDV 的轻量级消息传递总线。 NanoMQ 的嵌入式 Actor 架构扩展了 NNG 的内部异步 I/O，以及增强的消息传递和调度系统，以最大限度地提高整体容量。针对嵌入式环境和任务关键型场景进行微调。 NanoMQ 完全支持 MQTT V3.1.1/3.1 和 MQTT V5.0。 其主要特性如下:

- [功能特性](https://nanomq.io/docs/zh/latest/#功能特性)

  **完整支持 MQTT 5.0**：完整支持 MQTT 5.0/3.1.1，与所有标准 MQTT 开源 SDK 兼容。

  [**MQTT 桥接**](https://nanomq.io/docs/zh/latest/bridges/introduction.html)：内置 MQTT 多云桥接功能，简单配置即可接入各类流行的云服务建立云边通道进行多地数据同步。

  [**规则引擎**](https://nanomq.io/docs/zh/latest/rule/introduction.html)：内置简单规则引擎可以灵活处理边缘数据。也可以与 eKuiper 集成在边缘进行流式数据分析。

  [**消息持久化**](https://nanomq.io/docs/zh/latest/rule/config-rule-engine.html#sqlite-规则)：针对物联网复杂网络环境场景设计数据持久化和数据缓存功能，桥接消息可以本地缓存后断网续传。重要数据可以在边缘持久化滚动更新。

  [**丰富的集成 API**](https://nanomq.io/docs/zh/latest/api/introduction.html)：提供可拓展的事件驱动型 WebHook 接口和运维友好的 HTTP APIs，帮助减少边缘计算应用开发和集成的成本。

  [**多协议网关**](https://nanomq.io/docs/zh/latest/gateway/introduction.html)：通过协议网关支持 nanomsg/ZeroMQ/Websocket 等常用的消息协议，方便在边缘建立灵活的数据路由拓扑。并提供连接加密和安全保障。

  [**MQTT STREAM**](https://nanomq.io/docs/zh/latest/mqtt-stream/introduction.html): 对于同一 topic 的 MQTT 消息，可以看做一条数据流，并且这个数据流是可以进行落盘存储以及查询操作的，对于一些网络较差的环境下，为数据的完整性和可靠性提供了解决方案.

- [核心优势](https://nanomq.io/docs/zh/latest/#核心优势)

  **超轻量**：MQTT 服务可以低至 200Kb 的内存占用启动。

  **全异步 I/O**：针对物联网和 MQTT 内建 Actor 编程模型并行化计算负载。

  **多线程并行**：优秀的可拓展性，具有良好的多核 SMP（对称多处理）支持和多线程性能。

  **高性能**：边缘百万级消息处理能力。

  **跨平台**：可无缝移植到任何基于 POSIX 的系统平台。

  **互操作性**：数据透明，拓展接口丰富，易于和各类边缘计算框架集成。

# 2. 编译和启动

NanoMQ 有三个 [Docker Image](https://nanomq.io/docs/zh/latest/installation/docker.html), 其特性也是由少到多:

- NanoMQ 基础版（默认）
- NanoMQ Slim 版
- NanoMQ 完整版

但是本文不是用 Docker Image 的方式, 而是使用源码编译的方式, 因为很多细节的特性需要编译才能实现, 例如 AWS IoT Core 桥接特性. 另外经过验证源码在 macOS M2 的机器上编译不过, 所以本文以下的操作都是在 Ubuntu x86 架构的机器上进行的.

- 前置依赖

  ```bash
  brew install git-lfs
  brew install cmake
  ```

- 下载源码

  ```bash
  git clone https://github.com/nanomq/nanomq.git
  cd nanomq/
  git submodule update --init --recursive
  ```

- [编译](https://nanomq.io/docs/zh/latest/installation/build-options.html)

  ```bash
  mkdir build && cd build
  cmake -DENABLE_JWT=ON -DNNG_ENABLE_TLS=ON DNNG_ENABLE_SQLITE=ON ..
  make
  ```

  > [!NOTE]
  >
  > - -DENABLE_JWT=ON : 编译启用 HTTP Server 所需的 JWT 依赖项
  > - -DNNG_ENABLE_TLS=ON: 编译启用 TLS，依赖项：[mbedTLS](https://tls.mbed.org/)
  > - -DNNG_ENABLE_SQLITE=ON: 支持 SQLite

- 验证结果

  ```bash
  cd ..
  ./build/nanomq/nanomq start --conf etc/nanomq.conf
  NanoMQ Broker is started successfully!
  ```

  根据 nanomq.conf 的配置, 会监听 1883 端口实现 MQTT over TCP, 所以直接连接服务器这个端口发布消息即可, 验证工具可以使用[MQTTX](https://mqttx.app/zh/downloads).

# 3. 功能验证

## 3.1 认证

保持项目仓库现有的 etc/nanomq_pwd.conf 不变, 使用其默认配置的用户名和密码:

```toml
# #====================================================================
# # Configuration for Password
# #====================================================================

# # Write "username":"password" in this way.
"admin": "public"
"client": "public"
```

## 3.2 ACL

编辑现有的 etc/nanomq_acl.conf 文件配置认证信息, 以下配置是使用名单模式, 默认拒绝所有, 仅配置的用户和操作才被允许执行:

```toml
# #====================================================================
# # Configuration for Acl
# #====================================================================

rules = [
	# # Allow MQTT client using username "dashboard"  to subscribe to "$SYS/#" topics
	{"permit": "allow", "username": "admin", "action": "pubsub", "topics": ["#"]}
	{"permit": "allow", "username": "client", "action": "pubsub", "topics": ["client/only-one"]}

	# # Deny "All Users" subscribe to "$SYS/#" "#" Topics
	# {"permit": "deny", "username": "#", "action": "subscribe", "topics": ["$SYS/#", "#"]}

	# # Allow any other publish/subscribe operation
	{"permit": "deny"}
]
```

## 3.3 NanoMQ 配置

- 配置 TCP 来支持 MQTT
- 配置 TSL 来支持 MQTTS 和 证书认证
- 配置 Auth 来支持访问认证和鉴权
- 配置 Webhook 支持消息转发为 HTTP

```toml
# NanoMQ Configuration 0.18.0

# #============================================================
# # NanoMQ Broker
# #============================================================

mqtt {
    property_size = 32
    max_packet_size = 260MB
    max_mqueue_len = 2048
    retry_interval = 10s
    keepalive_multiplier = 1.25

    # Three of below, unsupported now
    max_inflight_window = 2048
    max_awaiting_rel = 10s
    await_rel_timeout = 10s
}

listeners.tcp {
    bind = "0.0.0.0:1883"
}

listeners.ssl {
	bind = "0.0.0.0:8883"
	keyfile = "./etc/certs/key.pem"
	certfile = "./etc/certs/cert.pem"
	cacertfile = "./etc/certs/cacert.pem"
	verify_peer = false
	fail_if_no_peer_cert = false
}

listeners.ws {
    bind = "0.0.0.0:8083/mqtt"
}

# http_server {
#     port = 8081
#     limit_conn = 2
#     username = admin
#     password = public
#     auth_type = basic
#     jwt {
#         public.keyfile = "./etc/certs/jwt/jwtRS256.key.pub"
#     }
# }

webhook = {
  url = "http://192.168.110.200:3000/mqtt"        # URL where webhooks will send HTTP requests to
  headers.content-type = "application/json" # HTTP header specifying request content type
  headers.authorization = "eyJhbGciOiJIUzI1NiJ9.eyJlbnZOYW1lIjoibG9jYWwifQ.oWJU9JhpdASfv_9XDHm1wZ2w7IA86N2YAr3aX8NUNEY"
  body.encoding = "plain"            # Encoding format of the payload field in HTTP body
  pool_size = 32                     # Connection process pool size
  events = [
    {
      event = "on_message_publish"   # Event type
      topic = "#"                # The specific topic that this event applies to
    }
  ]
}

log {
    to = [file, console]
    level = warn
    dir = "/tmp"
    file = "nanomq.log"
    rotation {
        size = 10MB
        count = 5
    }
}

auth {
    allow_anonymous = false
    no_match = allow
    deny_action = ignore

    cache = {
        max_size = 32
        ttl = 1m
    }

    # password = {include "./etc/nanomq_pwd.conf"}
    # acl = {include "./etc/nanomq_acl.conf"}
    http_auth = {
        auth_req {
            url = "http://192.168.110.200:3000/mqtt/auth"
            method = "POST"
            headers.content-type = "application/x-www-form-urlencoded"
            headers.authorization = "eyJhbGciOiJIUzI1NiJ9.eyJlbnZOYW1lIjoibG9jYWwifQ.oWJU9JhpdASfv_9XDHm1wZ2w7IA86N2YAr3aX8NUNEY"
            params = {clientid = "%c", username = "%u", password = "%P"}
        }

        super_req {
            url = "http://192.168.110.200:3000/mqtt/superuser"
            method = "POST"
            headers.content-type = "application/x-www-form-urlencoded"
            headers.authorization = "eyJhbGciOiJIUzI1NiJ9.eyJlbnZOYW1lIjoibG9jYWwifQ.oWJU9JhpdASfv_9XDHm1wZ2w7IA86N2YAr3aX8NUNEY"
            params = {clientid = "%c", username = "%u", password = "%P"}
        }

        acl_req {
            url = "http://192.168.110.200:3000/mqtt/acl"
            method = "POST"
            headers.content-type = "application/x-www-form-urlencoded"
            headers.authorization = "eyJhbGciOiJIUzI1NiJ9.eyJlbnZOYW1lIjoibG9jYWwifQ.oWJU9JhpdASfv_9XDHm1wZ2w7IA86N2YAr3aX8NUNEY"
            params = {clientid = "%c", username = "%u", access = "%A", ipaddr = "%a", topic = "%t", mountpoint = "%m"}
        }

        timeout = 5s
        connect_timeout = 5s
        pool_size = 32
  }
}
```

> [!NOTE]
> 虽然 Webhook 的文档中没有提及可以增加多个 header, 但是从数据结构上确实可以加, 这里可以增加一个 JWT Token, 来增加一些客户端访问认证的安全检查.

## 3.4 验证

首先启动程序, 在项目根目录运行如下命令:

```bash
./build/nanomq/nanomq start --conf etc/nanomq.conf
```

在 webhook-server 上启动写好的程序, 这里使用 typescript 编写 HTTP 服务如下:

```typescript
import express from "express";
import bodyParser from "body-parser";
import asyncHandler from "express-async-handler";
import * as jose from "jose";

const app = express();
const port = 3000;

app.use(bodyParser.json());
app.use(bodyParser.urlencoded({ extended: true }));

const users = [
  {
    username: "admin",
    password: "public",
    isSupper: true,
    // TIP: supper user can do enverything, so acl is no required
    acl: {
      "1": ["#"], // sub
      "2": ["#"], // pub
    },
  },
  {
    username: "client",
    password: "public",
    isSupper: false,
    // TIP: general user should have acl
    acl: {
      "1": ["client/only-one"], // sub
      "2": ["client/only-one"], // pub
    },
  },
];

app.use(
  asyncHandler(async (req, res, next) => {
    const auth = req.headers.authorization || "";
    await jose.jwtVerify(auth, Buffer.from("nanoqm-webhook-local"));
    next();
  })
);

app
  // Health check
  .get("/health", (req, res) => {
    console.log("health check");
    res.json({ now: Date.now() });
  })
  // Webhook receiver
  .post(
    "/mqtt",
    asyncHandler(async (req, res) => {
      console.log("/mqtt");
      console.log(req.body);
    })
  )
  // Authorization
  .post(
    "/mqtt/auth",
    asyncHandler(async (req, res) => {
      console.log("/mqtt/auth");
      const payload = req.body as {
        clientid: string;
        username: string;
        password: string;
      };
      console.log("auth paylaod:", payload);
      const user = users.find(
        (user) =>
          user.username === payload.username &&
          user.password === payload.password
      );
      console.log("auth find user:", !!user);
      if (!user) {
        res.status(401).send();
        return;
      }
      res.status(200).send();
    })
  )
  // Super user authorization
  .post(
    "/mqtt/superuser",
    asyncHandler(async (req, res) => {
      console.log("/mqtt/superuser");
      const payload = req.body as {
        clientid: string;
        username: string;
        password: string;
      };
      console.log("superuser paylaod:", payload);
      const user = users.find(
        (user) =>
          user.username === payload.username &&
          user.password === payload.password &&
          user.isSupper
      );
      console.log("superuser find user:", !!user);
      if (!user) {
        res.status(401).send();
        return;
      }
      res.status(200).send();
    })
  )
  // ACL
  .post(
    "/mqtt/acl",
    asyncHandler(async (req, res) => {
      console.log("/mqtt/acl");
      const payload = req.body as {
        clientid: string;
        username: string;
        access: string;
        topic: string;
      };
      console.log(payload);
      const user = users.find((user) => user.username === payload.username)!;
      const allowedTopics = user.acl[+payload.access as 1 | 2] || [];
      let invalidTopic = true;

      for (const topic of allowedTopics) {
        const topicRegExp = topic.replace("#", ".*");
        if (payload.topic.match(new RegExp(`^${topicRegExp}$`))) {
          invalidTopic = false;
        }
      }
      console.log("invalid: ", invalidTopic);

      if (invalidTopic) {
        res.status(403).send();
        return;
      }
      res.status(200).send();
    })
  );

app.listen(port, () => {
  console.log(`Http server app listening on port ${port}`);
});
```

经过测试和验证, 测试结果如下:

- [MQTT over TCP](https://nanomq.io/docs/zh/latest/config-description/listener.html) :white_check_mark:
- [MQTT over TLS/SSL](https://nanomq.io/docs/zh/latest/config-description/listener.html) :white_check_mark:
- [用户名密码认](https://nanomq.io/docs/zh/latest/config-description/acl.html):white_check_mark:
- [ACL 鉴权](https://nanomq.io/docs/zh/latest/config-description/acl.html):white_check_mark:
- [Webhook](https://nanomq.io/docs/zh/latest/config-description/webhook.html)
  - MQTT 转 HTTP :white_check_mark:
  - 配置 HTTP `authorization` 头 :white_check_mark:
- [HTTP 认证和鉴权](https://nanomq.io/docs/zh/latest/access-control/http.html) :white_check_mark:

> [!NOTE]
> HTTP 身份验证完整配置参考[这里](https://nanomq.io/docs/zh/latest/config-description/v014.html#http%E8%BA%AB%E4%BB%BD%E9%AA%8C%E8%AF%81), 虽然文档上说明 acl_req 的 params 没有 %P(密码), 但是经过测试 %P 可以配置, 如此就就可以在 pub/sub 的时候进行认证逻辑的开发.

# 4. 总结

NanoMQ 功能丰富, 支持最基本的 TPC 链接, 也支持 TLS/SSL, 走 TLS/SSL 的情况下, 支持仅 CA 单向认证, 也支持设备证书双向认证; 其本身的配置可以通过自身定义和提供的 HTTP API 进行操作, 且支持热更新; 更惊艳的功能是 HTTP Auth, 可以自己动态实现认证和鉴权.

推荐在开发验证阶段使用 NanoMQ, 其单机方式性能也很不错, 详情可见其[测试报告](https://nanomq.io/docs/zh/latest/test-report.html); 如果在生产环境, NanoMQ 是 MIT License, 可以进行二次开发, 但是自己需要搭建高可用的集群以提供稳定的服务.
