---
title: 如何本地针对多个Github秘钥并行使用
date: 2019-10-21 11:18:10
tags:
- Tips
- Git
- 工具
excerpt: 解决本地多个GitHub账号配置和使用的一些技巧
---

### 1. 背景介绍
大部分情况下开发者会同时有个人项目和公司项目, 如果公司和个人的仓库都在同一个平台比如GitHub, 这个时候后如何配置可以使得不同的 key 对应不同 GitHub 账号进行使用?

### 2. 解决方案
使用别名来控制 git 在使用 ssh 时加载不同的配置.

### 3. 实现步骤

假设已经针对个人和公司的 GitHub 生成了不同的 key 如下:
```
~/.ssh/id_rsa.pub ## 个人
~/.ssh/neuralgalaxy.pub ## 公司
```

#### 3.1 在~.ssh/config中新建不同的别名:

```
Host github.com
HostName github.com
PreferredAuthentications publickey
IdentityFile ~/.ssh/id_rsa

Host company.github.com
HostName github.com
PreferredAuthentications publickey
IdentityFile ~/.ssh/neuralgalaxy
```
其中第一个为个人使用,第二个为公司使用.

#### 3.2 修改 origin url
每个项目都有 origin, 利用上一步配置的Host别名配置 origin, 这里我修改公司项目的配置,个人配置保持不变, 具体操作如下:
```
cd your-project # a.进入项目根目录

git remote -v # b.查看现有 origin 地址
origin  git@github.com:XXCompany/ngiq-app-server.git (fetch)
origin  git@github.com:XXCompany/ngiq-app-server.git (push)

git remote set-url origin git@company.github.com:XXCompany/ngiq-app-server.git # c. 修改 url
```
从b步骤中可以看到原来的 url 中 domain 是 github.com, 我们需要将其替换为上一步配置的别名, 即: company.github.com

#### 3.3 测试
修改完毕后可以通过 git pull / git push测试, 可以通过 ssh 进行测试, 这里用ssh测试一下 company:
```
ssh -T git@company.github.com
Hi jianwang-beijing! You've successfully authenticated, but GitHub does not provide shell access.
```

### 4. 参考资料
- [Multiple SSH keys for different accounts on Github or Gitlab](https://coderwall.com/p/7smjkq/multiple-ssh-keys-for-different-accounts-on-github-or-gitlab)