---
title: 迁移aws lambda到vpc
date: 2023-03-31 19:18:49
tags: 
  - Tips
  - DevOps
excerpt: 解决vpc lambda访问各种外部资源和服务问题
---

# 1. 问题背景

​	目前后端架构是lambda不在vpc网络, 依赖的计算资源有: sqs/dynamodb/s3/第三方服务, 最近有一个新的需求是需要增加一个资源: redis. 增加redis的目的是使用缓存逻辑, 经过调研技术选型后的服务是: aws elasticache for redis(以下简称redis). 而redis必须在vpc内才可以创建, 所以在访问性上架构面临选择:

- 方案一: lambda放到vpc里面, 这个时需要解决的问题有: 
  - 访问vpc外的dynamodb
  - 访问vpc外的sqs
  - 访问vpc外的s3
  - 访问公网(第三方服务)
  

​		<img src="https://raw.githubusercontent.com/nnsay/gist/main/imgimage-20230331191211434.png" alt="image-20230331191211434" style="zoom: 33%;" />
 	

- 方案二: lambda继续保留在vpc外, 这时需要解决的问题是:
  - 如何访问vpc里的redis


​	经过一系列的讨论和研究, 从长远计未来我们的服务是要放到vpc中的, 宜早不宜迟, 所以我们选择迁移lambda进入vpc, 本文即讨论如何解决迁移后方案一中的4个访问问题.

# 2. 解决问题

方案一种四个需要解决的问题可以分为两类: 访问vpc外aws服务和访问外网, 两类问题最终调研和实行的解决方案如下:

- 访问vpc外aws服务: 使用`vpc endpoint`(vpc终端节点)的方式解决, 截取一段vpc endpoint的介绍如下

  > VPC 终端节点支持 Virtual Private Cloud (VPC) 与受支持的服务之间的连接，而无需您使用互联网网关、NAT 设备、VPN 连接或 AWS Direct Connect 连接。因此，您的 VPC 不会暴露在公共互联网上

  具体操作如下:

  - 创建s3 gateway类型的vpc终端节点
  - 创建dynamodb gateway类型的vpc终端节点
  - 创建sqs interface类型的vpc终端节点

  三种终端节点创建完后可以解决: vpc内lambda访问原有aws资源的问题.

- 访问外网: 使用nat gateway的方式解决, 具体操作如下:

  - 获取lambda vpc的subnet(子网) 

    简单起见假设vpc的public和island子网各有一个

  - 创建eip(弹性ip), 创建public类型nat gateway: 使用eip分配的ip + vpc pubic subnet

  - 创建route table(路由表), 并关联vpc

  - 编辑route table 子网关联, 将vpc的island子网关联上

  - 编辑route table  路由, 添加路由: `0.0.0.0/0`-> nat gateway

  创建和配置nat gateway结束后vpc lambda即可以访问公网, 访问路径大致是: vpc island subnet-> route -> nat gateway -> Internet

# 3. NAT Gateway vs VPC Endpoint

理论上有了NAT Gateway之后vpc里的lambda就可以像重构之前一样通过公网访问aws的各类服务, 为什么还需要创新vpc endpoint? 这里以sqs interface终端节点为例讲解:

- interface类型的终端节点是按量收费的, gateway的终端节点是免费的
- [终端节点的收费](https://www.amazonaws.cn/privatelink/pricing/)标准比[nat gateway](https://www.amazonaws.cn/vpc/pricing/)低
- 终端节点访问aws服务走的aws内网, 而不是走公网, 所以效率比较高

综上选择vpc终端节点是比较好的选择(在阿里云中例如OSS,创建后会有三个地址: vpc/public/intranet, 个人感觉这个vpc终端节点起的作用类似阿里云的vpc地址)

