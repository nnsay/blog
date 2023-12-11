---
title: 浏览器中zip的处理
date: 2023-12-11 16:42:39
tags:
  - Code
  - 研究
excerpt: 使用js解压/抽取/创建zip
---

# 1. 背景说明(由 chatgpt 生成)

文本文件的压缩技术在数据传输、存储和处理方面发挥着关键作用。压缩文本文件的过程通过各种算法和技术，将文件的大小减小到原始大小的一部分，这带来了诸多好处，特别是在上传文件、网络传输和存储方面。以下是关于针对文本文件做压缩的诸多好处的主要内容：

减少文件大小：压缩文本文件可以显著地减小文件的大小。通过使用压缩算法，文件的数据可以被有效地压缩，节省存储空间。

提高传输速度：较小的文件大小意味着更快的上传和下载速度。在网络传输过程中，压缩文件可以减少数据传输的时间，特别是对于带宽受限或网络速度较慢的情况下更为明显。

节省带宽和资源：在互联网传输数据时，压缩文本文件可以减少数据流量，降低带宽使用，同时也能减少服务器和客户端等系统资源的占用。

提高效率：压缩文本文件能够带来更高的效率，因为在处理大量文本数据时，更小的文件大小意味着更快的数据传输速度和更少的资源消耗。

节省成本：减少文件大小和提高传输效率可以降低存储成本和网络传输成本。这对于需要大规模存储和传输数据的组织和个人来说尤其重要。

方便存储和分享：压缩文本文件使得文件更易于存储和分享。更小的文件尺寸意味着更容易将文件传输到不同的位置或通过电子邮件等方式分享给其他人。

总体而言，压缩文本文件不仅可以节省空间和时间，还能提高数据传输的效率，降低成本，并提供更便捷的文件存储和分享方式。这使得文本文件压缩成为现代信息技术中不可或缺的一环

# 2. 技术选型

- [gildas-lormeau/zip.js](https://github.com/gildas-lormeau/zip.js)
- [Stuk/jszip](https://github.com/Stuk/jszip)

以上两个包支持浏览器和 nodejs, 但是建议选择第一个, 原因如下:

- 支持多核压缩
- 支持加密
- 协议友好
  - zip.js: BSD-3-Clause license
  - jszip: MIT license and GPLv3 license

# 3. 代码实现

代码示例使用: vue + ts

```html
<template>
  <h1>读取zip</h1>
  <input type="file" name="zipFile" />
  <button type="button" @click="listZipFile">解压缩&读取</button>

  <h1>创建zip</h1>
  <input type="file" name="sourceFile" multiple />
  <button type="button" @click="createZip">创建压缩</button>
</template>

<script setup lang="ts">
  import { BlobReader, BlobWriter, ZipReader, ZipWriter } from "@zip.js/zip.js";

  const listZipFile = async () => {
    const zipFileEle = document.querySelector<HTMLInputElement>(
      'input[name="zipFile"]'
    );
    if (!zipFileEle || !zipFileEle.files || zipFileEle.files!.length === 0) {
      console.error("no found zip");
      return;
    }
    const zipFile = zipFileEle.files[0];
    const zipFileReader = new BlobReader(zipFile);
    const zipReader = new ZipReader(zipFileReader, {
      filenameEncoding: "utf8",
      commentEncoding: "utf8",
    });
    const entries = await zipReader.getEntries();
    for (const entry of entries) {
      if (entry.directory) {
        continue;
      }
      if (entry.filename.startsWith("__")) {
        continue;
      }
      if (entry.filename.startsWith(".")) {
        continue;
      }
      if (entry.filename.endsWith(".DS_Store")) {
        continue;
      }
      // 处理业务: 读取obj文件
      if (entry.filename.endsWith(".obj")) {
        const dataStream = new BlobWriter();
        await entry.getData!(dataStream);
        const dateaBlob = await dataStream.getData();
        const text = await dateaBlob.text();
        console.log(`${entry.filename}的文件内容如下:`);
        console.log(text);
      }
    }
    await zipReader.close();
  };

  const createZip = async () => {
    const sourceFileEle = document.querySelector<HTMLInputElement>(
      'input[name="sourceFile"]'
    );
    if (!sourceFileEle || !sourceFileEle.files) {
      console.error("no found source files");
      return;
    }

    const dataWriter = new BlobWriter();
    const zipWriter = new ZipWriter(dataWriter, { level: 5 });
    for (const file of sourceFileEle.files) {
      await zipWriter.add(file.name, file.stream());
    }
    await zipWriter.close();
    const dataBlock = await dataWriter.getData();

    // 下载
    const downloadEle =
      document.querySelector<HTMLAnchorElement>("#zipDownload") ||
      document.createElement("a");
    downloadEle.id = "zipDownload";
    downloadEle.download = "objfile.zip";
    downloadEle.href = URL.createObjectURL(dataBlock);
    downloadEle.textContent = "点击下载压缩文件";
    // 下载方式二选一
    // 下载方式一: 添加页面元素, 用户点击下载
    sourceFileEle.parentNode?.insertBefore(
      downloadEle,
      downloadEle.nextSibling
    );
    // 下载方式二: 自动下载
    downloadEle.click();
  };
</script>
```

# 4. 参考文档

- [zip.js api](https://gildas-lormeau.github.io/zip.js/api/index.html)
- [create zip file](https://gildas-lormeau.github.io/zip.js/demos/demo-create-file.html): 可查看页面源码
- [read zip file](https://gildas-lormeau.github.io/zip.js/demos/demo-read-file.html): 可查看页面源码
