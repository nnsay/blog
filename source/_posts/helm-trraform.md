---
title: Helm结合Trraform使用
date: 2023-02-17 12:57:42
tags: 
- DevOps
- CloudNative
excerpt: helm+terraform实现资源编排
---

# 1. 什么是 [helm](https://helm.sh/)?

kubernetes的包管理器

- 定义/安装/升级 kubernetes应用程序(helm chart)
- 容易创建/发布/分享

# 2. 使用Helm

## 2.1 [安装](https://helm.sh/docs/intro/install/)

- from the homebrew

  ```
  brew install helm
  ```

- for apt

  ```
  curl https://baltocdn.com/helm/signing.asc | sudo apt-key add -
  sudo apt-get install apt-transport-https --yes
  echo "deb https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
  sudo apt-get update
  sudo apt-get install helm
  ```

## 2.2 [使用](https://helm.sh/docs/intro/using_helm/)

- 添加本地本地仓库

  ```
  helm repo add bitnami https://charts.bitnami.com/bitnami
  ```

- 查看仓库中的chart

  ```
  helm search repo bitnami
  ```

- 查看chart信息

  ```
  helm show chart bitnami/mysql
  helm show readme bitnami/mysql
  helm show all bitnami/mysql
  ```

- 安装一个chart

  ```
  helm repo update              # Make sure we get the latest list of charts
  helm install bitnami/mysql --generate-name
  ```

- 列表已安装的chart

  ```
  helm list
  ```

- 查看已安装chart状态

  ```
  helm status mysql-1612624192
  ```

- 删除chart

  ```
  helm uninstall mysql-1612624192
  ```

## 2.3 [结合Terraform使用](https://registry.terraform.io/providers/hashicorp/helm/latest/docs)

helm provider仅有一个resource: helm_release, 可以实现如下的效果:

- 安装chart

- 安装本地chart

- 安装仓库 
  helm provider使用前需要配置kubernetes权限, 这个配置和kubenetes provider的配置类似, 例如:

  ```yaml
  # 使用kubeconfig文件
  provider "helm" {
  kubernetes {
    config_path = "~/.kube/config"
  }
  }
  
  # 使用可执行插件
  
  data "aws_eks_cluster" "main" {
    name = "test-eks-cluster"
  }
  provider "helm" {
    kubernetes {
      host                   = data.aws_eks_cluster.main.endpoint
      cluster_ca_certificate = base64decode(data.aws_eks_cluster.main.certificate_authority[0].data)
      exec {
        api_version = "client.authentication.k8s.io/v1alpha1"
        args        = ["eks", "get-token", "--cluster-name", data.aws_eks_cluster.main.name]
        command     = "aws"
      }
    }
  }
  
  # 使用临时token
  data "aws_eks_cluster_auth" "main" {
    name = "test-eks-cluster"
  }
  provider "helm" {
    host                   = data.aws_eks_cluster.main.endpoint
    token                  = data.aws_eks_cluster_auth.main.token
    cluster_ca_certificate = base64decode(data.aws_eks_cluster.main.certificate_authority[0].data)
  }
  ```

## 2.4 使用案例

```yaml
# 使用chart url
resource "helm_release" "example1" {
  name  = "redis"
  chart = "https://charts.bitnami.com/bitnami/redis-10.7.16.tgz"
}
# 使用本地地址
resource "helm_release" "example2" {
  name       = "my-local-chart"
  chart      = "./charts/example"
}
# 使用chart仓库
resource "helm_release" "example" {
  name       = "my-redis-release"
  repository = "https://charts.bitnami.com/bitnami"
  chart      = "redis"
  version    = "6.0.1"

  values = [
    "${file("values.yaml")}"
  ]
  set {
    name  = "cluster.enabled"
    value = "true"
  }
}
```

# 3. 总结

helm本质上解决了重复代码的问题, 将 mysql/minio/cert manage/dashboard 等诸多常用的kubernetes应用和配置当成包一样的进行管理, 避免了我们去查资料, 拷贝yaml, 修改yaml最后应用yaml的诸多麻烦, 而云厂商例如aws也有自己的chart仓库, 实践中能增加效率, 让开发者主要聚焦到自有应用程序的发布, 而不是疲惫与基础kubernetes环境的搭建.
