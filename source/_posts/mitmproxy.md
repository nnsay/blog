---
title: mitmproxy
date: 2024-12-26 11:51:13
tags:
  - 工具
  - 研究
excerpt: 使用开源代理工具 mitproxy 替换 fiddler everywhere
---

# 1. mitmproxy 介绍

[mitmproxy](https://mitmproxy.org/) 是款开源代理工具, 如果之前用过 [fiddler everywhere](https://www.telerik.com/fiddler/fiddler-everywhere) 那就很容易理解此类工具. 此类工具多用于代理查看分析软件应用的网络请求, 调试 API, 或者了解应用通信逻辑和规则.

本文以如何了解一个小程序应用的数据通信为例, 来了解如何使用 mitmproxy 来捕获其请求和响应.

# 2. 安装和配置

## 2.1 安装应用

```bash
brew install mitmproxy
```

安装后会有三个核心应用:

- **mitmproxy**: 交互式的命令行工具

- **mitmweb**: 和 mitmproxy 类似, 但是基于 web 浏览器界面

- **mitmdump**: 命令行工具, 可以 dump 请求

## 2.2 安装证书

因为现在大部分应用都走 HTTPS 协议了, 所以安装 mitmproxy 证书这个步骤必须要做, 具体步骤如下:

- 启动代理

  ```bash
  mitmproxy
  ```

- 配置系统代理<a id='systemproxy'></a>

  打开系统网络配置, 配置 http 和 https 代理地址为 mitmproxy 的默认代理地址: http://127.0.0.1:8080

  ![](https://img.picgo.net/2024/12/25/42f8b295a09c2802ffc5ee536f25cce5056f85f9a902ee5b.png)

- 安装证书

  打开浏览器访问: [http://mitm.it](http://mitm.it/), 下载对应平台的证书并安装

  ![](https://img.picgo.net/2024/12/26/e270cb94693b269bfbc968441cde5be833db8b469b2a9a47.png)
  安装和信任证书的状态如下:

  ![](https://img.picgo.net/2024/12/26/8b3cf84e3071edc13393f7da03f36863f52c4e61eb9093b4.png)

# 3. 使用和验证

虽然 mitmproxy 命令行工具支持交互的操作, 但是 GUI 更简单和方便, 所以在实践案例(查看微信小程序数据交互)中我们使用 mitmweb.

## 3.1 启动代理

在终端中使用 mitmweb 命令行启动, 在浏览器会自动打开代理页面, 之后称这个页面为代理控制台, 因为这个页面可以像浏览器开发者工具中的网络工具一样, 筛选和查看某个请求和其响应.

```bash
mitmweb
```

浏览器中的代理控制台如下:

![](https://img.picgo.net/2024/12/26/1c59571eb74f1a863c175ca501130615ed46c42302161af3.png)

## 3.2 配置代理

想要看到应用的数据请求需要提配置网络代理, 一般有两种情况:

- 系统代理

  会代理所有应用的网络请求, 可以过滤出目标请求来查看, macos 的系统网络代理可以参考上一章节中[配置系统代理](#systemproxy)

- 应用代理

  此种方式需要应用支持, 比如微信桌面版本就支持配置代理, 如果应用本身不支持那就只能配置系统代理了, 如下图是微信的代理界面:

  ![微信代理](https://img.picgo.net/2024/12/26/e559310c2167daf27cd3ba99cf8170e4423a918e3ab8fa6c.png)

> [!TIP]
>
> 如果查看微信小程序数据请求配置应用代理是不行的, 经过测试小程序本身的网络请求不受微信应用代理影响, 所以还是需要配置系统网络代理.

## 3.3 查看数据

在桌面微信中打开小程序, 运行小程序触发一些数据请求, 然后在 mitmweb 的代理控制台中查看请求数据即可, 例如查看腾讯天气数据请求如下:

![](https://img.picgo.net/2024/12/26/a2c83d68adf907d95d9cc2b667386e4b26b8e4cf923c746d.png)

> [!CAUTION]
>
> 经过测试发现很多小程序数据交互没有认证保护, 也就是说知道了 API 和参数就可以直接请求数据了, 这是一种有安全隐患.

> [!IMPORTANT]
>
> 本文使用的方案和案例也是为了使用和验证 mitmproxy 功能, 请勿将该工具和方法用于非法或者不道德的技术目的.

# 4. 总结

[mitmproxy](https://mitmproxy.org/) 是一款优秀的开源的代理工具, 在基础使用层面完全可以替代 [fiddler everywhere](https://www.telerik.com/fiddler/fiddler-everywhere) , 后者作为一款商业软件功能很丰富但是只能试用 10 天, 价格是订阅制也很贵, 所以推荐使用 mitmproxy.

在使用 mitmproxy 或者 fiddler everywhere 等其他代理工具时如果您发现了安全漏洞和隐患请告知应用开发者, **不要利用此类漏洞和隐患从事非法事宜**.
