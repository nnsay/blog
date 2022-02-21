---
title: 阿里云知识点
date: 2021-03-10 17:47:41
tags:
- Aliyun
- Tips
excerpt: 阿里云DevOps技巧列表
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

### 8. KMS
- 获取Secret版本
```
ali kms ListSecretVersionIds --SecretName="/stack/research/db_password"                                                   
{
	"PageNumber": 1,
	"PageSize": 10,
	"RequestId": "e9bf3ff1-d3b1-44e1-abef-96416a52cd7d",
	"SecretName": "/stack/research/db_password",
	"TotalCount": 2,
	"VersionIds": {
		"VersionId": [
			{
				"CreateTime": "2021-05-07T02:13:58Z",
				"VersionId": "v20210507191252",
				"VersionStages": {
					"VersionStage": [
						"ACSPrevious"
					]
				}
			},
			{
				"CreateTime": "2021-05-07T02:15:45Z",
				"VersionId": "v20210510191252",
				"VersionStages": {
					"VersionStage": [
						"ACSCurrent"
					]
				}
			}
		]
	}

```
- 可以使用VersionStage获取最近的Secret值
目前值发现两个VersionStage: ACSCurrent/ACSPrevious
```
ali kms GetSecretValue --SecretName="/stack/research/db_password" --VersionStage=ACSCurrent
ali kms GetSecretValue --SecretName="/stack/research/db_password" --VersionStage=ACSPrevious
```

- 使用Key加密
`keyId`可以是id也可以是alias
```
ali kms Encrypt --KeyId="8da42b24-f1f7-450b-8ccb-f568b1e555fe" --Plaintext=helloworld
ali kms Encrypt --KeyId=alias/NG-Common --Plaintext=helloworld
```
- 解密

```
ali kms Decrypt --CiphertextBlob="ZWZiOTMxMDMtOTUyNi00OGUxLWI5MzEtOTYxZjU0OGUwMWU0T3BDf6PdXvJIHzMKCIxYKr7N3PngT2aS07Rk7TiFx4Me+u0QIMk="
{
	"KeyId": "8da42b24-f1f7-450b-8ccb-f568b1e555fe",
	"KeyVersionId": "efb93103-9526-48e1-b931-961f548e01e4",
	"Plaintext": "helloworld",
	"RequestId": "691ce8ec-f5e5-41dd-bc55-d8a06e2d99e2"
}
```

### 9. 使用digest拉取Docker镜像

image tag如分支名不能精确代表最新的镜像, 可以使用digest, digest使用时用@而不是, 如下:
```
docker pull registry.cn-hangzhou.aliyuncs.com/ngiq-cr/app_server@sha256:c4e1cbe395d87e4c9d94d831c689e7c01531d457f3d1bbbb83d6f1a698fac8e4
```
digest如何获取呢? 在阿里云中可以使用`GetRepoTag`方法:
```
aliyun cr GetRepoTag --RepoName=app_server --RepoNamespace=ngiq-cr --Tag=feature-aliyun
{
	"data": {
		"digest": "7de0f4b43286daef25f2c3ffadb9a78796e9aabde45feb16199528357b731791",
		"imageCreate": 1620436982000,
		"imageId": "de991a1a65f1eda17878cbb326e39e092d135f60bed068a983f750925ad6ce31",
		"imageSize": 346647563,
		"imageUpdate": 1620641463000,
		"status": "NORMAL",
		"tag": "feature-aliyun"
}
```

配合jq则可以快速获取digest值:
```
aliyun cr GetRepoTag --RepoName=app_server --RepoNamespace=ngiq-cr --Tag=feature-aliyun | jq -r '.data.digest'
```

需要注意的是这个值在`docker pull`的时候需要增加`sha256:`


### 10. 获取CLI STS TOKEN
aliyun 有些资源可以通过配置的profile自动获取STS但是有些资源不可以，例如CR、CSK
```
export CRED=$(curl http://100.100.100.200/latest/meta-data/Ram/security-credentials/CICDRole)
export AK=$(echo $CRED | jq -r '.AccessKeyId')
export AK_SEC=$(echo $CRED | jq -r '.AccessKeySecret')
export AK_STS=$(echo $CRED | jq -r '.SecurityToken')
export ALI_CRED="--sts-token=$AK_STS --access-key-id=$AK --mode=StsToken --access-key-secret=$AK_SEC"

aliyun cr $ALI_CRED GetAuthorizationToken | jq -r .data.authorizationToken | docker login --username=cr_temp_user --password-stdin "${PROC_BUILDER_URI}"

```

### 11. 获取account id

```
ali ram GetAccountAlias
{
	"AccountAlias": "1886910356xxxxx",
	"RequestId": "797603B6-F858-48B5-8D63-5DF508CF4713"
}
```

### 12. 同步文件
```
-- 同步 ng-system
mc cp --continue --recursive s3/ng-system/ oss/ng-system/
-- 比较
mc diff s3/ng-system/ oss/ng-system/

```

### 13. 命令行 ECR
个人版本的CR存在限流问题, 当个人仓库流量到达一定的限制, 触发限流, 当值镜像拉取很慢, 此时应该开启企业版CR即ECR, 当时阿里云CLI CR默认是没有对ECR做支持的, 但是可以通过`force`和`version`使用OpenAPI, 进而达到管理企业版CR的目的:
```
#获取实例列表
ali cr ListInstance --force --version 2018-12-01
#获取仓库列表
ali cr ListRepository --InstanceId=cri-xxxx--RepoStatus=NORMAL --version 2018-12-01 --force | jq -r '.Repositories[]|select(.RepoNamespaceName=="ngiq-prod-cr")|{RepoId,RepoNamespaceName,RepoName}'
```