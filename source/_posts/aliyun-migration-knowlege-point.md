---
title: 阿里云知识点
date: 2021-03-10 17:47:41
tags:
- Aliyun
- 技巧
---

### 1. 使用临时token登录阿里云CR

```
aliyun cr GetAuthorizationToken | jq .data.authorizationToken | tr -d '"' | docker login --username=cr_temp_user --password-stdin registry.cn-hangzhou.aliyuncs.com
```

### 2. 获取Kubernetes集群的kuberctl配置文件
```
aliyun cs DescribeClusterUserKubeconfig --ClusterId=c90bf856c68364386a07384a6a2bb176b
```

### 3. 使用use-context切换管理多个kubernetes集群
  
多个集群kubectl配置如下:
```
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: DATA+OMITTED
    server: https://192.168.132.236/k8s/clusters/c-8s9dc
  name: ng-demo
- cluster:
    certificate-authority-data: DATA+OMITTED
    server: https://192.168.132.236:6443
  name: ng-demo-server01
contexts:
- context:
    cluster: ng-demo
    user: ng-demo
  name: ng-demo
- context:
    cluster: ng-demo-server01
    user: ng-demo
  name: ng-demo-server01
current-context: ng-demo
kind: Config
preferences: {}
users:
- name: ng-demo
  user:
    token: REDACTED
```

切换命令如下:
```
$ kubectl config use-context ng-demo
Switched to context "ng-demo".
```

### 4. 设置OSS生命周期

Terraform目前(2021-3-10)不支持对于分片文件的生命周期, 可以考虑通过阿里云CLI实现.
```bash
# get and view the bucket lifecycel
aliyun oss lifecycle --method get oss://research-ng-temp
# set the lifecycle
aliyun oss lifecycle --method put oss://research-ng-temp lifecycle.xml
```

上面命令中的lifecycle.xml如下:
```xml
<?xml version="1.0" encoding="UTF-8"?>
  <LifecycleConfiguration>
      <Rule>
          <ID>857e37e3-0a3e-4250-a408-88f5b35f9714</ID>
          <Prefix></Prefix>
          <Status>Enabled</Status>
          <Expiration>
              <Days>16</Days>
          </Expiration>
          <AbortMultipartUpload>
              <Days>16</Days>
          </AbortMultipartUpload>
      </Rule>
  </LifecycleConfiguration>
```

### 5. ICP

- [ICP接入](https://help.aliyun.com/knowledge_detail/36924.html): ICP可以同别的平台迁移到阿里云, 这个过程比在做一次ICP快很多

### 6. [免费DV证书](https://help.aliyun.com/document_detail/156645.html)

阿里云每个实名账号个人或企业主体在一年内可以有20个免费DV证书可以使用, 每个证书仅支持绑定一个具体的域名, 20个的数量可以支持常见的测试和验证需求. 这种证书可以用于:
- CDN自定义域名
- SLB启用HTTPS