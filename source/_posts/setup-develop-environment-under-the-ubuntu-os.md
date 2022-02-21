---
title: Ubuntu 20.02 搭建开发环境 
date: 2020-11-04 13:19:01
tags:
- Tips
excerpt: 使用Ubuntu 20.02搭建办公/开发环境
---

#### 0 介绍
最近因为自己的MBP太老的性能有点跟不上了，所以申请了公司自的电脑， 公司的电脑是ThinkPadT490, 16G的内存极大的提高的开发效率， 特别是跑单元测试提高了两倍速度。

根据最近一段时间的使用，我基于Ubuntu系统搭建了一套自己的开发环境，并且还比较顺手从MBP切换到Ubuntu从开发角度看并没有太多的不适应， 所以总结了这篇文章， 希望可以帮助到使用Ubuntu开发的朋友！

Ubuntu OS:  Ubuntu 20.04.1 LTS。

#### 1 [通过dpk安装软件](https://linoxide.com/ubuntu-how-to/install-deb-file-from-command-line/)
dpk包是常见的安装软件方式，而这些包基本都可以从软件的各自官网下载。
```
#安装
sudo dpkg -i ./xxx.deb
#安装出错修复， 修复后再安装
sudo apt -f
#卸载
sudo dpkg -r xxx
```
使用该方法可安装：
- 有道词典
- DBearer
- 网易云音乐
- VS Code
- WPS
- 搜狗输入法

#### 2 通过Snap安装软件
这类软件可以在Snap的商店里面找到你需要的软件： https://snapcraft.io/store
```
sudo snap install xxx
```
使用该方法可以安装：
- Microsoft Teams
- Zenhub
- Drawio

#### 3 使用AppImage软件
AppImage通用平台软件，这类软件可以从官网或者Github下载， 下载后只要增加可执行权限双击即可使用，使用该方式可以安装使用如下软件：
- VNote

#### 4 [使用Wine安装Windows软件](https://zhuanlan.zhihu.com/p/144286142)
使用该方式可安装一些常用Windows软件：
- 微信

#### 5 使用Docker安装软件
使用docker-componse可以管理基础软件，这是本人经常使用的方式， 主要是方便快速管理和升级软件， 使用该方式可以安装的软件服务如下：
-  mysql
-  postgres
-  redis
- mongodb
