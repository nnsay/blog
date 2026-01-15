---
title: GitHub Actions 使用 OIDC 安全访问 AWS 指南
date: 2026-01-15 18:29:15
tags:
  - AWS
  - DevOps
excerpt: 告别 AK/SK，在流水线中使用 OIDC 安全访问 AWS
---

# 声明

本文由 🤖AI 协作完成, 内容已过实际测试.

## 1. 前言

在 CI/CD 流水线中访问 AWS 资源时，传统的做法是创建一个 IAM User，生成 Access Key ID 和 Secret Access Key (AK/SK)，然后将其存储在 GitHub Secrets 中。

这种做法存在显著的安全隐患：

- **密钥泄露风险**：如果 Secrets 管理不当或代码泄露，长期有效的密钥可能被盗用。
- **管理成本高**：需要定期轮换密钥，维护麻烦。

**更好的解决方案是使用 OpenID Connect (OIDC)。** 通过配置 OIDC，GitHub Actions 可以在运行时向 AWS 申请**临时的**访问凭证。这是一种“无密钥 (Keyless)”的安全实践，完全避免了长期凭证的存储问题。

本文将详细介绍如何配置 AWS 和 GitHub Actions 来实现这一目标。

## 2. AWS 端配置：建立信任

我们需要在 AWS IAM 中配置一个“信任锚点”，告诉 AWS：“我信任 GitHub 的 OIDC 服务，并允许特定的 GitHub 仓库扮演我的 IAM 角色”。

### 第一步：创建 OIDC 身份提供商 (Identity Provider)

1.  登录 AWS 控制台，进入 **IAM** 服务。
2.  在左侧菜单点击 **Identity providers** (身份提供商)，点击 **Add provider**。
3.  配置如下信息：
    - **Provider type**: 选择 `OpenID Connect`。
    - **Provider URL**: 输入 `https://token.actions.githubusercontent.com` (必须完全一致，不要带多余的斜杠)。
    - **Audience**: 输入 `sts.amazonaws.com`。
4.  点击 **Get thumbprint** (获取指纹)，AWS 会验证证书。
5.  点击 **Add provider** 完成创建。

### 第二步：创建 IAM 角色 (IAM Role)

接下来，创建一个赋予 GitHub Action 权限的角色。

1.  在 IAM 控制台，点击 **Roles** -> **Create role**。
2.  **Trusted entity type** 选择 `Web identity`。
3.  **Identity provider** 选择上一步创建的 `token.actions.githubusercontent.com`。
4.  **Audience** 选择 `sts.amazonaws.com`。
5.  **GitHub organization/repository**：此处可先跳过，我们稍后通过策略精确控制。
6.  点击 **Next**，添加你需要的权限（例如 `AmazonS3FullAccess` 或自定义策略）。
7.  给角色起个名字（例如 `GitHubAction-DeployRole`），点击创建。

### 第三步：配置信任策略 (Trust Policy) —— **关键步骤**

为了防止其他人的 GitHub 仓库冒充你的身份，我们需要严格限制信任范围。

1.  进入刚创建的角色详情页，点击 **Trust relationships** 标签页。
2.  点击 **Edit trust policy**。
3.  将内容修改为以下 JSON。

> **注意**：如果你使用的是 **AWS 中国区**（如 `cn-northwest-1`），ARN 的前缀是 `arn:aws-cn`；如果是全球区，则是 `arn:aws`。请根据实际情况修改。

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        // 如果是全球区，使用 arn:aws:iam::...
        // 如果是中国区，使用 arn:aws-cn:iam::...
        "Federated": "arn:aws-cn:iam::YOUR_AWS_ACCOUNT_ID:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          // 🔒 核心安全限制：仅允许特定仓库的特定分支
          // 格式：repo:<OrgName>/<RepoName>:ref:refs/heads/<BranchName>
          "token.actions.githubusercontent.com:sub": "repo:YOUR_GITHUB_ORG/YOUR_REPO_NAME:ref:refs/heads/main"
        }
      }
    }
  ]
}
```

**配置说明：**

- `sub` (Subject) 字段决定了谁有权限。
- 允许特定分支：`repo:org/repo:ref:refs/heads/main`
- 允许所有分支：`repo:org/repo:*`
- 允许特定 Tag：`repo:org/repo:ref:refs/tags/*`
- 允许组织下所有: repo:org/\*

## 3. GitHub 端配置：编写流水线

### 第一步：准备 Role ARN

获取刚才创建的 IAM 角色 ARN，例如 `arn:aws-cn:iam::123456789012:role/GitHubAction-DeployRole`。
建议将其放入 GitHub 仓库的 **Variables** 或 **Secrets** 中，或者直接在 env 中定义。

### 第二步：编写 Workflow YAML

在 `.github/workflows/xxx.yml` 中，有两个关键点：

1.  **Permissions**: 必须声明 `id-token: write`，否则无法申请 OIDC 令牌。
2.  **Action**: 使用 `aws-actions/configure-aws-credentials`。

```yaml
name: AWS OIDC Deploy

on:
  push:
    branches:
      - main

# 【关键配置 1】申请 OIDC 令牌必须的权限
permissions:
  id-token: write
  contents: read # 显式声明读取代码权限，符合最小权限原则

env:
  # 注意：AWS 中国区 Region 为 cn-northwest-1 或 cn-north-1
  AWS_DEFAULT_REGION: cn-northwest-1
  # 注意：中国区 ARN 以 arn:aws-cn 开头
  AWS_DEPLOY_ROLE_ARN: arn:aws-cn:iam::123456789012:role/GitHubAction-DeployRole

jobs:
  aws-access:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      # 【关键配置 2】交换临时凭证
      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ env.AWS_DEPLOY_ROLE_ARN }}
          aws-region: ${{ env.AWS_DEFAULT_REGION }}
          # role-session-name: GitHubActions (可选)
          # role-duration-seconds: 3600    (可选，默认 1 小时)

      # 验证身份
      - name: Verify AWS identity
        run: aws sts get-caller-identity

      # 执行业务操作 (示例：列出 S3)
      - name: List S3 buckets
        run: aws s3 ls
```

## 4. 原理解析

为什么这套流程是安全的？让我们通过流程图来看一下幕后发生了什么：

1.  **启动**：GitHub Action 启动，检测到 `permissions: id-token: write`。
2.  **颁发令牌**：GitHub OIDC Provider 生成一个 JWT 令牌，其中包含当前仓库的元数据（如 `sub: repo:user/repo:ref:refs/heads/main`）。
3.  **请求交换**：`configure-aws-credentials` Action 拿着这个 JWT 令牌去请求 **AWS STS** 服务。
4.  **校验**：AWS STS 进行双重检查：
    - **验签**：检查令牌是否由信任的 GitHub IdP 签发。
    - **鉴权**：检查 IAM Role 的 Trust Policy 中的 `Condition` 是否匹配令牌中的 `sub` 信息。
5.  **授权**：如果匹配，AWS 返回临时的 Access Key、Secret Key 和 Session Token。
6.  **执行**：GitHub Action 使用这些临时凭证在有效期内操作 AWS 资源。

## 5. 常见问题 (FAQ)

- **Q: 报错 `Not authorized to perform sts:AssumeRoleWithWebIdentity`？**

  - A: 检查 IAM Role 的 Trust Policy 中 `sub` 字段是否拼写正确，GitHub 组织名和仓库名区分大小写。
  - A: 检查 `provider` 的 ARN 是否正确（注意全球区 `aws` 和中国区 `aws-cn` 的区别）。

- **Q: 临时凭证有效期是多久？**

  - A: 默认 1 小时。可以通过 `role-duration-seconds` 参数调整，范围是 15 分钟到 12 小时。注意 IAM Role 本身也有一个“最大会话时长”设置，请求的时长不能超过该设置。

- **Q: AWS 中国区有什么特别注意的？**
  - A: 主要是 ARN 格式（`arn:aws-cn`）和 Region 代码（`cn-northwest-1` / `cn-north-1`）。OIDC Provider URL 依然是全球通用的 `token.actions.githubusercontent.com`。

通过以上步骤，你就可以优雅地移除 GitHub Secrets 中的 AK/SK，实现更安全、更合规的云端部署流水线。
