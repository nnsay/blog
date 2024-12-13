---
title: skopeo
date: 2024-06-13 11:37:41
tags:
  - DevOps
excerpt: 使用skopeo同步镜像
---

# 1. 介绍

如今大部分服务的交付都是通过镜像进行交付的, 例如: 私有化部署场景, 需要将打好的镜像放到客户的 CR(容器镜像仓库)中, 或者我们有多个环境(dev, test, staging, prod)每个环境因为隔离都会保存一份镜像用于部署. 其实保存多分是避免不了的, 但是在每个 CR 的镜像需要构建就太浪费时间, 以多环境为例, 完全可以将 test 测试通过的镜像无需构建同步到到 staging 和 prod 进行使用即可.

本文介绍一种针对容器镜像操作的工具: [skopeo](https://github.com/containers/skopeo). 该工具可以是一个命令行工具, 可以操作容器镜像和镜像仓库, 当然上面场景提到的多个仓库同步镜像正式该工具的核心功能, 我们可以像拷贝文件一样将一个 CR 的镜像拷贝到另外一个 CR. 以下是 skopeo 的简介:

Skopeo 支持 API V2 容器镜像注册表，如 docker.io 和 quay.io 注册表、私有注册表、本地目录以及本地 OCI 布局目录。Skopeo 可以执行以下操作：

- 从各种存储机制中复制镜像。例如，你可以在不需要权限的情况下将镜像从一个注册表复制到另一个注册表。
- 检查远程镜像，显示其属性，包括其层，而无需将镜像拉取到主机。
- 从镜像仓库中删除镜像。
- 将外部镜像仓库同步到内部注册表以用于隔离部署。
- 在注册表要求时，skopeo 可以传递适当的凭据和证书进行身份验证。

# 2. 实践

## 2.1 [安装](https://github.com/containers/skopeo/blob/main/install.md)

```bash
brew install skopeo
```

## 2.2 登录

```bash
# docker hub
~ skopeo login --username=nnsay docker.io
Password:
Login Succeeded!
# github cr
~ skopeo login --username=nnsay ghcr.io
Password:
Login Succeeded!
# quay
~ skopeo login --username=nnsay quay.io
Password:
Login Succeeded!
```

查看当前登录的 CR 的用户:

```bash
~ skopeo login --get-login docker.io
nnsay
~ skopeo login --get-login ghcr.io
nnsay
```

## 2.3 镜像检查

```
skopeo inspect docker://docker.io/nnsay/rest-nodejs
skopeo inspect docker://docker.io/nnsay/rest-nodejs:latest
```

检查结果:

```json
{
  "Name": "docker.io/nnsay/rest-nodejs",
  "Digest": "sha256:894e14608ea6748686f76f1a389e4be4eaff83363cf8d6720ab60a007e6881d8",
  "RepoTags": ["01eb900", "b4dc8c9-dirty", "b54e9e6", "latest"],
  "Created": "2024-05-08T10:32:19.570088304Z",
  "DockerVersion": "",
  "Labels": null,
  "Architecture": "arm64",
  "Os": "linux",
  "Layers": [
    "sha256:bca4290a96390d7a6fc6f2f9929370d06f8dfcacba591c76e3d5c5044e7f420c",
    "sha256:6f96711d72f7a535656ea52a3109d569cde16c6d218d52e04957afd2402afa0e"
  ],
  "LayersData": [
    {
      "MIMEType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
      "Digest": "sha256:bca4290a96390d7a6fc6f2f9929370d06f8dfcacba591c76e3d5c5044e7f420c",
      "Size": 3347715,
      "Annotations": null
    },
    {
      "MIMEType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
      "Digest": "sha256:6f96711d72f7a535656ea52a3109d569cde16c6d218d52e04957afd2402afa0e",
      "Size": 62254,
      "Annotations": null
    }
  ],
  "Env": [
    "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
    "NODE_VERSION=18.20.2",
    "YARN_VERSION=1.22.19"
  ]
}
```

> [!IMPORTANT]
> inspect 命令重镜像地址是否有 tag 的唯一区别是 Digest 结果不一样

## 2.4 查询镜像 Tag

inspect 命令可以检查镜像 tag, 如上面结果中的`RepoTags`, 也可以直接获取 tag:

```bash
~ skopeo list-tags docker://docker.io/nnsay/rest
{
    "Repository": "docker.io/nnsay/rest",
    "Tags": [
        "01eb900",
        "b4dc8c9-dirty",
        "b54e9e6",
        "latest"
    ]
}
```

## 2.5 拷贝

目前支持的 CR 有: Quay, Docker Hub, OpenShift, GCR, Artifactory...

### 2.5.1 CR 到 CR

```bash
# docker.io -> quay.io
skopeo copy --multi-arch=all docker://docker.io/nnsay/rest-nodejs:latest docker://quay.io/nnsay/rest-nodejs:latest
```

> [!NOTE]
>
> - 拷贝时 quay.io 不需要提前创建镜像仓库
> - 拷贝是可以通过`multi-arch`指定平台: system, all, or index-only

### 2.5.2 CR 到 Local

```bash
mkdir rest-nodejs
skopeo copy docker://docker.io/nnsay/rest-nodejs:latest dir:./rest-nodejs
```

查看拷贝到本地的镜像文件:

```bash
# 目录结构
➜  rest-nodejs tree .
.
├── 505c806c8fed7a9d50910aef54f57a067bb1703f868e5e851e85664f96b671ab
├── ...
├── ed2e2467392d69d8fcbdc4cf428a936bd933c0546a89f34afd9afe2a150b4f73
├── manifest.json
└── version

# 版本
➜  rest-nodejs cat version
Directory Transport Version: 1.1

# manifest
➜  rest-nodejs cat manifest.json
{
   "schemaVersion": 2,
   "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
   "config": {
      "mediaType": "application/vnd.docker.container.image.v1+json",
      "size": 7690,
      "digest": "sha256:505c806c8fed7a9d50910aef54f57a067bb1703f868e5e851e85664f96b671ab"
   },
   "layers": [
      {
         "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
         "size": 3347715,
         "digest": "sha256:bca4290a96390d7a6fc6f2f9929370d06f8dfcacba591c76e3d5c5044e7f420c"
      },
      ...
      {
         "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
         "size": 62254,
         "digest": "sha256:6f96711d72f7a535656ea52a3109d569cde16c6d218d52e04957afd2402afa0e"
      }
   ]
}

# 层文件类型: JSON和gzip(也可以从manifest.json中知晓)
➜  rest-nodejs file 505c806c8fed7a9d50910aef54f57a067bb1703f868e5e851e85664f96b671ab
505c806c8fed7a9d50910aef54f57a067bb1703f868e5e851e85664f96b671ab: JSON data
➜  rest-nodejs file 6f96711d72f7a535656ea52a3109d569cde16c6d218d52e04957afd2402afa0e
6f96711d72f7a535656ea52a3109d569cde16c6d218d52e04957afd2402afa0e: gzip compressed data, original size modulo 2^32 18636
```

### 2.5.3 Local 到 CR

```bash
skopeo copy dir:./rest-nodejs docker://docker.io/nnsay/rest-nodejs:copytest
Getting image source signatures
Copying blob bca4290a9639 skipped: already exists
Copying blob ed2e2467392d skipped: already exists
Copying blob 8409080807cd skipped: already exists
Copying blob 8fabd45d2d86 skipped: already exists
Copying blob 58ae15b6393a skipped: already exists
Copying blob 9cfe8bf71bbe skipped: already exists
Copying blob b1befdbe30a8 skipped: already exists
Copying blob eb547ff4872f skipped: already exists
Copying blob 6f96711d72f7 skipped: already exists
Copying config 505c806c8f done   |
Writing manifest to image destination
```

上面的例子虽然层都存在了, 但是还是新增了一个 tag, 去 docker hub 可以查看到:

<img src="https://img.picgo.net/2024/06/12/image-20240612143650799fa26ab665abcf1c2.png" alt="image-20240612143650799" style="zoom:50%;" />

## 2.6 删除镜像

```bash
skopeo delete docker://quay.io/nnsay/rest-nodejs
```

> [!NOTE]
>
> - 不提供 tag 是 tag 是 latest
> - 如果镜像有依赖删除不掉

## 2.8 同步

### 2.8.1 CR 到 Local

```
skopeo sync --src docker --dest dir --scoped docker.io/nnsay/rest-nodejs ./rest-nodejs
```

`sync`比`copy`的优势是批量, 可以将一个镜像的所有的 tag 都同步目标. 拷贝后的文件目录树如下:

```
├── docker.io
│   └── nnsay
│       ├── rest-nodejs:01eb900
│       │   ├── 5730c637dadbf0ad9b9e4903c62e30d9f239a79626ebec5cfaeba38861638678
│       │   ├── 6a75fad5c66c981a46081a9f10761c4066b3bc553eb18ce4f298d45aed2f1991
│       │   ├── 6f5a403356d97a11bfa5349e025293b8d634ff7d6e0ebc5df63351e87cd7f5cc
│       │   ├── 7049a133eb54d652049fa44657d24cef7298dc371ffab6e27f5cd5e53e62159f
│       │   ├── 70c397145e382faacb37dd6122975c7de70024f24dc832853bbd7fd96d916170
│       │   ├── 717ea7c8282c75dcb840c9de1b608545a8b9464ef255885f361b2b2ec4b7ee7c
│       │   ├── 7f16613f31035b565d5cf89e9af3fa8890136e7799bddca2a06f6f432e52709e
│       │   ├── 90577567f13c6046278cbfa4a557cb5a19ed5f9c281ab540e84ddd8f74c71922
│       │   ├── c303524923177661067f7eb378c3dd5277088c2676ebd1cd78e68397bb80fdbf
│       │   ├── eb7895eb72c1a145d166631d5a81c61bffd747783c52e8a1de9fa0e7d5871817
│       │   ├── manifest.json
│       │   └── version
│       └── rest-nodejs:latest
│           ├── 505c806c8fed7a9d50910aef54f57a067bb1703f868e5e851e85664f96b671ab
│           ├── 58ae15b6393a39efbae7529155a59479af3f7ed318524c6eda40e7dac0fded7e
│           ├── 6f96711d72f7a535656ea52a3109d569cde16c6d218d52e04957afd2402afa0e
│           ├── 8409080807cdb96ff66696e7ee976364d3ecbf0e297a79e472460ad4f983aeff
│           ├── 8fabd45d2d86badb5acb8689ed8665436d31b2b00e40e51a003960507c234488
│           ├── 9cfe8bf71bbea49c32593429360ce2e678838611977df04274ddef5426551048
│           ├── b1befdbe30a812a37ada093c2f0b39b47089dd4a7ee1497aaf9c24aca0e008af
│           ├── bca4290a96390d7a6fc6f2f9929370d06f8dfcacba591c76e3d5c5044e7f420c
│           ├── eb547ff4872f702330c428133d4056ed3cf33d305cbde77fb0a472790894e332
│           ├── ed2e2467392d69d8fcbdc4cf428a936bd933c0546a89f34afd9afe2a150b4f73
│           ├── manifest.json
│           └── version
```

### 2.8.2 Local 到 CR

```bash
# cr 到 local
skopeo sync --src docker --dest dir --scoped docker.io/nnsay/rest ./rest
# local 到 cr
skopeo sync --src dir --dest docker ./rest quay.io/nnsay/rest
```

> [!WARNING]
>
> - cr 同步到 local, 然后利用同步的结果重新返回同步到 cr 报错(skopeo sync --src dir --dest docker ./rest docker.io/nnsay/rest):
>
>   FATA[0008] Error copying ref "dir:rest/docker.io/nnsay/rest:latest": trying to reuse blob sha256:c303524923177661067f7eb378c3dd5277088c2676ebd1cd78e68397bb80fdbf at destination: checking whether a blob sha256:c303524923177661067f7eb378c3dd5277088c2676ebd1cd78e68397bb80fdbf exists in docker.io/nnsay/rest/rest: requested access to the resource is denied
>
> - cr 同步到 local, 再次执行报错(skopeo sync --src docker --dest dir --scoped docker.io/nnsay/rest ./rest):
>
>   FATA[0002] Refusing to overwrite destination directory "rest/docker.io/nnsay/rest:latest
>
> - local 同步到另一个 CR 成功(skopeo sync --src dir --dest docker ./rest quay.io/nnsay/rest)

建议在具体使用 sync 保持同步目录干净, 且不要试图覆盖, 不管是 cr 到 local 还是 local 到 cr

### 2.8.3 CR 到 CR

```bash
~ skopeo sync --src docker --dest docker docker.io/nnsay/aliyundrive-webdav quay.io/nnsay/aliyundrive-webdav
INFO[0000] Tag presence check                            imagename=docker.io/nnsay/aliyundrive-webdav tagged=false
INFO[0000] Getting tags                                  image=docker.io/nnsay/aliyundrive-webdav
INFO[0002] Copying image ref 1/1                         from="docker://nnsay/aliyundrive-webdav:latest" to="docker://quay.io/nnsay/aliyundrive-webdav/aliyundrive-webdav:latest"
Getting image source signatures
Copying blob 9d6aed3dd5e1 done   |
Copying blob 213ec9aee27d done   |
Copying blob 114a5ea8ad83 done   |
Copying blob a661876d736d done   |
Copying blob 4f4fb700ef54 done   |
Copying blob 988498f5135b done   |
Copying config 847ce028e6 done   |
Writing manifest to image destination
INFO[0017] Synced 1 images from 1 sources
```

> [!WARNING]
> 如果镜像是多平台同步且和本地系统默认平台不一致则报错(skopeo copy docker://docker.io/nnsay/rest-nodejs:latest docker://quay.io/nnsay/rest-nodejs:latest):
>
> ERRO[0002] Couldn't get cpu architecture: getCPUInfo for OS darwin not implemented
>
> FATA[0002] choosing an image from manifest list docker://nnsay/rest-nodejs:latest: no image found in image index for architecture arm64, variant "", OS darwin
>
> 此时, 建议使用 copy: skopeo copy --multi-arch=all docker://docker.io/nnsay/rest-nodejs:latest docker://quay.io/nnsay/rest-nodejs:latest

# 3. 总结

skopeo 在多个 cr 中操作镜像还是比较方便的: 针对私有化部署场景, 如果是内网 cr, 则可以使用 copy 或者 sync 命令将镜像保存在本地, 然后再从本地转存到内网 cr, 但是要特别注意本地文件目录要干净且不要覆盖; 如果 cr 可访问则可以直接使用 cr 到 cr 的方式同步镜像.

一般情况下我们会有的放矢的同步镜像, 中小型的项目镜像也不会太多, 即使很多也可以写 Shell 脚本, 所以综合考虑个人觉得 copy 比 sync 更好用, 因为语法上也更简洁和清晰, 限制和错误也更少.
