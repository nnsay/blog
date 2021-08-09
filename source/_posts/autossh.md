---
title: 优秀工具推荐--autossh
date: 2021-08-09 09:28:36
tags:
- Linux
---

### 1. 背景
SSH不管是远程登录还是创建一个SSH通道总是掉线很不爽, 查到到一个工具autossh可以自动重连, 再结合SSH本身的一些高级特性基本再使用过程再也不需要担心SSH断开的问题了.

### 2. [autossh](https://linux.die.net/man/1/autossh) 基本功能介绍

- 自动重连
- 可后台运行
- 兼容SSH高级特性

### 3. 经典使用场景
- SSH通道: 自测一周随时使用通道都是打开的, 强烈在通道需求下使用
- SSH远程登录: 虽然会自动重连, 但是建议还是在目标端配合screen使用

### 4. 举个例子

```
autossh -f -M 0 -N -D 9999 -o "ServerAliveInterval 10" -o "ServerAliveCountMax 3" -o ExitOnForwardFailure=yes tunnel@xxx.xxx.xxx.xxx
```