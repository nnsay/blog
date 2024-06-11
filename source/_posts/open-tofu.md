---
title: open tofu
date: 2024-06-11 14:54:13
tags:
  - DevOps
excerpt: 使用tofu替换terraform入门指导
---

# 1. 什么是 ToFu

tofu 是 terraform 的一个开源分支, 因为在 2023 年 8 月 10 日 terraform 的许可从 Mozilla Public License (v2.0) (the “MPL”)变为了 Business Source License (v1.1) (the “BUSL”), 新的许可对商业不友好, 所以社区创建了 tofu, 详情可以查看[这里](https://opentofu.org/manifesto/). 我们私有化部署也将采用 tofu 在 kubernetes 进行服务的部署和维护.

本文不介绍太详细的理论知识, 要学习了解细节的同学可先查看[官网文档](https://opentofu.org/docs/). 本文针对有 terraform 经验的同学, 学习替换 terraform 到 open tofu 如何进行开发, 涉及以下内容:

- 主模块配置
- Provider
- 状态存储
- 模块化
- 技巧

# 2. 工程实践

## 2.1 状态

要实现 IaC 状态文件肯定是不能缺少, 否则就不可能产生对比, 进而产生对资源的增删改操作, 最简单的状态是将状态文件存在本地, 但是本地容易丢失, 如果部署多个对象是本地就要维护个状态文件, 所以这里我们采用将状态文件保存在远程, tofu 支持 kubernetes 作为 backend 所以我们将状态文件实际上保存在了 kubernetes 中.

```yaml
terraform {
  backend "kubernetes" {
  secret_suffix = "state"
  namespace     = "nnsay-com"
  config_path   = "~/.kube/config"
  }
}
```

## 2.2 初始化

因为要保存状态且我们的服务也需要一个命名空间, 这里约定我们的命名空间是: nnsay-com, 所以提前创建好命名空间

```bash
kubectl create namespace nnsay-com
```

在第一次或者一个新的环境中首先需要初始化, 初始化会下载 provider 和初始化状态, 这个操作可以频繁执行

```bash
export TF_VAR_image_registry=$(jq -c -n --arg password $GITHUB_TOKEN '{server:"ghcr.io",username:"cicd",password:$password,email:"cicd@visualdynamics.cn"}')

tofu init -var-file="local.tfvars"
```

初始化之后就可以查看状态了, 按照如上的配置 kubernetes 中的状态是:

```bash
kubectl get secret tfstate-default-state  -n nnsay-com -o jsonpath='{.data.tfstate}' | base64 --decode | gunzip

{
  "version": 4,
  "terraform_version": "1.7.2",
  "serial": 1,
  "lineage": "00318f9b-0f0e-2b46-8bea-a0e24978403d",
  "outputs": {},
  "resources": [],
  "check_results": null
}
```

## 2.3 模块化实践

模块化是常见的抽象方式, 也便于并行开发和协作, 以下是模块化方式和文件命名都是按照个人实践经验总结, 开发者可参考这个结构和命名规范, 不做强制要求.

- root: 根模块就是在根目录配置了`terraform`的.tf 文件所在的地方
  - main.tf: 语义上的入口文件, 在其中配置`terraform`, 核心配置如下
    - backend: 定义状态文件保存位置
    - required_providers: 定义 providers
  - output.tf: 定义主模块的输出
  - variable.tf: 定义主模块变量
  - \*.tfvars: 定义变量, 例如 local.tfvars, 使用 -var-file 指定
  - \*.auto.tfvars: 自动加载变量
- 根目录中可以调用子模块
  - \*-resource.tf: 定义模块下具体的资源
  - output.tf: 定义主模块的输出
  - variable.tf: 定义主模块变量

结合上面的模块化说明, 举例说明一个还有两个子模块的 main.tf 代码如下:

```yaml
locals {
  k8s_namespace              = "nnsay-com"
  image_registry_secret_name = "regcred"
}

terraform {
  backend "kubernetes" {
    secret_suffix = "state"
    namespace     = "nnsay-com"
    config_path   = "~/.kube/config"
  }
  required_providers {
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "2.30.0"
    }
  }
}

provider "kubernetes" {
  config_path = "~/.kube/config"
}

module "common" {
  source = "./common"

  namespace_name             = local.k8s_namespace
  image_registry_secret_name = local.image_registry_secret_name
}

module "serving" {
  source = "./serving"

  namespace_name = local.k8s_namespace
  secret_name    = module.common.secret_name # 这里有隐式的模块依赖关系, serving依赖common
}
```

# 3. 经验技巧

- 已经初始化了希望重新初始化重

  ```bash
  tofu init -reconfigure
  ```

- 定义复杂类型的变量

  ```yaml
  variable "image_registry" {
  type = object({
  server   = string
  username = string
  password = string
  email    = string
  })
  description = "docker image registry credentials"
  default = {
  server   = "ghcr.io"
  username = "cicd",
  password = "",
  email    = "docker@visualdynamics.cn"
  }
  sensitive = true
  }
  ```

- 定义复杂类型的输出

  ```yaml
  output "secret_name" {
  value = {
  regcred        = kubernetes_secret.regcred.metadata[0].name
  awslocal       = kubernetes_secret.awslocal.metadata[0].name
  common         = kubernetes_secret.common.metadata[0].name
  apikey         = kubernetes_secret.apikey.metadata[0].name
  database       = kubernetes_secret.database.metadata[0].name
  object_storage = kubernetes_secret.object_storage.metadata[0].name
  bucket         = kubernetes_secret.bucket_name.metadata[0].name
  lambda         = kubernetes_secret.lambda_name.metadata[0].name
  service        = kubernetes_secret.service_name.metadata[0].name
  }
  description = "all secret name object which is used in container environment variables"
  }
  ```

- 查看执行计划

  ```bash
  tofu plan -var-file="local.tfvars"
  ```

- 跳过交互性批准

  ```bash
  tofu apply -auto-approve -var-file="local.tfvars"
  ```

- 自动加载变量

  - 使用 .auto.tfvars 结尾的文件定义变量
  - 使用`terraform.tfvars`文件定义变量

  > [!NOTE]
  >
  > tofu 加载变量的顺序:
  >
  > - Environment variables
  > - The `terraform.tfvars` file, if present.
  > - The `terraform.tfvars.json` file, if present.
  > - Any `*.auto.tfvars` or `*.auto.tfvars.json` files, processed in lexical order of their filenames.
  > - Any `-var` and `-var-file` options on the command line, in the order they are provided. (This includes variables set by an HCP Terraform workspace.)

- 强制解锁

  锁的 ID 一般会在提示信息中指出:

  ```
  ╷
  │ Error: Error acquiring the state lock
  │
  │ Error message: the state is already locked by another tofu client
  │ Lock Info:
  │   ID:        81b32749-3f9d-eadc-bc16-c81278ca9414
  │   Path:
  │   Operation: OperationTypeApply
  │   Who:       wangjian@JimmyOfficeMacBookPro.local
  │   Version:   1.7.2
  │   Created:   2024-06-07 06:08:14.68546 +0000 UTC
  │   Info:
  │
  │
  │ OpenTofu acquires a state lock to protect the state from being written
  │ by multiple users at the same time. Please resolve the issue above and try
  │ again. For most commands, you can disable locking with the "-lock=false"
  │ flag, but this is not recommended.
  ╵
  ```

  通过下面的命令强制解锁:

  ```bash
  tofu force-unlock 81b32749-3f9d-eadc-bc16-c81278ca9414
  ```
