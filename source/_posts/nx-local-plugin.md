---
title: 自定义Nx插件
date: 2023-04-11 12:15:40
tags:
  - Tips
  - DevOps
excerpt: 使用自定义Nx生成器/修改器满足特殊的工程代码需求
---

# 1. 创建自定义插件项目

Nx作为Monorepo的一个框架方便了monorepo的开发和协作, 如果现有的生成器无法满足业务需求, 可以通过自定义插件的方式自定义生成器, 解决特殊的工程需求. 自定义插件首先需要安装`@nrwl/nx-plugin`, 使用它可以创建一个新`Library`的空项目, 在此项目上进行自定义开发, 具体操作如下:

```bash
yarn add @nrwl/nx-plugin@latest --dev
yarn nx g @nrwl/nx-plugin:plugin albedo-nx-plugin --importPath @albedo-inc/albedo-nx-plugin
```

生成自定义生成器后核心代码如下:

```bash
libs/albedo-nx-plugin
├── README.md
├── executors.json
├── generators.json
├── jest.config.ts
├── package.json
├── project.json
├── src
│   ├── executors
│   │   └── build
│   │       ├── executor.spec.ts
│   │       ├── executor.ts
│   │       ├── schema.d.ts
│   │       └── schema.json
│   ├── generators
│   │   └── albedo-nx-plugin
│   │       ├── files
│   │       │   └── src
│   │       │       └── index.ts__template__
│   │       ├── generator.spec.ts
│   │       ├── generator.ts
│   │       ├── schema.d.ts
│   │       └── schema.json
│   └── index.ts
├── tsconfig.json
├── tsconfig.lib.json
└── tsconfig.spec.json
```

需要注意的是创建plugin会引起`package.json`新增依赖项:

```json
{
	"dependencies": {
		"@swc/helpers": "~0.4.11"
	},
  "devDependencies": {
    "jsonc-eslint-parser": "^2.1.0",
    "@swc-node/register": "^1.4.2",
    "@swc/cli": "~0.1.55",
    "@swc/core": "^1.2.173",
    "@types/node": "16.11.7"
  }
}

```

# 2. 自定义业务逻辑

为了讲解和说明这里定义一个用户故事: 使用生成器选择已经存在应用类型的项目对其`project.json`中的build taget进行多环境配置修改. 具体操作如下:

- 删除必要的文件

  - src/executors: 根据用户故事插件是一个生成器/修改器不是执行器, 所以这个目录可以删除

  - executors.json: 同上
  
  - src/generators/files: 不需要生成新文件
  
- 修改生成器的名字

  `generators.json` L6

  ```json
  {
    "$schema": "http://json-schema.org/schema",
    "name": "albedo-nx-plugin",
    "version": "0.0.1",
    "generators": {
      "env-config": {
        "factory": "./src/generators/env-config/generator",
        "schema": "./src/generators/env-config/schema.json",
        "description": "env-config generator"
      }
    }
  }
  ```

- 修改schema, 选择project

  修改生成器文件`src/generators/schema.json`

  ```json
  {
    "$schema": "http://json-schema.org/schema",
    "cli": "nx",
    "$id": "EnvConfig",
    "title": "",
    "type": "object",
    "properties": {
      "projectName": {
        "type": "string",
        "description": "Application project name",
        "$default": {
          "$source": "argv",
          "index": 0
        },
        "x-prompt": "Which project to change?",
        "x-dropdown": "projects"
      }
    },
    "required": ["projectName"]
  }
  ```

  > 同时需要修: `src/generators/schema.d.ts` 这个类型需要和`src/generators/schema.json`对应

- 加载project的配置/修改/保存

  修改自定义生成器`generator.ts`文件, 编写业务逻辑

  ```typescript
  import { readProjectConfiguration, Tree, updateProjectConfiguration } from '@nrwl/devkit'
  import { EnvConfigGeneratorSchema } from './schema'
  
  export default async function (tree: Tree, options: EnvConfigGeneratorSchema) {
    const projectConfig = readProjectConfiguration(tree, options.projectName)
    if (projectConfig.projectType === 'application') {
      projectConfig.targets.build.configurations = {
        sandbox: {
          mode: 'development',
        },
        dev: {
          mode: 'development',
        },
        staging: {
          mode: 'proudction',
        },
        prod: {
          mode: 'proudction',
        },
      }
      updateProjectConfiguration(tree, options.projectName, projectConfig)
    }
  }
  ```

# 3. 测试及使用

 - 使用交互式方式

   ```bash
   # frontend
   yarn nx g @albedo-inc/albedo-nx-plugin:env-config
   ```
   
 - 直接传递参数方式

   ```
   yarn nx g @albedo-inc/albedo-nx-plugin:env-config --projectName albedo-dashboard
   ```

# 4. 参考文档

- [nx generators](https://nx.dev/recipes/generators)
- [nrwl_devkit](https://nx.dev/packages/devkit/documents/nrwl_devkit)
