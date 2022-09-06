---
title: acme.sh
date: 2022-09-06 12:04:12
tags:
  - Aliyun
  - Tips
  - 研究
excerpt: 使用证书管理客户端(acme.sh)快速管理和维护证书
---

# 1. 介绍

证书管理客户端常见的有很多, 具体可以查看: [ACME客户端](https://letsencrypt.org/zh-cn/docs/client-options/)列表, 社区中比较流行有以下两个:

- [acme.sh](https://github.com/acmesh-official/acme.sh)
- [certbot](https://github.com/certbot/certbot)

certbot在国内使用不顺畅, 主要是不支持国内厂商的一些权限验证比如阿里云的DNS权限验证, 更重要的是如果无法使用DNS授权则不能自动更新证书, 只能手动验证或者结合[社区方案](https://github.com/tengattack/certbot-dns-aliyun); cerbot需要root权限, 而acme.sh不需要; cerbot是letencrypt推荐的客户端, 但是acme.sh不仅支持letencrypt还支持别的CA; 个人在简单体验后在自定义程度上acme.sh更好; 综上所以本教程主要介绍acme.sh

# 2. 安装

```bash
git clone https://github.com/acmesh-official/acme.sh.git
cd ./acme.sh
./acme.sh --install -m jimmy.w@aliyun.com
```

# 3. 生成证书: 手动DNS TXT验证

[使用DNS TXT记录方式验证域名](https://github.com/acmesh-official/acme.sh/wiki/dns-manual-mode)

- 生成TXT记录

  ```bash
  ➜  ~ acme.sh --issue --domain exmaple.cn --dns --yes-I-know-dns-manual-mode-enough-go-ahead-please --server letsencrypt 
  [Mon Sep  5 17:51:04 CST 2022] Using CA: https://acme-v02.api.letsencrypt.org/directory
  [Mon Sep  5 17:51:04 CST 2022] Create account key ok.
  [Mon Sep  5 17:51:04 CST 2022] Registering account: https://acme-v02.api.letsencrypt.org/directory
  [Mon Sep  5 17:51:08 CST 2022] Registered
  [Mon Sep  5 17:51:08 CST 2022] ACCOUNT_THUMBPRINT='Rw3wugyTUzVuXoVI5n1lnbRk16iauaTg4-aG2KxQe4k'
  [Mon Sep  5 17:51:08 CST 2022] Creating domain key
  [Mon Sep  5 17:51:08 CST 2022] The domain key is here: /Users/jw/.acme.sh/exmaple.cn/exmaple.cn.key
  [Mon Sep  5 17:51:08 CST 2022] Single domain='exmaple.cn'
  [Mon Sep  5 17:51:08 CST 2022] Getting domain auth token for each domain
  [Mon Sep  5 17:51:12 CST 2022] Getting webroot for domain='exmaple.cn'
  [Mon Sep  5 17:51:12 CST 2022] Add the following TXT record:
  [Mon Sep  5 17:51:12 CST 2022] Domain: '_acme-challenge.exmaple.cn'
  [Mon Sep  5 17:51:12 CST 2022] TXT value: 'gj74Nh8vl8wkSyWeHwXHVldS2wzPF5laLiytCYxt6x8'
  [Mon Sep  5 17:51:12 CST 2022] Please be aware that you prepend _acme-challenge. before your domain
  [Mon Sep  5 17:51:12 CST 2022] so the resulting subdomain will be: _acme-challenge.exmaple.cn
  [Mon Sep  5 17:51:12 CST 2022] Please add the TXT records to the domains, and re-run with --renew.
  [Mon Sep  5 17:51:12 CST 2022] Please add '--debug' or '--log' to check more details.
  [Mon Sep  5 17:51:12 CST 2022] See: https://github.com/acmesh-official/acme.sh/wiki/How-to-debug-acme.sh
  ```

  - [server](https://github.com/acmesh-official/acme.sh/wiki/Server): 支持很多种server, 默认是zerossl,国内网络不稳定, 使用letsencrypt较好, 也可以设置默认的CA

    ```bash
    acme.sh --set-default-ca --server letsencrypt
    ```

    

- 手动添加TXT

- 验证TXT, 生成证书

  ```bash
  ➜  ~ acme.sh --issue --domain exmaple.cn --dns --yes-I-know-dns-manual-mode-enough-go-ahead-please --server letsencrypt --renew
  [Mon Sep  5 17:54:03 CST 2022] Renew: 'exmaple.cn'
  [Mon Sep  5 17:54:03 CST 2022] Renew to Le_API=https://acme-v02.api.letsencrypt.org/directory
  [Mon Sep  5 17:54:05 CST 2022] Using CA: https://acme-v02.api.letsencrypt.org/directory
  [Mon Sep  5 17:54:05 CST 2022] Single domain='exmaple.cn'
  [Mon Sep  5 17:54:05 CST 2022] Getting domain auth token for each domain
  [Mon Sep  5 17:54:05 CST 2022] Verifying: exmaple.cn
  [Mon Sep  5 17:54:08 CST 2022] Pending, The CA is processing your order, please just wait. (1/30)
  [Mon Sep  5 17:54:13 CST 2022] Success
  [Mon Sep  5 17:54:13 CST 2022] Verify finished, start to sign.
  [Mon Sep  5 17:54:13 CST 2022] Lets finalize the order.
  [Mon Sep  5 17:54:13 CST 2022] Le_OrderFinalize='https://acme-v02.api.letsencrypt.org/acme/finalize/717834657/122698486827'
  [Mon Sep  5 17:54:15 CST 2022] Downloading cert.
  [Mon Sep  5 17:54:15 CST 2022] Le_LinkCert='https://acme-v02.api.letsencrypt.org/acme/cert/046d26bb81d66ad65eaba74a41d1c7944cdd'
  [Mon Sep  5 17:54:16 CST 2022] Cert success.
  -----BEGIN CERTIFICATE-----
  MIIFFTCCA/2gAwIBAgISBG0mu4HWatZeq6dKQdHHlEzdMA0GCSqGSIb3DQEBCwUA
  ...
  YMHtkQUEV0zV
  -----END CERTIFICATE-----
  [Mon Sep  5 17:54:16 CST 2022] Your cert is in: /Users/jw/.acme.sh/exmaple.cn/exmaple.cn.cer
  [Mon Sep  5 17:54:16 CST 2022] Your cert key is in: /Users/jw/.acme.sh/exmaple.cn/exmaple.cn.key
  [Mon Sep  5 17:54:17 CST 2022] The intermediate CA cert is in: /Users/jw/.acme.sh/exmaple.cn/ca.cer
  [Mon Sep  5 17:54:17 CST 2022] And the full chain certs is there: /Users/jw/.acme.sh/exmaple.cn/fullchain.cer
  ```

  # 4. [生成证书: 使用DNS API验证](https://github.com/acmesh-official/acme.sh/blob/master/dnsapi/dns_ali.sh)

  ```bash
  export Ali_Key="access key id"
  export Ali_Secret="access key secret"
  
  ➜  ~ acme.sh --issue --dns dns_ali -d *.exmaple.cc --force
  [Mon Sep  5 19:53:24 CST 2022] Using CA: https://acme-v02.api.letsencrypt.org/directory
  [Mon Sep  5 19:53:24 CST 2022] Creating domain key
  [Mon Sep  5 19:53:24 CST 2022] The domain key is here: /Users/jw/.acme.sh/exmaple.cc/exmaple.cc.key
  [Mon Sep  5 19:53:24 CST 2022] Single domain='exmaple.cc'
  [Mon Sep  5 19:53:25 CST 2022] Getting domain auth token for each domain
  [Mon Sep  5 19:53:34 CST 2022] Getting webroot for domain='exmaple.cc'
  [Mon Sep  5 19:53:34 CST 2022] Adding txt value: WW37w1vmirOPCJcB0YlqAwQgnnc3UN84n7wAPp3dY84 for domain:  _acme-challenge.exmaple.cc
  [Mon Sep  5 19:53:39 CST 2022] The txt record is added: Success.
  [Mon Sep  5 19:53:39 CST 2022] Let's check each DNS record now. Sleep 20 seconds first.
  [Mon Sep  5 19:54:00 CST 2022] You can use '--dnssleep' to disable public dns checks.
  [Mon Sep  5 19:54:00 CST 2022] See: https://github.com/acmesh-official/acme.sh/wiki/dnscheck
  [Mon Sep  5 19:54:00 CST 2022] Checking exmaple.cc for _acme-challenge.exmaple.cc
  [Mon Sep  5 19:54:02 CST 2022] Domain exmaple.cc '_acme-challenge.exmaple.cc' success.
  [Mon Sep  5 19:54:02 CST 2022] All success, let's return
  [Mon Sep  5 19:54:02 CST 2022] Verifying: exmaple.cc
  [Mon Sep  5 19:54:05 CST 2022] Pending, The CA is processing your order, please just wait. (1/30)
  [Mon Sep  5 19:54:15 CST 2022] Success
  [Mon Sep  5 19:54:15 CST 2022] Removing DNS records.
  [Mon Sep  5 19:54:15 CST 2022] Removing txt: WW37w1vmirOPCJcB0YlqAwQgnnc3UN84n7wAPp3dY84 for domain: _acme-challenge.exmaple.cc
  [Mon Sep  5 19:54:23 CST 2022] Removed: Success
  [Mon Sep  5 19:54:23 CST 2022] Verify finished, start to sign.
  [Mon Sep  5 19:54:24 CST 2022] Lets finalize the order.
  [Mon Sep  5 19:54:24 CST 2022] Le_OrderFinalize='https://acme-v02.api.letsencrypt.org/acme/finalize/717834657/122722852227'
  [Mon Sep  5 19:54:30 CST 2022] Downloading cert.
  [Mon Sep  5 19:54:30 CST 2022] Le_LinkCert='https://acme-v02.api.letsencrypt.org/acme/cert/03986d5724d4c06ea2c89914b5de37b659cf'
  [Mon Sep  5 19:54:34 CST 2022] Cert success.
  -----BEGIN CERTIFICATE-----
  MIIFFzCCA/+gAwIBAgISA5htVyTUwG6iyJkUtd43tlnPMA0GCSqGSIb3DQEBCwUA
  ....
  EJjkA/OhSkleGqg=
  -----END CERTIFICATE-----
  [Mon Sep  5 19:54:34 CST 2022] Your cert is in: /Users/jw/.acme.sh/exmaple.cc/exmaple.cc.cer
  [Mon Sep  5 19:54:34 CST 2022] Your cert key is in: /Users/jw/.acme.sh/exmaple.cc/exmaple.cc.key
  [Mon Sep  5 19:54:34 CST 2022] The intermediate CA cert is in: /Users/jw/.acme.sh/exmaple.cc/ca.cer
  [Mon Sep  5 19:54:34 CST 2022] And the full chain certs is there: /Users/jw/.acme.sh/exmaple.cc/fullchain.cer
  ```

  对比手动方式:

  - 手动方式生成的证书服务自动renew, 而api方式可以renew证书

  # 5. 安装证书

  以NGINX为例:

  ```bash
  acme.sh --install-cert -d exmaple.cn --key-file ~/temp/nginx/exmaple.cn.key --fullchain-file ~/temp/nginx/fullchain.cer --reloadcmd "service nginx force-reload"
  ```

  - install-cert: 实际是拷贝证书
  - reloadcmd: 自定义命令, 通常是重启服务

  # 6. 自动刷新证书

  ```bash
  ➜  ~ acme.sh --cron --force
  [Mon Sep  5 20:00:14 CST 2022] ===Starting cron===
  [Mon Sep  5 20:00:14 CST 2022] Renew: 'exmaple.cc'
  [Mon Sep  5 20:00:14 CST 2022] Renew to Le_API=https://acme-v02.api.letsencrypt.org/directory
  [Mon Sep  5 20:00:19 CST 2022] Using CA: https://acme-v02.api.letsencrypt.org/directory
  [Mon Sep  5 20:00:19 CST 2022] Single domain='exmaple.cc'
  [Mon Sep  5 20:00:19 CST 2022] Getting domain auth token for each domain
  [Mon Sep  5 20:00:34 CST 2022] Getting webroot for domain='exmaple.cc'
  [Mon Sep  5 20:00:34 CST 2022] exmaple.cc is already verified, skip dns-01.
  [Mon Sep  5 20:00:34 CST 2022] Verify finished, start to sign.
  [Mon Sep  5 20:00:34 CST 2022] Lets finalize the order.
  [Mon Sep  5 20:00:34 CST 2022] Le_OrderFinalize='https://acme-v02.api.letsencrypt.org/acme/finalize/717834657/122724236097'
  [Mon Sep  5 20:00:36 CST 2022] Downloading cert.
  [Mon Sep  5 20:00:36 CST 2022] Le_LinkCert='https://acme-v02.api.letsencrypt.org/acme/cert/035a862e306aa2155f43a86b47c2af23389a'
  [Mon Sep  5 20:00:47 CST 2022] Cert success.
  -----BEGIN CERTIFICATE-----
  MIIFFjCCA/6gAwIBAgISA1qGLjBqohVfQ6hrR8KvIziaMA0GCSqGSIb3DQEBCwUA
  ...
  t0LWMLZT0U/Y1A==
  -----END CERTIFICATE-----
  [Mon Sep  5 20:00:47 CST 2022] Your cert is in: /Users/jw/.acme.sh/exmaple.cc/exmaple.cc.cer
  [Mon Sep  5 20:00:47 CST 2022] Your cert key is in: /Users/jw/.acme.sh/exmaple.cc/exmaple.cc.key
  [Mon Sep  5 20:00:47 CST 2022] The intermediate CA cert is in: /Users/jw/.acme.sh/exmaple.cc/ca.cer
  [Mon Sep  5 20:00:47 CST 2022] And the full chain certs is there: /Users/jw/.acme.sh/exmaple.cc/fullchain.cer
  [Mon Sep  5 20:00:48 CST 2022] Renew: 'exmaple.cn'
  [Mon Sep  5 20:00:48 CST 2022] Renew to Le_API=https://acme-v02.api.letsencrypt.org/directory
  [Mon Sep  5 20:00:48 CST 2022] It seems that you are using dns manual mode. Read this link first: https://github.com/acmesh-official/acme.sh/wiki/dns-manual-mode
  [Mon Sep  5 20:00:48 CST 2022] Error renew exmaple.cn.
  [Mon Sep  5 20:00:48 CST 2022] ===End cron===
  ```

  # 7. acme更新

  acme更新是为了保证和CA更新同步, 应该时常做

  - 手动升级

    ```bash
    ➜  ~ acme.sh --upgrade
    [Tue Sep  6 09:33:08 CST 2022] Installing from online archive.
    [Tue Sep  6 09:33:08 CST 2022] Downloading https://github.com/acmesh-official/acme.sh/archive/master.tar.gz
    [Tue Sep  6 09:33:09 CST 2022] Extracting master.tar.gz
    [Tue Sep  6 09:33:10 CST 2022] It is recommended to install socat first.
    [Tue Sep  6 09:33:10 CST 2022] We use socat for standalone server if you use standalone mode.
    [Tue Sep  6 09:33:10 CST 2022] If you don't use standalone mode, just ignore this warning.
    [Tue Sep  6 09:33:10 CST 2022] Installing to /Users/jw/.acme.sh
    [Tue Sep  6 09:33:10 CST 2022] Installed to /Users/jw/.acme.sh/acme.sh
    [Tue Sep  6 09:33:10 CST 2022] Good, bash is found, so change the shebang to use bash as preferred.
    [Tue Sep  6 09:33:12 CST 2022] OK
    [Tue Sep  6 09:33:12 CST 2022] Install success!
    [Tue Sep  6 09:33:13 CST 2022] Upgrade success!
    ```

  - 自动升级

    ```
    acme.sh --upgrade  --auto-upgrade
    ```

    - --auto-upgrade [0|1]: Valid for '--upgrade' command, indicating whether to upgrade automatically in future. Defaults to 1 if argument is omitted.

  # 8. 删除证书

  - 删除定时任务

    ```bash
    acme.sh --uninstall-cronjob
    ```

  - 删除证书

    ```bash
    ➜  ~ acme.sh --remove --domain exmaple.cn 
    [Tue Sep  6 09:44:49 CST 2022] exmaple.cn is removed, the key and cert files are in /Users/jw/.acme.sh/exmaple.cn
    [Tue Sep  6 09:44:49 CST 2022] You can remove them by yourself.
    ```

  - 清除安装

    包括acme自身和cron job

    ```bash
    acme.sh --uninstall
    ```

  