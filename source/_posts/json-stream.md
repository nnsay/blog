---
title: json stream
date: 2024-04-12 15:57:22
tags:
  - 研究
  - Code
excerpt: 使用ndjson实现超大json数据流操作
---

# 1. 需求背景

在日常开发中经常会遇到大对象或者大文件处理, 比如在 nodejs 开发中, 一个算法包可能范围了一个长度为好几万长度的一个对象, 这个对象使用 Restful API 不好传递, 肯定会把这个处理结果保存为文件, 然后通过通过文件传递数据; 相反的业务中也会存在一个很大的文件里面可能是一个 JSON 结构的数据, 但是做完反序列化后在代码中变成了一个大对象. 所以本文讨论大对象处理的两个实践:

- 大对象如何保存为数据文件
- 大数据文件如何发返序列化小对象

如果上面的实践有解决方案那么我们的代码将对内存的要求变低, 让代码更健壮的处理数据.

# 2. 分析

通常一碰到大数据的处理, 第一想到的就是流, 第一个想到的数据格式就是 csv, 业务数据一般不是数组就是对象, 一个大的数组可以可以通过多行文件来存储, 一行保存数据组中的一个元素;而对象可能需要收抽象一下比如狠一点我们将文件划分为 m\*n 的矩阵而对象数据的值分别存到矩阵的某个位置, 只要不重复就好, 解析数据时将文件数据按行转为内存对象, 在用矩阵位置和对象 Key 的 map 关系进行取用就行.

其实不管是 CSV 和还是矩阵可以都不太通用因为太 trick 了, 在 JS 中不管是对象还是数组都是 JSON, 是否有一种数据格式可以保存大量 JSON 数据呢? 答案当然是有, 这就是: [JSON Stream](https://en.wikipedia.org/wiki/JSON_streaming).

JSON Stream 是一个思路, 通过维基百科的介绍, 其本质是带分隔符的 JSON, 它的落地实现方案有:

- Newline-delimited JSON
- Record separator-delimited JSON
- Concatenated JSON
- Length-prefixed JSON

最简单和理解是第一种, 也是经过调研生态圈使用比较多的一种, 这种方案也称为: **ndjson**, 关于 ndjson 还有一个议案, 可以从[这里](https://github.com/ndjson/ndjson-spec)查看, 而起本质规定 ndjson 如何序列化和反序列化, 这里不做太多讨论.

在维基百科中的最后给出了一些应用程序和工具, 常见的比如`jq`, 可以读取行分隔的 JSON 文本. 其中 nodejs 相关的有:

- [ldjson-stream](https://www.npmjs.org/package/ldjson-stream) module for [Node.js](https://en.wikipedia.org/wiki/Node.js)
- [ld-jsonstream](https://www.npmjs.com/package/ld-jsonstream) dependency free module for [Node.js](https://en.wikipedia.org/wiki/Node.js)

经过查看文档和使用, 最后本文决定采用 [ndjson.js](https://github.com/ndjson/ndjson.js), 该包其实是 ldjson-stream 的持续维护版本, 不选择 ld-jsonstream 是因为其没有序列化功能.

# 3. 代码实践

使用 ndjson 实现本文开头的两个业务场景, 具体来说就是:

- 从一个 ndjson 文件中反序列化话得到小对象
- 将大量小对象序列化并写入文件(ndjson 文件)

上面两个实践如果可以跑通, 则我们找到了一种对内存友好的大对象处理方案了.

```typescript
import * as ndjson from "ndjson";
import * as fs from "fs";

(async () => {
  const file = "test/ndjson.txt";
  // 将js对象序列化为流进而写入文件
  const writable = fs.createWriteStream(file, {
    encoding: "utf8",
    highWaterMark: 10,
  });
  const serialize = ndjson.stringify();
  for (let i = 0; i < 10000; i++) {
    serialize.write({
      id: i,
      key: "change1_0.6995461115147918",
      value: { rev: "1-e240bae28c7bb3667f02760f6398d508" },
      doc: {
        _id: "change1_0.6995461115147918",
        _rev: "1-e240bae28c7bb3667f02760f6398d508",
        hello: 1,
      },
    });
  }
  serialize.end();
  serialize.pipe(writable);
  writable.on("finish", () => {
    console.log("all data file size: ", fs.statSync(file).size);
  });

  // 从流水读取数据并parse为js对象
  const alldata = [];
  fs.createReadStream("test/ndjson.txt", {
    encoding: "utf8",
    highWaterMark: 10,
  })
    .pipe(ndjson.parse())
    .on("data", function (obj) {
      alldata.push(obj);
    })
    .on("end", () => {
      console.log("data size: ", alldata.length);
    });
})();
```

输出结果:

```bash
all data file size:  1968890
data size:  10000
```

# 4. 结论

**ndjson 是一种当前较为成熟的一种 JSON Stream 的实现方案, 可以运用于前后端的大对象大数据场景.**

# 5. 其他参考

- [在 Node.js 中如何处理一个大型 JSON 文件?](https://github.com/qufei1993/Nodejs-Roadmap/blob/master/docs/nodejs/advanced/json-stream.md)
- [Oboe.js](https://www.npmjs.com/package/oboe)
