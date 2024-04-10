---
title: zx脚本工具
date: 2024-04-10 15:27:12
tags:
  - DevOps
  - 工具
excerpt: 简单高效的使用JavaScript编写脚本
---

# 1. [ZX](https://google.github.io/zx/)介绍

zx 是 google 的一个脚本工具, 可以帮助开发者轻松的编写脚本, 主要有以下优势:

- 使用 js 书写脚本
- 可以利用丰富的 js 库

# 2. 代码演练

example.mjs

```js
#!/usr/bin/env zx

import { $, echo, os, chalk } from "zx";

// 调用shell命令
try {
  const projectName = await $`jq -r '.name' < package.json`;
  echo`1. projectName: ${projectName}`;
} catch (e) {
  const { stderr, exitCode } = e;
  console.error({ stderr, exitCode });
}

// 打印: console/echo
const date = await $`date`;
console.log(`2. Current date is ${date.stdout.trim()}.`);
echo`3. Current date is ${date}.`;
const echoResult = await $`echo 4. Current date is ${date}.`; // 调用shell中的echo如果不获取结果,在脚本输出结果中看不到
echo`${echoResult}`;

// 颜色
console.log("console.log", chalk.blue("蓝色"));
console.log(`console.log ${chalk.blue("蓝色")} template`);
echo("echo", chalk.blue("蓝色"));
echo(`echo ${chalk.blue("蓝色")} template `);

// 批量操作
const batchResult = await Promise.all([
  $`sleep 1; echo 1`,
  $`sleep 2; echo 2`,
  $`sleep 3; echo 3`,
]);
for (const result of batchResult) {
  echo`batch result: ${result}`;
}

// 文件和流
await $`echo "Hello, stdout!"`.pipe(
  fs.createWriteStream(`${__dirname}/output.txt`)
);

// 系统
echo`home目录: ${os.homedir()}`;
echo`日志目录: ${path.resolve(path.join(os.homedir(), "tmp", "output.log"))}`;
echo`当前目录: ${__dirname}`;
```

> [!note]
> 第三行的引用实则未必, 但是在写脚本时候可以提供很多类型感知防止写错;
> 另外如果脚本依赖第三方包也是需要在前面引用的.

结果输出:

```
1. projectName: albedo-services
2. Current date is Wed Apr 10 15:03:36 CST 2024.
3. Current date is Wed Apr 10 15:03:36 CST 2024.
4. Current date is Wed Apr 10 15:03:36 CST 2024.
console.log 蓝色
console.log 蓝色 template
echo 蓝色
echo 蓝色 template
batch result: 1
batch result: 2
batch result: 3
home目录: /Users/wangjian
日志目录: /Users/wangjian/tmp/output.log
当前目录: /Users/wangjian/github/vd/albedo-services/tmp/zxshell
```

# 3. 执行方式

- 可执行文件执行

  ```bash
  # 增加可执行权限
  chmod +x example.mjs
  # 执行
  ./example.mjs
  ```

  > [!note]
  >
  > 确保 shebang 已配置, zx 的 shengbang 是: `#!/usr/bin/env zx`

- zx 执行

  ```bash
  # 安装 zx, 使用npm或者brew
  npm install -g zx
  brew install zx

  # 执行
  zx ./example.mjs
  ```

# 4. 学习更多

- [执行 Promise](https://google.github.io/zx/process-promise)
- [内置 API](https://google.github.io/zx/api)
- [配置](https://google.github.io/zx/configuration)
