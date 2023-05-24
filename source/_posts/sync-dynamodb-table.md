---
title: DynamoDB表数据同步
date: 2023-05-24 16:49:11
tags:
  - Tips
  - DevOps
excerpt: 一行脚本同步DynamoDB表数据
---

<style>
pre {
overflow: auto;
white-space: pre;
white-space: pre-wrap;
word-wrap: break-word;
}
</style>

### 用户故事

同步 dev 环境官方钢材库数据到 sandbox 环境

### 依赖

- 安装 `aws` 和 `jq`
- aws 配置了`sandbox`和 dev 两个`profile`及 credentials

### 脚本

```bash
aws dynamodb scan --table-name steel-table-dev --scan-filter '{"userID":{"ComparisonOperator":"EQ","AttributeValueList":[{"S":"63047568ade7f31ad13c641a"}]}}' --profile dev --no-paginate | jq -c '.Items | _nwise(25) | {"steel-table-dev": map({"PutRequest": {"Item": . }})}' | while read -r line; do echo $line > chunk.json & aws dynamodb batch-write-item --request-items file://chunk.json --profile sandbox; done
```

### 脚本工作原理

- 第一个管道拉取同步数据

  - 使用 dynamodb scan 命令查询数据
  - 使用--table-name 指定源数据表名: steel-table-dev
  - 使用--scan-filter 设置过滤条件: userID = 63047568ade7f31ad13c641a

  本步骤获取数据格式如下:

  ```json
  {
    "Items": [
      {
        "col1": {
          "N": "3.925"
        },
        "col2": {
          "N": "2.5"
        }
        ...
      }
      ...
    ]
  }
  ```

- 第二个管道按照写入最大限制(25)分片数据

  - 使用`jq`解析数据
  - 使用`jq`的`_nwise`函数分片数据
  - 使用`jq`的管道重新定义 JSON, 其中 Root Key 是目标数据表名: steel-table-dev

  本步骤改造数据格式如下, 这种格式是 aws 要求的, 详情可以参考: [BatchWriteItem](https://docs.aws.amazon.com/amazondynamodb/latest/APIReference/API_BatchWriteItem.html)

  ```json
  {
    "steel-table-dev": [
      {
        "PutRequest": {
          "Item": {
            "col1": {
              "N": "3.925"
            },
            "col2": {
              "N": "2.5"
            }
            ...
          }
        }
      }
      ...
    ]
  }
  ```

- 第三个管道提交同步数据

  - 遍历每个`jq`产生的分片, 每个分配数据写入临时文件
  - 使用临时文件配合 aws cli 进行写入
