---
title: picgo
date: 2022-04-07 11:06:53
tags:
- 工具
- Tips
excerpt: 使用PicGo提供图床服务使Typora协作更优雅
---

# 1. [PicGo-Core](https://github.com/PicGo/PicGo-Core)

> A tool for pictures uploading. Both CLI & API supports

PicGo-Core其实PicGo下的一个项目, PicGo是一个基于Electron的GUI客户端而PicGo-Core是做图床实现的核心,如其GitHub的描述, Core只是CLI和API, 其中API就是NPM package, API方式使用时更像是一个做图床的SDK. 本问主要介绍CLI的使用.

# 2. PicGo CLI 使用

## 2.1 安装
```
yarn global add picgo # 或者 npm install picgo -g
```

## 2.2 配置文件

默认picgo cli的配置文件位置在: ~/.picgo
```
.picgo
├── config.json
├── node_modules
├── package-lock.json
├── package.json
└── picgo.log
```

## 2.3 配置

- GitHub: 使用GitHub作为图床, picgo内置, 直接进行配置

  - [直接编辑config.json进行配置](https://picgo.github.io/PicGo-Core-Doc/zh/guide/config.html#picbed)
  - [使用命令行交互命令进行配置](https://picgo.github.io/PicGo-Core-Doc/zh/guide/commands.html#config-set)

  ```
  picgo set uploader github
  ```

- Gitee: GitHub因为网络等问题可能服务访问, 国内建议使用Gitee作为图床, 安装步骤如下:

  - 安装[gitee-uploader](https://github.com/lizhuangs/picgo-plugin-gitee-uploader)

    ```
    picgo install gitee-uploader
    ```

    只有安装了该插件, gitee的uploader才能被使用

  - 配置gitee-uploader

    ````
    picgo set uploader github
    ````

- [其他插件: 这里介绍了CLI和GUI的各种PicGo插件](https://github.com/PicGo/Awesome-PicGo)

## 2.4 配置结果展示

```
➜  ~ cat ~/.picgo/config.json
{
  "picBed": {
    "current": "gitee",
    "uploader": "gitee",
    "github": {
      "branch": "main",
      "customUrl": "",
      "path": "img/",
      "repo": "xxxname/xxxrepo",
      "token": "ghp_xxxx"
    },
    "transformer": "path",
    "picgo-plugin-gitee-uploader": {},
    "gitee": {
      "repo": "xxxname/xxxrepo",
      "branch": "master",
      "token": "xxxx",
      "path": "",
      "customPath": "default",
      "customUrl": ""
    }
  },
  "picgoPlugins": {
    "picgo-plugin-gitee-uploader": true
  },
  "picgo-plugin-gitee-uploader": {
    "lastSync": "2022-04-07 10:17:54"
  }
}
```

## 2.5 切换图床

如果有多个图床, 可以选择一个作为默认, 也可能根据情况切换, 方法如下:

```
➜  ~ picgo use uploader
? Use an uploader (Use arrow keys)
❯ gitee 
  smms 
  tcyun 
  github 
  qiniu 
  imgur 
  aliyun
  (Move up and down to reveal more choices)
```

这是一个交互的命令, 选择你需要的图床, 选择的图床名字会在配置文件中的: `picBed.current`和`picBed.uploader`zhong , 随意可以接直接编辑配置.

# 3. 测试上传

- 使用操作系统或者其他各类截图工具如微信, 选择一个区域截图, 截图会报错在粘贴板

- 上传粘贴板截图

  ```
  ➜  ~ picgo u
  [PicGo INFO]: Before transform
  [PicGo INFO]: Transforming... Current transformer is [path]
  [PicGo INFO]: Before upload
  [PicGo INFO]: Uploading... Current uploader is [gitee]
  [PicGo SUCCESS]: 
  https://gitee.com/xxxname/xxxrepo/raw/master/20220407105320.png
  ```

# 4. 与Typora配合使用

- 选择图像, 选择自定义命令
- 输入自定义命令

![image-20220407110157776](https://raw.githubusercontent.com/nnsay/gist/main/img/20220407113115.png)

其中`命令`中的全文如下, 主要是增加了环境变量`PATH`的修改, 让其可以找到`picgo`命令

```
export PATH=/usr/local/bin/:$PATH; picgo upload
```

注意: 命令picgo upload是picgo u的简写, 和上面测试步骤中的命令一致

配置完成后, 当你截图复制到Typora的时候就自动会把图片上传到默认图床, 并且把图片地址从本地替换为图床文件.


