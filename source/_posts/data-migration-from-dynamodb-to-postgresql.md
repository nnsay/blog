---
title: DynamoDB数据迁移至PostgreSQL
date: 2024-03-08 16:36:22
tags:
  - Tips
  - DevOps
excerpt: 使用Typescript脚本从DynamoDB迁移数据到PostgreSQL
---

# 1. 业务场景

将业务数据从 DynamoDB 迁移到 PostgreSQL. 本文代码假定以下条件:

- 使用 dynamoose 作为 DynamoDB 的 ORM
- 在现有代码工程中有可以复用的 dynamoose schema
- 使用 prisma 作为 PostgreSQL 的 ORM

虽然本文基于以上代码工程实践, 但是仅有第一项是必须的, 剩余两项非必须, 可以根据实际业务场景实现,比如以下两个场景:

- 使用了 prisma 但是数据库不是 PostgreSQL 而是 MySQL
- 没有使用 prisma 使用了别的数据库和 ORM

本文重点是提供一个思路来做 DynamoDB 的数据迁移.

# 2. 迁移脚本

```ts
import dynamoose from "dynamoose";
import { DongleHeartbeatSchema } from "@schemas/dongle";
import { PVSchema } from "@schemas/pv";
import {
  DynamoDBTableName,
  getDynamoDBTableName,
} from "@albedo-inc/infra-config";
import { ObjectType } from "nestjs-dynamoose";
import { PrismaClient } from "@prisma/client";

// Postgres instance
const db = new PrismaClient();

//  DynamoDB instance
const ddb = new dynamoose.aws.ddb.DynamoDB({
  credentials: {
    accessKeyId: "ak",
    secretAccessKey: "as",
  },
  region: "cn-northwest-1",
});
dynamoose.aws.ddb.set(ddb); // Set DynamoDB instance to the Dynamoose DDB instance

const ENV_NAME = "dev";
const sourceDB = {
  heartbeat: dynamoose.model(
    DynamoDBTableName.DONGLE_HEARTBEAT_TBL,
    DongleHeartbeatSchema,
    {
      tableName: getDynamoDBTableName(
        DynamoDBTableName.DONGLE_HEARTBEAT_TBL,
        ENV_NAME
      ),
    }
  ),
  pv: dynamoose.model(DynamoDBTableName.PV_TBL, PVSchema, {
    tableName: getDynamoDBTableName(DynamoDBTableName.PV_TBL, ENV_NAME),
  }),
};

const testPostgresConnnect = async () => {
  const versionResult = await db.$queryRaw`select version()`;
  console.log("version: ", versionResult[0].version);
};

const exec = async () => {
  // 测试 Postgres 数据库连接
  await testPostgresConnnect();

  console.log("同步pv");
  let lastKey: ObjectType = null;
  let pvCount = 0;
  do {
    let pvItems = await sourceDB.pv.scan().startAt(lastKey).exec();
    lastKey = pvItems.lastKey;
    pvCount += pvItems.count;
    // console.log({
    //   count: scanResult.count,
    //   scanCount: scanResult.scannedCount,
    //   lastKey: lastKey,
    // })

    // 遍历当前批次获取到的 DynamoDB Table 数据, 按照业务需求插入到 Postgres 中
    // for (let item of pvItems) {
    //   console.log(item.toJSON())
    //   await db.demoPost.create()
    // }
  } while (lastKey);
  console.log(`同步pv完成, 同步个数: ${pvCount}`);
};

exec()
  .then(() => {
    console.log("done");
  })
  .catch((err) => console.error(err));
```

# 3. 执行

使用`ts-node`运行脚本:

```bash
ts-node --project=tmp/sync-dynamodb/tsconfig.json tmp/sync-dynamodb/migration.ts
```

> [!NOTE]
> 因为脚本使用了`monorepo`的部分包, 所以在使用`ts-node`执行的时候需要添加==tsconfig.json==文件到脚本目录, tsconfig.json 参考内容如下:

```json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "module": "commonjs",
    "esModuleInterop": true
  },
  "include": ["*.ts"],
  "exclude": [],
  "ts-node": {
    "swc": true
  }
}
```
