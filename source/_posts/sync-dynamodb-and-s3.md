---
title: DynamoDB和S3数据同步
date: 2023-10-26 15:56:42
tags:
  - Code
  - 研究
  - DevOps
excerpt: 优雅同步DynamoDB和S3数据
---

# 1. 业务场景

最近因为微服务迁移的需要需要将现有微服务中的 DynamoDB Table 和 S3 Bucket 做一个迁移, 新老资源结构不变只需要从一个位置迁移到另外一个位置. 因为是在线系统所以在设计方案时需要考虑以下两点:

- 实时数据迁移
- 历史数据迁移

实时数据迁移优先上线, 历史数据迁移可以慢慢同步, 等到历史数据同步结束, 新的微服务就可以使用新资源, 在同步时需要注意数据准确性和完整性.

# 2. 方案设计

## 2.1 历史数据同步

历史数据同步相对比较简单, 最笨的办法可以通过写程序来实现, 通过调研相关的技术文档详细的设计方法如下:

### 2.1.1 DynamoDB Table 历史数据同步: 编程

DynamoDB 在国内市场似乎用的比较少, 资料和工具都比较少, 经过调研决定使用编程方式解决, 单表同步逻辑如下:

```typescript
import {
  DynamoDBClient,
  ScanCommand,
  AttributeValue,
  PutItemCommand,
  ScanCommandOutput,
} from "@aws-sdk/client-dynamodb";
import { fromIni } from "@aws-sdk/credential-provider-ini";

export const syncTable = async (
  sourceTableName: string,
  targetTableName: string,
  envName: string
) => {
  const devCredential = fromIni({ profile: envName });
  const devClient = new DynamoDBClient({ credentials: devCredential });

  let lastEvaluatedKey: Record<string, AttributeValue> | undefined = {
    __default: { N: "default" },
  };

  // 循环获取所有数据
  do {
    const data: ScanCommandOutput = await devClient.send(
      new ScanCommand({
        TableName: sourceTableName,
        ExclusiveStartKey: lastEvaluatedKey?.__default
          ? undefined
          : lastEvaluatedKey,
      })
    );
    lastEvaluatedKey = data.LastEvaluatedKey;
    console.log(
      `${sourceTableName}--> ${targetTableName} lastEvaluatedKey: %j currentCount: %d`,
      lastEvaluatedKey,
      data.Count
    );
    // 同步
    const tasks =
      data.Items?.map((item) => {
        return devClient.send(
          new PutItemCommand({
            TableName: targetTableName,
            Item: item,
          })
        );
      }) || [];
    await Promise.all(tasks);
  } while (lastEvaluatedKey?.__default || lastEvaluatedKey);
};
```

上面的代码使用 Typescript 编写, 使用 DynamoDB 的 SDK, 使用 Scan 方法循环查询表直到所有的数据查找完; 另外在权限部分使用`fromIni`获取本地配置的 AWS Profile 权限.

### 2.1.2 S3 Bucket 历史数据同步: CLI

AWS CLI 提供的 `sync` 非常适合做文件迁移, 并且支持增量, 所以 S3 Bucket 的历史数据同步直接使用 Shell 脚本 + AWS CLI 即可, 代码片段如下:

```bash
#! /bin/bash
set -e
sourcebucket="xxx"
targetbucket="yyy"
profile=dev
echo "$(date +'%Y-%m-%d %H:%M:%S') 开始同步: $sourcebucket->$targetbucket"
aws s3 sync --only-show-errors s3://source-dev-dev s3://target-dev --profile=$profile
echo "$(date +'%Y-%m-%d %H:%M:%S') 结束同步: $sourcebucket->$targetbucket"
```

## 2.2 实时数据同步

实时数据同步是数据迁移方案不停服务无缝迁移的核心, 经过查看 AWS 的相关服务决定使用 DynamoDB Stream 和 S3 Event 来实现.

### 2.2.1 DynamoDB Table 实时数据同步: DynamoDB Stream

要实现 DynamoDB Table 实时数据同步, 首先要开启 Table Stream, 然后监听 Table 事件, 通过 Lambda 来实现具体的同步逻辑, 这里以 Serverless 框架为例来做代码演示.

首先开启 Table 的 Stream 功能, 并配置事件处理函数, 可以在`serverless.ts`中设置:

```typescript
import { AWS } from '@serverless/typescript'

const config: AWS = {
  service: 'serverless-tutorial-service',
  frameworkVersion: '3',
  provider: {
    name: 'aws',
    region: 'cn-northwest-1',
    stage: "${opt:stage, 'local'}",
    stackName: '${self:service}-${self:provider.stage}',
  }
  functions: {
    tableStream: {
      name: '${self:provider.stackName}-tablestream',
      handler: 'dist/lambda.dataSync',
      events: [
        {
          stream: {
            type: 'dynamodb',
            arn: { 'Fn::GetAtt': ['ExampleTable', 'StreamArn'] },
          },
        },
      ],
    },
    s3Sync: {
      name: '${self:provider.stackName}-s3sync',
      handler: 'dist/lambda.s3Sync',
      events: [
        {
          s3: {
            bucket: '${env:EXAMPLE_S3}',
            event: 's3:ObjectCreated:*',
            existing: true,
          },
        },
        {
          s3: {
            bucket: '${env:EXAMPLE_S3}',
            event: 's3:ObjectRemoved:*',
            existing: true,
          },
        },
      ],
    },
  },
  resources: {
    Description: '${self:provider.stackName} resource',
    Resources: {
      ExampleTable: {
        Type: 'AWS::DynamoDB::Table',
        Properties: {
          TableName: '${env:EXAMPLE_TBL}-${self:provider.stage}',
          AttributeDefinitions: [
            {
              AttributeName: 'userID',
              AttributeType: 'S',
            },
            {
              AttributeName: 'pvID',
              AttributeType: 'S',
            },
          ],
          KeySchema: [
            {
              AttributeName: 'userID',
              KeyType: 'HASH',
            },
            {
              AttributeName: 'pvID',
              KeyType: 'RANGE',
            },
          ],
          ProvisionedThroughput: {
            ReadCapacityUnits: 5,
            WriteCapacityUnits: 5,
          },
          PointInTimeRecoverySpecification: {
            PointInTimeRecoveryEnabled: '${self:custom.enabledPITR.${self:provider.stage}}',
          },
          StreamSpecification: {
            StreamViewType: 'NEW_AND_OLD_IMAGES',
          },
        },
      },
      ExampleTableTarget: {
        Type: 'AWS::DynamoDB::Table',
        Properties: {
          TableName: '${env:EXAMPLE_TBL}-${self:provider.stage}-target',
          AttributeDefinitions: [
            {
              AttributeName: 'userID',
              AttributeType: 'S',
            },
            {
              AttributeName: 'pvID',
              AttributeType: 'S',
            },
          ],
          KeySchema: [
            {
              AttributeName: 'userID',
              KeyType: 'HASH',
            },
            {
              AttributeName: 'pvID',
              KeyType: 'RANGE',
            },
          ],
          BillingMode: 'PAY_PER_REQUEST',
        },
      },
      ExampleS3: {
        Type: 'AWS::S3::Bucket',
        Properties: {
          BucketName: '${env:EXAMPLE_S3}',
          LifecycleConfiguration: {
            Rules: [{ Id: 'DeleteAfter1Day', Status: 'Enabled', ExpirationInDays: 1 }],
          },
          CorsConfiguration: {
            CorsRules: [
              {
                AllowedOrigins: ['*'],
                AllowedHeaders: ['*'],
                AllowedMethods: ['GET', 'PUT', 'POST', 'DELETE', 'HEAD'],
                MaxAge: 3000,
              },
            ],
          },
        },
      },
      ExampleS3Target: {
        Type: 'AWS::S3::Bucket',
        Properties: {
          BucketName: '${env:EXAMPLE_S3}-target',
          LifecycleConfiguration: {
            Rules: [{ Id: 'DeleteAfter1Day', Status: 'Enabled', ExpirationInDays: 1 }],
          },
          CorsConfiguration: {
            CorsRules: [
              {
                AllowedOrigins: ['*'],
                AllowedHeaders: ['*'],
                AllowedMethods: ['GET', 'PUT', 'POST', 'DELETE', 'HEAD'],
                MaxAge: 3000,
              },
            ],
          },
        },
      },
    },
  },
}
module.exports = config
```

该`serverless.ts` 不完整, 但是核心的资源创建和函数配置是完整的, 实际情况按照自己业务进行调整, 需要关注的有以下几点:

- Dynamodb Table `ExampleTable` 是数据源, 同步到 `ExampleTableTarget`, 其中`ExampleTable`设置了`StreamSpecification`配置
- `StreamSpecification` 的 `StreamViewType` 配置的是`NEW_AND_OLD_IMAGES`, 表示新老数据修改删除都会通知
- Stream 事件的监听处理函数是: `functions.tableStream` 中定义的`dist/lambda.dataSync` 函数

Stream 事件的处理函数如下:

```typescript
import { S3Event } from 'aws-lambda'
import AWS from 'aws-sdk'
import { DynamoDBStreamEvent } from 'aws-lambda'

// 新老表映射关系自行提供
const tableNameMap = {
  aaa: 'bbb'
}
const docClient = new AWS.DynamoDB.DocumentClient()
const syncTable = async (event: DynamoDBStreamEvent) => {
  for (const record of event.Records) {
    const sourceTableName = record.eventSourceARN.split('/')[1]
    const targetTableName = tableNameMap[sourceTableName]
    if (!sourceTableName || !targetTableName) {
      const errorMessage = `找不到同步表关系: ${sourceTableName}->${targetTableName}`
      console.error(errorMessage)
      throw new Error(errorMessage)
    }
    if (record.eventName === 'INSERT' || record.eventName === 'MODIFY') {
      const newItem = AWS.DynamoDB.Converter.unmarshall(record.dynamodb.NewImage)
      // 将 newItem 写入到另一个表
      console.log('new item: %j', newItem)
      const params = {
        TableName: targetTableName, // 更改为目标表的表名
        Item: newItem,
      }
      await docClient.put(params).promise()
    } else if (record.eventName === 'REMOVE') {
      // 处理删除操作
      console.log('oldItem key: %j', AWS.DynamoDB.Converter.unmarshall(record.dynamodb.Keys))
      const params = {
        TableName: targetTableName, // 更改为目标表的表名
        Key: AWS.DynamoDB.Converter.unmarshall(record.dynamodb.Keys), // 根据键值删除
      }
      await docClient.delete(params).promise()
    }
  }
}
export const dataSync: Handler = async (event: DynamoDBStreamEvent, context) => {
  console.log('dataSync event: ')
  console.log(JSON.stringify(event))
  console.log('dataSync context: ')
  console.log(JSON.stringify(context))

  await syncTable(event)
}),
```

开启 DynamoDB Table 的 Stream 事件后通过 Lambda 函数处理实现表的实时数据同步, 这个同步方法与表无关, 任何表都可以通过该处理函数.

### 2.2.2 S3 Bucket 实时数据同步: Bucket Event

同 DynamoDB 类似, S3 也有 Bucket 的事件, 这个事件不需要特别开启, 我们直接创建处理函数即可, 至于资源的创建可以查看 **2.2.1** 中 serverless.ts 中的配置:

- ExampleS3 是数据源, ExampleS3Target 是同步的目标
- 处理函数是: `functions.s3Sync` 中定义的 `dist/lambda.s3Sync`

Bucket 事件处理函数和新代码如下:

```typescript
import AWS from "aws-sdk";
import { S3Event } from "aws-lambda";

const bucketNameMap = {
  exmpales3: "exmaples3target",
};
const s3 = new AWS.S3();
export const syncBucket = async (event: S3Event) => {
  for (const record of event.Records) {
    const sourceBucketName = record.s3.bucket.name;
    const targetBucketName = bucketNameMap[sourceBucketName];

    if (!sourceBucketName || !targetBucketName) {
      const errorMessage = `找不到同步桶关系: ${sourceBucketName}->${targetBucketName}`;
      console.error(errorMessage);
      throw new Error(errorMessage);
    }
    if (record.eventName.startsWith("ObjectCreated")) {
      console.log(`copyObject ${record.s3.object.key}`);
      console.log({
        Bucket: targetBucketName,
        Key: record.s3.object.key,
        CopySource: `${sourceBucketName}/${record.s3.object.key}`,
      });
      await s3
        .upload({
          Bucket: targetBucketName,
          Key: record.s3.object.key,
          Body: s3
            .getObject({
              Bucket: sourceBucketName,
              Key: record.s3.object.key,
            })
            .createReadStream(),
        })
        .promise();
    }
    if (record.eventName.startsWith("ObjectRemoved")) {
      console.log(`deleteObject:`);
      console.log({
        Bucket: targetBucketName,
        Key: record.s3.object.key,
      });
      await s3
        .deleteObject({
          Bucket: targetBucketName,
          Key: record.s3.object.key,
        })
        .promise();
    }
  }
};
export const s3Sync: Handler = async (event: S3Event, context) => {
  console.log("s3Sync event: ");
  console.log(JSON.stringify(event));
  console.log("s3Sync context: ");
  console.log(JSON.stringify(context));

  await syncBucket(event);
};
```

这个处理函数需要关注一下`ObjectCreated`的逻辑处理, 代码中没有事用`copyObject`而是使用 `getObject`和`upload`这是为了保证准确性, 因为`copyObject` 不支持对现有的文件进行修改, 所以这里实际是先把原数据下载然后再覆盖目标文件.

# 3. 总结

本文通过 DynamoDB Table 的 Stream / S3 Bucket Event / Lambda, 实现针对热(实时)数据的同步, 通过 AWS SDK 和 CLI 实现了冷(历史)数据的同步. 其中热数据同步的方法比较有价值, 可以稍微做一些扩展及可以实现很多高级功能, 比如将老表数据拆分到多个新的子表.

在实现过程中需要注意的是: Stream 事件处理函数可以设置多个表的事件监听, 而 Bucket 事件处理有限制, 同一个 Bucket 同一个事件只能被分配给一个处理函数, 所以如果现有代码中已经针对 Bucket Event 有业务处理时需要格外注意.

# 4. FQA

- 同一个 Bucket 事件不能分享给不同目标

  原始报错: Configurations on the same bucket cannot share a common event type.

  要解决这个问题需要检查现有 Bucket 的属性, 检查 **Event notifications** 看是否已经有相同的事件被分配了其他函数, AWS 不支持相同的 Bucket 事件分配给不同的函数, 所以:

  - 不同的函数使用尽量具体的事件
  - 合并事件处理逻辑到同一个函数
