---
title: 简单三步发布本地服务或网站
date: 2020-11-13 16:25:17
tags: 
- Tips
- 工具

excerpt: 使用`diode`发布本地网站或者API到公网, 方便演示或者调式API
---

## 1. 背景

在日常开发中经常需要演示和调试未上线的网站或者API，[diode](https://support.diode.io/) 应运而生，简单的快速的可以发布本地的网站或者API服务到外网，然后通过diode.link的二级域名访问你发布的东西，十分方便。本文介绍如何简单快速的通过diode发布本地的API服务。

## 2. 三步发布本地API服务

### 2.1 启动一个本地服务

我本地有一个开发的中的项目， 这个项目有一个健康检查的API： GET /health, 该项目是基于nodejs框架express的，这里不详细介绍了总之它是一个提供RESTful API的服务，使用如下命令启动：

```
npm run start
```

我的服务启动后监听本地3000端口。

### 2.2 安装[diode](https://support.diode.io/article/lsr4tkzz8t)

```
curl -Ssf https://diode.io/install.sh | sh
```

### 2.3 发布本地服务

```
diode publish -public 80:3000 
```

上面这个命令的意思是： 本地3000端口的服务被发布到diode的80端口上，且这个发布是公开的（当然也可以更多安全发布策略）。

首次发布比较慢，diode会初始化一个client的ID， 这个ID作为diode.link的自域名可以在外网被访问。上面命令的控制台输出大致如下：

```
[INFO] Diode Client version : v0.6.1 02 Nov 2020                         
[INFO] Client address       : 0xaf9e2d540232d394997480889042a2e9a3d11beb 
[INFO] Fleet address        : 0x6000000000000000000000000000000000000000 
[INFO] Connected to host: as1.prenet.diode.io:41046, validating... 
[INFO] Network is validated, last valid block: 1001299 0x000024f36df70e4b0694bd7f604ed326dddd72cc25cbe8068af9934f348e11d6 
[INFO]  
[INFO] Port      <name>     : <extern>     <mode>    <protocol>     <allowlist> 
[INFO] Port         80      :     3000      public       any             
[INFO] Port was not published port = 80         server=as1.prenet.diode.io:41046       server=as1.prenet.diode.io:41046
[INFO] Port was not published port = 80         server=as1.prenet.diode.io:41046
^C[INFO] 1/5 Stopping socksserver 
[INFO] 2/5 Stopping proxyserver 
[INFO] 3/5 Stopping configserver 
[INFO] 4/5 Cleaning pool 
[INFO] 5/5 Closing logs 
➜  ~ diode publish -public 3000:80  
[INFO] Diode Client version : v0.6.1 02 Nov 2020                         
[INFO] Client address       : 0xaf9e2d540232d394997480889042a2e9a3d11beb 
[INFO] Fleet address        : 0x6000000000000000000000000000000000000000 
[INFO] Connected to host: as1.prenet.diode.io:41046, validating... 
[INFO] Network is validated, last valid block: 1001331 0x0000793c211a3a7707fcf13b5275a84a961c7f19857ce03c9c0e8d3026bc0b16 
[INFO]  
[INFO] HTTP Gateway Enabled : http://0xaf9e2d540232d394997480889042a2e9a3d11beb.diode.link/ 
[INFO] Port      <name>     : <extern>     <mode>    <protocol>     <allowlist> 
[INFO] Port       3000      :       80      public       any             

```

## 3. 验证

通过上面的三步，可以从最后一步最后看到本地程序已经被发布了，从控制台可以看到一个`Client address`, 现在你可以在外网使用： http://<client address>.diode.link 如同使用本地的服务 http://localhost:3000 ， 使用curl测试一下health接口：

```
curl -s -XGET https://0xaf9e2d540232d394997480889042a2e9a3d11beb.diode.link/health
```

测试结果符合预期， 测试成功！


## 4. 总结

Diode是我看到第一个免费的且超级方便的本地服务发布工具，在安全性它还有一些高级的设置可以继续学习和研究， 比如只允许特定的地址访问被发布的外网服务；diode也可以发布本地静态文件所以网站也是可以,这对前端的同学也很有帮助。

今天介绍的Diode希望可以帮助开发的朋友。