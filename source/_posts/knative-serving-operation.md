---
title: 阿里云Knative Serving路由控制
date: 2022-01-26 14:35:06
tags:
  - Aliyun
  - DevOps
  - CloudNative
  - 研究
excerpt: Knative的路由是如何工作
---

# 1. Service
- knative有两个主要的功能: serving 和 eventing
- serving主要由有`Service`提供功能
    - service 的创建和变更产生配置 configuration / revision / route
    - route有一个主路由其负载指向一个revision
    - 每个revision可以设置tag
    - 每个revision可以设置负载占比(可以实现:蓝绿/金丝雀/渐进式部署)

## 1.1 方式一: kubctl+yaml
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
      containerConcurrency: 10 # ContainerConcurrency specifies the maximum allowed in-flight (concurrent) requests per container of the Revision
      timeoutSeconds: 1000 # TimeoutSeconds is the maximum duration in seconds that the request routing layer will wait for a request delivered to a container to begin replying (send network traffic)
      containers:
        - image: registry.cn-hangzhou.aliyuncs.com/knative-sample/helloworld-go:160e4dc8
          ports:
            - name: http1
              containerPort: 8080
          env:
            - name: TARGET
              value: "World"
```

## 1.2 方法二: kn

```
$ kn -n demo service create helloworld --image registry.cn-hangzhou.aliyuncs.com/knative-sample/helloworld-go:160e4dc8 --env TARGET=Jian

```

## 1.3 kubectl vs kn
- kubectl + yaml 可以做更细节的设置, 查看 [serving.knative.dev/v1](https://knative.dev/docs/reference/api/serving-api/#serving.knative.dev/v1.Service)
    - containerConcurrency
    - timeoutSeconds
- kn简单方便

<!-- more -->

# 2.  设置Tag

上步骤已经创建了一个serving: helloworld, 更新服务再生成一个revision:
```bash
$ kn -n demo service update helloworld --env TARGET=Jimmy
```

查看所有的revision
```
$ kn -n demo revision list -s helloworld
NAME               SERVICE      TRAFFIC   TAGS   GENERATION   AGE   CONDITIONS   READY   REASON
helloworld-00002   helloworld   100%             2            12h   3 OK / 4     True    
helloworld-00001   helloworld                    1            12h   3 OK / 4     True 

```

为两个revision设置tag

```
$ kn -n demo service update helloworld-go --tag helloworld-go-00001=world --tag helloworld-go-00002=jimmy
```

查看路由详情
```
$ kn -n demo route describe helloworld-go
Name:       helloworld
Namespace:  demo
Age:        4m
URL:        http://helloworld.demo.xxxxxxxx.app.alicontainer.com
Service:    helloworld

Traffic Targets:  
  100%  @latest (helloworld-00002)
    0%  helloworld-00001 #cn
        URL:  http://cn-helloworld.demo.xxxxxxxx.app.alicontainer.com
    0%  helloworld-00002 #en
        URL:  http://en-helloworld.demo.xxxxxxxx.app.alicontainer.com

Conditions:  
  OK TYPE                      AGE REASON
  ++ Ready                     30s 
  ++ AllTrafficAssigned         3m 
  ++ CertificateProvisioned     3m TLSNotEnabled
  ++ IngressReady              30s
```

其中 `Traffic Targets` 是关键, 这里的理论知识如下:
- `@latest`是knative默认增加的tag,在configuration生成和修改时自动指向最新的revision
- 主路由地址: http://<service name>.<namsapce>.xxxxxxxx.app.alicontainer.com
- Tag路由地址: http://<tag>-<service name>.<namsapce>.xxxxxxxx.app.alicontainer.com
- 以上结果如果请求主路由, 100%流量导向 helloworld-00002
- 以上结果如果请求Tag路由, 分别指向具体helloworld-00001(cn)和helloworld-00002(en)

# 3. 设置负载

```
$ kn -n demo service update helloworld --traffic cn=80 --traffic en=20
$ kn -n demo route describe helloworld-go
Name:       helloworld
Namespace:  demo
Age:        13h
URL:        http://helloworld.demo.xxxxxxxx.app.alicontainer.com
Service:    helloworld

Traffic Targets:  
   80%  helloworld-00001 #cn
        URL:  http://cn-helloworld.demo.xxxxxxxx.app.alicontainer.com
   20%  helloworld-00002 #en
        URL:  http://en-helloworld.demo.xxxxxxxx.app.alicontainer.com

Conditions:  
  OK TYPE                      AGE REASON
  ++ Ready                     31s 
  ++ AllTrafficAssigned        13h 
  ++ CertificateProvisioned    13h TLSNotEnabled
  ++ IngressReady              31s
```
当一个revision有自定义tag后默认的@latest tag就没有了, 开发者完全接管负载控制, 下面的操作:
```
# 生成第三个revision
$ kn -n demo service update helloworld --env TARGET=mate
Updating Service 'helloworld' in namespace 'demo':

 15.695s Ready to serve.

Service 'helloworld' updated to latest revision 'helloworld-00003' is available at URL:
http://helloworld.demo.xxxxxxxx.app.alicontainer.com

# 查看路由
$ kn -n demo route describe helloworld
Name:       helloworld
Namespace:  demo
Age:        13h
URL:        http://helloworld.demo.xxxxxxxx.app.alicontainer.com
Service:    helloworld

Traffic Targets:  
   80%  helloworld-00001 #cn
        URL:  http://cn-helloworld.demo.xxxxxxxx.app.alicontainer.com
   20%  helloworld-00002 #en
        URL:  http://en-helloworld.demo.xxxxxxxx.app.alicontainer.com

Conditions:  
  OK TYPE                      AGE REASON
  ++ Ready                      9m 
  ++ AllTrafficAssigned        13h 
  ++ CertificateProvisioned    13h TLSNotEnabled
  ++ IngressReady
```

上面的操作可以发现虽然第三个revision有了, 但是没有分配负载, 使用curl可以验证不会返回`mate`, 从revision列表也可应征:
```
$ kn -n demo revision list -s helloworld
NAME               SERVICE      TRAFFIC   TAGS   GENERATION   AGE     CONDITIONS   READY   REASON
helloworld-00003   helloworld                    3            5m53s   3 OK / 4     True    
helloworld-00002   helloworld   50%       en     2            13h     4 OK / 4     True    
helloworld-00001   helloworld   50%       cn     1            13h     4 OK / 4     True 
```

如果需要revision3接管流量, 需要手动修改:
```
# revision 3 接管全部流量
$ kn -n demo service update --traffic helloworld-00003=100
Updating Service 'helloworld' in namespace 'demo':

  0.043s Ingress has not yet been reconciled.
 15.670s Ready to serve.

Service 'helloworld' with latest revision 'helloworld-00003' (unchanged) is available at URL:
http://helloworld.demo.xxxxxxxx.app.alicontainer.com

# 查看revision列表
NAME               SERVICE      TRAFFIC   TAGS   GENERATION   AGE     CONDITIONS   READY   REASON
helloworld-00003   helloworld   100%             3            8m19s   3 OK / 4     True    
helloworld-00002   helloworld             en     2            13h     3 OK / 4     True    
helloworld-00001   helloworld             cn     1            13h     3 OK / 4     True

# 查看路由
$ kn -n demo route describe helloworld
Name:       helloworld
Namespace:  demo
Age:        13h
URL:        http://helloworld.demo.xxxxxxxx.app.alicontainer.com
Service:    helloworld

Traffic Targets:  
    0%  helloworld-00001 #cn
        URL:  http://cn-helloworld.demo.xxxxxxxx.app.alicontainer.com
    0%  helloworld-00002 #en
        URL:  http://en-helloworld.demo.xxxxxxxx.app.alicontainer.com
  100%  helloworld-00003

Conditions:  
  OK TYPE                      AGE REASON
  ++ Ready                      3m 
  ++ AllTrafficAssigned        13h 
  ++ CertificateProvisioned    13h TLSNotEnabled
  ++ IngressReady               3m
```
