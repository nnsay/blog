---
title: github action cache
date: 2023-03-24 11:58:46
tags:
  - Tips
excerpt: 使用github action cache加快流水线速度
---

# 1. 问题场景

最近在使用 Github Action 做 CICD 的过程中发现了一个问题: 构建很慢. 除了 Runner 冷启动外还有一些流水线业务本身的问题, 例如: 安装依赖.

当使用 Monorepo 后依赖的安装增多了, 使得 yarn/npm install 的速度变得很慢, 经过探索得知: 可以通过使用缓存安装依赖包的速度, 本文介绍缓存方式.

# 2. [缓存全局包数据](https://github.com/actions/setup-node#caching-global-packages-data)

这是一种非缓存`node_modules`目录的方式, 缓存的是全局包数据, 底层使用 action/cache, 但是封装的比较好, 自动化程度高, 配置少, 具体的做法是在 github workflow 使用`actions/setup-node`并启用缓存:

```yaml
- name: Setup Node.js
  uses: actions/setup-node@v3
  with:
  node-version: 16
  cache: yarn
```

cache 支持三种缓存: `npm`/`yarn`/`pnpm`, 且自动寻找缓存依赖文件: `package-lock.json`/`npm-shrinkwrap.json`/ `yarn.lock`

# 3. [缓存依赖文件(目录)](https://github.com/actions/cache)

直接使用`actions/cache`进行缓存, 完整的测试例子如下, 其中`actions/cache`是核心:

```yaml
name: action catch test

run-name: "action: ${{ inputs.action }}"

on:
  workflow_dispatch:
    inputs:
      action:
        type: string
        description: operation what?

jobs:
  yarn:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - uses: actions/setup-node@v3
        with:
          node-version: "16.x"

      - uses: actions/cache@v3
        id: cache-node-module
        with:
          key: ${{ runner.os }}-modules-${{ hashFiles('yarn.lock') }}
          path: node_modules

      - name: Install Depedency
        if: steps.cache-node-module.outputs.cache-hit != 'true'
        run: yarn install
```

[`actions/cache`属性介绍](https://github.com/actions/cache#inputs):

- input:

  - key: 缓存的名字标识

  - path: 缓存文件或者目录

- output
  - cache-hit: 缓存是否命中

# 4. 测试和验证

测试验证针对的是`actions/cache`方式缓存, 而不是`actions/setup-node`, 这种日常已经使用加速效果不大, 需要两个文件:

- package.json: 里面配置了`dependencies`/`devDependencies`, 尽量多一点
- yarn.lock: 和上面的 package.json 对应起来

- 测试步骤: 分别手动 run 两次`yarn`这个`job`, 断言如下:

  - 第一次: 没有命中缓存执行了 yarn install, 时间较长

  - 第二次: 命中的缓存没有执行 yarn install, 时间较短

- 测试结果

![image-20230324123135048](https://raw.githubusercontent.com/nnsay/gist/main/imgimage-20230324123135048.png)

- 测试结论

  - 通过缓存目录极大的加快了依赖安装的时间, 该方式加速效果明显
  - 该方式缓存文件较多, 需要注意不要触发 Github 的限制: 10GB/repo
  - 缓存较大时下载缓存可能耗时

  总体上该方式值得尝试!
