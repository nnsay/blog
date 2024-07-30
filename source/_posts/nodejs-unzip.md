---
title: NodeJs Unzip
date: 2024-07-30 13:43:19
tags:
  - Code
  - 研究
excerpt: 调用nodejs大文件解压解决方案
---

# 1. 问题背景

在 aws 中使用以 lamdba 为基础的 serverless 架构的话在处理大文件时有一些痛点问题:

- lambda 的最大执行时间 900 秒
- lambda 的最大临时存储空间 10 GB

如果一个业务较为复杂, 例如: a. 从 s3 下载文件; b. 本地解压; c. 算法运行; d. 压缩算法结果; e. 上传算法结果到 s3.
这种业务在文件较小时是一个在一次 lambda 中执行完成, 但是当文件较大时, 下载/解压等都会形成瓶颈问题导致 lamdba 超时; 如何解决这个问题呢? 常见的一个方案是拆分: 将原本一个长时的 lambda 根据其处理逻辑, 拆分为若干个小单元, 每个单元使用一个 lambda 来执行.

以上的设计思路要实现, 就要保证每个单元不超时, 经过实际业务分析, 我发现在 nodejs 进行解压时耗时较多, 本文主要关注: 使用 nodejs 如何高效的解压大文件.

# 2. 选型和测试

经过一些调研, 选择一下三个 NPM 包来测试对大文件的解压:

- jszip
- unzipper
- node-stream-zip 👍

通过 nodejs 脚本测试针对一个 7.5G 的 zip 文件进行解压操作, 测试脚本如下:

```ts
import JSZip from "jszip";
import { mkdir, readFile, writeFile } from "fs/promises";
import { existsSync } from "fs";
import unzipper from "unzipper";
import path from "path";
import StreamZip from "node-stream-zip";

const fileName = "包钢";
const zipFile = `${process.env.HOME}/Downloads/${fileName}.zip`;

// 可以解压小zip, 可以列出压缩文件, 但是大zip遍历过程中解压有问题(不推荐)
const unzipperTest = async () => {
  const directory = await unzipper.Open.file(zipFile);
  for (const file of directory.files) {
    if (file.type === "Directory" || file.path.includes("__MACOSX")) continue;
    console.log(file.path);
    const destFilePath = `${process.env.HOME}/Downloads/tmp/${file.path}`;
    const destFileDir = path.dirname(destFilePath);
    if (!existsSync(destFileDir)) await mkdir(destFileDir, { recursive: true });
    await writeFile(destFilePath, file.stream());
  }
};
// 可以解压小zip, 可以列出压缩文件, 不支持流式解压(不推荐)
const jszipTest = async () => {
  const zipBuffer = await readFile(zipFile);
  const zip = await JSZip.loadAsync(zipBuffer);
  for (const [fileName, fileData] of Object.entries(zip.files)) {
    console.log(fileName);
  }
};
// 推荐, 大文件解压成功且支持流
const nodeStreamTest = async () => {
  const zip = new StreamZip.async({ file: zipFile });
  const entries = await zip.entries();
  for (const entry of Object.values(entries)) {
    if (entry.isDirectory || entry.name.includes("__MACOSX")) continue;
    console.log(entry.name);
    const destFilePath = `${process.env.HOME}/Downloads/tmp/${entry.name}`;
    const destFileDir = path.dirname(destFilePath);
    if (!existsSync(destFileDir)) await mkdir(destFileDir, { recursive: true });
    await zip.extract(entry.name, destFilePath);
  }
  await zip.close();
};
const exec = async () => {
  // await jszipTest()
  // await unzipperTest()
  await nodeStreamTest();
};

exec().catch((err) => {
  console.error("catch error: ", err);
});
```

经过测试, 发现 node-stream-zip 这个包比较好原因:

- 解压大文件没有报错, 其他两个包在小文件时工作正常, 但是大 zip 遍历过程中简单增加部分文件另存逻辑就异常了
- 支持流式解压, 这意味着不需要给 lambda 分配较大的内存空间

# 3. 总结

本文在长时 lamdab 提供了一个解决思路: 拆分.

在具体的解压问题上, 可以使用 node-stream-zip 这个包来解压大文件是一个不错的选择.

> [!note]
> 如果读者细心观察可以得知, 7.5G 的文件解压后可能更大,源文件+解压结果总大小很容易超过 lamdba 最大临时存储空间的限制(10G), 所以如果真实的 lambda 环境需要给 lamdba 配置 efs 挂载.
