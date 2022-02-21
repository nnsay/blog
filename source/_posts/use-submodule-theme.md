---
title: Git子模块妙用
date: 2021-09-15 14:05:41
tags:
- Tips
excerpt: 如何使用子模块的方式管理代码依赖, 以hexo博客主题为例
---

## 1. 背景

本站点使用hexo搭建, hexo提供了很多主题, 我选取的是一个叫做: [concise](https://github.com/sanonz/hexo-theme-concise)的主题. 根据这个主题作者的readme, 需要`git clone`这个主题, 然后放到自己博客的`themes`目录下然后进行主题应用. 这带了一个问题, 为了远程打包我不得不把这个主题代码当成我的博客代码加入版本库提交到远端. 但是问题是如果主题作者更新了其主题我将不能很方便的获取最新的主题, 笨办法就是删除当前主题目录`themes/concise`然后重复第一次加载主题的步骤, 这显然不够优雅. 这问题本质是给代码复用的问题, 我看到git子模块这个解决方案, 尝试之后发现这正是我寻找的方案, 分享出来希望大家可以借鉴.

## 2. [Git子模块](https://git-scm.com/book/zh/v2/Git-%E5%B7%A5%E5%85%B7-%E5%AD%90%E6%A8%A1%E5%9D%97)的一些理论

## 3. 如何使用Git子模块解决主题问题

- 添加适合的主题
```
git submodule add https://github.com/sanonz/hexo-theme-concise.git ./themes/concise
```

- 配置主题的分支(版本)
```
git config -f .gitmodules submodule.themes/concise.branch master
```

- 配置git pull自动拉取子模块
```
git config submodule.recurse true
```

- 忽略子模块更新(因为本地调试时需要替换图片)
```
git config -f .gitmodules submodule."themes/concise".ignore dirty 
```

除此之外, 需要把配置安装到主题中, 主题配置放在themes目录根目录, 然后copy到主题中, 所以修改package script如下:
```
"scripts": {
    "build": "npm run cp && hexo generate",
    "clean": "hexo clean",
    "deploy": "npm run cp && hexo deploy",
    "server": "npm run cp && hexo server",
    "cp": "cp themes/_config.yml themes/concise/ && cp images/* themes/concise/source/images"
  },
```

## 3. 总结

使用git子模块能很好的复用第三方的代码, 且自己的代码不会对其侵入操作, 实现了在一个项目总的协同工作. 在我例子中子模块作为我应用主题代码的一种方式, 一旦我运行`git pull`则子模块也获得了更新, 是我可以快速应用主题, 这种方式非常优雅.

这个例子也为项目集成和协作提供了一个思路: 使用子模块作为包(通用代码)管理, 把共用的代码都通过子模块的方式进行管理. 当然这里有一些思考:
- 项目如果有100个包(通用代码)要做100个子模块?
- 如果子模块引入的代码和主项目是异构的怎么办? 比如主项目是JavaScript, 但是模块内容是TypeScript, 或者更异构的东西