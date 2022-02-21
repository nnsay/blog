---
title: 阿里云Knative Serving一撇
date: 2022-01-21 16:36:20
tags:
  - Aliyun
  - DevOps
  - CloudNative
  - 研究
excerpt: 在阿里云创建Knative服务并练手
---

## 1. [搭建Knative环境](https://help.aliyun.com/document_detail/185057.html)

### 1.1 创建服务
- 方式一: 创建一个ASK/ACK专有版集群, 创建时开启Knative选项
- 方式二: 现有ASK/ACK专有版集群上安装Knative组件
    - 部署Knative-Serving组件
    - 部署Knative-Eventing组件
 
搭建Knative后在阿里云上会得到两个东西:
- 访问网关, 是个IP
- 公网测试域名, 例如: *.xxxxxxxx.app.alicontainer.com

### 1.2 安装CLI
- kubectl
- kn
- hey

以上都可以使用`brew`安装

<!-- more -->

## 2.  Hello World
- 准备服务文件helloworld-service.yaml
```yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: helloworld-go
spec:
  template:
    metadata:
      labels:
        app: helloworld-go
      annotations:
        autoscaling.knative.dev/target: "10"
    spec:
      containers:
        - image: registry.cn-hangzhou.aliyuncs.com/knative-sample/helloworld-go:160e4dc8
          ports:
            - name: http1
              containerPort: 8080
          env:
            - name: TARGET
              value: "World"
```
- 创建knative服务
```bash
$ kubectl create namespace demo
namespace/demo created
$ kubectl apply -n demo -f helloword-service.yaml
service.serving.knative.dev/helloworld-go created
```
- 查看knative服务
```bash
$ kn -n demo service list
NAME            URL                                                                  LATEST   AGE   CONDITIONS   READY     REASON
helloworld-go   http://helloworld-go.demo.xxxxxxxx.app.alicontainer.com            10s   0 OK / 3     Unknown   RevisionMissing : Configuration "helloworld-go" is waiting for a Revision to become ready.
$ kubectl -n demo get pod -w
NAME                                                      READY   STATUS        RESTARTS   AGE
helloworld-go-00001-deployment-7f7d75c465-88n4h           1/2     Terminating   0          96s
helloworld-go-00001-deployment-reserve-7989bd95cf-2cn4r   2/2     Running       0          36s
helloworld-go-00001-deployment-7f7d75c465-88n4h           0/2     Terminating   0          110s
helloworld-go-00001-deployment-7f7d75c465-88n4h           0/2     Terminating   0          111s
helloworld-go-00001-deployment-7f7d75c465-88n4h           1/2     Terminating   0          114s
helloworld-go-00001-deployment-7f7d75c465-88n4h           1/2     Terminating   0          2m11s
helloworld-go-00001-deployment-7f7d75c465-88n4h           1/2     Terminating   0          2m11s
$ kubectl -n demo get service 
NAME                                  TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)                                      AGE
helloworld-go                         ClusterIP   None           <none>        80/TCP                                       32m
helloworld-go-00001                   ClusterIP   172.19.3.27    <none>        80/TCP                                       32m
helloworld-go-00001-private           ClusterIP   172.19.5.43    <none>        80/TCP,9090/TCP,9091/TCP,8022/TCP,8012/TCP   32m
helloworld-go-00001-reserve-private   ClusterIP   172.19.7.159   <none>        80/TCP,9090/TCP,9091/TCP,8022/TCP,8012/TCP   32m
```

  从上面的检查命令可以得知: 一个服务会有一个常驻的reserve pod和若干server pod, 峰谷时server pod降低为0, 当有请求时server pod增加, 奇怪的是从0请求到1请求时reserve pod也进行了重建:
  ![service pod](https://gitee.com/nnsay/imagehost/raw/master/vnotebooks/default/it/study/knative%20on%20aliyun.md/114815115268392.png)

- 测试knative服务
```
$ curl -i -XGET http://helloworld-go.demo.xxxxxxxx.app.alicontainer.com
HTTP/1.1 200 OK
Date: Fri, 21 Jan 2022 07:27:19 GMT
Content-Type: text/plain; charset=utf-8
Content-Length: 13
Connection: keep-alive

Hello World
```

- 压测
```bash
$ hey -z 30s -c 50 http://helloworld-go.demo.xxxxxxxx.app.alicontainer.com/ && kubectl -n demo get pod 
Summary:
  Total:	30.0857 secs
  Slowest:	0.6134 secs
  Fastest:	0.0360 secs
  Average:	0.0622 secs
  Requests/sec:	802.3732
  
  Total data:	313820 bytes
  Size/request:	13 bytes

Response time histogram:
  0.036 [1]	|
  0.094 [21158]	|■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■
  0.151 [2731]	|■■■■■
  0.209 [84]	|
  0.267 [46]	|
  0.325 [22]	|
  0.382 [0]	|
  0.440 [46]	|
  0.498 [4]	|
  0.556 [5]	|
  0.613 [43]	|


Latency distribution:
  10% in 0.0406 secs
  25% in 0.0435 secs
  50% in 0.0510 secs
  75% in 0.0724 secs
  90% in 0.0961 secs
  95% in 0.1023 secs
  99% in 0.1580 secs

Details (average, fastest, slowest):
  DNS+dialup:	0.0001 secs, 0.0360 secs, 0.6134 secs
  DNS-lookup:	0.0000 secs, 0.0000 secs, 0.0199 secs
  req write:	0.0000 secs, 0.0000 secs, 0.0020 secs
  resp wait:	0.0620 secs, 0.0358 secs, 0.6134 secs
  resp read:	0.0001 secs, 0.0000 secs, 0.0037 secs

Status code distribution:
  [200]	24140 responses



NAME                                              READY   STATUS    RESTARTS   AGE
helloworld-go-00001-deployment-7f7d75c465-fl4g5   2/2     Running   0          82s
helloworld-go-00001-deployment-7f7d75c465-xdr5k   2/2     Running   0          60s
```
期望看到pod是5, 因为yaml文件中`autoscaling.knative.dev/target`是10, 但是这里是2, 需要特别注意的是reserve pod没有在最后的结果里, 猜测在流量持续稳定时reserve pod没有存在意义, 所以reserve pod情况是:
- 没有流量: 仅有reserve pod
- 有少许流量: reserve pod 和 service pod 都有, 但是 reserve pod总试图退出
- 持续刘流量: 没有reserve pod, 此时没有意义
综上reserve pod是波峰到波谷是最后一个用于反馈下一次流量新建service pod的一种守护pod. 这种机制是knavite工作的重点, 其中serving组件的工作原理如下图:
![components of serviing](https://gitee.com/nnsay/imagehost/raw/master/vnotebooks/default/it/study/knative%20on%20aliyun.md/35640196825920.png)
从0流量到1流量的工作细节如下:
![0-1](https://gitee.com/nnsay/imagehost/raw/master/vnotebooks/default/it/study/knative%20on%20aliyun.md/396041405615012.png)


## 3. 帮助手册
### 3.1  开启集群PrivateZone
```
# 编辑eci-profile文件
kubectl -n kube-system edit configmap eci-profile
# 修改 `enablePrivateZone` 为 `true`
enablePrivateZone: "true"
```

### 3.2 卸载Knative
- 在集群卸载Knative组件
- 恢复`eci-profile`文件`enablePrivateZone`属性为`false`

## 4. 参考文件
1. [Knative概述](https://help.aliyun.com/document_detail/184831.html)
2. Serverless容器集群ASK [Knative](https://help.aliyun.com/document_detail/184830.html)
3. [Knative云原生应用开发指南](https://knative-sample.com/)