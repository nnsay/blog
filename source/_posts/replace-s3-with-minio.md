---
title: 使用minio替换s3
date: 2024-01-05 19:58:25
tags:
  - 研究
excerpt: 使用minio替换s3的可行性
---

# 1. 探求 minio 和 s3 兼容性

调研使用`minio`替换`s3`的可行性分析. 根据现有业务特点测试内容如下:

- 对象操作: 可以使用现有的 s3 sdk 操作 minio 文件对象
  - putObject
  - deleteObject
  - copyObject
  - getObject
  - headObject
  - presignedUrl
- 对象事件: minio 的 bucket 可以添加事件监控对象操作
  - 上传
  - 删除
- awscli: 可以使用现有 s3 工具(awscli)操作 minio
- 生命周期: 可以设置生命周期, 例如自动过期删除

以上操作如可以通过 aws s3 sdk 和其他工具顺利操作 minio, 则证明可行.

# 2. 搭建环境

使用 docker-compose 启动搭建环境, 有两个主要的服务:

- minio 服务
- webhook 服务: 通过调研 minio 支持 Bucket 通知事件, 通知对象有很多种: AMQP/MQTT/NATS/Kafka/Mysql/PostgreSQL/Redis/Webhook 等, 这里选择 webhook

docker-compose.yaml 如下:

```yaml
version: "3.8"
services:
  ### minio (https://hub.docker.com/r/minio/minio)
  minio:
    container_name: minio
    image: minio/minio
    command: server /miniodata --console-address ":9001"
    ports:
      - "9000:9000"
      - "9001:9001"
    restart: always
    volumes:
      - data:/miniodata
  miniowebhook:
    container_name: miniowebhook
    build: /tmp/minio # 这里需要指定合适的webhook服务的源码地址
    image: local/miniowebhook
    command: node webhook.js
volumes:
  data:
```

## 2.1 webhook 服务

因为 minio 部署在 docker 中, 为了 webhook 可以访问方便, 所以这里也把 webhook 服务部署到 docker, 让 wehhook 容器和 minio 容器在同一个网络下. webhook 相关代码如下:

- webhook.js

  ```js
  const express = require("express");
  const jwt = require("jsonwebtoken");

  const app = express();
  app.use(express.json()); // for parsing application/json
  app.use(express.urlencoded({ extended: true }));
  const port = 6000;
  const secret = "minio webhook secret";

  app.post("/webhook", (req, res) => {
    if (req.headers.authorization?.startsWith("Bearer")) {
      const token = req.headers.authorization.split(" ")[1];
      // 验证token
      const data = jwt.verify(token, secret);
      console.log("verify data: ", data);
      // 获取参数, 例如删除对象事件如下:
      /**
       * 上传事件:
       * {"EventName":"s3:ObjectCreated:Put","Key":"vd-main/Tile_+002_+002.mtl","Records":[{"eventVersion":"2.0","eventSource":"minio:s3","awsRegion":"","eventTime":"2024-01-05T11:26:33.408Z","eventName":"s3:ObjectCreated:Put","userIdentity":{"principalId":"minioadmin"},"requestParameters":{"principalId":"minioadmin","region":"","sourceIPAddress":"192.168.65.1"},"responseElements":{"x-amz-id-2":"dd9025bab4ad464b049177c95eb6ebf374d3b3fd1af9251148b658df7ac2e3e8","x-amz-request-id":"17A76FE05CEBA939","x-minio-deployment-id":"c56dc51b-4b0a-41c0-ab16-7b990c2d51ad","x-minio-origin-endpoint":"http://172.19.0.2:9000"},"s3":{"s3SchemaVersion":"1.0","configurationId":"Config","bucket":{"name":"vd-main","ownerIdentity":{"principalId":"minioadmin"},"arn":"arn:aws:s3:::vd-main"},"object":{"key":"Tile_%2B002_%2B002.mtl","size":210,"eTag":"775ff5def9531772a2bb12853cd46fce","contentType":"application/octet-stream","userMetadata":{"content-type":"application/octet-stream"},"sequencer":"17A76FE05CF3469B"}},"source":{"host":"192.168.65.1","port":"","userAgent":"MinIO (linux; arm64) minio-go/v7.0.66 MinIO Console/(dev)"}}]}
       * 删除事件:
       * {"EventName":"s3:ObjectRemoved:Delete","Key":"vd-main/Tile_+002_+002.mtl","Records":[{"eventVersion":"2.0","eventSource":"minio:s3","awsRegion":"","eventTime":"2024-01-05T11:27:59.964Z","eventName":"s3:ObjectRemoved:Delete","userIdentity":{"principalId":"minioadmin"},"requestParameters":{"principalId":"minioadmin","region":"","sourceIPAddress":"127.0.0.1"},"responseElements":{"content-length":"164","x-amz-id-2":"dd9025bab4ad464b049177c95eb6ebf374d3b3fd1af9251148b658df7ac2e3e8","x-amz-request-id":"17A76FF48418FD2E","x-minio-deployment-id":"c56dc51b-4b0a-41c0-ab16-7b990c2d51ad","x-minio-origin-endpoint":"http://172.19.0.2:9000"},"s3":{"s3SchemaVersion":"1.0","configurationId":"Config","bucket":{"name":"vd-main","ownerIdentity":{"principalId":"minioadmin"},"arn":"arn:aws:s3:::vd-main"},"object":{"key":"Tile_%2B002_%2B002.mtl","sequencer":"17A76FF4843A978E"}},"source":{"host":"127.0.0.1","port":"","userAgent":"MinIO (linux; arm64) minio-go/v7.0.66 MinIO Console/(dev)"}}]}
       */
      console.log(JSON.stringify(req.body));
      // TODO: 添加自定义业务
    }
    res.end();
  });

  app.listen(port, () => {
    const token = jwt.sign(
      {
        user: 1,
      },
      secret
    );
    console.log(`token: ${token}`);
    console.log(`webhook start at ${port}`);
  });
  ```

- Dockerfile

  ```dockerfile
  from node

  WORKDIR /app
  COPY ./webhook.js ./webhook.js
  RUN npm install express jsonwebtoken

  CMD node webhook.js

  ```

> ⚠️ 注意: 这两个文件的位置要和 docker-compose.yaml 中的 miniowebhook build 地址一样

## 2.2 启动服务

```
docker compose up -d
```

# 3. 测试

## 3.1 测试对象操作

使用 typescript 脚本测试对象的各种操作:

```ts
import { PutObjectCommand, GetObjectCommand, S3 } from "@aws-sdk/client-s3";
import { getSignedUrl } from "@aws-sdk/s3-request-presigner";
import axios from "axios";

const bucket = "vd-main";
// HINT: 该配置来自.mc/config.json中的一个minio配置, 其中AK/SK可以配置为用户名和密码
// HINT: AK/SK也可以从 http://localhost:9001/access-keys 中生成一份
const minioConfig = {
  url: "http://localhost:9000",
  accessKey: "x01paW1tNk9pbim9VH2D",
  secretKey: "D76Xc3aJ1yVrTcYPVkm6pOnAclBcCXo7YgRIcFlK",
  api: "S3v4",
  path: "auto",
};

const exec = async () => {
  const s3 = new S3({
    credentials: {
      accessKeyId: minioConfig.accessKey,
      secretAccessKey: minioConfig.secretKey,
    },
    endpoint: minioConfig.url,
    forcePathStyle: true,
  });

  // 创建/修改
  console.log(">>>>>>> putObject <<<<<<<");
  const putResult = await s3.putObject({
    Bucket: bucket,
    Key: "putobject.json",
    Body: JSON.stringify({ hello: "中国" }),
  });
  console.log("put object result: ", putResult.ETag);
  // 复制
  console.log(">>>>>>> copyObject <<<<<<<");
  const copyResult = await s3.copyObject({
    Bucket: bucket,
    Key: "putobject-copy.json",
    CopySource: `${bucket}/putobject.json`,
  });
  console.log("copy object result: ", copyResult.CopyObjectResult?.ETag);

  // put signed url
  console.log(">>>>>>> put signed url <<<<<<<");
  const putSignedURL = await getSignedUrl(
    s3,
    new PutObjectCommand({
      Bucket: bucket,
      Key: "signedurlobject.json",
    }),
    {
      expiresIn: 600,
    }
  );
  console.log("put signed url result: ", putSignedURL);

  console.log(">>>>>>> upload by signedurl <<<<<<<");
  await axios.put(putSignedURL, { data: JSON.stringify({ hello: "Jimmy" }) });

  console.log(">>>>>>> get signed url <<<<<<<");
  const getSignedURL = await getSignedUrl(
    s3,
    new GetObjectCommand({
      Bucket: bucket,
      Key: "signedurlobject.json",
    }),
    {
      expiresIn: 600,
    }
  );
  console.log("get signed url result: ", getSignedURL);

  console.log(">>>>>>> download by signedurl <<<<<<<");
  const getSignedURLResult = await axios.get(getSignedURL);
  console.log("download by signedurl result: ", getSignedURLResult.data);

  // 读取
  console.log(">>>>>>> getObject <<<<<<<");
  const getResult = await s3.getObject({
    Bucket: bucket,
    Key: "signedurlobject.json",
  });
  console.log(
    "get object(signedurl upload) result: %s",
    await getResult.Body?.transformToString("utf8")
  );

  // 删除
  console.log(">>>>>>> deleteObject <<<<<<<");
  const deleteResult = await s3.deleteObject({
    Bucket: bucket,
    Key: "putobject-copy.json",
  });
  console.log(
    "delete object result: %s",
    deleteResult.$metadata.httpStatusCode
  );

  await s3
    .headObject({
      Bucket: bucket,
      Key: "putobject-copy.json",
    })
    .catch((err) => {
      console.log("head object result: %s", err.message);
    });
};

exec()
  .then(() => {
    console.log("done");
  })
  .catch((err) => {
    console.log(err);
  });
```

运行该脚本查看测试结果是否 OK.

## 3.2 测试对象事件

这里需要一些操作: 1. 添加事件; 2. 将事件绑定到 bucket. 可以通过控制台配置, 配置方式如下:

![image-20240105194545975](https://raw.githubusercontent.com/nnsay/gist/main/imgimage-20240105194545975.png)

> ‼️ 注意: Auth Token 可以自己使用`jsonwebtoken`生成一个不过期的 jwt token, 用于在 webhook 服务中验证来源是否合法

![image-20240105194626214](https://raw.githubusercontent.com/nnsay/gist/main/imgimage-20240105194626214.png)

手动上传文件/删除文件查看 webhook 容器日志检查事件是否 OK

## 3.2 aws-cli 操作

添加 aws profile:

```bash
[default]
region=cn-northwest-1
outputh=json
cli_pager=
endpoint_url=http://localhost:9000
```

添加 credentials:

```bash
[default]
aws_access_key_id=x01paW1tNk9pbim9VH2D
aws_secret_access_key=D76Xc3aJ1yVrTcYPVkm6pOnAclBcCXo7YgRIcFlK
```

> ℹ️ AK/SK 也可以从 http://localhost:9001/access-keys 中生成一份

测试 awscli 命令如下:

```bash
-> aws s3 ls
2024-01-05 16:40:20 vd-main

-> aws s3 ls s3://vd-main --recursive --summarize
2024-01-05 18:01:12         18 putobject.json
2024-01-05 18:01:12         32 signedurlobject.json

Total Objects: 2
   Total Size: 50
```

## 3.3 生命周期

通过控制台配置生命周期:

![image-20240105201754664](https://raw.githubusercontent.com/nnsay/gist/main/imgimage-20240105201754664.png)

# 4. 总结

minio 基本上可以替换 s3, 并且在 s3 对象操作和事件上也有比较好的兼容, 推荐在云下使用.
