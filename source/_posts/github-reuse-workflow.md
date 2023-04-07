---
title: Github Action复用工作流
date: 2023-04-07 17:54:27
tags:
  - DevOps
  - Tips
excerpt: 使用Github Action复用工作流优化代码工程
---

# 1. 问题介绍

在使用Github Action做CICD的过程中会编写很多Workflow脚本, 很多脚本又有很多相似性, 例如针对开发(dev)/预生成(staging)和生成(prod)三个场景中只有[触发工作流时事件(Event)](https://docs.github.com/zh/actions/using-workflows/events-that-trigger-workflows)和[环境(Environment)](https://docs.github.com/zh/actions/deployment/targeting-different-environments/using-environments-for-deployment)不同, 而核心工作流步骤(Step)是差不多的, 如下:

- dev

  .github/workflows/dev-release.yml:

  ```yaml
  name: Dev(master) release
  
  on:
    push:
      branches:
        - master
  
  jobs:
  	version-bump:
  		# ...
  	deploy-aws:
      name: Deploy AWS - Dev environment
      runs-on: ubuntu-latest
      needs: version-bump
      environment: dev
  
      steps:
        - name: Check out git repository
          uses: actions/checkout@v3
          with:
            fetch-depth: 0
            ref: ${{ needs.version-bump.outputs.release-sha }}
  
        - name: Setup Node.js
          uses: actions/setup-node@v3
          with:
            node-version: 16
            cache: yarn
  
        - name: Install CI dependencies
          run: yarn install --frozen-lockfile
  
        - name: Build and deploy applications to AWS
          run: npx nx affected --target=deploy --configuration=${{ vars.ENV_NAME }}
          env:
            AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
            AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
            ENV_NAME: ${{ vars.ENV_NAME }}
            AWS_DEFAULT_REGION: cn-northwest-1
  ```

- staging

  .github/workflows/staging-release.yml:

  ```yaml
  name: Staging release
  
  on:
    push:
      branches:
        - release/*
  
  jobs:
  	deploy-aws:
      name: Deploy AWS - Staging environment
      runs-on: ubuntu-latest
      environment: staging
  
      steps:
        - name: Check out git repository
          uses: actions/checkout@v3
          with:
            fetch-depth: 0
            
        # ...
        # be same with the dev workflow yaml L24~L39
  ```

- prod

  .github/workflows/prod-release.yml:

  ```yaml
  name: Production release
  
  on:
    workflow_dispatch:
  
  jobs:
  	deploy-aws:
      name: Deploy AWS - Production environment
      runs-on: ubuntu-latest
      environment: production
  
      steps:
        - name: Check out git repository
          uses: actions/checkout@v3
          with:
            fetch-depth: 0
            
        # ...
        # be same with the dev workflow yaml L24~L39
  ```

结合上面的例子分析可知:

- 不同点:

  - dev是代码push到master触发; staging是push release分支触发; prod是手动触发

  - dev/staging/prod三个工作流中的`environment`是不一样的`

- 相同点

  - L24~L39是完全相同的

- 略有不同

  - dev和checkout逻辑和staging/prod不同, 多了一个`ref`

综上, `相同点`+`略有不同` 这两部分其实可以想办法复用, 而这个技术就是: [复用工作流](https://docs.github.com/zh/actions/using-workflows/reusing-workflows)

# 2. [复用工作流](https://docs.github.com/zh/actions/using-workflows/reusing-workflows)

复用工作流简而言之: 将类似的工作流, 这里主要就是相似的Step抽出到一个新的工作流, 而这个新的工作流可以被重复调用. 在实践之前需要简单介绍些理论:

- 被调用方
  - 复用工作流的触发事件是: workflow_call, 即被调用方
  - 在复用工作流可以继续调用其他服用工作流, 但是最多四层, 如: root->reuse1->reuse2记作2层
  - 复用工作流定义在事件处定义: `inputs`和`secrets`两种变量用于传递参数
  - 复用工作流永远仅使用调用方的环境和上下文信息, 与自己所在位置无关
- 调用方
  - 使用复用工作流的一方称为调用方, 调用方使用`uses`关键词来调用复用工作流
  - 调用方使用`with`关键词设置`inputs`参数设置
  - 调用方使用`secrets`进行机密参数设置
    - secrets变量支持继承: `inherit`

结合上面的例子实现步骤如下:

- 被调用方: 定义复用工作流

  github/workflows/deploy-aws.yml:

  ```yaml
  name: AWS deployment
  
  on:
    workflow_call:
      inputs:
        envName: # 支持环境切换
          type: string
          description: the deployment environment
          required: true
        releaseSha: # 支持两种checkout
          type: string
          description: the git head
  
  jobs:
    deploy:
      name: Deploy AWS
      runs-on: ubuntu-latest
      environment: ${{ inputs.envName }} # 切换环境
  
      steps:
        - name: Check out git repository
          if: inputs.releaseSha != '' # 支持ref的checkout
          uses: actions/checkout@v3
          with:
            fetch-depth: 0
            ref: ${{ inputs.releaseSha }}
  
        - name: Check out git repository
          if: inputs.releaseSha == '' # 支持默认的checkout
          uses: actions/checkout@v3
          with:
            fetch-depth: 0
  
        - name: Setup Node.js
          uses: actions/setup-node@v3
          with:
            node-version: 16
  
        - name: Install CI dependencies
          run: yarn install --frozen-lockfile
  
        - name: Build and deploy applications to AWS
          run: npx nx affected --target=deploy --configuration=${{ inputs.envName }} # 引用变量
          env:
            AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
            AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
            ENV_NAME: ${{ inputs.envName}} # 引用变量
            AWS_DEFAULT_REGION: cn-northwest-1
  
  ```

  > 以上yaml中`#`注释部分是需要着重观察的地方

 - dev调用方改造

   ```yaml
   name: Dev(master) release
   
   on:
     push:
       branches:
         - master
   
   jobs:
   	version-bump:
   		# ...
   	deploy-aws:
   		uses: ./.github/workflows/deploy-aws.yml # 调用复用工作流
       with: # 复用工作流参数设置
         envName: dev
         releaseSha: ${{ needs.version-bump.outputs.release-sha }}
       secrets: inherit # 复用工作流机密参数设置
       name: Deploy AWS - Dev environment
       needs: version-bump
   ```

 - staging调用方改造

   ```yaml
   name: Staging release
   
   on:
     push:
       branches:
         - release/*
   
   jobs:
     deploy-aws:
       uses: ./.github/workflows/deploy-aws.yml # 调用复用工作流
       with: # 复用工作流参数设置
         envName: staging
       secrets: inherit # 复用工作流机密参数设置
       name: Deploy AWS - Staging environment
   ```

- prod调用方改造

  ```yaml
  name: Production release
  
  on:
    workflow_dispatch:
  
  jobs:
    deploy-aws:
      uses: ./.github/workflows/deploy-aws.yml # 调用复用工作流
      with: # 复用工作流参数设置
        envName: prod
      secrets: inherit # 复用工作流机密参数设置
      name: Deploy AWS - Production environment
  ```

# 3. 总结

复用工作流不仅支持在同一个代码仓库中创建和调用复用工作流, 也支持跨仓库调用复用工作流; 

复用工作流的`secrets`继承机制可以让我们在定义复用工作流时少些很多机密参数,  又因为复用工作永远使用调用方的上下文所以结合Github Envivronment使用切换环境和参数很方便; 

复用工作流在代码工程/并行协作上也增加了质量和效率;

总之, 复用工作流是Github Action的一个最佳实践, 推荐使用!

