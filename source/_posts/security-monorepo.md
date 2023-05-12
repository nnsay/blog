---
title: Monorepo敏感代码保护
date: 2023-05-12 11:30:38
tags:
  - 研究
  - Tips
excerpt: 调研如何针对monorepo做敏感代码保护
---

# 1. 背景介绍

[monorepo](https://monorepo.tools/#what-is-a-monorepo)融合了多个`应用`或者`库`到一个代码仓库中, 但是通常某个开发者仅负责其中几个部分, 一次性拿到整个仓库的代码不安全, 核心代码容易泄露. 本文旨在寻找一种途径, 可以从源码上拆分 monorepo 但不破坏 monoreop 结构, 或者说拆出去的部分任然可以组合成 monorepo, 不丢失 monorepo 其优势.

# 2. 解决方案

分析上面的背景介绍, 单仓库想要控制部分代码的权限, 解决思路就是将敏感的代码块拆分出去, 然后分别管理; 或者有途径可以直接单仓库的部分内容进行权限设置.

- 基于文件夹的代码管理

  - [Git Server Hook](https://git-scm.com/book/zh/v2/%E8%87%AA%E5%AE%9A%E4%B9%89-Git-%E4%BD%BF%E7%94%A8%E5%BC%BA%E5%88%B6%E7%AD%96%E7%95%A5%E7%9A%84%E4%B8%80%E4%B8%AA%E4%BE%8B%E5%AD%90): 指定基于用户的访问权限控制列表（ACL）系统, 需要通过`服务器端钩子`实现

  - 拆分为 multirepo

    - [git submodules(子模块)](https://git-scm.com/book/zh/v2/Git-%E5%B7%A5%E5%85%B7-%E5%AD%90%E6%A8%A1%E5%9D%97)
    - git-subtree

      - [Git subtree: the alternative to Git submodule](https://www.atlassian.com/git/tutorials/git-subtree)
      - [Git Subtree Basics](https://gist.github.com/SKempin/b7857a6ff6bddb05717cc17a44091202)
      - [git-man/git-subtree](https://manpages.debian.org/testing/git-man/git-subtree.1.en.html)

    - [Google repo](https://gerrit.googlesource.com/git-repo): Repo is a tool built on top of Git. Repo helps manage many Git repositories, does the uploads to revision control systems, and automates parts of the development workflow. Repo is not meant to replace Git, only to make it easier to work with Git. The repo command is an executable Python script that you can put anywhere in your path

  - multirepo+自定义集成脚本: 将 monorepo 拆分为 multirepo 分别管理权限, 但是在提服务时通过脚本把 multirepo 组合成 monorepo

# 3. 实践: git subtree

## 3.1 创建 subtree 项目

使用 nx 创建空项目, 然后添加一个应用和一个库项目:

```bash
# 创建空项目
yarn create nx-workspace
cd nx-tree

# 添加项目
git remote add shorturl git@github.com:nnsay/short-url.git
git subtree add --prefix=apps/shorturl shorturl main
git remote add kiosk git@github.com:nnsay/ubuntu-kiosk.git
git subtree add --prefix=libs/kiosk kiosk main

# 提交
git remote add origin git@github.com:nnsay/nx-tree.git
git push origin main

# 测试clone
cd $(mktemp -d)
git clone git@github.com:nnsay/nx-tree.git
tree nx-tree
➜  tmp.vhf7UTm6 tree nx-tree -L 2
nx-tree
├── README.md
├── apps
│   └── shorturl
├── libs
│   └── kiosk
├── nx.json
├── package.json
├── packages
└── yarn.lock
```

## 3.2 subtree 实践

- 更新 sub repo 代码

  ```bash
  git subtree pull --prefix=apps/shorturl shorturl main
  ```

- sub repo 本地修改和提交

  ```bash
  echo $(date) >>apps/shorturl/README.md
  git commit -am 'change 5'
  git subtree push --prefix=apps/shorturl shorturl main
  ```

- sub repo 远程修改本地更新

  ```bash
  git fetch shorturl main
  git subtree merge --prefix=apps/shorturl shorturl/main
  # 或者
  git merge -s subtree shorturl/main
  ```

- 删除已经添加的 sub repo

  - 直接从代码删除目录(prefix)
  - 提交 commit 或者 reset commit

## 3.3 subtree 总结

- subtree 通过添加 remote 的方式添加第三方代码源地址

- subtree add 不会产生.submodule 文件, 而是通过添加 commit 实现的

  ```bash
  commit fc83858fb6cc54ca05aacea44f41fa880730691b (HEAD -> main)
  Merge: 370b3a0 10cb6de
  Author: Jimmy Wang <jimmy.w@aliyun.com>
  Date:   Tue May 9 14:38:20 2023 +0800

      Add 'libs/kiosk/' from commit '10cb6de903b358fb82aaade142846c8301ae8156'

      git-subtree-dir: libs/kiosk
      git-subtree-mainline: 370b3a06a95b67d96193e5669f382432a896a56c
      git-subtree-split: 10cb6de903b358fb82aaade142846c8301ae8156

  commit 370b3a06a95b67d96193e5669f382432a896a56c
  Merge: a1c176d 209c1ca
  Author: Jimmy Wang <jimmy.w@aliyun.com>
  Date:   Tue May 9 14:35:34 2023 +0800

      Add 'apps/shorturl/' from commit '209c1cad3f0ea59d9358e34329a5fb67e0b1cc27'

      git-subtree-dir: apps/shorturl
      git-subtree-mainline: a1c176d09b7ad3a4a61793c1158d245095ef8e42
      git-subtree-split: 209c1cad3f0ea59d9358e34329a5fb67e0b1cc27
  ```

- subtree 项目克隆和单体项目一样

## 3.4 现有项目改造 subtree(_鸡肋_)

```bash
git remote add shorturl git@github.com:nnsay/short-url.git
git subtree split --prefix=apps/shorturl -b shorturl/main
git subtree push --prefix=apps/shorturl shorturl shorturl/main # git push shorturl shorturl/main:main -f
git fetch shorturl shorturl/main
```

> 注意
>
> subtree split 比较鸡肋, 如果无法找到一个合适的 commit 很难仅仅使用 prefix 的部分, 适合空 repo 或者 repo 前期 commit 不太多的时候

# 4. 实践: [git submodules](https://git-scm.com/book/zh/v2/Git-%E5%B7%A5%E5%85%B7-%E5%AD%90%E6%A8%A1%E5%9D%97)

## 4.1 创建 submodules 项目

```bash
# 创建空项目
yarn create nx-workspace
cd nx-tree
git remote add origin git@github.com:nnsay/nx-tree.git

# 添加submodules
git submodule add -b main --name shorturl git@github.com:nnsay/short-url.git ./apps/shorturl
git submodule add -b main --name kiosk git@github.com:nnsay/ubuntu-kiosk.git libs/kiosk

# 提交
git commit -am 'add submodule'
git push
```

## 4.2 submodules 实践

- 主 repo 更新子模块

  ```bash
  # 1. 产生变更
  echo `date` >> apps/shorturl/README.md
  git status

  # 2-1 进入子目录提交变更
  cd apps/shorturl
  git commit -am 'change 1'
  git push
  # 2-2 在主目录提交变更
  git submodule foreach "git commit -am 'new change' || true"
  git submodule foreach "git push || true"

  # 3. 子模块变更提交到主模块
  cd ../../
  git commit -am 'change 1 on main repo'
  git push
  ```

- 子模块远程更新

  子模块升级或者变更后, 主模块引用更新

  ```bash
  # 在shorturl中提交新的commit
  todo
  # 在主模块中更新
  git submodule status
  git submodule update --init --remote --rebase # 或者 git submodule update --init --remote --merge
  git submodule status
  git commit -am 'apply the short url update'
  git push
  ```

  > 注意: 仅仅拉取查看子模块变更:
  > git pull --recurse-submodules

- 克隆

  克隆主 repo, 分别拉取有权限的子模块

  ```bash
  cd $(mktemp -d)
  git clone git@github.com:nnsay/nx-tree.git
  cd nx-tree
  # apps/shorturl和libs/kiosk都是空的
  ls apps/shorturl
  # 获取授权的子模块代码
  git submodule update --init apps/shorturl
  ```

  > 拉取所有子模块代码, 不加路径即可:
  >
  > git submodule update --init 或者 git clone git@github.com:nnsay/nx-tree.git --recurse-submodules

- 修改子模块路径

  ```bash
  # 使用git mv, submodules文件自动更新
  git mv apps/shorurl apps/shorturl
  ```

- 清除 submodules

  ```bash
  git submodule deinit apps/shorturl
  git submodule deinit libs/kiosk
  git commit -am 'remove submodules'
  git push
  ```

- Could not find section in .gitmodules

  ```bash
  git rm --cached libs/kiosk
  git submodule update --init
  ```

- 推送检查

  ```bash
  git push --recurse-submodules=check
  The following submodule paths contain changes that can
  not be found on any remote:
    apps/shorturl

  Please try

          git push --recurse-submodules=on-demand

  or cd to the path and use

          git push

  to push them to a remote.

  # 如果子模块没有推送则推送子模块
  git push --recurse-submodules=on-demand
  # 最后推送主模块
  git push
  ```

## 4.3 submodules 总结

- submodules 是通过`.gitmodules`文件记录子模块

  ```toml
  [submodule "shorturl"]
  	path = apps/shorurl
  	url = git@github.com:nnsay/short-url.git
  	branch = main
  [submodule "kiosk"]
  	path = libs/kiosk
  	url = git@github.com:nnsay/ubuntu-kiosk.git
  	branch = main
  ```

- submodules 本质是 multirepo, 可以在子模块目录更新代码, 最终提交到主目录变更即可

## 4.4 现有项目改造

```bash
cd apps/shorturl
git init
git remote add origin git@github.com:nnsay/short-url.git
rm -rf apps/shorturl
git commit -am 'remote existed apps/shorturl'
git push

git submodule add -b main --name shorturl --force git@github.com:nnsay/short-url.git ./apps/shorturl
git commit -am 'add submodule'
git push
git submodule update --remote -- apps/shorturl
git commit 'apply latest submodule'
git push
```

# 5. 实践: ~~Google Repo~~

## 5.1 安装命令

```bash
curl https://storage.googleapis.com/git-repo-downloads/repo > ~/.local/bin/repo
chmod +x ~/.local/bin/repo
export PATH="${HOME}/.local/bin:${PATH}"
repo --help
```

> 注意
>
> 如果报错: env: python: No such file or directory, 是因为找不到 python, 可以安装 python 或者修改~/.local/bin/repo 的声明行, 例如修改为: \#!/usr/bin/env python3

## 5.2 repo 实践

```bash
# 初始化
mkdir nx-tree  && cd nx-tree
repo init -u git@github.com:nnsay/nx-tree.git

# TODO
```

## 5.3 总结

- google repo 初始化会下载文件, 下载文件走 google 的域名, 国内使用不便, 未来在 CICD 的流水线中也会遇到这个问题
- 文档较少
- init 时必须`default.xm`从[主要文档](https://wladimir-tm4pda.github.io/source/git-repo.html)上看比较适合 Android 开发

综上, 暂不研究.

# 6. 方案总结

- 方案分析

  - 基于 Git Server Hook 的方式最优雅, 但是一般都使用现有的商业代码托管平台如 Github,不能自定义 Git Server Hook, 故该方案不可取
  - 无论哪工具或者解决方案, 文档要规范, 常见问题容易找到解决方案, 故 google repo 工具不可取, 实际上也不适用
  - 不重复造轮子, 尽量不写胶水代码, 使用现有成熟工具, 所以选择: git subtree`和 `git submodules 比较好

- 结论

  从社区资料看 git subtree`和 `git submodules 是成熟的方案, 在所调研的资料做中 substree 作为 submoudles 的一种替代, 在口碑上比较受大家欢迎; submodules 有一些问题, 但是将其看为 multi repo 比较容易理解和使用, 且`.gitmodules`文件也是一种依赖说明. 两者均可作为 Monorepo 的拆分方案.

  从实践上看个人倾向使用 submodules, 因为其子命令`foreach`可以在批量处理多个子项目时提供便利, 例如 Release 时升级版本->提交->推送, 使用 submodules 更简单.

  综合推荐: git submodules
