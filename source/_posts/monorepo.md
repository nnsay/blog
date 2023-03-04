---
title: monorepo
date: 2023-03-04 12:59:59
tags: 
- Code
excerpt: 代码工程管理--monorepo实践
---

# 1. 前提

**单体应用架构** vs **多仓多模块管理** vs **单仓多模块管理** 三种代码组织和维护的方式各有有缺点:

- 单体应用架构(monolithic): 软件开发前期, 业务不复杂, 依赖较少, 使用分层方式组织代码,  简单易用, 维护效率较高
- 多仓多模块管理(multirepo): 因为业务复杂和代码量增多, 单体架构难以有效组织代码, 编译时间过长, code review和代码维护也面临困难, 通常从单体架构拆分为微服务, 每个微服务是一个代码仓库, 独立升/编译/维护/上线
- 单仓多模块管理(monorepo): 多仓库模式在仓库数量持续增加后面临公共资源重复引用, CI/CD等脚本重复分散到各个项目维护, 单仓多模块管理的组织方式减轻了重复项维护的痛点, 另外在主干开发的GitFlow模式中单仓库更简单, 无需在多仓库模式中分别创建和维护主干分支

参考文档:

- [Monorepo，大型前端项目管理模式实践](https://developer.aliyun.com/article/1067018)
- [Why Monorepos](https://nx.dev/more-concepts/why-monorepos#monorepos)
- [monorepo.tools](https://monorepo.tools/): 一个专门讲monorepo的网站

# 2. 演练

nx命令行不需要额外, npm安装后默认自带这个子命令, 可以使用一下命令验证:
```bash
npx nx --version
```
演练主要使用[nest插件](https://nx.dev/packages/nest/documents/overview)作为脚手架工具

- 创建工作区
```bash
npx create-nx-workspace nx-tutorial --preset=nest --appName tutotial --nxCloud false
cd nx-tutorial/ && npm install -D @nrwl/nest

# or
yarn create nx-workspace nx-tutorial --preset=nest --appName tutotial --nxCloud false
cd nx-tutorial/ && yarn add -D @nrwl/nest

tree . -L 1
.
├── README.md
├── node_modules
├── nx.json
├── package-lock.json
├── package.json
├── apps
├── libs
├── tools
└── tsconfig.base.json
```
上面的`apps`和`libs`是存放自项目的位置

- 创建演练项目
```bash
yarn nx generate @nrwl/nest:app main-api

yarn nx generate @nrwl/nest:lib auth --publishable --importPath=@org/auth --buildable

```

- 编写测试代码

在main-api中调用auth lib中的一个方法

- 运行和验证
```bash
# 手动执行
yarn nx build auth
yarn nx build main-api
# 一次性执行多个
yarn nx run-many  --projects=auth,main-api --target=build
# 使用affect执行
yarn nx affected --target=build
# 启动服务
yarn nx serve main-api
# or
yarn nx affected --target=serve

```

  > **演练代码仓库: **[**nx-tutorial**](https://github.com/nnsay/nx-tutorial)

# 3. [Nx核心](https://nx.dev/core-features)

- 工作区 workspace
   - `工作区`是monorepo组织多个`项目`框架, 这个概念和golang/yarn等技术中的工作区概念一致
   - 工作区是`nx.json`所在的文件夹, 而项目一般存放到 packages/apps/libs等目录下
   - 项目存放的目录名称可以在在nx.json中配置, 并不一定使用自动生成的目录
```bash
"workspaceLayout": {
  "appsDir": "demos",
  "libsDir": "packages"
},
```

- 项目 project
   - 项目是组织单仓库多项目架构的基础概念, 项目会被放置在一个workspace中
- 命令 target
   - 提供一种命令的定义, 可以定义在工作区(nx.json)和项目(project.json)级别
   - 优先使用项目级别的, 项目级别的target是和工作区的target做合并的
- 执行任务
   - 任务定义位置有两处
      - package.json中的scripts
      - project.json中的targets
   - 执行任务的方式
      - nx [project] [target]
      - nx run [project][:target][:configuration] [_..]
      - nx run-many: 执行全部
   - 支持并行
   - 支持依赖执行: nx affected --targets=a,b,c,d
- 缓存任务结果
   - 目的: 方式代码未变更却重复的执行任务, 减少执行次数, 增加执行效率
   - 执行任务缓存可配置(nx.json)
```json
{
  "tasksRunnerOptions": {
    "default": {
      "runner": "nx/tasks-runners/default",
      "options": {
        "cacheableOperations": ["build", "test"]
      }
    }
  }
}
```

   - 使用缓存: 任务执行之前通过任务的`input`结合缓存判断代码是否发生变更, 如果未发生变更则从缓存中恢复`output`, 例如: 文件/终端输出
   - 缓存位置和文件: node_modules/.cache/nx/run.json
   - 清除缓存: nx reset
- 生成器 generator
   - 对工作区和项目进行代码组织和修改的工具
- 命令行 CLI
   - nx affected:graph: 可以开启一个本地website,在该界面上可以查看依赖关系和`target`
- [自动更新依赖](https://nx.dev/core-features/enforce-project-boundaries)
```bash
# Step1: 检查和更新依赖, 修改package.json生成migrations.json
yarn nx migrate latest
# Step2: 执行更新依赖
yarn nx migrate --run-migrations
# Step3: 清除
rm migrations.json
```

- [强制限定项目边界](https://nx.dev/core-features/enforce-project-boundaries)

为了方式项目引用到其他项目内部的的对象, 让依赖关系变得凌乱, 为此可以通过强制限定项目边界限定依赖关系仅通过项目引用.

   - 安装插件
```bash
yarn add npm i @nrwl/eslint-plugin-nx @nrwl/devkit -D
```

   - 设置tags, 可选
```bash
{
	"tags": [
    "scope:lib",
    "scope:app"
  ]
}
```

   - 配置eslint检查规则
```bash
{
  // ... more ESLint config here

  // @nrwl/nx/enforce-module-boundaries should already exist within an "overrides" block using `"files": ["*.ts", "*.tsx", "*.js", "*.jsx",]`
  "@nrwl/nx/enforce-module-boundaries": [
    "error",
    {
      "allow": [],
      // update depConstraints based on your tags
      "depConstraints": [
        {
          "sourceTag": "scope:lib",
          "onlyDependOnLibsWithTags": ["scope:lib"]
        },
        {
          "sourceTag": "scope:app",
          "onlyDependOnLibsWithTags": ["scope:app", "scope:lib"]
        }
      ]
    }
  ]

  // ... more ESLint config here
}
```

