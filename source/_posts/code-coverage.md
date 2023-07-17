---
title: 基于 Monorepo 的自定义代码覆盖率报告
date: 2023-07-17 08:47:58
tags:
  - Tips
  - DevOps
excerpt: Monorepo 中代码测试覆盖率报告的使用和实践
---

## 1. 业务背景

如果组织代码使用 Monorepo 的架构则代码仓库中有很多项目, 以 Nx 框架为例, 有些项目是应用类, 有些工具类或者库, 每个项目都有单元测试, 进而有不同的代码覆盖率, 如何将多个项目的覆盖率报告按照项目分组并结合 Github PR 使用是本文的实践, 主要包含以下内容:

- 介绍两个脚本, 分别针对`Istanbul`的`json`和`json-summary`报告格式进行二次统计, 进而生成自己的覆盖率报告, 达到按照项目分组和更高度自定义的需求
- 介绍两个 Gihub Workflow 的步骤
  - 如何在 PR 中提交 Comment, 实现自定义覆盖率的报告在 PR 上显示
  - 如何在流水线中检查阈值, 覆盖率低于阈值流水线失败, 这对 PR 质量要求严格的团队是必须要的特性

## 2. 脚本

覆盖率报告有很多格式, 常见的有 json/json-summary/clover/lcov/text 等, 具体可以查[这里](https://istanbul.js.org/docs/advanced/alternative-reporters/)查看详情. 对于 nodejs 编程比较友好是 json 格式, 以下是具体两种 json 格式的自定义统计报告, 报告**以 Monorepo 中的应用程序分类统计**, 而不是整体统计.

### 2.1 json

```js
/**
 * Copyright © https://github.com/pveyes/byzantine
 */

import * as fs from 'fs'
import * as path from 'path'

interface ReportType {
  coverageMap: {
    [path: string]: {
      path: string
      s: { [k: string]: number }
      f: { [k: string]: number }
      b: { [k: string]: number[] }
    }
  }
}

type AggregateReportItemType = {
  covered: number
  all: number
}

type AggregateResult = {
  branches: number
  statements: number
  functions: number
}

type CoverageResult = {
  path: string
  branches: AggregateReportItemType
  statements: AggregateReportItemType
  functions: AggregateReportItemType
}

function generateEmptyReport(): AggregateReportItemType {
  return {
    covered: 0,
    all: 0,
  }
}

function aggregateCoverage(coverages: CoverageResult[]): AggregateResult {
  let statements = 0
  let statementCount = 0
  let branches = 0
  let branchCount = 0
  let functions = 0
  let functionCount = 0

  if (coverages.length === 0) {
    return {
      statements: 0,
      branches: 0,
      functions: 0,
    }
  }

  coverages.forEach(coverage => {
    statements += coverage.statements.covered
    statementCount += coverage.statements.all

    branches += coverage.branches.covered
    branchCount += coverage.branches.all

    functions += coverage.functions.covered
    functionCount += coverage.functions.all
  })

  if (statementCount === 0) {
    statements = 100
  } else {
    statements = (statements / statementCount) * 100
  }

  if (branchCount === 0) {
    branches = 100
  } else {
    branches = (branches / branchCount) * 100
  }

  if (functionCount === 0) {
    functions = 100
  } else {
    functions = (functions / functionCount) * 100
  }

  return {
    statements,
    branches,
    functions,
  }
}

function parseCoverage(json: ReportType): CoverageResult[] {
  const paths = Object.keys(json.coverageMap)

  return paths.map(path => {
    const coverage = json.coverageMap[path]

    const branches = generateEmptyReport()
    const statements = generateEmptyReport()
    const functions = generateEmptyReport()

    Object.keys(coverage.b).forEach(id => {
      const result = coverage.b[id]
      result.forEach(r => {
        branches.covered += r > 0 ? 1 : 0
        branches.all += 1
      })
    })

    Object.keys(coverage.s).forEach(id => {
      statements.covered += coverage.s[id] > 0 ? 1 : 0
      statements.all += 1
    })

    Object.keys(coverage.f).forEach(id => {
      functions.covered += coverage.f[id] > 0 ? 1 : 0
      functions.all += 1
    })

    // We don't return line coverage because it's no longer relevant
    // https://github.com/gotwarlost/istanbul/issues/639
    return {
      path,
      branches,
      statements,
      functions,
    }
  })
}

const baseDir = path.resolve(__dirname, '../coverage/packages')

const paths: string[] = []
fs.readdirSync(baseDir).forEach(app => {
  const report = `${baseDir}/${app}/report.json`
  if (fs.existsSync(report)) {
    paths.push(report)
  }
})

if (paths.length) {
  console.log('# Coverage report')
  console.log('|App|Statements|Branches|Functions|')
  console.log('|---|---|---|---|')
  paths.forEach(path => {
    const coverages = parseCoverage(require(path))
    const { statements, branches, functions } = aggregateCoverage(coverages)
    const app = path.split('/').at(-2)
    console.log(
      `|${app}|${statements.toFixed(2)}%|${branches.toFixed(2)}%|${functions.toFixed(2)}%|`
    )
  })
}

```

测试结果:

```markdown
# Coverage report

| App  | Statements | Branches | Functions |
| ---- | ---------- | -------- | --------- |
| app1 | 100.00%    | 100.00%  | 100.00%   |
| app2 | 89.90%     | 66.06%   | 90.61%    |
| app3 | 100.00%    | 100.00%  | 100.00%   |
| lib1 | 87.83%     | 66.67%   | 93.33%    |
```

### 2.2 json-summary

该方式是本人团队最后使用的方式, 具体脚本封装在 nx 自定义插件:

```ts
import { Tree, readJsonFile } from "@nx/devkit";
import * as fs from "fs";
import * as path from "path";
import { CodeCoverageReportGeneratorSchema } from "./schema";

export type CoverageSummary = {
  total: {
    lines: {
      total: number;
      covered: number;
      skipped: number;
      pct: number;
    };
    statements: {
      total: number;
      covered: number;
      skipped: number;
      pct: number;
    };
    functions: {
      total: number;
      covered: number;
      skipped: number;
      pct: number;
    };
    branches: {
      total: number;
      covered: number;
      skipped: number;
      pct: number;
    };
    branchesTrue: {
      total: number;
      covered: number;
      skipped: number;
      pct: number;
    };
  };
};

// red: < 65
// yellow: >=65 && <80
// green: > 80
export const formatCoverageValue = (v: number) => {
  if (v < 65) {
    return `🔴 ${v.toFixed(2)}%`;
  } else if (v > 80) {
    return `🟢 ${v.toFixed(2)}%`;
  } else {
    return `🟡 ${v.toFixed(2)}%`;
  }
};

export interface CodeCoverageReportGeneratorSchema {
  coverageDir: string;
  reportPath: string;
  limitTarget: number;
}

export async function codeCoverageReportGenerator(
  tree: Tree,
  options: CodeCoverageReportGeneratorSchema
) {
  const baseDir = path.resolve(process.cwd(), options.coverageDir);
  const reportJsonPaths: string[] = [];
  fs.readdirSync(baseDir).forEach((nxProject) => {
    const report = `${baseDir}/${nxProject}/coverage-summary.json`;
    if (fs.existsSync(report)) {
      reportJsonPaths.push(report);
    }
  });
  const exitFile = path.resolve(process.cwd(), `${options.reportPath}.exit`);
  fs.rmSync(exitFile, { force: true });
  if (reportJsonPaths.length) {
    const reportMarkdown: string[] = [];
    reportMarkdown.push("# Coverage report");
    reportMarkdown.push("|App|Lines|Statements|Branches|Functions|");
    reportMarkdown.push("|---|---|---|---|---|");
    reportJsonPaths.forEach((reportJsonPath) => {
      const coverages = readJsonFile(reportJsonPath) as CoverageSummary;
      const {
        total: { lines, statements, branches, functions },
      } = coverages;
      const app = reportJsonPath.split("/").at(-2);
      reportMarkdown.push(
        `|${app}|${formatCoverageValue(lines.pct)}|${formatCoverageValue(
          statements.pct
        )}|${formatCoverageValue(branches.pct)}|${formatCoverageValue(
          functions.pct
        )}|`
      );
      if (statements.pct < options.limitTarget) {
        const msg = `${app} statements: ${formatCoverageValue(
          statements.pct
        )} < ${options.limitTarget}`;
        const exitFile = path.resolve(
          process.cwd(),
          `${options.reportPath}.exit`
        );
        fs.writeFileSync(exitFile, `${msg}\n`, { flag: "a" });
      }
    });
    const reportFile = path.resolve(process.cwd(), options.reportPath);
    fs.writeFileSync(reportFile, reportMarkdown.join("\n"));
  }
}

export default codeCoverageReportGenerator;
```

测试结果如下, 比`json`报告多了一个行覆盖:

```markdown
# Coverage report

| App  | Lines      | Statements | Branches   | Functions  |
| ---- | ---------- | ---------- | ---------- | ---------- |
| app1 | 🟢 100.00% | 🟢 100.00% | 🟢 100.00% | 🟢 100.00% |
| app2 | 🟢 90.13%  | 🟢 89.90%  | 🟡 66.06%  | 🟢 90.61%  |
| app3 | 🟢 100.00% | 🟢 100.00% | 🟢 100.00% | 🟢 100.00% |
| lib1 | 🟢 89.22%  | 🟢 87.82%  | 🟡 66.66%  | 🟢 93.33%  |
```

## 3. 配合 Github Workflow 使用

### 3.1 [使用 GH 给 PR 写 Comment](https://cli.github.com/manual/gh_pr_comment)

这个功能主要是在 PR 运行完单元测试后例如上面的提到的脚本实现对覆盖率报告的收集, 并生成 markdown 文件, 这个 markdown 文件可以作为评论(Comment)写在 PR 上,方便查看覆盖率报告. 结合 2.2 中的使用 nx 生成器方式生产报告的方式如下:

```bash
- name: Generate custom code coverage report
  run: |
    mkdir -p coverage/packages
    mkdir -p tmp/
    yarn nx g nx-plugin:code-coverage-report --coverageDir coverage/packages --reportPath tmp/report.md

- name: Submit custom code coverage report
  run: |
    if [ -f tmp/report.md ]; then
      commentId=$(gh api \
        -H "Accept: application/vnd.github+json" \
        -H "X-GitHub-Api-Version: 2022-11-28" \
        "/repos/nnsay/core/issues/${{ github.event.number }}/comments" | jq -r '.[]|select(.body|test("Coverage report"))|.id')
      if [ -n "$commentId" ]; then
        echo "existed comment id: $commentId"
        gh api \
          --method DELETE \
          -H "Accept: application/vnd.github+json" \
          -H "X-GitHub-Api-Version: 2022-11-28" \
          "/repos/nnsay/core/issues/comments/$commentId"
      fi
      cat tmp/report.md
      gh pr comment ${{ github.event.number }} --body-file tmp/report.md
    fi
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}


```

### 3.2 覆盖率阈值检检查

```yaml
- name: Check the coverage limit target
  run: |
    if [ -f tmp/report.md.exit ]; then
      cat tmp/report.md.exit
      exit 1
    fi
```

‼️ 该步骤需要在生产自定义报告之后做, 步骤中的`report.md.exit`就是自定义 nx 生成器中生成的文件, 开发者可以参考这个思路实现更复杂和符合业务需求的逻辑.

## 4. 总结 (© claude.ai)

- 可以按照项目分类统计覆盖率,而不是仓库整体统计,更加准确反映每个项目的测试质量。
- 可以自定义报告格式,生成 markdown 等格式,便于在 CI/CD 流程中使用。
- 可以在 PR 评论中实时展示覆盖率报告,帮助 review。
- 可以设置覆盖率阈值,不达标时流水线失败,保证代码质量。
- 借助代码生成器,可以使报告更易定制,也方便与 CI/CD 系统集成。
- 总体来说,自定义报告让开发团队更加关注测试质量,也为持续改进测试奠定基础。
