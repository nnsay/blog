---
title: volta
date: 2022-05-07 10:52:18
tags:
  - Tips
  - 工具
excerpt: 全局和项目级别的javascrpt工具版本控制工具
---

# 1. [volta 介绍](https://volta.sh/)

volta 是一种 JavaScript 命令行工具管理者工具. 可以管理: node/npm/yarn 三种工具版本. 其特点有:

- 快速

  对比 nvm 来说 volta 还可以管理 yarn, nvm 安装后会在 shell 中增加逻辑,使得 shell 终端开启很慢, volta 采用 shims 的技术不会比 nvm 快速不少

- 可基于不同项目进行版本的无缝切换

  每个项目或者代码仓库可以在 package.json 中配置不同的 node/npm/yarn 版本, 然后在这个项目使用时就会自动切换到对应工具版本使用; 当然 volta 也支持全局默认的 node 版本.

- 跨平台

# 2. volta 使用

## 2.1 安装

```bash
# 通过homebrew
brew install volta
# 通过shell脚本
curl https://get.volta.sh | bash
```

## 2.2 使用

### 2.2.1 查看当前已经安装的工具

```bash
➜  ~ volta list all
⚡️ User toolchain:

    Node runtimes:
        v14.19.2 (default)
        v16.15.0

    Package managers:
```

这里已经安装了两个, 如果是空, 可以使用以下命令安装

### 2.2.2 安装工具

```bash
# 精确工具版本安装
volta install node@14.19.2
# 主版本工具安装
volta install node@14
# 启用volta
➜  ~ volta setup
success: Setup complete. Open a new terminal to start using Volta!
```

**注意:** 每次`install` 除了安装, 也会设置这个版本为默认版本, 即**全局**默认版本, 新建终端即可发现工具新版本是最近一次`install`的版本. 所以如果需要切换全局默认版本也是用 install 命令:

```bash
# 切换全局默认版本
➜  ~ volta install node@16
success: installed and set node@16.15.0 (with npm@8.5.5) as default
```

其次注意执行 setup 命令启用, 启用后可以在$PATH 中看到 volta 的设置, 这也是其启用后的标记:

```bash
➜  ~ echo $PATH
/Users/xxxx/.volta/bin:/...
```

### 2.2.3 禁用 volta

上面 PATH 中 volta 的设置一般在.zshrc/.bashrc 文件中, 所以如果你禁用 volta 使用系统原有的工具, 可以注释掉这个试着, 配置一般如下:

```bash
export VOLTA_HOME="$HOME/.volta"
export PATH="$VOLTA_HOME/bin:$PATH"
```

### 2.2.4 基于项目设置版本

```bash
# 进入项目执行pin, 进行版本绑定
➜  demo-project git:(master) volta pin node@16
success: pinned node@16.15.0 (with npm@8.5.5) in package.json

# 查看变化
➜  demo-project git:(master) ✗ git diff
diff --git a/package.json b/package.json
+  },
+  "volta": {
+    "node": "16.15.0"
   }
 }
```

通过上面命令可以发现 package.json 进行了 node 版本锁定, 之后在这个项目里用到 node 的时候都会使用锁定的版本.

# 3. 总结

volta 对比 nvm 不会对终端启动速度产生影响, 且支持项目级别的工具版本锁定, 这样团队开发时只要每个人装了 volta, 则大家都会使用项目中锁定的版本的工具进行开发, 提高了一致性, volta 是一款非常优秀的工具.

# 4. 文档

- [volta 官方网站](https://volta.sh/)
- [volta Github 地址](https://github.com/volta-cli/volta)
