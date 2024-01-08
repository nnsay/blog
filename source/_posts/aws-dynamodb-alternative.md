---
title: AWS DynamoDB 替代方案
date: 2024-01-08 18:47:06
tags:
  - 研究
excerpt: 调研DynamoDB替换的可行性
---

# 1. 探究 AWS DynamoDB 替代品

之前探求过 AWS S3 的替代品, 发现可以使用 Minio 替换 S3, 本次继续探究 AWS DynamoDB 替代品, 这些替代品的探求目的是为了下云的私有化部署, 在私有化部署中就没有 DynamodDB 了, 如何较少的修改使现有的代码来迁移是本文的目的.

因为要较少的修改代码, 所以替代品的首要特性就是兼容 DynamoDB API, 除此之外其他一些特性还有:

- 全局二级索引
- TTL(time to live)

# 2. 替代品分析

从官方资料和社区经过初步调研初步入围的方案有:

- [dynamodb-local](https://hub.docker.com/r/amazon/dynamodb-local): 针对开发人员开发和测试使用的一个版本, 缺少很多功能例如: 预配吞吐量/数据转化/TTL, 详情可以查看: [DynamoDB local usage notes](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/DynamoDBLocal.UsageNotes.html)
- [云数据库 MongoDB 版](https://help.aliyun.com/zh/mongodb/): 该服务属于阿里云, [云数据库 MongoDB 分片集群实例兼容 DynamoDB 协议](https://help.aliyun.com/zh/mongodb/user-guide/compatibility-details-of-dynamodb-compatible-apsaradb-for-mongodb-instances)
- [ScyllaDB](https://www.scylladb.com/open-source-nosql-database/): ScyllaDB is a [drop-in replacement for DynamoDB](https://www.scylladb.com/product/benchmarks/dynamodb-benchmark/). No need to change your client code — you can keep all your existing data models and queries when using our DynamoDB-compatible API

dynamodb-local: 是开发测试使用的, 不是一个生产环境级别的东西, 所以不采用.

云数据库 MongoDB 版: 是阿里云的一个服务, 但是当私有化部署环境不能接入外网的时候也就不行, 所以不采用.

ScyllaDB: 一种兼容 DynamoDB 和 Cassandra 数据库的新数据库, 在 [db-engins](https://db-engines.com/en/ranking/)  排名 66 位(2024-1-8), 其官方文档也有[全局二级索引](https://opensource.docs.scylladb.com/stable/cql/secondary-indexes.html)和[TTL](https://opensource.docs.scylladb.com/stable/cql/time-to-live.html)的特性, 整体感觉比较合适优先采纳.

# 3. 验证

## 3.1 ScyllaDB 本地环境

通过 [Alternator: DynamoDB API In Scylla](https://opensource.docs.scylladb.com/stable/alternator/alternator.html), 配置 docker-compose 服务如下:

```yaml
version: "3.8"
services:
  scylla:
    container_name: scylla
    image: scylladb/scylla:latest
    ports:
      - "8000:8000"
    command: --alternator-port=8000 --alternator-write-isolation=always
volumes:
  data:
```

启动服务:

```bash
docker compose up -d scylla
```

使用 [csqhhrc](https://opensource.docs.scylladb.com/stable/cql/cqlsh.html):

```bash
docker exec -it scylla cqlsh
```

## 3.2 基础测试

在开发中一般使用 dynamoose 这个 ORM 来操作 DynamoDB, 所以这里也使用这个作为代码验证的一个条件:

```ts
import dynamoose, { Schema } from "dynamoose";

// Create new DynamoDB instance
const ddb = new dynamoose.aws.ddb.DynamoDB({
  // credentials: {
  //   accessKeyId: 'ak',
  //   secretAccessKey: 'as',
  // },
  region: "cn-northwest-1",
  endpoint: "http://localhost:8000",
  // endpoint: 'http://localstack:4566',
});
dynamoose.aws.ddb.set(ddb);

export interface DongleHeartbeat {
  dongleID: string;
  userID: string;
  heartbeatAt: number;
  createdAt: number;
  ttl: number;
}

export const DongleHeartbeatSchema = new Schema(
  {
    dongleID: { type: String, hashKey: true },
    userID: { type: String, rangeKey: true },
    heartbeatAt: {
      type: Number,
      default: () => Math.round(Date.now() / 1000),
    },
    ttl: {
      type: Number,
      // index: {
      //   name: 'idx_dongleheartbeat_ttl',
      //   type: 'global',
      //   project: false,
      // },
    },
  },
  {
    timestamps: {
      createdAt: {
        createdAt: {
          type: {
            value: Date,
            settings: {
              storage: "seconds",
            },
          },
        },
      },
    },
  }
);

const HeartbeatModel = dynamoose.model("Heartbeat", DongleHeartbeatSchema, {
  tableName: "scylla-dongle-heartbeat-tbl-local",
});

const exec = async () => {
  const key: Pick<DongleHeartbeat, "dongleID" | "userID"> = {
    dongleID: "dongleid1",
    userID: "user1",
  };
  // 查找
  const queryResult = await HeartbeatModel.query(key)
    .sort("ascending")
    .limit(1)
    .exec();
  console.log("query result: ", queryResult);
  const scanResult = await HeartbeatModel.scan(key).limit(1).exec();
  console.log("scan result: ", scanResult);
  const getOneResult = await HeartbeatModel.get(key);
  console.log("get one result: ", getOneResult);
  if (getOneResult) {
    await getOneResult.delete();
  }

  // 新建
  const heartbeat: Partial<DongleHeartbeat> = {
    ...key,
    heartbeatAt: Date.now(),
    ttl: Date.now(),
  };
  const createResult = await HeartbeatModel.create(heartbeat);
  console.log("createResult: ", createResult);
  // 修改
  const newHeartbeat: Partial<DongleHeartbeat> = {
    ...key,
    heartbeatAt: Date.now(),
    ttl: Date.now(),
  };
  let updateResult = await HeartbeatModel.update(newHeartbeat);
  console.log("updateResult by object: ", updateResult);
  updateResult = await HeartbeatModel.update(key, { ttl: 0 });
  console.log("updateResult by key: ", updateResult);
  // 删除
  const deleteResult = await updateResult.delete();
  console.log("delete result: ", deleteResult);
  await updateResult.save();
};

exec()
  .then(() => {
    console.log("done");
  })
  .catch((err) => console.error(err));
```

以上代码通过 dynamoose 测试针对 DynamoDB 的一下特性的测试:

- query
- scan
- create
- update
- delete
- hash key
- sort key

从测试脚本看, 以上特性都 OK, scylla 基本满足了需求. 查看 cqlsh 结果如下:

```sql
cqlsh> select * from "alternator_scylla-dongle-heartbeat-tbl-local"."scylla-dongle-heartbeat-tbl-local";

 dongleID  | userID | :attrs
-----------+--------+-----------------------------------------------------------------------------------------------------
 dongleid1 |  user1 | {'createdAt': 0x0300000000659bad00, 'heartbeatAt': 0x0300000000018ce81bc96f, 'ttl': 0x030000000000}
```

> ⚠️ keyspace 和 table 都是自动创建, 这里直接通过: describe tables 可以查找到脚本创建的表信息.

## 3.3 全局索引测试

如果取消上面代码中`ttl`的`index`的配置:

```ts
ttl: {
  type: Number,
  index: {
    name: 'idx_dongleheartbeat_ttl',
    type: 'global',
    project: false,
  },
},
```

基础测试脚本跑不过, 通过 cqlsh 查看结果:

```sql
cqlsh> describe table "alternator_scylla-dongle-heartbeat-tbl-local"."scylla-dongle-heartbeat-tbl-local"
CREATE TABLE "alternator_scylla-dongle-heartbeat-tbl-local"."scylla-dongle-heartbeat-tbl-local" (
    "dongleID" text,
    "userID" text,
    ":attrs" map<text, blob>,
    ttl decimal,
    PRIMARY KEY ("dongleID", "userID")
) WITH CLUSTERING ORDER BY ("userID" ASC)
    AND bloom_filter_fp_chance = 0.01
    AND caching = {'keys': 'ALL', 'rows_per_partition': 'ALL'}
    AND comment = ''
    AND compaction = {'class': 'SizeTieredCompactionStrategy'}
    AND compression = {'sstable_compression': 'org.apache.cassandra.io.compress.LZ4Compressor'}
    AND crc_check_chance = 1.0
    AND dclocal_read_repair_chance = 0.0
    AND default_time_to_live = 0
    AND gc_grace_seconds = 864000
    AND max_index_interval = 2048
    AND memtable_flush_period_in_ms = 0
    AND min_index_interval = 128
    AND read_repair_chance = 0.0
    AND speculative_retry = '99.0PERCENTILE';

CREATE MATERIALIZED VIEW "alternator_scylla-dongle-heartbeat-tbl-local"."scylla-dongle-heartbeat-tbl-local:idx_dongleheartbeat_ttl" AS
    SELECT *
    FROM "alternator_scylla-dongle-heartbeat-tbl-local"."scylla-dongle-heartbeat-tbl-local"
    WHERE ttl IS NOT NULL
    PRIMARY KEY (ttl, "dongleID", "userID")
    WITH CLUSTERING ORDER BY ("dongleID" ASC, "userID" ASC)
    AND bloom_filter_fp_chance = 0.01
    AND caching = {'keys': 'ALL', 'rows_per_partition': 'ALL'}
    AND comment = ''
    AND compaction = {'class': 'SizeTieredCompactionStrategy'}
    AND compression = {'sstable_compression': 'org.apache.cassandra.io.compress.LZ4Compressor'}
    AND crc_check_chance = 1.0
    AND dclocal_read_repair_chance = 0.0
    AND default_time_to_live = 0
    AND gc_grace_seconds = 864000
    AND max_index_interval = 2048
    AND memtable_flush_period_in_ms = 0
    AND min_index_interval = 128
    AND read_repair_chance = 0.0
    AND speculative_retry = '99.0PERCENTILE'

scylla_tags = {};

scylla_tags = {}
```

对于`ttl`的处理 dynamodb 的处理如下:

```json
{
  "AttributeDefinitions": [
    {
      "AttributeName": "dongleID",
      "AttributeType": "S"
    },
    {
      "AttributeName": "userID",
      "AttributeType": "S"
    },
    {
      "AttributeName": "ttl",
      "AttributeType": "N"
    }
  ],
  "TableName": "scylla-dongle-heartbeat-tbl-local",
  "KeySchema": [
    {
      "AttributeName": "dongleID",
      "KeyType": "HASH"
    },
    {
      "AttributeName": "userID",
      "KeyType": "RANGE"
    }
  ],
  "TableStatus": "ACTIVE",
  "CreationDateTime": "2024-01-08T08:01:12.337Z",
  "ProvisionedThroughput": {
    "LastIncreaseDateTime": "1970-01-01T00:00:00.000Z",
    "LastDecreaseDateTime": "1970-01-01T00:00:00.000Z",
    "NumberOfDecreasesToday": 0,
    "ReadCapacityUnits": 1,
    "WriteCapacityUnits": 1
  },
  "TableSizeBytes": 66,
  "ItemCount": 1,
  "TableArn": "arn:aws:dynamodb:cn-northwest-1:000000000000:table/scylla-dongle-heartbeat-tbl-local",
  "TableId": "3740c127-381c-4759-ae2c-216163108ff4",
  "GlobalSecondaryIndexes": [
    {
      "IndexName": "idx_dongleheartbeat_ttl",
      "KeySchema": [
        {
          "AttributeName": "ttl",
          "KeyType": "HASH"
        }
      ],
      "Projection": {
        "ProjectionType": "KEYS_ONLY"
      },
      "IndexStatus": "ACTIVE",
      "ProvisionedThroughput": {
        "ReadCapacityUnits": 1,
        "WriteCapacityUnits": 1
      },
      "IndexSizeBytes": 66,
      "ItemCount": 1,
      "IndexArn": "arn:aws:dynamodb:ddblocal:000000000000:table/scylla-dongle-heartbeat-tbl-local/index/idx_dongleheartbeat_ttl"
    }
  ],
  "Replicas": []
}
```

从以上 dynamodb 和 scylla 两个表的结果看, scylla 对于 `key schame` 会转为自己的列, 其他的转为`:attrs`, 如果非 key 字段设置索引则 scylla 则会处理失败, 原因应该是其找到 `ttl` 列, 这点可以通过直接使用 cqlsh 创建索引来证明:

```sql
# 1. 删除现有 keyspace
drop keyspace "alternator_scylla-dongle-heartbeat-tbl-local";
# 2. 注释掉 ttl.index 部分, 重新运行脚本
# 3. 创建索引
cqlsh>
cqlsh> select * from "alternator_scylla-dongle-heartbeat-tbl-local"."scylla-dongle-heartbeat-tbl-local";

 dongleID  | userID | :attrs
-----------+--------+-----------------------------------------------------------------------------------------------------
 dongleid1 |  user1 | {'createdAt': 0x0300000000659bbe21, 'heartbeatAt': 0x0300000000018ce85eb12b, 'ttl': 0x030000000000}

(1 rows)
cqlsh> create index idx_dongleheartbeat_ttl on "alternator_scylla-dongle-heartbeat-tbl-local"."scylla-dongle-heartbeat-tbl-local"(ttl);
InvalidRequest: Error from server: code=2200 [Invalid query] message="No column definition found for column ttl"
cqlsh>
```

总结: scylla 索引基于一个 column, 如果想用全局二级索引得想办法解决 dynomoose schema 到 scylla table 创建的逻辑, 或者不使用 dynomoose schema 自动创建 table. 关于如何不使用自动创建的 table, 单独一节讲解, 向后查看章节 4.

## 3.4 [TTL](https://opensource.docs.scylladb.com/stable/cql/time-to-live.html)

该特性不进行测试了, 根据[官方文档](https://opensource.docs.scylladb.com/stable/cql/time-to-live.html), 这个特性 Cassandra 也有, Scylla 继承了过来, 有较为详细的说明, 所以忽略该测试.

# 4. 使用 Scylla 表

## 4.1 创建

```sql
cqlsh> CREATE KEYSPACE IF NOT EXISTS alternator_demo WITH REPLICATION = {'class': 'SimpleStrategy','replication_factor':1};

Warnings :
Using Replication Factor replication_factor=1 lower than the minimum_replication_factor_warn_threshold=3 is not recommended.

cqlsh> use alternator_demo;
cqlsh:alternator_demo> CREATE TABLE demo (
user_id int,
str text,
mtime timestamp,
PRIMARY KEY (user_id, mtime)
) WITH CLUSTERING ORDER BY (mtime DESC);
cqlsh:alternator_demo> DESCRIBE TABLES

demo
```

## 4.2 测试代码

```ts
const DemoSchema = new Schema({
  user_id: { type: Number, hashKey: true },
  str: {
    type: String,
  },
  mtime: Number,
});
const DemoModel = dynamoose.model("Demo", DemoSchema, {
  tableName: "demo",
});

const test = async () => {
  const createResult = await DemoModel.create({
    user_id: 1,
    str: "user1",
    mtime: Date.now(),
  });
  console.log("createResult: ", createResult);
};

const test = async () => {
  const getOneResult = await DemoModel.get({ user_id: 1 });
  if (getOneResult) {
    await getOneResult.delete();
  }
  const createResult = await DemoModel.create({
    user_id: 1,
    str: "user1",
    mtime: Date.now(),
  });
  console.log("createResult: ", createResult);
  await createResult.delete();
  createResult.mtime = Date.now();
  const newResult = await createResult.save();
  console.log("newResult: ", newResult);
};
```

运行脚本如下:

```bash
ValidationException: Type mismatch: expected type S for key column user_id, got type "N"
```

如果将脚本的中 user_id 赋值为字符串"1", 错误为:

```
ValidationException: Key column mtime not found
```

这里需要 Troubleshooting 一下, 如果使用 dynamoose 创建 demo2 表, 对比 demo 和 demo2 两个表:

```sql
cqlsh> describe table alternator_demo2.demo2

CREATE TABLE alternator_demo2.demo2 (
    user_id decimal PRIMARY KEY,
    ":attrs" map<text, blob>
) WITH bloom_filter_fp_chance = 0.01
    AND caching = {'keys': 'ALL', 'rows_per_partition': 'ALL'}
    AND comment = ''
    AND compaction = {'class': 'SizeTieredCompactionStrategy'}
    AND compression = {'sstable_compression': 'org.apache.cassandra.io.compress.LZ4Compressor'}
    AND crc_check_chance = 1.0
    AND dclocal_read_repair_chance = 0.0
    AND default_time_to_live = 0
    AND gc_grace_seconds = 864000
    AND max_index_interval = 2048
    AND memtable_flush_period_in_ms = 0
    AND min_index_interval = 128
    AND read_repair_chance = 0.0
    AND speculative_retry = '99.0PERCENTILE';

scylla_tags = {}

cqlsh> describe table alternator_demo.demo

CREATE TABLE alternator_demo.demo (
    user_id int,
    mtime timestamp,
    str text,
    PRIMARY KEY (user_id, mtime)
) WITH CLUSTERING ORDER BY (mtime DESC)
    AND bloom_filter_fp_chance = 0.01
    AND caching = {'keys': 'ALL', 'rows_per_partition': 'ALL'}
    AND comment = ''
    AND compaction = {'class': 'SizeTieredCompactionStrategy'}
    AND compression = {'sstable_compression': 'org.apache.cassandra.io.compress.LZ4Compressor'}
    AND crc_check_chance = 1.0
    AND dclocal_read_repair_chance = 0.0
    AND default_time_to_live = 0
    AND gc_grace_seconds = 864000
    AND max_index_interval = 2048
    AND memtable_flush_period_in_ms = 0
    AND min_index_interval = 128
    AND read_repair_chance = 0.0
    AND speculative_retry = '99.0PERCENTILE';
```

通过对比测试发现在这对非 Key 列的处理大有不同, dynamoose 似乎不识别 demo 的列结构, 似乎它会从 `:attr`  寻找非 Key 列, 所以 Dynamoose 和 Scylla 是不兼容的, 在迁移时需要做取舍.

# 5. 结论

Scylla 在协议上确实兼容了 DynamoDB, 增删改查以及 hash key 和 range key 都完美支持, 但是在非 key 字段上设置索引上不兼容, 这个不兼容不是特性上的不兼容, 是 ORM 层级上需要兼容, 这个兼容性可以通过代码解决但成本较大, 比如可能需要换 ORM 或者修改数据库操作写法. 总体上 Scylla 替换 DynamoDB 可行, 但如果有强框架绑定比如 ORM 需要虚取舍.
