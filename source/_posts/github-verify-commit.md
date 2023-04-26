---
title: Github提交验证
date: 2023-04-26 17:37:39
tags:
  - Tips
excerpt: 开启提交验证让你的代码更被信任
---

# 1. 什么提交验证

一言以蔽之: 验证提交的代码是作者写的. 因为`git` 允许配置`user` 的`name`和`email`如果 A 用户和 B 用户都拥有对仓库 R 的提交权限, 那么 A 可以把其`user`配置为 B 的`user` 的`name`, 这时就不知道某个`commit`到底是谁提交的, 从另一个方面说也无法证明某些代码是作者写的, 而提交验证正是这么一个措施去验证代码提交作者的身份. 如下图:

![image-20230426170220144](https://raw.githubusercontent.com/nnsay/gist/main/imgimage-20230426170220144.png)

第一个`77f22aa`是一个验证过的, 状态是: `Verified`, 而`cdcf74d`是未验证状态: `Unverified`

# 2. 如何实现提交验证

在这里仅说明`SSH`提交验证的实现, 除此之外 Github 还支持[更多验证方式](https://docs.github.com/zh/authentication/managing-commit-signature-verification/about-commit-signature-verification). 具体的实现步骤如下:

- 开启`git`提交验证

  ```
  git config --global commit.gpgsign true
  git config --global gpg.format ssh
  git config --global user.signingkey ~/.ssh/id_ed25519.pub
  ```

  > 注意
  >
  > user.signingkey 根据个人实际情况进行配置

- [上传签名公钥](https://docs.github.com/zh/authentication/connecting-to-github-with-ssh/adding-a-new-ssh-key-to-your-github-account)

  ```
  # 拷贝公钥到剪切板
  pbcopy < ~/.ssh/id_ed25519.pub
  ```

  - 设置`Vigilant mode` , 该步骤可选

    登录 Github, 进入`SSH and GPG Keys`页面(https://github.com/settings/keys), 勾选下面的标记

    - [x] Flag unsigned commits as unverified

  - 新建 Key

    在当前页面点击`New SSH Key`或者直接访问: https://github.com/settings/ssh/new 这个链接进入新建 Key 页面, 填写如下信息:

    - Title: Key 标题, 通过标题表达这个 Key 的用户

    - Key type: 选择`Signing Key`

    - Key: 粘贴拷贝的公钥内容即可

    最后`Add SSH Key`即可.

  > 注意
  >
  > Key type 一定要选择 Signing Key 而不是 ~~Authentication Key~~

# 3. 测试

因为上一步配置了`commit.gpgsign` `gpg.format` `user.signingkey`, 所以验证就很简单了, 签名会在 commit 的时候自动进行.

```bash
touch temp.log
git add .
git commit -m 'test verify commit'
git push
```

push 代码后在 Gtihub 上对应的仓库中查看 commit 列表中查看 commit 的验证状态即可, 如果没有设置`Vigilant mode`则以前为做签名验证的不显示`Unverified`标签, 否则会显示.
