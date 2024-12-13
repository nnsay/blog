---
title: "PostgreSQL数据同步利器--pgsync"
date: 2024-11-11 18:20:19
tags:
  - Tips
  - 工具
excerpt: 使用 pgsync 同步 PostgreSQL 数据
---

# 1. 同步 PostgreSQL 数据

同步数据是常见的需求, 例如在 sandbox 环境做了一些数据配置, 然后将这些数据配置同步到另外的环境. 复杂而稳定的同步方案通常来说是副本集; 如果不考虑实时性仅仅同步一些历史数据那可以采用 SQL 文件, 本文主要研究后者.

普通的 SQL 方案即: 将源数据库导出为 SQL 文件, 然后将 SQL 文件在目标数据库执行. 小数据量的时候这种方式手动来做也还行, 但是数据库量大了后执行效率就比较低, 本文介绍一种 CLI 工具来完成手动工作, 并可通过配置并发来增加执行效率, 这个工具就是: [pgsync](https://github.com/ankane/pgsync)

[pgsync](https://github.com/ankane/pgsync) 在 github 上有 2.3k 个 star, 属于高分项目. 在 mac 上安装 pgsync 如下:

```bash
brew install pgsync
```

因为这个项目是 ruby 写的, 也可以通过 gem 安装:

```bash
gem install pgsync
```

# 2. 实践

## 2.1 初始化

初始化会产生一个配置模板, 然后通过编辑这个文件提供数据同步的数据库信息. 这个步骤不是必须的, 因为可以通过命令行参数提供相同的信息, 但是我建议还是创建一个, 毕竟修改命令行参数不是很方便.

```bash
pgsync --init
```

## 2.2 配置

按照初始化之后的配置模板文件中的提示完成相关的配置, 配置很简单不复杂. 主要的配置项有: from/to/exclude, 如果需要分组可以配置: group, 如果配置的地址不是 localhost 还需要设置: to_safe: true.

这里给出我的一个本地测试配置:

```yaml
# source database URL
# database URLs take the format of:
#   postgres://user:password@host:port/dbname
#
# we recommend a command which outputs a database URL
# so sensitive information is not included in this file
#
# we *highly recommend* you use sslmode=verify-full when possible
# see https://ankane.org/postgres-sslmode-explained for more info
from: postgresql://root:1qaz2wsx@localhost:5432/solarsketch

# destination database URL
to: postgresql://root:1qaz2wsx@localhost:5432/solarsketch-debug

# exclude tables
exclude:
  - "*"

# define groups
groups:
  demo:
    - DemoPost
    - DemoUser

# sync specific schemas
# schemas:
#   - public

# protect sensitive information
data_rules:
  email: unique_email
  phone: unique_phone
  last_name: random_letter
  birthday: random_date
  encrypted_*: null

to_safe: true
```

> [!NOTE]
>
> exclude 配置为"\*", 同步的时候我倾向使用白名单, 但是该工具似乎没有, 所以我先排除所有的表, 然后在 group 中配置自己需要的表, 使用这种方式间接实现白名单功能

验证配置是否合法:

```bash
pgsync --list
```

如果可以正常输出 from/to 的数据库连接信息就说明配置正确了, 如果没有将 exclude 设置为"\*"则还可以看到要同步的表的名称列表.

## 2.3 执行同步

执行同步命令格式: pgsync [tables,groups] [sql] [options]. 如果对命令不熟悉可以通过: pgsync --help 来了解如何使用.

```bash
pgsync demo --overwrite
```

以上命令作用是 执行 demo 分组中的所有表的同步, 并且在数据行上使用 overwrite 策略. 行策略有三种:

- overwrite: 如果目标存在, 则使用源数据重新目标数据
- preserve: 如果目标存在, 则不做任何修改
- truncate: 默认行为, 先 truncate 目标表, 然后执行同步

> [!NOTE]
> 如果有多个表需要同步, 默认情况顺序同步, 可以通过 --jobs 设置并行同步表的个数.

# 3. 总结

经过在 mac 平台的测试, 从一个远程的数据库同步到另外一个远程数据库且不做表并发, 同步速度: 740 row/s. 个人感觉对于百万行的数据同步足以使用该工具解决, 推荐使用.
