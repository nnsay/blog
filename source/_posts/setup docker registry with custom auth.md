---
title: 搭建自定义授权的Docker Registory
date: 2021-04-16 15:43:47
tags:
---

[toc]
### 1. 鉴权介绍
自建docker registry比较简单, 但是docker registry的鉴权如何能和现有的系统结合就很优雅了, 我们可以自定义一些规则允许/拒绝某些账户拉取和推送或者只推送. 实现的基本思路:
- 自定义一个docker registry镜像, 可以基于现有官方镜像, 在定义镜像中指定鉴权服务地址
- 自定一个鉴[toc]
### 1. 鉴权介绍
自建docker registry比较简单, 但是docker registry的鉴权如何能和现有的系统结合就很优雅了, 我们可以自定义一些规则允许/拒绝某些账户拉取和推送或者只推送. 实现的基本思路:
- 自定义一个docker registry镜像, 可以基于现有官方镜像, 在定义镜像中指定鉴权服务地址
- 自定一个鉴权的镜像, 在这个镜像中可以包含自定义的鉴权逻辑和仓库的ACL
- 部署上面两个服务


### 2. 自定义Docker Registory
```
FROM centos:7.6.1810

RUN mkdir -p /var/lib/registry

EXPOSE 5000

ENV REGISTRY_AUTH token
ENV REGISTRY_AUTH_TOKEN_REALM https://${DOMAIN}/auth
ENV REGISTRY_AUTH_TOKEN_SERVICE "Docker registry"
ENV REGISTRY_AUTH_TOKEN_ISSUER "Auth Service"
ENV REGISTRY_AUTH_TOKEN_ROOTCERTBUNDLE /ssl/domain.crt
ENV REGISTRY_HTTP_TLS_CERTIFICATE /ssl/domain.crt
ENV REGISTRY_HTTP_TLS_KEY /ssl/domain.key
ENV REGISTRY_HTTP_SECRET w87reoqweifgqoe

COPY config/config.yml /etc/docker/registry/config.yml
COPY ssl/ /ssl/
COPY ./cert/digicert-ng.pem.trust /etc/pki/ca-trust/source/anchors/digicert-ng.pem
RUN update-ca-trust extract

CMD ["/bin/registry", "serve", "/etc/docker/registry/config.yml"]
```

### 3. 自定义权限镜像(重点)

#### 3.1 镜像
这里用到了一个开源的功能很全的基础镜像: [cesanta/docker_auth](https://github.com/cesanta/docker_auth), 为了理解镜像配置需要学习这个镜像的基本知识.
```
FROM cesanta/docker_auth as cesanta
FROM python:2.7.18-alpine
COPY --from=cesanta /docker_auth/ /docker_auth/

ARG CICD_SECRET
ENV CICD_SECRET=${CICD_SECRET}

ENTRYPOINT ["/docker_auth/auth_server"]
CMD ["-alsologtostderr=true", "-log_dir=/logs", "/config/extAuth.yml"]
EXPOSE 5001

COPY config/ /config/
COPY ssl/ /ssl/
COPY extensions/ /extensions/
```

#### 3.2 镜像配置
上面的镜像中有两个配置config/extensions
- [config配置参](https://github.com/cesanta/docker_auth/blob/main/examples/reference.yml), 如下配置配置了auth和registry通信是https的证书,token的issuer, 扩展授权方式ext_auth, 还有acl, 这些都是关键的配置
```yaml
server: # Server settings.
  addr: ":5001"

  certificate: "/ssl/domain.crt"
  key: "/ssl/domain.key"

token: # Settings for the tokens.
  issuer: "Auth Service" # Must match issuer in the Registry config.
  expiration: 900
ext_auth:
  command: "/extensions/ext_auth.sh" # Can be a relative path too; $PATH works.
  args: [""]
acl:
  - match: { ip: "127.0.0.0/8" }
    actions: ["*"]
    comment: "Allow everything from localhost (IPv4)"
  - match: { account: "/.+/", type: "registry", name: "catalog" }
    actions: ["*"]
    comment: "Logged in users can query the catalog."
  - match: { labels: { "group": "CICD" } }
    actions: ["*"]
    comment: "User assigned to group 'CICD' is able to push"
  - match: { account: "/.+/" }
    actions: ["pull"]
    comment: "Logged in users can pull all images."
```
- extensions配置是基于上面的配置的ext_auth,例如是个脚本:
```sh
read u p

if [ "$u" == "cicd" ]; then
  if [ "$p" == "$CICD_SECRET" ]; then
    echo '{"labels": {"group": ["CICD"]}}'
    exit 0
  fi
  exit 1
fi

python /extensions/boxlogin.py "$u" "$p"

exit $?
```

#### 3.3 鉴权镜像项目目录

```
.
├── Dockerfile
├── build.sh
├── config
│   └── extAuth.yml
├── extensions
│   ├── boxlogin.py
│   └── ext_auth.sh
└── ssl
    ├── domain.crt
    ├── domain.csr
    └── domain.key
```
ext_auth.sh是配置所需要的shell脚本, 脚本不要配置标准输出, 退出的code是0或者1. boxlogin.py的Python脚本可以自定更灵活的逻辑, 比如发送Restful请求到现有服务, 判断登录是否成功, 成功则exit(0),否则exit(1).

### 4. 部署

有了镜像部署就简单了:
- 部署registry并暴露一个访问地址
- 部署auth并部署并暴露一个访问地址
- 两个访问地址IP或者域名一致,但是auth的端口是5001
部署完了保证两个服务各自都可以访问, 可以通过如下命令分别查看两个服务是否正常:
```
curl https://my.registory.cn/v2/
curl https://my.registory.cn:5001/auth
```

### 5. Troubleshooting

- [docker login returns 400 Bad Request](https://github.com/goharbor/harbor/issues/7159) 
如果用了负载均衡或者代理需要注意代理的后端必须是https, 可以参考[issue](https://github.com/goharbor/harbor/issues/7159)
- [auth failure](https://github.com/cesanta/docker_auth#troubleshooting)
可以把auth服务的日志级别调高,[显示更多日志](https://github.com/cesanta/docker_auth#troubleshooting)