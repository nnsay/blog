---
title: 阿里云知识点
date: 2021-03-10 17:47:41
tags:
- Aliyun
- Tips
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

### 7. 第三方账号证书导入
阿里云支持上传证书, 可以将一个已经在其他平台和账号的证书上传到阿里云, 操纵步骤:
- 下载购买证书, 如果在阿里云购买的话, 下载的时候有很多种下载方式, 选择 nginx 下载
![cert download](https://gitee.com/nnsay/public/raw/master/1618555774_20210416144918809_2123515414.png)
- 解压下载文件, 可以获得.pem(certificate)和.key(provide key)文件
- 上传, 在需要证书的阿里云账号的证书管理控制台, 粘贴复制.pem和.key的内容
- 使用, 上传完后证书即可以像其他购买的证书一样正常使用, 例如配置OSS自定义域名,代码15和16行
```
resource "alicloud_cdn_domain_new" "docker_storage_domain" {
  domain_name = local.ng_docker_file_domain
  cdn_type    = "download"
  scope       = "domestic"

  sources {
    content = local.docker_storage_bucket_domain
    type    = "oss"
    port    = 443
  }

  certificate_config {
    server_certificate_status = "on"
    force_set                 = 1
    cert_name                 = local.cert_name
    cert_type                 = "upload"
  }

  tags = {
    Name  = "${var.stack_name}-docker-storage-domain"
    Stack = var.stack_name
  }

  depends_on = [
    alicloud_oss_bucket.docker_storage
  ]
}
```