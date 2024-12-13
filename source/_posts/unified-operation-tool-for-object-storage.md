---
title: 统一文件对象存储操作
date: 2024-03-27 14:24:22
tags:
  - Tips
  - Code
excerpt: 使用AWS S3 SDK统一操作对象存储
---

# 1. 痛点

一个微服支持不同的原厂商提供的文件存储, 假设我们的服务要适配:

- AWS S3
- Aliyun OSS
- Minio

一般情况下我们会在服务中因为不同厂商的 SDK 进行文件对象操作, 那么此时要引入三种 SDK, 其次为了不在代码中写很多 If-Else 我们需要抽象为工具包, 业务中使用工具包而不是直接使用 SDK 进行操作. 这就使得代码仓库依赖变多体积变大. 如果和解决这个问题, 是否有统一化的文件对象操作?

# 2. 事实上统一兼容性

如果仔细查看阿里云/腾讯云/Minio 的文档, 我们可以得到结论: S3 API 已经成为实际上的文件对象操作标准. 各个厂商都对自己的文件操作服务做了 S3 API 兼容性操作, 具体可以参考一下文档:

- [OSS 兼容 S3 API 说明](https://help.aliyun.com/zh/oss/developer-reference/compatibility-with-amazon-s3)
- [AWS SDK 操作 Aliyun OSS](https://help.aliyun.com/zh/oss/developer-reference/use-amazon-s3-sdks-to-access-oss)
- [MINIO AWS S3 Compatibility](https://min.io/product/s3-compatibility)
- [使用 AWS S3 SDK 访问 COS](https://cloud.tencent.com/document/product/436/37421#node.js)

经过文档的调研分析我们可以进一步总结: S3/OSS/Minio/COS 在绝大部分文件操作上有统一性. 所以要解决痛点问题, 可以尝试使用 AWS S3 SDK 作为统一的文件对象操作工具.

# 3. 代码实践

验证的文件操作有:

- GetObject
- PutObject
- CopyObject
- HeadObject
- DeleteObject
- XxxSignedUrl

除了上面的常用 API 以外, 根据阿里云的文兼容性说明文档, 分片操作也是兼容的, 但是不在本文测试验证的代码中, Typescript 测试验证代码如下:

```typescript
import {
  S3Client,
  PutObjectCommand,
  CopyObjectCommand,
  GetObjectCommand,
  DeleteObjectCommand,
  HeadObjectCommand,
} from "@aws-sdk/client-s3";
import { fromIni } from "@aws-sdk/credential-provider-ini";
import { getSignedUrl } from "@aws-sdk/s3-request-presigner";
import axios from "axios";
import fs from "fs";

enum OSSProviderType {
  S3,
  OSS,
  MINIO,
}
const getClient = (providerType = OSSProviderType.S3) => {
  if (providerType === OSSProviderType.OSS) {
    const aliConfig: {
      profiles: { access_key_id: string; access_key_secret: string }[];
    } = JSON.parse(
      fs.readFileSync(`${process.env.HOME}/.aliyun/config.json`).toString()
    );
    return new S3Client({
      region: "oss-cn-hangzhou",
      credentials: {
        accessKeyId: aliConfig.profiles[0].access_key_id,
        secretAccessKey: aliConfig.profiles[0].access_key_secret,
      },
      endpoint: "https://oss-cn-hangzhou.aliyuncs.com",
      apiVersion: "2006-03-01",
    });
  } else if (providerType === OSSProviderType.MINIO) {
    const minioConfig: {
      aliases: {
        [k: string]: {
          url: string;
          accessKey: string;
          secretKey: string;
          api: string;
          path: string;
        };
      };
    } = JSON.parse(
      fs.readFileSync(`${process.env.HOME}/.mc/config.json`).toString()
    );
    return new S3Client({
      credentials: {
        accessKeyId: minioConfig.aliases.local.accessKey,
        secretAccessKey: minioConfig.aliases.local.secretKey,
      },
      endpoint: minioConfig.aliases.local.url,
      forcePathStyle: true,
    });
  }
  return new S3Client({
    region: "cn-northwest-1",
    credentials: fromIni({ profile: "sandbox" }),
  });
};

const exec = async () => {
  // get object storage client for AWS S3 or Aliyun OSS
  const client = getClient(OSSProviderType.MINIO);
  const bucket = "nnsay-cn";

  // 创建/修改
  console.log(">>>>>>> putObject <<<<<<<");
  const putResult = await client.send(
    new PutObjectCommand({
      Bucket: bucket,
      Key: "putobject.json",
      Body: "hello",
    })
  );
  console.log("put object result: ", putResult.ETag);
  // 复制
  console.log(">>>>>>> copyObject <<<<<<<");
  const copyResult = await client.send(
    new CopyObjectCommand({
      Bucket: bucket,
      Key: "putobject-copy.json",
      CopySource: `${bucket}/putobject.json`,
    })
  );
  console.log("copy object result: ", copyResult.CopyObjectResult?.ETag);

  // put signed url
  console.log(">>>>>>> put signed url <<<<<<<");
  const putSignedURL = await getSignedUrl(
    client,
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
  await axios.put(putSignedURL, JSON.stringify({ hello: "Jimmy" }));

  console.log(">>>>>>> get signed url <<<<<<<");
  const getSignedURL = await getSignedUrl(
    client,
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
  const getResult = await client.send(
    new GetObjectCommand({
      Bucket: bucket,
      Key: "signedurlobject.json",
    })
  );
  console.log(
    "get object(signedurl upload) result: %s",
    await getResult.Body?.transformToString("utf8")
  );

  // 删除
  console.log(">>>>>>> deleteObject <<<<<<<");
  const deleteResult = await client.send(
    new DeleteObjectCommand({
      Bucket: bucket,
      Key: "putobject-copy.json",
    })
  );
  console.log(
    "delete object result: %s",
    deleteResult.$metadata.httpStatusCode
  );

  // Head
  const headResult = await client.send(
    new HeadObjectCommand({
      Bucket: bucket,
      Key: "putobject.json",
    })
  );

  console.log("head object result: %s", headResult.ETag);
};

exec()
  .then(() => {
    console.log("done");
  })
  .catch((err) => {
    console.log(err);
  });
```

同理, 我们验证一下 Python SDK 具体代码如下:

```Python
from io import BufferedReader
from typing import Union

import boto3
from botocore.config import Config

default_signed_url_expires = 3600


class ObjectStorage:
  def __init__(self, provider: str, config: dict[str, Union[str, None]]):
    if provider == 'OSS':
      self.client = boto3.client(
        's3',
        aws_access_key_id=config['access_key_id'],
        aws_secret_access_key=config['access_key_secret'],
        endpoint_url=f"https://{config['region']}.aliyuncs.com",
        config=Config(s3={'addressing_style': 'virtual', 'signature_version': 's3v4'}),
      )
    elif provider == 'MINIO':
      self.client = boto3.client(
        's3',
        aws_access_key_id=config['access_key_id'],
        aws_secret_access_key=config['access_key_secret'],
        endpoint_url=config['endpoint'],
        config=Config(s3={'addressing_style': 'path', 'signature_version': 's3v4'}),
      )
    else:
      self.client = boto3.client('s3')

  def download_file(self, local_file: str, bucket: str, key: str):
    with open(local_file, 'wb') as data:
      self.client.download_fileobj(Bucket=bucket, Key=key, Fileobj=data)

  def put_object(self, bucket: str, key: str, body: Union[bytes, str, BufferedReader]):
    res = self.client.put_object(Bucket=bucket, Key=key, Body=body)
    return res['ETag']

  def get_object(self, bucket: str, key: str):
    res = self.client.get_object(Bucket=bucket, Key=key)
    return res['Body'].read()

  def get_object_signed_url(self, bucket: str, key: str, expires: int = default_signed_url_expires):
    return self.client.generate_presigned_url(
      'get_object', Params={'Bucket': bucket, 'Key': key}, ExpiresIn=expires
    )
```

> [!NOTE] > [boto3 s3 文档](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/s3.html)

> [!NOTE]
> 该包已开源: Github 项目[python-object-storage](https://github.com/nnsay/python-object-storage), PyPI 包 [nnsay.object-storage](https://pypi.org/project/nnsay.object-storage/)

测试验证代码如下:

```python
bucket = 'nnsay-cn'
provider = os.environ.get("OBJECT_STORAGE_PROVIDER", "OSS")
config = {
    "access_key_id": os.environ.get("OBJECT_STORAGE_AK"),
    "access_key_secret": os.environ.get("OBJECT_STORAGE_SK"),
    "region": os.environ.get("OBJECT_STORAGE_REGION"),
    "endpoint": os.environ.get("OBJECT_STORAGE_ENDPOINT")
}
object_storage = ObjectStorage(provider, config)

# 上传操作
with open('hello.log', 'rb') as data:
    etag = object_storage.put_object(bucket, 'hello.log', data)
    print(f"etag: {etag}")

# 下载操作
object_storage.download_file('download-hello-minio.log', bucket, 'hello.log')

# 获取对象
bytes = object_storage.get_object(bucket, 'hello.log')
data = bytes.decode("utf-8")
print(f"get data: {data}")

# 获取下载地址
url = object_storage.get_object_signed_url(bucket, 'hello.log')
print(f"download signed url: {url}")
```

# 4. 总结

以上代码已经测试验证通过, 可以证明: AWS S3 SDK 可以在绝大部分文件操作中作为统一的文件操作工具使用.

以上代码中其中 Typescript 并没有封装成类, 在实践中可以可靠具体的封装, 比如使用 Nest 框架时封装为 Nest Service; 而 Python 的简单封装已经发布到了 PyPI

本文也提及了腾讯云 COS 也对 S3 API 有兼容, 但是因为没有测试环境所以代码中没有封装和测试, 感兴趣的同学可以自行尝试.
