---
title: 阿里云盘挂载(MacOS)
date: 2022-07-01 16:24:56
tags:
 - 工具
 - Tips
excerpt: 挂载阿里云盘到本地(MacOS)
---

# 阿里云盘挂载(MacOS)

阿里云盘不限速吸引了很多用户, 早起用户有1T左右的永久空间, 这个空间办公足够了. 如果有影音需要可能这个空间可能不足但是可以通过做任务获取空间. 百度网盘之前有一个很好的体验就是可以把网盘映射到本地, 同步和浏览十分方便, 网盘如同本地磁盘一样使用体验上无感知, 本文主要介绍如何挂载阿里云盘到本地.

[基本原理介绍](https://github.com/messense/aliyundrive-webdav): 首先把阿里云盘包装成一个WebDAV 服务, 然后在操作系统上直接使用该服务.

## 1. [阿里云盘WebDAV服务](https://github.com/messense/aliyundrive-webdav)

### 1.1 获取阿里云盘Token

登录网页版阿里云盘, 打开发者工具, 在控制台执行如下命令可以获取一个`自己的阿里云盘TOKEN`

```javascript
JSON.parse(localStorage.token).refresh_token
```

![image-20220701150434063](https://raw.githubusercontent.com/nnsay/gist/main/img/image-20220701150434063.png)

### 1.2 搭建WebDAV服务

- 方法一: 使用docker部署一个webdav服务, docker-compose.yaml文件如下:

  ```yaml
  version: "3"
  services:
    aliyundrive:
      container_name: aliyundrive
      restart: unless-stopped
      ports:
        - 4000:8080
      environment:
        - REFRESH_TOKEN="自己的阿里云盘TOKEN"
        - WEBDAV_AUTH_USER=yourAccount # 自定义
        - WEBDAV_AUTH_PASSWORD=yourAccountPassword # 自定义
      image: messense/aliyundrive-webdav
  ```

  启动命令如下:

  ```bash
  docker-compose up -d aliyundrive
  ```

- 方法二: 使用二进制命令

  - 下载安装 [aliyundrive-webdav](https://github.com/messense/aliyundrive-webdav/releases) 

  - 启动命令

    ```bash
    # 前台方式
    aliyundrive-webdav --workdir ~/temp/aliyundrive --auth-user aliyun --auth-password aliyun@pwd --refresh-token d63aa96109c442d8ad427035c960505b --port 8888
    
    # 后台方式
    nohup aliyundrive-webdav --workdir ~/temp/aliyundrive --auth-user aliyun --auth-password aliyun@pwd --refresh-token d63aa96109c442d8ad427035c960505b --port 8888 &
    ```
    

两种方式均OK, 启动Docker可能占用资源, 二进制方式对资源使用更友好一些.

## 2. 挂载云盘到本地

### 2.1 连接到远程服务器

打开`访达`菜单栏, 选择`前往`, 选择`链接服务器`, 如下图:

![image-20220701151036938](https://raw.githubusercontent.com/nnsay/gist/main/img/image-20220701151036938.png)

输入服务地址: http://localhost:4000

![image-20220701151322535](https://raw.githubusercontent.com/nnsay/gist/main/img/image-20220701151322535.png)

最后输入1.2中第10,11行的账号和密码, 等待链接完成即可.

### 2.2 查看挂载效果

上一步链接到http://localhost:4000后则本地`桌面`和`位置`就会出现一个`localhost`的磁盘, 可以通过一下命令确认:

```bash
➜  ~ ls -l /Volumes
total 4
lrwxr-xr-x  1 root      wheel     1 Jun 28 07:28 Macintosh HD -> /
drwx------  1 xxxxxxxx  staff  2048 Jul  1 15:16 localhost
```

当然也可以通过`访达`左侧`位置`菜单使用该网盘. 至此即可以向使用本地文件一样使用阿里云盘.