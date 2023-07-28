---
title: Github自托管Runner实践
date: 2023-07-28 11:58:19
tags:
  - DevOps
excerpt: 使用 Mac mini 搭建自托管的 Github Runner
---

# 1. 运行自托管 Runner

本文档调研基于 Mac OS 系统 M 系列芯片的硬件测试, 测试过硬件有:

- MacBook Pro, macOS 13.4.1 (c) (22F770820d) Apple M2 Pro
- Mac mini, macOS 13.4.1 (c) (22F770820d), Apple M2

## 1.1 下载脚本

```bash
# Create a folder
mkdir actions-runner && cd actions-runner

# Download the latest runner package
curl -o actions-runner-linux-arm64-2.306.0.tar.gz -L https://github.com/actions/runner/releases/download/v2.306.0/actions-runner-linux-arm64-2.306.0.tar.gz

# Extract the installer
mkdir runner1
tar xzf ./actions-runner-linux-arm64-2.306.0.tar.gz -C runner1/

```

## 1.2 配置和运行

```bash
./config.sh --name demo-runner --url https://github.com/Albedo-Inc --token xxxxx --labels demo

./run.sh
```

如果运行交互性的操作一律使用回车默认即可

# 2. 性能对比

## 2.2 Pull Request 工作流

主要任务是运行单元测试和集成测试, 性能对比如下:

|                                   | Self hosted |         | Github hosted |         |
| --------------------------------- | ----------- | ------- | ------------- | ------- |
|                                   | 首次        | 第 N 次 | 首次          | 第 N 次 |
| Set up job                        | 21          | 21      | 2             | 2       |
| Restore nx cache                  | 72          | 68      | 67            | 90      |
| Check out git repository          | 15          | 16      | 6             | 5       |
| Apply the node_modules cache      | 2           | 55      | 13            | 13      |
| Install dependencies              | 539         | 1       | 1             | 1       |
| Run Jest tests                    | 8           | 0       | 52            | 0       |
| Run Cypress tests                 | 124         | 0       | 322           | 1       |
| Post Apply the node_modules cache | 98          | 0       | 0             | 0       |
| Total duration                    | 907         | 187     | 404           | 166     |

首次含义:
本地无 nx 缓存(1.1GiB)
本地无 yarn 缓存(360 MB)
远程无 node_module 缓存

## 1.2 Build 工作流

主要任务是编译, 将 Typescript 打包为 JavaScript, 性能对比如下:

|                              | Self hosted | Github hosted |
| ---------------------------- | ----------- | ------------- |
| Set up job                   | 23          | 4             |
| Restore nx cache             | 193         | 118           |
| Check out git repository     | 16          | 7             |
| Apply the node_modules cache | 31          | 16            |
| Install dependencies         | 3           | 1             |
| Build affected projects      | 26          | 568           |
| Total duration               | 172         | 720           |

## 1.3 总结

- 启动调度速度: Self hosted 逊于 Github hosted
- 下载速度: Self hosted 稍逊 Github hosted
- 编译/测试任务处理: Self hosted 优于 Github hosted

**如何一个工作流的耗时大头是任务处理类的, 可以考虑使用 Self hosted, 否则效率上无优势**

# 3. 单机并行方案

根据日常 CICD 的观察, 很多场景下对 CPU 的要求不好, 主要是内存的限制, 特别是 TypeScript 构建过程, 所以如果单机性能较好只作为一个 Runner 运行太浪费, 所以单机开启多个 Runner 是理想的选择, 具体探索和尝试的思路如下:

## 3.1 单进程 vs 多任务

本地仅开启一个 Runner 进程, 但是并行有多个流水线任务都使用自托管 Runner. 测试结果:

**本来的并行多任务无法并行, 因为争抢 Runner 变成串行**

## 3.2 多进程 vs 多任务

本地运行多个 Runner 进程, 模拟注册多个 Runner. 测试结果:

- 在同目录下

  不能再次运行 config.sh 重新配置, 除非删除之前的配置

- 在不同目录下

  可以注册另一个 Runner, 需要注意的是:

  - 提供不一样的机器名称
  - 提供相同的标签

## 3.3 多容器 vs 多任务

根据[Github awesome runner](https://github.com/myoung34/docker-github-actions-runner)的文章, 选择`myoung34/github-runner`作为 Runner 镜像, 启动多个容器实例来模拟多 Runner. Compose 配置如下:

```yaml
version: '2.3'
services:
  worker1:
    image: myoung34/github-runner:latest
    container_name: worker1
    environment:
      REPO_URL: https://github.com/nnsay/github-action-tutorial
      RUNNER_NAME: worker1
      RUNNER_TOKEN: ADNLAD2P5KRJJCBQJFWGVF3EW5PRU
      # RUNNER_GROUP: my-group
      LABELS: demo,docker-runner
    security_opt:
      # needed on SELinux systems to allow docker container to manage other docker containers
      - label:disable
    volumes:
      - '/var/run/docker.sock:/var/run/docker.sock'
      - '/tmp/nx:/tmp/nx'
  worker2:
  	... ...
```

验证结果:

- docker compose up 自动注册 Runner

- docker compose down 自动删除 Runner

- 多任务执行异常

  - `Build工作流`稳定失败且无错误日志, 推测是内存原因上调 Docker 资源亦无法解决

  - Cypress e2e 测试失败

    ```
    Your system is missing the dependency: Xvfb

    Install Xvfb and run Cypress again.

    Read our documentation on dependencies for more information:

    https://on.cypress.io/required-dependencies

    If you are using Docker, we provide containers with all required dependencies installed.

    ----------

    Error: spawn Xvfb ENOENT

    ----------

    Platform: linux-arm64 (Ubuntu - 20.04)
    Cypress Version: 12.14.0
    ```

- 缺陷

  - 案例中的镜像内虽然包含 docker, 但是 github action 自己不支持在 self hosted 使用 docker, 详情查看[这里](https://github.com/myoung34/docker-github-actions-runner/tree/master#docker-support)

## 3.4 总结

个人推荐**多进程**的方式, 根据实际测试和使用**多容器**有诸多问题例如: 镜像的个性化维护, 不支持 Job Service, 莫名其妙的退出等; 其次容器或者虚拟机本身是要占用资源了, **多进程**方式资源利用率更高.

# 4. 深入调研

## 4.1 自定义 Runner 镜像

```bash
# 下载代码
git clone https://github.com/myoung34/docker-github-actions-runner.git
# 镜像自定义, 例如安装一些没有的命令: yarn
...
# 构建base镜像
docker build -f Dockerfile.base -t myoung34/github-runner-base:latest .
# 构建runner镜像
docker build -t local/github-runner:latest .
```

## 4.2 [将自托管的运行应用程序配置为服务](https://docs.github.com/zh/actions/hosting-your-own-runners/managing-self-hosted-runners/configuring-the-self-hosted-runner-application-as-a-service)

github runner 脚本已经包含了将应用程序服务化的帮助脚本, 直接使用即可

```bash
./svc.sh install
./svc.sh start
```

# 5. Troubleshooting

## 5.1. 克隆代理

~/.gitconfig:

```toml
[http]
	postBuffer = 524288000
	version = HTTP/1.1
	proxy = 127.0.0.1:7890
[https]
	proxy = 127.0.0.1:7890
```

## 5.2 找不到`Yarn`命令

```bash
sudo npm install yarn@1 -g
```

## 5.3 AWS-CLI SSL 证书验证失败

```yaml
rules:
  - "DOMAIN-SUFFIX,amazonaws.cn,DIRECT"
  - "DOMAIN-SUFFIX,amazonaws.com.cn,DIRECT"
```

## 5.4. Docker login 失败

错误: Error saving credentials: error storing credentials - err: exit status 1, out: `User interaction is not allowed.`

解决方案: 编辑 runsvc.sh, 增加 session 级别的 unlock

```bash
security unlock-keychain -p xxxx ~/Library/Keychains/login.keychain-db
```

## 5.5 环境变量设置

查看本地安装目录`env.sh`文件, 可以在本地生产.env 然后写入环境变量, 例如:

```toml
https_proxy=http://127.0.0.1:7890
http_proxy=http://127.0.0.1:7890
```

可以通过执行 env.sh 查看最后生成的.env 中的环境变量是否 OK

## 5.6 Mac mini 保持活动状态

问题: 即使配置从不睡眠和锁屏, 退出远程桌面后发现 Runner 还是会掉线, 即使在终端放置一些诸如 htop 的命令运行, 发现其实也不是真正的运行, 似乎被挂起了, 这个挂起可以使用 nodejs 的 setInterval 输出时间的方法证明.

解决方案: 使用第三方软件保持活动: [Amphetamine](https://apps.apple.com/cn/app/amphetamine/id937984704)

# 6. 远程连接

## 6.1 远程桌面方式

- Runner 端设置

  系统设置->通用->共享, 打开 屏幕共享

- 发起连接

  Finder->前往->连接服务器, 输入: vnc://officerunner1, 连接成功后还需要输入用户名/密码即可远程桌面操作 Runner

  ℹ️officerunner1 是 Runner 主机名, 这个名字也可以在: 系统设置->通用->共享->本地主机名 中设置

## 6.2 SSH

- Runner 端设置

  系统设置->通用->共享, 打开 远程登录

- 发起连接

  Finder->前往->连接服务器, 输入: ssh [username]@[ip/machineName], 连接成功后还需要输入密码随后即可操作

# 6. 参考文档

- [Github awesome runner](https://jonico.github.io/awesome-runners/)

- [Unlock keychain on headless system does not work on BigSur](https://developer.apple.com/forums/thread/690665)
