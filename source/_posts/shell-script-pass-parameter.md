---
title: shell 脚本传参
date: 2019-10-24 23:40:00
tags:
  - Shell
  - Tips
excerpt: shell脚本相互调用时常见的传参方式
---

### 1. 如何在 Shell 脚本间传递参数
- 通过脚本执行参数
- 通过脚本执行环境变量

### 2. 具体实现
假设有两个脚本: exe.sh 和 exe1.sh, 前者调用后者, 并且需要传递一个平台参数, 例如:w代表window, m代表mac. 另外这两脚本都是经过 `chmod +x` 操作后的, 所以直接可以运行.

#### 2.1 通过脚本执行参数
将需要的信息参数当成可执行脚本的命令行参数传入, 具体如下:
`exe.sh`
```sh
#!/usr/bin/env bash
if [ -z $1 ]; then
  P=w
else
  P=$1
fi

./exe1.sh $P
```

`exe1.sh`
```sh
#!/usr/bin/env bash
echo "method 1 in exe1 shell:" $1
```
执行结果:
```sh
./exe.sh
method 1 in exe1 shell: w

./exe.sh m
method 1 in exe1 shell: m
```

#### 2.2 通过脚本执行环境变量
将需要的信息参数当成可执行脚本的命令行参数传入, 具体如下:
`exe.sh`
```sh
#!/usr/bin/env bash
if [ -z $1 ]; then
  P=w
else
  P=$1
fi

PLATFORM=$P ./exe1.sh
```

`exe1.sh`
```sh
#!/usr/bin/env bash
echo "method 2 in exe1 shell:" $PLATFORM
```
执行结果:
```sh
./exe.sh
method 2 in exe1 shell: w

./exe.sh m
method 2 in exe1 shell: m
```

### 3. 思考

虽然两个方式都可以解决问题, 个人建议还会第二种比较好, 理由是: 如果传递参数较多第二个因为是变量, 有语义,而第一种单纯的靠顺序,没有语义,如果顺变量维护成本很大.