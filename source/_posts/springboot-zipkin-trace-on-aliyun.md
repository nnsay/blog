---
title: SpringBoot链路追踪服务在阿里云上的实践
date: 2022-08-03 18:10:19
tags:
  - Tips
  - Aliyun
  - DevOps
  - CloudNative
excerpt: 如何在阿里云上搭建Springboot微服务链路追踪系统
---

# 0. 背景介绍

springboot集成了zipkin, 而阿里云上`日志服务`中可以新建Trace服务, 所以需要把trace数据推送到这个服务即可. 难点是如何上推送? 阿里云支持的trace推送的方式有两种: 1. 直接发送; 2. 通过collector发送. 相关文档可以看[这里](https://help.aliyun.com/document_detail/208913.html). 

第一种是需要AccessKey ID和AccessKey Secre, 维护这个显然没有维护role方便(现实情况中已有所有服务都是用的role方式获取权限, 不想在新增一种), 其次第一种业务和云厂商绑定死了, 因为直接发送的地址就是阿里云Trace服务暴露的地址.

本文档使用第二种实现和搭建Trace, 这里借助了[opentelemetry-collector-contrib](https://github.com/open-telemetry/opentelemetry-collector-contrib); collector支持role方式授权推送trace数据, 不直接和云厂商服务对接, 即opentelemetry-collector-contrib屏蔽了云厂商, 以后用别的厂商只需要修改collector的配置文件即可.

# 1. 准备工作

该文档使用opentelemetry-collector-contrib镜像自建deployment/service等资源, 与使用controller方式相比这么做的优势是:

- 添加云厂商的特殊注解. eg: k8s.aliyun.com/eci-ram-role-name
- 更精细化设置collector pod

## 1.1  Role授权
- 授权日志服务
- 授权日志服务使用的对象存储服务

```json
{
  "Action": "log:*",
  "Effect": "Allow",
  "Resource": "acs:log:*:*:*"
},
{
  "Action": "oss:*",
  "Effect": "Allow",
  "Resource": [
		...
    "acs:oss:*:*:ngiq-cn-*-log",
    "acs:oss:*:*:ngiq-cn-*-log/*"
  ]
},
```

在这个文档, 这个Role是: `pnt-CodeRole`

## 1.2 Collector镜像

测试过程中发现官方opentelemetry-collector-contrib镜像有被不能加载的情况, 另外也每次使用官方镜像都走外网, 所以先把官方镜像推送自有仓库中, 例如使用CICD服务上传镜像到阿里云仓库
```bash
docker pull otel/opentelemetry-collector-contrib

docker tag otel/opentelemetry-collector-contrib ngiq-registry-vpc.cn-hangzhou.cr.aliyuncs.com/ngiq-cr/opentelemetry-collector-contrib

aliyun cr GetAuthorizationToken --InstanceId cri-xxxxx --force --version 2018-12-01 | jq -r .AuthorizationToken | docker login --username=cr_temp_user --password-stdin ngiq-registry-vpc.cn-hangzhou.cr.aliyuncs.com

docker push ngiq-registry-vpc.cn-hangzhou.cr.aliyuncs.com/ngiq-cr/opentelemetry-collector-contrib
```

## 1.3 [创建Trace实例](https://help.aliyun.com/document_detail/208892.html)

# 2. Collector服务

创建collector服务yaml文件opentelemetry-app.yaml, 内容如下:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: otel-collector
data:
  collector.yaml: |
    receivers:
      zipkin:
        endpoint: "0.0.0.0:9411"
    exporters:
      alibabacloud_logservice/sls-trace:
        endpoint: "cn-hangzhou-intranet.log.aliyuncs.com"   
        project: "ngiq-cn-pnt-log"               
        logstore: "ngiq-cn-pnt-trace-traces"              
        ecs_ram_role: "pnt-CodeRole"
    service:
      pipelines:
        traces:
          receivers: [zipkin]
          exporters: [alibabacloud_logservice/sls-trace]

---

apiVersion: apps/v1
kind: Deployment
metadata:
  name: otel-collector
spec:
  replicas: 1
  selector:
    matchLabels:
      app: otel-collector
  template:
    metadata:
      labels:
        app: otel-collector
      annotations:
        k8s.aliyun.com/eci-ram-role-name: pnt-CodeRole
        k8s.aliyun.com/eci-image-cache: "true"
    spec:
      volumes:
        - name: otc-internal
          configMap:
            name: otel-collector
            items:
              - key: collector.yaml
                path: collector.yaml
            defaultMode: 420
      containers:
        - name: otel-collector
          image: ngiq-registry.cn-hangzhou.cr.aliyuncs.com/ngiq-cr/opentelemetry-collector-contrib:latest
          args:
            - '--config=/conf/collector.yaml'
          resources:
            limits:
              cpu: '1'
              memory: 2Gi
            requests:
              cpu: '1'
              memory: 1Gi
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
          volumeMounts:
            - name: otc-internal
              mountPath: /conf
---

apiVersion: v1
kind: Service
metadata:
  name: otel-collector
spec:
  selector:
    app: otel-collector
  ports:
  - port: 9411
    targetPort: 9411
  type: ClusterIP

---

apiVersion: autoscaling/v2beta1
kind: HorizontalPodAutoscaler
metadata:
  name: otel-collector
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: otel-collector
  minReplicas: 1
  maxReplicas: 3
  metrics:
  - type: Resource
    resource:
      name: memory
      targetAverageUtilization: 70
```
创建collector:

```bash
kubectl -n pnt apply -f opentelemetry-app.yaml
```



以下内容需要按需变化:

- ConfigMap.data.collector.yaml.exporters.alibabacloud_logservice/sls-trace
  - project: 设置为您在创建Trace实例时所选择的Project
  - logstore: 格式为: {trace_instance_id}-traces
  - ecs_ram_role: 需提前设置role有sls和oss的权限
- Deployment
  - metadata.annotations.k8s.aliyun.com/eci-ram-role-name

# 3. 测试结果
- 开启zipkin的服务

  ```properties
  spring.zipkin.enabled=true
  spring.zipkin.base-url=http://otel-collector:9411/
  ```

  调用几个服务提供的接口然后观察, 例如: get 请求一下 /box/org/detail接口

- 查看结果

	![image-20220801160121197](https://raw.githubusercontent.com/nnsay/gist/main/img/image-20220801160121197.png)


