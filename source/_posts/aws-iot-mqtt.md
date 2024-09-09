---
title: AWS IoT MQTT通信
date: 2024-09-09 17:26:57
tags:
  - Code
  - 研究
excerpt: 使用AWS IoT服务进行MQTT通信
---

# 1. 需求描述

**用户故事**: 一个发电站中有很多设备, 一些设备可以发送数据到 SAAS 报告一些数据比如发电量, 而 SAAS 会进行数据分析进而决定电站应该如何运行, 所以会下发一些指令给电站设备, 简而言之: 电站设备和 SAAS 有数据来往.

**需求分析**: 数据的上行或者下行走不走 MQTT 意味着是否要和客户的设备直连, 针对这两种方案的考虑如下:

- 不走 MQTT 理由
  - 安全性, 客户可能直接控制他们的设备, 或者客户的设备无法公网访问
  - 兼容性, 下行命令可能因为设备厂商和型号有不同
  - 适应性, 客户的设备可能不止一台, 或者不固定有伸缩的可能
  - 实施性, 客户的设备必须二次开发以适应 AWS IoT MQTT5 的标准和安全, 培训门槛较高(语言/工具)
- 走 MQTT 理由
  - 响应快, 直接设备没有中间系统
  - 标准, MQTT 是 IoT 设备信息传输标准

**结论**: 抛开实际的具体决策, 以技术准备的视角看需要调研如何基于 MQTT 数据实现上下行, 以及如何在目前的基础设施(AWS)上实现.

基于上述用户故事, 本文讲涉及以下方面的调研或者实践:

- 如何基于 AWS 实现 IoT 通信
  - AWS IoT Core
  - MQTT
- 验证 AWS IoT Core MQTT 可行性
- 代码实现数据上下行
- 常见问题探究, 例如如何实现用户名密码认证/如何自定义域名......

# 2. 基础知识

本章节的内容主要是介绍 AWS IoT Core MQTT 通信相关的知识, 其中全部内容均来自官方文档, 只是挑选了重点内容并重新简单组织了一下知识体系, 需要深入了解的同学可以查看官方文档: https://docs.amazonaws.cn/iot/index.html

## 2.1 [AWS IoT Core 通信协议](https://docs.aws.amazon.com/iot/latest/developerguide/protocols.html)

AWS IoT Core 支持使用 MQTT 的设备和客户端，并且 MQTT 用于发布和订阅消息和设备的 WebSocket 安全 （WSS） 协议 以及使用 HTTPS 协议发布消息的客户端。所有协议支持 IPv4 和 IPv6。

设备支持的通讯协议如下:

| Protocol            | Operations supported | Authentication           | Port | ALPN protocol name |
| :------------------ | :------------------- | :----------------------- | :--- | :----------------- |
| MQTT over WebSocket | Publish, Subscribe   | Signature Version 4      | 443  | N/A                |
| MQTT over WebSocket | Publish, Subscribe   | Custom authentication    | 443  | N/A                |
| MQTT                | Publish, Subscribe   | X.509 client certificate | 443† | `x-amzn-mqtt-ca`   |
| MQTT                | Publish, Subscribe   | X.509 client certificate | 8883 | N/A                |
| MQTT                | Publish, Subscribe   | Custom authentication    | 443† | `mqtt`             |
| HTTPS               | Publish only         | Signature Version 4      | 443  | N/A                |
| HTTPS               | Publish only         | X.509 client certificate | 443† | `x-amzn-http-ca`   |
| HTTPS               | Publish only         | X.509 client certificate | 8443 | N/A                |
| HTTPS               | Publish only         | Custom authentication    | 443  | N/A                |

以上协议对应的端点如下:

| Protocol      | Endpoint or URL               |
| :------------ | :---------------------------- |
| MQTT          | `iot-endpoint`                |
| MQTT over WSS | `wss://iot-endpoint/mqtt`     |
| HTTPS         | `https://iot-endpoint/topics` |

其中 HTTPS 不能保证连接的持续时间长于接收和响应请求所需的时间。

MQTT 连接持续时间取决于您使用的身份验证功能。下表列出了每个特征在理想条件下的最长连接持续时间。

| Feature                  | Maximum duration \* |
| :----------------------- | :------------------ |
| X.509 client certificate | 1–2 weeks           |
| Custom authentication    | 1–2 weeks           |
| Signature Version 4      | Up to 24 hours      |

## 2.2 [MQTT](https://docs.aws.amazon.com/iot/latest/developerguide/mqtt.html)

### 2.2.1 工具包

[MQTT 协议](http://mqtt.org/)（Message Queuing Telemetry Transport） 是一个 专为受限设备设计的轻量级且广泛采用的消息传送协议。 AWS IoT Core 对 MQTT 的支持基于[MQTT 版本 3.1.1 规范](http://docs.oasis-open.org/mqtt/mqtt/v3.1.1/os/mqtt-v3.1.1-os.html)和[MQTT 版本 5.0 规范](http://docs.oasis-open.org/mqtt/mqtt/v5.0/mqtt-v5.0.html)，但存在一些差异，如 [AWS IoT 与 MQTT 规范的差异](https://docs.aws.amazon.com/iot/latest/developerguide/mqtt.html#mqtt-differences)中所述

AWS IoT Core 还支持跨 MQTT 版本（MQTT 3 和 MQTT 5）通信。一个 MQTT 3 发布者可以将 MQTT 3 消息发送给将接收 MQTT 的 MQTT 5 订阅者 5 发布消息，反之亦然。

目前支持通过 MQTT 的 AWS IoT Device SDK 的编程语言有:

- [C++](https://github.com/aws/aws-iot-device-sdk-cpp-v2)
- [Python](https://github.com/aws/aws-iot-device-sdk-python-v2)
- [JavaScript](https://github.com/aws/aws-iot-device-sdk-js-v2)
- [Java](https://github.com/aws/aws-iot-device-sdk-java-v2)
- [嵌入式 C](https://github.com/aws/aws-iot-device-sdk-embedded-C)

虽然 AWS 建议使用 AWS IoT 设备开发工具包连接到 AWS IoT，但这不是必需的。 但是，如果不使用 AWS IoT 设备开发工具包，则必须提供必要的连接 以及通信安全。客户端必须发送[服务器名称指示 （SNI） TLS 外延](https://tools.ietf.org/html/rfc3546#section-3.1)在 Connection 请求中。不包含 SNI 被拒绝。有关更多信息，请参阅[运输 AWS IoT 中的安全性](https://docs.aws.amazon.com/iot/latest/developerguide/transport-security.html)。使用 IAM 用户和 AWS 凭证的客户端 验证客户端必须提供正确的[签名版本 4](https://docs.aws.amazon.com/general/latest/gr/signature-version-4.html) 验证。

### 2.2.2 服务质量 (QoS) 选项

AWS IoT 和 AWS IoT 设备开发工具包支持[MQTT 服务质量 （QoS） 级别 `0` 和 `1`](http://docs.oasis-open.org/mqtt/mqtt/v3.1.1/os/mqtt-v3.1.1-os.html#_Toc385349263). MQTT 协议还定义级别 2, 但是 AWS IoT 没有实现.

| QoS 级别   | 信息是                                             | 说明                                                                     |
| :--------- | :------------------------------------------------- | :----------------------------------------------------------------------- |
| QoS 级别 0 | 不发送或发送多次                                   | 此级别应该用于通过可靠通信链接发送的消息，或者可以毫无问题地错过的消息。 |
| QoS 级别 1 | 至少发送一次，然后重复发送，直到收到 `PUBACK` 响应 | 在发送方收到指示成功传递的 `PUBACK` 响应之前，该消息不被认为是完整的。   |

### 2.2.3 持久性会话

- 创建持久化行对话

  在 MQTT 3 中，您可以通过发送 `CONNECT` 消息并将 `cleanSession` 标记设置为 `0` 来创建 MQTT 持久性会话。如果发送 `CONNECT` 消息的客户端不存在会话，则会创建一个新的持久性会话。如果客户端已存在会话，则客户端会恢复现有会话。要创建干净会话，您需要发送一条 `CONNECT` 消息，并将 `cleanSession` 标志设置为 `1`，当客户端断开连接时，代理将不存储任何会话状态。

  在 MQTT 5 中，您可以通过设置 `Clean Start` 标志和 `Session Expiry Interval` 来处理永久性会话。干净启动控制连接会话的开始和上一会话的结束。当您设置 `Clean Start` = `1` 时，会创建一个新会话，如果之前的会话存在，将终止之前的会话。当您设置 `Clean Start` = `0` 时，连接会话将恢复之前的会话（如果存在之前的会话）。会话过期间隔控制连接会话的结束。会话过期间隔指定断开连接后会话将持续的时间（以秒为单位，4 字节整数）。设置 `Session Expiry interval`=`0` 可使会话在断开连接后立即终止。如果未在 CONNECT 消息中指定会话过期间隔，默认值为 0。

  | MQTT 5 干净启动和会话过期      |                                                    |
  | :----------------------------- | :------------------------------------------------- |
  | 属性值                         | 描述                                               |
  | `Clean Start`= `1`             | 创建新会话并终止之前的会话（如果存在之前的会话）。 |
  | `Clean Start`= `0`             | 如果存在之前的会话，则恢复会话。                   |
  | `Session Expiry Interval`> `0` | 持续会话。                                         |
  | `Session Expiry interval`= `0` | 不持续会话。                                       |

  在 MQTT 5 中，如果设置 `Clean Start` = `1` 和 `Session Expiry Interval` = `0`，这相当于 MQTT 3 清理会话。如果设置 `Clean Start` = `0` 和 `Session Expiry Interval` > `0`，这相当于 MQTT 3 永久会话。

- 持久化期间操作

  客户端使用连接已确认 (`CONNACK`) 消息中的 `sessionPresent` 属性确定是否存在持久性会话。如果 `sessionPresent` 为 `1`，则存在持久性会话，并且在客户端收到 `CONNACK` 后将为客户端存储的所有消息传送给客户端，如[重新连接到持久性会话后的消息流量](https://docs.amazonaws.cn/iot/latest/developerguide/mqtt.html#persistent-session-reconnect)中所述。如果 `sessionPresent` 是 `1`，则客户端无需重新订阅。但是，如果 `sessionPresent` 为 `0`，则不存在持久性会话，并且客户端必须重新订阅主题筛选条件。

  客户端加入持久性会话后，它可以发布消息并订阅主题筛选条件，而无需在每个操作上附加任何标记。

- 重新连接到持久性会话后的消息流量

  持久性会话表示客户端与 MQTT 消息代理之间的持续连接。当客户端使用持久性会话连接到消息代理时，消息代理会保存客户端在连接期间所做的所有订阅。当客户端断开连接时，消息代理将未确认的 QoS 1 消息和发布的新 QoS 1 消息存储到客户端订阅的主题。根据账户限制存储消息。超过限制的消息将被删除。有关持久消息限制的更多信息，请参阅 [Amazon IoT Core 终端节点和配额](https://docs.amazonaws.cn/general/latest/gr/iot-core.html#message-broker-limits)。当客户端重新连接到其持久性会话时，将恢复所有订阅，并以每秒 10 条消息的最大速率将所有已存储消息发送到客户端。在 MQTT 5 中，如果具有消息过期间隔的出站 QoS1 在客户端离线时过期，则连接恢复后，客户端将不会收到过期消息。

- 结束持久性会话

  持久性会话可通过以下方式结束：

  - 持久性会话到期时间已过。当消息代理检测到客户端已断开连接（通过客户端断开连接或连接超时）时，持久性会话到期计时器将启动。
  - 客户端发送将 `cleanSession` 标记设置为 `1` 的 `CONNECT` 消息。

  在 MQTT 3 中，持久性会话过期时间的默认值为一小时，此设置适用于账户中的所有会话。

  在 MQTT 5 中，您可以为 CONNECT 和 DISCONNECT 数据包上的每个会话设置会话过期间隔。

  对于 DISCONNECT 数据包上的会话过期间隔：

  - 如果当前会话的会话过期间隔为 0，则无法在 DISCONNECT 数据包上将会话过期间隔设置为大于 0。
  - 如果当前会话的会话过期间隔大于 0，并且您在 DISCONNECT 数据包上将会话过期间隔设置为 0，会话将在 DISCONNECT 上结束。
  - 否则，DISCONNECT 数据包的会话过期间隔将更新当前会话的会话过期间隔。

- 持久性会话到期后重新连接

  如果客户端在其持久性会话到期之前未重新连接到该会话，则该会话结束并丢弃其存储的消息。当客户端在会话到期后重新连接并将 `cleanSession` 标记设置为 `0` 时，服务会创建一个新的持久性会话。上一个会话中的任何订阅或消息都不可用于此会话，因为它们在上一个会话到期时被丢弃。

### 2.2.4 保留消息

Amazon IoT Core 保存设置了 RETAIN 标志的 MQTT 消息。这些 _保留消息_ 被作为普通的 MQTT 消息发送给已订阅该主题的所有客户端，同时也被存储起来发送给该主题的新订阅者。

- 创建保留消息
  客户端在发布 MQTT 消息时确定是否保留消息。客户端可以在发布消息时使用 设备 SDK 设置 RETAIN 标志。应用程序和服务可以在使用 Publish 操作 发布 MQTT 消息时设置 RETAIN 标志。

  每个主题名称只保留一条消息。发布到主题的带有 RETAIN 标志设置的新消息将替换先前发送到该主题的任何现有保留消息。

- 订阅保留消息主题

  客户端订阅保留消息主题，就像订阅任何其他 MQTT 消息主题一样。通过订阅保留消息主题接收的保留消息均有 RETAIN 标志设置。

  Amazon IoT Core 当客户端向保留的消息主题发布带有 0 字节消息负载的保留消息时，将删除保留的消息。已订阅保留消息主题的客户端也将收到该 0 字节消息。

  订阅包含保留消息主题的通配符主题筛选条件可使客户端接收发布到保留消息主题的后续消息，但在订阅时不会传递保留消息。

  **注意：** 要在订阅时接收保留消息，订阅请求中的主题筛选条件必须与保留消息主题完全匹配。

  订阅保留消息主题时收到的保留消息有 RETAIN 标志设置。订阅客户端在订阅后收到的保留消息则没有。

- 删除保留消息

  通过将设置了 RETAIN 标志的消息和空（0 字节）消息负载发布到要删除的保留消息的主题名称，设备、应用程序和服务可以删除保留消息。此类消息会从中删除保留的消息 Amazon IoT Core，发送给订阅了该主题的客户端，但它们不会被保留 Amazon IoT Core。

  还可以通过访问 [Amazon IoT 控制台](https://console.amazonaws.cn/iot/home#/retainedMessages) 中的保留消息以交互的方式删除保留消息。通过 [Amazon IoT 控制台](https://console.amazonaws.cn/iot/home#/retainedMessages) 删除的保留消息也会向已订阅保留消息主题的客户端发送 0 字节的消息。

  保留消息在删除后无法恢复。客户端需要发布新的保留消息才能取代已删除的消息。

- Last Will and Testament（LWT）消息
  Last Will and Testament（LWT）是 MQTT 中的一项特征。借助 LWT，客户端可以指定一条消息，当出现未启动的断开连接时，代理将该消息发布到客户定义的主题，并发送给订阅该主题的所有客户端。客户端指定的消息称为 LWT 消息或 Will 消息，客户端定义的主题称为 Will 主题。您可以指定当设备连接到代理时的 LWT 消息。通过在连接期间在 Connect Flag bits 字段中设置 Will Retain 标志，可以保留这些消息。例如，如果 Will Retain 标志设置为 1，则 Will 消息将存储在代理中的关联 Will 主题中。有关更多信息，请参阅 Will 消息。

- MQTT 5 支持的特征

  **Amazon IoT Core 支持以下 MQTT 5 功能：**

  - [共享订阅](https://docs.amazonaws.cn/iot/latest/developerguide/mqtt.html#mqtt5-shared-subscription)
  - [干净启动和会话过期](https://docs.amazonaws.cn/iot/latest/developerguide/mqtt.html#mqtt5-clean-start)
  - [所有 ACK 上的原因代码](https://docs.amazonaws.cn/iot/latest/developerguide/mqtt.html#mqtt5-reason-code)
  - [主题别名](https://docs.amazonaws.cn/iot/latest/developerguide/mqtt.html#mqtt5-topic-alias)
  - [消息过期](https://docs.amazonaws.cn/iot/latest/developerguide/mqtt.html#mqtt5-message-expiry)
  - [其它 MQTT 5 特征](https://docs.amazonaws.cn/iot/latest/developerguide/mqtt.html#mqtt5-other-features)

- MQTT 5 属性

  | 属性               | 描述                                                                                                                                                                                           | 输入类型       | 数据包                                                                |
  | :----------------- | :--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | :------------- | :-------------------------------------------------------------------- |
  | 负载格式指示符     | 一个布尔值，指示负载是否格式化为 UTF-8。                                                                                                                                                       | 字节           | PUBLISH、CONNECT                                                      |
  | 内容类型           | 一个描述负载内容的 UTF-8 字符串。                                                                                                                                                              | UTF-8 字符串   | PUBLISH、CONNECT                                                      |
  | 回复主题           | 一个 UTF-8 字符串，描述接收方作为请求-响应流程的一部分应发布到的主题。主题不得包含通配符。                                                                                                     | UTF-8 字符串   | PUBLISH、CONNECT                                                      |
  | 关联数据           | 请求消息的发送者用来指示回复消息回复哪个请求的二进制数据。                                                                                                                                     | 二元           | PUBLISH、CONNECT                                                      |
  | 用户属性           | 一对 UTF-8 字符串。此属性可以在一个数据包中出现多次。接收者将按照键-值对的发送顺序接收键-值对。                                                                                                | UTF-8 字符串对 | CONNECT、PUBLISH、Will Properties、SUBSCRIBE、DISCONNECT、UNSUBSCRIBE |
  | 消息过期间隔       | 一个 4 字节整数，表示消息过期间隔（以秒为单位）。如果此数值不存在，消息永远不会过期。                                                                                                          | 4 字节整数     | PUBLISH、CONNECT                                                      |
  | 会话过期间隔       | 一个 4 字节的整数，表示会话到期时间间隔（以秒为单位）。 Amazon IoT Core 最多支持 7 天，默认最长为 1 小时。如果您设置的值超过了账户的最大值，则 Amazon IoT Core 会在 CONNACK 中返回调整后的值。 | 4 字节整数     | CONNECT、CONNACK、DISCONNECT                                          |
  | 分配的客户端标识符 | 设备未指定客户端 ID Amazon IoT Core 时生成的随机客户端 ID。随机客户端 ID 必须是代理当前管理的任何其他会话都未使用的新客户端标识符。                                                            | UTF-8 字符串   | CONNACK                                                               |
  | 服务器保持活动     | 一个 2 字节整数，表示服务器分配的保持活动时间。如果客户端处于非活动状态的时间超过保持活状态的动时间，服务器将断开客户端的连接。                                                                | 2 字节整数     | CONNACK                                                               |
  | 请求问题信息       | 一个布尔值，表示在失败时是发送原因字符串还是用户属性。                                                                                                                                         | 字节           | CONNECT                                                               |
  | 接收最大值         | 一个 2 字节整数，表示在未收到 PUBACK 的情况下可以发送的 PUBLISH QOS > 0 数据包的最大数量。                                                                                                     | 2 字节整数     | CONNECT、CONNACK                                                      |
  | 主题别名最大值     | 此值表示将被接受作为主题别名的最大值。默认值为 0。                                                                                                                                             | 2 字节整数     | CONNECT、CONNACK                                                      |
  | 最大 QoS 数        | 支持的 QoS 的最大值。 Amazon IoT Core 默认值为 1。 Amazon IoT Core 不支持 QoS2。                                                                                                               | 字节           | CONNACK                                                               |
  | 保留可用           | 一个布尔值，用于指示 Amazon IoT Core 消息代理是否支持保留的消息。默认 为 1。                                                                                                                   | 字节           | CONNACK                                                               |
  | 最大数据包大小     | Amazon IoT Core 接受和发送的最大数据包大小。不能超过 128 KB。                                                                                                                                  | 4 字节整数     | CONNECT、CONNACK                                                      |
  | 通配符订阅可用     | 一个布尔值，用于指示 Amazon IoT Core 消息代理是否支持可用的通配符订阅。默认 为 1。                                                                                                             | 字节           | CONNACK                                                               |
  | 订阅标识符可用     | 一个布尔值，用于指示 Amazon IoT Core 消息代理是否支持可用订阅标识符。默认值是 0。                                                                                                              | 字节           | CONNACK                                                               |

- Amazon IoT 与 MQTT 规格的区别

  尽管消息代理的实施基于 [MQTT v3.1.1 规范](http://docs.oasis-open.org/mqtt/mqtt/v3.1.1/os/mqtt-v3.1.1-os.html)和 [MQTT v5.0 规范](http://docs.oasis-open.org/mqtt/mqtt/v5.0/mqtt-v5.0.html)，但与这些规范有如下区别：

  - Amazon IoT 不支持 MQTT 3 的以下数据包：PUBREC、PUBREL 和 PUBCOMP。
  - Amazon IoT 不支持 MQTT 5 的以下数据包：PUBREC、PUBREL、PUBCOMP 和 AUTH。
  - Amazon IoT 不支持 MQTT 5 服务器重定向。
  - Amazon IoT 仅支持 MQTT 服务质量 (QoS) 级别 0 和 1。 Amazon IoT 不支持以 QoS 级别 2 发布或订阅。在请求 QoS 级别 2 时，消息代理不会发送 PUBACK 或 SUBACK。
  - 在中 Amazon IoT，订阅 QoS 级别为 0 的主题意味着消息的传送次数为零次或多次。消息可能会多次发送。多次发送的消息在发送时可能会使用不同的数据包 ID。在这些情况下，不会设置 DUP 标志。
  - 在响应连接请求时，消息代理将发送 CONNACK 消息。此消息包含一个标志，用于指明该连接是否会恢复上一个会话。
  - 在发送其它控制数据包或断开连接请求之前，客户端必须等待从 Amazon IoT 消息代理发送到设备的 CONNACK 消息。
  - 当客户端订阅主题时，在消息代理开始发送 SUBACK 和客户端开始收到新的匹配消息之间存在时间延迟。
  - 当客户端在订阅主题的主题过滤条件中使用通配符 `#` 时，主题层次结构中处于及其级别以下的所有字符串均会匹配。但是，父主题不匹配。例如，订阅主题 `sensor/#` 接收发布到主题`sensor/`、`sensor/temperature`、`sensor/temperature/room1` 的消息，但不是发布到 `sensor` 的消息。有关通配符的更多信息，请参阅 [主题筛选条件](https://docs.amazonaws.cn/iot/latest/developerguide/topics.html#topicfilters)。
  - 消息代理使用客户端 ID 标识每个客户。客户端 ID 作为 MQTT 负载的一部分从客户端传递到消息代理。客户端 ID 相同的两个客户端无法同时连接到消息代理。当某个客户端使用另一客户端正在使用的客户端 ID 连接到消息代理时，会接受新的客户端连接，而之前连接的客户端会断开连接。
  - 在极少数情况下，消息代理可能会使用不同的数据包 ID 再次发送相同的逻辑 PUBLISH 消息。
  - 订阅包含通配符的主题筛选条件无法接收保留消息。要接收保留消息，订阅请求必须包含与保留消息主题完全匹配的主题筛选条件。
  - 消息代理并不保证收到消息和 ACK 的顺序。
  - Amazon IoT 可能有与规格不同的限制。有关更多信息，请参阅《Amazon IoT 参考指南》中的[Amazon IoT Core 消息代理和协议限制与限额](https://docs.amazonaws.cn/general/latest/gr/iot-core.html#message-broker-limits)。
  - 不支持 MQTT DUP 标志。

# 3. 连接到 AWS IoT Core

以个人开发电脑为 IoT 设备, 以 MQTT 为通信协议, 实施如果在本地连接到 AWS IoT, 编程代码以 nodejs 为例.

## 3.1 [创建安全策略](https://docs.aws.amazon.com/iot/latest/developerguide/create-iot-resources.html#create-iot-policy)

安全策略和权限类似, 定义哪些资源和行为是否运行执行和方案. 策略会被证书附加上, 而证书又可以附加设备, 通过设备通过证书和策略产生了关系, 进而控制设备上行数据合法性.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "iot:*",
      "Resource": "*"
    }
  ]
}
```

> [!warning]
> 这里仅是测试, 生产环境还应该尽量细粒度定义策略, 比如从 Resource 的 topic/topicfilter/client 上限制, 如下:
>
> - "arn:aws-cn:iot:cn-northwest-1:000000000000:topic/sdk/test/js"
> - "arn:aws-cn:iot:cn-northwest-1:000000000000:topicfilter/sdk/test/js"
> - "arn:aws-cn:iot:cn-northwest-1:000000000000:client/sdk-nodejs-\*"
>
> 当然那也可以从 Action 上限制, 比如:
>
> - iot:Connect
> - iot:Publish iot:RetainPublish
> - iot:Receive iot:Subscribe
> - iot:GetRetainedMessage iot:ListRetainedMessages
> - iot:GetThingShadow iot:ListNamedShadowsForThing iot:UpdateThingShadow iot:DeleteThingShadow

在 AWS IoT 中，策略对 iot:Subscribe 和 iot:Receive 的权限要求更为严格。即使你的代码中订阅的是特定的主题而不是主题过滤器，AWS IoT 的策略依然需要授权订阅和接收的操作分别对应 topicfilter 和 topic 资源。

**原因分析**

- **iot:Subscribe** **和** **iot:Receive** **的分工**:

  - iot:Subscribe 用于允许设备订阅某个主题或主题过滤器，因此需要 ==topicfilter== 资源。
  - iot:Receive 用于允许设备接收来自某个具体主题的消息，因此需要 ==topic== 资源。

- **策略的匹配机制**:

  - 当你订阅一个特定的主题时，AWS IoT 会检查 iot:Subscribe 的权限是否包含对 topicfilter 的授权。
  - 当你接收到消息时，AWS IoT 会检查 iot:Receive 的权限是否包含对该具体 topic 的授权。

- **安全性考虑**

  AWS IoT 在设计中将订阅和接收消息的权限分开管理，可能是出于安全考虑。要求明确指定 topicfilter 和 topic 资源，以确保订阅和接收消息的操作都有明确的权限控制。

## 3.2 [创建设备](https://docs.aws.amazon.com/iot/latest/developerguide/create-iot-resources.html#create-aws-thing)

创建设备即在 AWS 上创建一个设备的逻辑对象, 创建设备的重要产物是设备证书:

- 私钥
- 公钥
- 设备证书
- 根证书

> [!warning]
>
> 1. AWS IoT 并不存储您的私钥，因此一旦您错过下载私钥的机会，您需要重新生成证书和密钥对来确保设备的安全通信
> 2. 创建设备(Thing)时选择选择上一步创建的安全策略, 如果忘记附加策略可以在设备关联的证书中附加安全策略

```bash
./certs
├── AmazonRootCA1.pem
├── researchmbp-certificate.pem.crt
├── researchmbp-private.pem.key
└── researchmbp-public.pem.key
```

## 3.3 运行官方 Demo 链接

这里的代码参考了控制台中连接的测试代码, 是一个 shell 脚本, 具体实现如下;

```bash
#!/usr/bin/env bash
# stop script on error
set -e

# Check to see if root CA file exists, download if not
if [ ! -f ./certs/AmazonRootCA1.pem ]; then
  printf "\nDownloading AWS IoT Root CA certificate from AWS...\n"
  curl https://www.amazontrust.com/repository/AmazonRootCA1.pem >certs/AmazonRootCA1.pem
fi

CWD=$(pwd)

# install AWS Device SDK for NodeJS if not already installed + pubsub sample
if [ ! -d ./aws-iot-device-sdk-js-v2 ]; then
  printf "\nInstalling AWS SDK...\n"
  git clone https://github.com/aws/aws-iot-device-sdk-js-v2.git --recursive
  cd aws-iot-device-sdk-js-v2
  npm install
  # samples require their own install
  cd samples/node/pub_sub
  npm install
  cd $CWD
fi

# run pub/sub sample app using certificates downloaded in package
printf "\nRunning pub/sub sample application...\n"
endpoint=$(aws iot describe-endpoint | jq -r '.endpointAddress')
node aws-iot-device-sdk-js-v2/samples/node/pub_sub/dist/index.js --endpoint "$endpoint" --key certs/researchmbp-private.pem.key --cert certs/researchmbp-certificate.pem.crt --ca_file certs/AmazonRootCA1.pem --client_id sdk-nodejs-v2 --topic sdk/test/js
```

> [!note]
>
> 1. 将设备的证书下载到本地目录, 在执行 node 命令时注意证书路径.
> 2. 访问 AWS IoT 需要的证书有: 设备证书/私钥/根证书

> [!tip]
>
> 1. 获取设备数据端点: aws iot describe-endpoint | jq -r '.endpointAddress'
> 2. 调试测试脚本: --verbosity debug

## 3.4 运行自定义连接

自定义链接主要是借助`aws-iot-device-sdk-v2`进行自定义场景的 MQTT5 的通信开发.

### 3.4.1 用户故事

有两个电站和一个 SAAS 平台, 两个电站可以上行数据到平台,例如发送发电量; 平台也下行数据到电站,例如命令充电或者放电
| 角色| 行为描述 | 代码|
| --- | --- | --- |
|SAAS 平台|接受发电量, 并下发命令|main.ts|
|电站 0|发送发电量, 并监听下行命令|device-station0.ts|
|电站 1|发送发电量, 并监听下行命令|device-station1.ts|

### 3.4.2 代码结构

代码结构如下:

```bash
.
├── README.md
├── args.ts
├── certs
│   ├── AmazonRootCA1.pem
│   ├── researchmbp-Policy
│   ├── researchmbp-certificate.pem.crt
│   ├── researchmbp-private.pem.key
│   ├── researchmbp-public.pem.key
│   ├── saas-certificate.pem.crt
│   ├── saas-private.pem.key
│   └── saas-public.pem.key
├── device-station0.ts
├── device-station1.ts
├── lib
│   └── client.ts
├── main.ts
├── package.json
├── start.sh
└── yarn.lock
```

#### args.ts

```typescript
export const args = {
  endpoint: process.env.AWS_IOT_ENDPOINT!,
  key: "certs/researchmbp-private.pem.key",
  cert: "certs/researchmbp-certificate.pem.crt",
};

export const saasArgs = {
  endpoint: process.env.AWS_IOT_ENDPOINT!,
  key: "certs/saas-private.pem.key",
  cert: "certs/saas-certificate.pem.crt",
};
```

#### lib/client.ts

```typescript
import { mqtt5, iot } from "aws-iot-device-sdk-v2";
import { ICrtError } from "aws-crt";

export const createMQTT5Client = (
  args: {
    endpoint: string;
    cert: string;
    key: string;
  },
  clientId?: string
) => {
  // 构建Mqtt5Client配置
  const builder =
    iot.AwsIotMqtt5ClientConfigBuilder.newDirectMqttBuilderWithMtlsFromPath(
      args.endpoint,
      args.cert,
      args.key
    );
  const connectionProperties: mqtt5.ConnectPacket = {
    keepAliveIntervalSeconds: 1200,
  };
  if (clientId) {
    connectionProperties.clientId = clientId;
  }
  const config = builder.withConnectProperties(connectionProperties).build();

  // 创建Mqtt5Client
  const client: mqtt5.Mqtt5Client = new mqtt5.Mqtt5Client(config);

  client.on("error", (error: ICrtError) => {
    console.log("Error event: " + error.toString());
  });

  client.on(
    "messageReceived",
    (eventData: mqtt5.MessageReceivedEvent): void => {
      console.log(
        "Message Received event: " + JSON.stringify(eventData.message)
      );
      if (eventData.message.payload) {
        console.log(
          "    with payload: " +
            Buffer.from(
              new Uint8Array(eventData.message.payload as ArrayBuffer)
            ).toString("utf-8")
        );
      }
    }
  );

  client.on("attemptingConnect", (eventData: mqtt5.AttemptingConnectEvent) => {
    console.log("Attempting Connect event");
  });

  client.on("connectionSuccess", (eventData: mqtt5.ConnectionSuccessEvent) => {
    console.log("Connection Success event");
    console.log("    Connack: " + JSON.stringify(eventData.connack));
    console.log("    Settings: " + JSON.stringify(eventData.settings));
  });

  client.on("connectionFailure", (eventData: mqtt5.ConnectionFailureEvent) => {
    console.log("Connection failure event: " + eventData.error.toString());
    if (eventData.connack) {
      console.log("    Connack: " + JSON.stringify(eventData.connack));
    }
  });

  client.on("disconnection", (eventData: mqtt5.DisconnectionEvent) => {
    console.log("Disconnection event: " + eventData.error.toString());
    if (eventData.disconnect !== undefined) {
      console.log(
        "    Disconnect packet: " + JSON.stringify(eventData.disconnect)
      );
    }
  });

  client.on("stopped", (eventData: mqtt5.StoppedEvent) => {
    console.log("Stopped event");
  });

  return client;
};
```

#### main.ts

```typescript
import { once } from "events";
import { mqtt5 } from "aws-iot-device-sdk-v2";
import { interval, mergeMap, take, lastValueFrom } from "rxjs";
import { createMQTT5Client } from "./lib/client";
import { saasArgs } from "./args";

// 运行
const run = async (shouldExit: boolean) => {
  const client = createMQTT5Client(saasArgs, "stmain");
  const connectionSuccess = once(client, "connectionSuccess");
  client.start();
  await connectionSuccess;

  const suback = await client.subscribe({
    subscriptions: [
      { qos: mqtt5.QoS.AtLeastOnce, topicFilter: "device/station/up/#" },
      // { qos: mqtt5.QoS.AtLeastOnce, topicFilter: "device/station/up/st000" },
      // { qos: mqtt5.QoS.AtMostOnce, topicFilter: "device/station/up/st001" },
    ],
  });
  console.log("Suback result: " + JSON.stringify(suback));

  // delay publish downstream command
  const publishTask = interval(1000 * 10).pipe(
    take(2),
    mergeMap(async (index) => {
      console.log(`publish downstream command ${index}`);
      const qos0PublishResult = await client.publish({
        qos: mqtt5.QoS.AtMostOnce,
        topicName: "device/station/down/st000",
        payload: JSON.stringify({
          action: "discharge",
          endDate: Date.now() + 1000,
        }),
        userProperties: [{ name: "comeFrom", value: "solarsketch" }],
      });
      console.log(
        `Topic(device/station/down/st000) ${mqtt5.QoS.AtMostOnce} Publish result: ` +
          JSON.stringify(qos0PublishResult)
      );
      const qos1PublishResult = await client.publish({
        qos: mqtt5.QoS.AtLeastOnce,
        topicName: "device/station/down/st001",
        payload: JSON.stringify({
          action: "charge",
          endDate: Date.now() + 2000,
        }),
      });
      console.log(
        `Topic(device/station/down/st001) ${mqtt5.QoS.AtMostOnce} Publish result: ` +
          JSON.stringify(qos1PublishResult)
      );
    })
  );
  await lastValueFrom(publishTask);

  // comment here for upstream testing
  // const unsuback = await client.unsubscribe({
  //   topicFilters: ["device/station/up/#"],
  // });
  // console.log("Unsuback result: " + JSON.stringify(unsuback));

  if (shouldExit) {
    const stopped = once(client, "stopped");
    client.stop();
    await stopped;
    client.close();
  }
};

run(true)
  // .then(() => {
  //   process.exit();
  // })
  .catch((err) => {
    console.error("catch error:", err);
  });
```

#### device-station0.ts

```typescript
import { once } from "events";
import { mqtt5 } from "aws-iot-device-sdk-v2";
import { createMQTT5Client } from "./lib/client";
import { args } from "./args";

// 运行
const run = async (shouldExit: boolean) => {
  const clientId = "st000";
  const client = createMQTT5Client(args, clientId);
  const connectionSuccess = once(client, "connectionSuccess");
  client.start();
  await connectionSuccess;

  const upstreamTopic = `device/station/up/${clientId}`;
  const downstreamTopic = `device/station/down/${clientId}`;
  const suback = await client.subscribe({
    subscriptions: [
      { qos: mqtt5.QoS.AtMostOnce, topicFilter: downstreamTopic },
    ],
  });
  console.log(`Suback(${downstreamTopic}) result: ` + JSON.stringify(suback));

  const publishResult = await client.publish({
    qos: mqtt5.QoS.AtMostOnce,
    topicName: upstreamTopic,
    payload: JSON.stringify({ power: 1000, unix: Date.now() }),
    userProperties: [{ name: "deviceID", value: clientId }],
  });
  console.log(
    `Topic(${upstreamTopic}) ${mqtt5.QoS.AtMostOnce} Publish result: ` +
      JSON.stringify(publishResult)
  );
  // const unsuback = await client.unsubscribe({
  //   topicFilters: [downstreamTopic],
  // });
  // console.log("Unsuback result: " + JSON.stringify(unsuback));

  if (shouldExit) {
    const stopped = once(client, "stopped");

    client.stop();

    await stopped;

    client.close();
  }
};

run(false).catch((err) => {
  console.error("catch error:", err);
});
```

#### device-station1.ts

```typescript
import { once } from "events";
import { mqtt5 } from "aws-iot-device-sdk-v2";
import { createMQTT5Client } from "./lib/client";
import { args } from "./args";

// 运行
const run = async (shouldExit: boolean) => {
  const clientId = "st001";
  const client = createMQTT5Client(args, clientId);
  const connectionSuccess = once(client, "connectionSuccess");
  client.start();
  await connectionSuccess;

  const upstreamTopic = `device/station/up/${clientId}`;
  const downstreamTopic = `device/station/down/${clientId}`;
  const suback = await client.subscribe({
    subscriptions: [
      { qos: mqtt5.QoS.AtMostOnce, topicFilter: downstreamTopic },
    ],
  });
  console.log(`Suback(${downstreamTopic}) result: ` + JSON.stringify(suback));

  const publishResult = await client.publish({
    qos: mqtt5.QoS.AtMostOnce,
    topicName: upstreamTopic,
    payload: JSON.stringify({ power: 1000, unix: Date.now() }),
    userProperties: [{ name: "deviceID", value: clientId }],
  });
  console.log(
    `Topic(${upstreamTopic}) ${mqtt5.QoS.AtMostOnce} Publish result: ` +
      JSON.stringify(publishResult)
  );

  // const unsuback = await client.unsubscribe({
  //   topicFilters: [upstreamTopic],
  // });
  // console.log("Unsuback result: " + JSON.stringify(unsuback));

  if (shouldExit) {
    const stopped = once(client, "stopped");

    client.stop();

    await stopped;

    client.close();
  }
};

run(false).catch((err) => {
  console.error("catch error:", err);
});
```

> [!note]
> 详情代码参考 Github 仓库: [awsiot-tutorial](https://github.com/nnsay/awsiot-tutorial)

### 3.4.3 测试

```bash
# 新建终端运行SAAS 平台
AWS_IOT_ENDPOINT=$(aws iot describe-endpoint | jq -r '.endpointAddress') ts-node main.ts

# 新建终端运行电站 0
AWS_IOT_ENDPOINT=$(aws iot describe-endpoint | jq -r '.endpointAddress') ts-node device-station0.ts

# 新建终端运行电站 1
AWS_IOT_ENDPOINT=$(aws iot describe-endpoint | jq -r '.endpointAddress') ts-node device-station1.ts
```

> [!warning]
> 因为要获取 AWS IoT Endpoint 所以在每个终端中需要有访问 aws 的访问权限

在三个终端分别查看上行下行数据情况即可.

# 4. 常见问题探究

## 4.1 设备身份证

当设备数据上行时, 如何判断一个设备真实是哪个设备? 现有的方案如下:

- **使用设备唯一标识符作为主题的一部分**

设备 1 发布消息到 devices/device1/data

设备 2 发布消息到 devices/device2/data

订阅者可以通过订阅 devices/+/data 来接收所有设备的消息，并从主题中提取出设备的唯一标识符

- **在消息的 Payload 中包含设备标识**

  ```json
  {
    "device_id": "device1",
    "timestamp": "2024-08-27T12:00:00Z",
    "data": {
      "temperature": 22.5,
      "humidity": 60
    }
  }
  ```

- **使用 MQTT 的 Client ID**

  每个 MQTT 客户端在连接时都会指定一个 Client ID。订阅者可以使用 MQTT Broker 提供的功能来查看 Client ID，从而确定消息是由哪个设备发送的。然而，这种方法通常不用于消息内容中的设备标识，更多是用于连接管理和控制。

- **使用自定义的 MQTT Headers**

  某些 MQTT Broker 可能支持自定义 headers（类似于 HTTP headers），可以通过这种方式附加设备的标识信息。不过，这种方法依赖于 Broker 的支持，不能在所有环境下使用。

- **使用证书或安全凭证**

  如果使用 TLS/SSL 连接或其他安全措施，每个设备通常有自己的证书或密钥对。通过验证消息来源的安全证书，可以确认是哪一个设备发送的消息。这种方法通常用于安全要求较高的场景。

可行性分析其中出最后一个外基本算是一类: 发送方通过可定义化的标识加载在 MQTT 的某些属性中生效, 然后接收方通过读取属性来确定设备. 大部分情况下该方法是可行的且是简单的, 但是如果设备属于第三方这种标识可能被篡改或者模拟. 最后一种看起来可取, 但是需要一些手段:

- ~~可以在 MQTT 接受到消息时可以获取其通信用的公钥~~
- ~~MQTT 服务本身可以提供一些中间件可以自定义实现设备身份确认的逻辑~~

在 AWS IoT 中，当设备通过 MQTT 发送消息时，虽然设备的通信是基于 X.509 证书加密的，但在消息的接收和处理过程中，AWS IoT 并不会直接提供设备证书（包括公钥）的信息; 而 AWS IoT MQTT 也无法进行自定义开发, 毕竟是黑盒的产品.

### 方案 1: 安全策略

通过 Topic 或 ClientId 可能被模拟或者篡改, 但是在 AWS IoT 中每一个设备的证书是可以附加安全策略, 可以严格设置安全策略这个设备是否可以连接且那些 Topic 可以订阅和发送, 如此即使 Topic 或者 ClientId 被篡改或者模拟也无法正常通信, 而接受者可以通过这两个信息来确认设备身份, 以下是一个设备安全策略的示例:

- 限制了哪些 Topic 可以发送消息
- 限制了哪些 Topic 可以订阅和接受到消息
- 限制了连接的 ClientId

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["iot:Publish", "iot:PublishRetain"],
      "Resource": "arn:aws-cn:iot:cn-northwest-1:*:topic/device/station/up/st000"
    },
    {
      "Effect": "Allow",
      "Action": ["iot:Receive"],
      "Resource": [
        "arn:aws-cn:iot:cn-northwest-1:*:topic/device/station/down/st000"
      ]
    },
    {
      "Effect": "Allow",
      "Action": ["iot:Subscribe"],
      "Resource": [
        "arn:aws-cn:iot:cn-northwest-1:*:topicfilter/device/station/down/st000"
      ]
    },
    {
      "Effect": "Allow",
      "Action": "iot:Connect",
      "Resource": [
        "arn:aws-cn:iot:cn-northwest-1:*:client/stmain",
        "arn:aws-cn:iot:cn-northwest-1:*:client/st000",
        "arn:aws-cn:iot:cn-northwest-1:*:client/st001"
      ]
    }
  ]
}
```

如**3.4**的测试用例, main.ts 的设备使用的是 allow all 的安全策略, 而 device-station0.ts 和 device-station1.ts 的设备使用上面这个安全策略, 测试结果如下:

- main.ts

  不受影响, 可以订阅发布, 因为其用的 allow all 的安全策略

- device-station0.ts

  不受影响, 可以订阅发布, 因为其用的安全策略允许该设备做这些事情

- device-station1.ts

  可以连接, 但是发送和订阅的没有结果, 虽然代码上看到 Publish 结果和 device-station0.ts 但是 main.ts 中并没有收到其消息, 其和 main.ts 的上下行数据被完全限制了. 如看初始的日志可以看到 Subscribe 的错误码是: 135, 在[MQTT 原因码](https://docs.aws.amazon.com/zh_cn/iot/latest/developerguide/mqtt.html#mqtt5-reason-codes)中正是为授权的意思

  ![](https://img.picgo.net/2024/09/09/9fc45658f448c57e3dfd34cfd8533b79c9b49f4cc31b2169.png)

> [!important]
> 建议在生产环境严格限制 IoT:Connect 中的 Client, 因为根据[AWS IoT MQTT 规格特性](https://docs.aws.amazon.com/zh_cn/iot/latest/developerguide/mqtt.html#mqtt-differences): 消息代理使用客户端 ID 标识每个客户。客户端 ID 作为 MQTT 负载的一部分从客户端传递到消息代理。客户端 ID 相同的两个客户端无法同时连接到消息代理。当某个客户端使用另一客户端正在使用的客户端 ID 连接到消息代理时，会接受新的客户端连接，而之前连接的客户端会断开连接

### 方案 2: 规则

AWS IoT[规则](https://docs.aws.amazon.com/zh_cn/iot/latest/developerguide/iot-rules.html)是为了设备信息可以 AWS 其他服务交互, 其中规则支持的第一个特性就是: 补充或筛选从设备接收的数据. 具体实现如下:

- AWS IoT 规则引擎可以使用 SQL 查询从 MQTT 消息中提取数据。在查询中，你可以使用 principalid() 函数来获取设备的证书 ID，这个 ID 是与每个设备的证书唯一绑定的，无法被模拟。
- 你可以将消息数据和 principalid 一起存储到 DynamoDB 或通过 Lambda 函数进一步处理

> [!note] > [principal()方法](https://docs.aws.amazon.com/zh_cn/iot/latest/developerguide/iot-sql-functions.html#iot-sql-function-principal)可以返回设备用于身份验证的委托人, 当客户端使用 X.509 设备证书时返回 X.509 证书指纹即设备证书 ID

> [!important]
> 方案一已经验证可行
> 方案二没有实际验证, 经过查看文档理论上也可行, 但是处理逻辑上是需要多走一步(principalid()获取证书), 实现起来比较复杂, 详情可以查看[文档](https://docs.aws.amazon.com/zh_cn/iot/latest/developerguide/iot-rules.html)

## 4.2 不同设备不同证书通信

AWS IoT 中每个一设备(Things)会有自己的证书, 证书只是安全保护和 HTTPS 类似, 但是证书不影响 pub/sub 的消息通信, 多个设备使用各自的证书链接到 MQTT 节点正常通信即可.

## 4.3 非 AWS IoT Device SDK 通信

这里使用 python 的 SDK 测试, 不过这个测试有点特殊, 这个测试依赖**4.6**的背景, 简单来说不是通过证书来认证而是通过用户名和密码连接到 AWS IoT.

安装包 python 包:

```bash
pip install paho-mqtt
```

执行脚本:

```python
import paho.mqtt.client as mqtt
import ssl
import os

# 设置AWS IoT连接参数
mqtt_endpoint = os.getenv("MQTT_ENDPOINT", "")  # 替换为你的 AWS IoT 端点
mqtt_port = 443  # 使用443端口
mqtt_username = os.getenv("MQTT_USERNAME")  # 替换为你的用户名（Token）
mqtt_password = os.getenv("MQTT_PASSWORD")  # 替换为你的密码（Token）
mqtt_client_id = mqtt_username  # 替换为你的客户端ID
topic = f"device/station/down/{mqtt_username}"


# The callback for when the client receives a CONNACK response from the server.
def on_connect(client, userdata, flags, reason_code, properties):
    print(f"Connected with result code {reason_code}")
    # Subscribing in on_connect() means that if we lose the connection and
    # reconnect then subscriptions will be renewed.
    client.subscribe(topic, qos=1)


# The callback for when a PUBLISH message is received from the server.
def on_message(client, userdata, msg):
    print("Received: " + msg.topic + " " + str(msg.payload))


mqttc = mqtt.Client(
    callback_api_version=mqtt.CallbackAPIVersion.VERSION2,  # type: ignore
    client_id=mqtt_client_id,
    clean_session=True,
    protocol=mqtt.MQTTv311,
)
mqttc.on_connect = on_connect
mqttc.on_message = on_message

# 设置用户名和密码
mqttc.username_pw_set(username=mqtt_username, password=mqtt_password)
# 配置自定义 TLS/SSL 上下文
context = ssl.create_default_context()
context.set_alpn_protocols(["mqtt"])  # 设置 ALPN 协议
mqttc.tls_set_context(context)

# 连接到 AWS IoT
mqttc.connect(
    host=mqtt_endpoint,
    port=mqtt_port,
    keepalive=60,
)

# 发布一条消息到主题
mqttc.publish(topic, payload="Hello from paho-mqtt!", qos=1, retain=False)

# Blocking call that processes network traffic, dispatches callbacks and
# handles reconnecting.
# Other loop*() functions are available that give a threaded interface and a
# manual interface.
mqttc.loop_forever()

```

测试结果:

```
Connected with result code Success
Received: device/station/down/mqtt b'Hello from paho-mqtt!'
```

## 4.4 设备断连重发

要实现这个特性需要创建[持久化会话](https://docs.amazonaws.cn/iot/latest/developerguide/mqtt.html#mqtt-persistent-sessions), 在 MQTT 5 中，您可以通过设置 `Clean Start` 标志和 `Session Expiry Interval` 来处理永久性会话。干净启动控制连接会话的开始和上一会话的结束。当您设置 `Clean Start` = `1` 时，会创建一个新会话，如果之前的会话存在，将终止之前的会话。当您设置 `Clean Start` = `0` 时，连接会话将恢复之前的会话（如果存在之前的会话）。会话过期间隔控制连接会话的结束。会话过期间隔指定断开连接后会话将持续的时间（以秒为单位，4 字节整数）。设置 `Session Expiry interval`=`0` 可使会话在断开连接后立即终止。如果未在 CONNECT 消息中指定会话过期间隔，默认值为 0.

```typescript
const builder =
  iot.AwsIotMqtt5ClientConfigBuilder.newDirectMqttBuilderWithMtlsFromPath(
    args.endpoint,
    args.cert,
    args.key
  );
const connectionProperties: mqtt5.ConnectPacket = {
  keepAliveIntervalSeconds: 1200,
  sessionExpiryIntervalSeconds: 86400,
  clientId: "custom-clientId",
};
const config = builder
  .withConnectProperties(connectionProperties)
  .withSessionBehavior(mqtt5.ClientSessionBehavior.RejoinPostSuccess)
  .build();
const client: mqtt5.Mqtt5Client = new mqtt5.Mqtt5Client(config);
```

## 4.5 自定义端点域名

```typescript
// 创建自定义域名配置
const mqttCertificate = new acm.Certificate(
  this,
  `${props.envName}-MQTT-Endpoint-Certificate`,
  {
    domainName: infraConfig[props.envName].mqtt.customDomain,
    certificateName: `${props.envName}-mqtt-endpoint-cert`,
    validation: acm.CertificateValidation.fromDns(),
  }
);
new iot.CfnDomainConfiguration(this, `${props.envName}-MqttDomainConfig`, {
  domainConfigurationName: `${props.envName}-mqtt-domain`,
  domainName: infraConfig[props.envName].mqtt.customDomain,
  serverCertificateArns: [mqttCertificate.certificateArn],
  domainConfigurationStatus: "ENABLED",
});
// 创建自定义域名映记录
const hostedZone = route53.HostedZone.fromHostedZoneAttributes(
  this,
  `MQTT-HostedZone`,
  {
    hostedZoneId: getHostedZoneId(infraConfig[props.envName].mqtt.customDomain),
    zoneName: infraConfig[props.envName].mqtt.customDomain
      .split(".")
      .slice(-2)
      .join("."),
  }
);
new route53.CnameRecord(this, `${props.envName}-MQTTCnameRecord`, {
  recordName: infraConfig[props.envName].mqtt.customDomain.split(".")[0],
  zone: hostedZone,
  domainName: infraConfig[props.envName].mqtt.defaultEndpointDomain,
});
```

其中 _infraConfig[props.envName].mqtt_ 配置类似如下:

```json
{
  "customDomain": "mqtt-sandbox.nnsay.cn",
  "defaultEndpointDomain": "aabbccddee-ats.iot.cn-northwest-1.amazonaws.com.cn"
}
```

## 4.6 用户名密码认证

AWS 推荐的认证方式是证书认证如**4.1**章节所述, 但是很多现有的设备或者服务还是喜欢用户名密码方式连接 MQTT, 这个特性需要使用[自定义身份验证](https://docs.amazonaws.cn/iot/latest/developerguide/custom-authentication.html), 大致流程是如下:

- 创建自定义 Authorizer
- 创建自定义域名配置使用自定义 Authorizer

具体实施步骤可以参考官网 Demo 的代码仓库: **[aws-iot-enhanced-custom-authorizer-demo](https://github.com/aws-samples/aws-iot-enhanced-custom-authorizer-demo)**

自定义认证的测试这里也采用 aws iot 的 SDK 来做, 如下:

```typescript
import { mqtt, io } from "aws-iot-device-sdk-v2";
import dotenv from "dotenv";

dotenv.config();

const mqttClient = new mqtt.MqttClient();
const tlsCtxOptions = new io.TlsContextOptions();
tlsCtxOptions.alpn_list = ["mqtt"];
const clientId = process.env.MQTT_USERNAME!;
const connection = mqttClient.new_connection({
  client_id: clientId,
  host_name: process.env.MQTT_ENDPOINT!,
  port: 443,
  use_websocket: false,
  clean_session: true,
  keep_alive: 60,
  username: process.env.MQTT_USERNAME!,
  password: process.env.MQTT_PASSWORD!,
  socket_options: new io.SocketOptions(
    io.SocketType.STREAM,
    io.SocketDomain.IPV4,
    1000 * 60
  ),
  tls_ctx: new io.ClientTlsContext(tlsCtxOptions),
});

connection
  .on("connection_success", (eventData) => {
    console.log("Connection Success event");
    console.log("    reason_code: " + eventData.reason_code);
    console.log("    session_present: " + eventData.session_present);
  })
  .on("error", (eventData) => {
    console.log("Error event: " + eventData.message);
  })
  .on("disconnect", () => {
    console.log("Disconnection Success event");
  })
  .on("connection_failure", (eventData) => {
    console.log("Connection failure event: " + eventData.error.toString());
  })
  .on("closed", () => {
    console.log("Closed event");
  });

const exec = async () => {
  await connection.connect();
};

exec().catch((err) => console.error("catch error: ", err));
```

> [!note]
> 切换到`mqtt-username-password`分支进行查看, 直接看文件夹 mqtt-username-password 中的内容即可

> [!important]
>
> 1. 如果设备不支持定制开发仅支持静态配置方式访问 MQTT 建议禁用自定义 Authorizer 的签名功能
> 2. [自定义授权对客户端有特殊要求](https://docs.aws.amazon.com/zh_cn/iot/latest/developerguide/custom-auth.html#custom-auth-mqtt): AWS IoT Core 通过使用 MQTT 和自定义身份验证连接的设备必须通过端口 443 进行连接。它们还必须通过应用层协议协商 (ALPN) TLS 扩展名，值为`mqtt`和服务器名称指示 (SNI) 扩展名及其 AWS IoT Core 数据端点的主机名

小结:

1. Amazon Web Service IoT 如果要支持用户名密码访问需要使用自定义认证.
2. 对于 MQTT 协议的自定义身份认证,要求 ALPN 设置 mqtt,以和证书认证使用的 ALPN x-amzn-mqtt-ca 区分
3. 如果现有设备只能使用 MQTT 协议,且不支持改 ALPN 的修改,则无法接入 Amazon Web Service IoT 服务.

# 5. 参考资料

## 5.1 AWS IoT 相关

- [aws-iot-device-dsk-v2](https://aws.github.io/aws-iot-device-sdk-js-v2/node/index.html)
- [MQTT5 Samples](https://github.com/aws/aws-iot-device-sdk-js-v2/tree/main/samples)

## 5.2 MQTT Broker 推荐

- 开源

  - [emitter](https://emitter.io/download/): High performance, distributed and low latency publish-subscribe platform.

  - [emqx](https://github.com/emqx/emqx): The most scalable open-source MQTT broker for IoT, IIoT, and connected vehicles
  - [bifromq](https://github.com/bifromqio/bifromq): A Multi-Tenancy MQTT broker adopting Serverless architecture

- 云产品

  - [云消息队列 MQTT 版](ttps://help.aliyun.com/zh/apsaramq-for-mqtt)
  - [EMQX](https://www.emqx.com/zh)

## 5.3 开源工具

- [MQTTX](https://github.com/emqx/MQTTX): Powerful and All-in-One MQTT 5.0 client toolbox for Desktop, CLI and WebSocket.
