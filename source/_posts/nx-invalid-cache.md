---
title: Nx缓存失效探究
date: 2023-07-20 18:03:19
tags:
  - DevOps
excerpt: 关于Nx缓存失效的Troubleshooting
---

# 1. Nx 缓存无法命中

复现步骤:

```bash
yarn nx run-many --target=build -p auth-guard
yarn nx run-many --target=build -p power-calculator
```

power-calculator 本身依赖 auth-guard, 先执行 auth-guard 会产生缓存, 但是 power-calculator 不能够使用前一个步骤产生的缓存, 而重新 build auth-guard

# 2. Troubleshooting

根据 Nx 源码得知 Hash 生成相关的两个文件:

- hash-task.js
- task-hasher.js

其中`TaskHasherImpl`是核心的 Hash 类, `hashTask`会对以下内容进行 Hash 计算, 最后将结果合并再做一次 Hash 作为最后的 Hash:

- 自身`target`的`input`文件
- 自身项目依赖,以及依赖的依赖
- `target`执行器

在`TaskHasherImpl`的`hashExternalDependency` 方法中增加日志:

```typescript
if (this.projectGraph.dependencies[projectName]) {
  // 跟踪rc-tabs变化
  projectName === "npm:rc-tabs" &&
    console.log(this.projectGraph.dependencies[projectName]);
  this.projectGraph.dependencies[projectName].forEach((d) => {
    if (!visited.has(d.target)) {
      // 跟踪rc-tabs变化
      projectName === "npm:rc-tabs" &&
        console.log(d.target, this.hashExternalDependency(d.target, visited));
      // 跟踪antd变化
      // projectName === 'npm:antd' && console.log(d.target, this.hashExternalDependency(d.target, visited))
      partialHashes.push(this.hashExternalDependency(d.target, visited));
    }
  });
}
```

通过增加的日志, 可以得出结论虽然复现步骤中的两条命令都会产生`auth-guard`但是其 hash 不一样, 其中一个原因是`antd`的 hash 不一样, 继续跟踪可以发现`rc-tabs`的 hash 不一样, 而`rc-tabs`的依赖是固定, 具体查看一下图片:

<img src="https://raw.githubusercontent.com/nnsay/gist/main/imgimage-20230720175352211.png" alt="auth-guard依赖项hash" style="zoom:50%;" />

<img src="https://raw.githubusercontent.com/nnsay/gist/main/imgimage-20230720175434407.png" alt="=antd依赖项hash" style="zoom:50%;" />

<img src="https://raw.githubusercontent.com/nnsay/gist/main/imgimage-20230720175529889.png" alt="rc-tabs依赖项hash" style="zoom:50%;" />

根据最后的图片, 依赖项居然少了一个, 所以问题出在如下地方:

```typescript
if (!visited.has(d.target)) {
```

如果我们把这行修改为:

```typescript
if (true || !visited.has(d.target)) {
```

就可以发现 hash 是一样了, 所以这里的`visited`影响了依赖的 hash 计算, **这里是个 Bug**. 出问题的 Nx 版本是: 16.3.2 但是切换源码到 16.5.1 发现这里已经修改为:

```typescript
// we want to calculate the hash of the entire dependency tree
if (this.projectGraph.dependencies[targetProjectName]) {
  this.projectGraph.dependencies[targetProjectName].forEach((d) => {
    if (
      !visited.has(
        this.computeExternalDependencyIdentifier(targetProjectName, d.target)
      )
    ) {
      partialHashes.push(
        this.hashExternalDependency(targetProjectName, d.target, visited)
      );
    }
  });
}
```

根据注释可以从计算某个包的整个树的 hash, 我们可以把新代码的修复方法放在老代码中, 例如:

```bash
hashDepsInputs(inputs, projectGraphDeps, visited) {
    // console.log({inputs, projectGraphDeps, visited})
    return tslib_1.__awaiter(this, void 0, void 0, function* () {
        return (yield Promise.all(inputs.map((input) => tslib_1.__awaiter(this, void 0, void 0, function* () {
            return yield Promise.all(projectGraphDeps.map((d) => tslib_1.__awaiter(this, void 0, void 0, function* () {
                if (visited.indexOf(d.target) > -1) {
                    return null;
                }
                else {
                    visited.push(d.target);
                    if (this.projectGraph.nodes[d.target]) {
                        return yield this.hashNamedInputForDependencies(d.target, input.input || 'default', visited);
                    }
                    else {
                        const hash = this.hashExternalDependencyV2(d.source, d.target);
                        return {
                            value: hash,
                            details: {
                                [d.target]: hash,
                            },
                        };
                    }
                }
            })));
        }))))
            .flat()
            .filter((r) => !!r);
    });
}
computeExternalDependencyIdentifier(
  sourceProjectName,
  targetProjectName
) {
  return `${sourceProjectName}->${targetProjectName}`;
}
hashExternalDependencyV2(
      sourceProjectName,
      targetProjectName,
      visited = new Set()
) {
  // try to retrieve the hash from cache
  if (this.externalDepsHashCache[targetProjectName]) {
    return this.externalDepsHashCache[targetProjectName];
  }

  visited.add(
    this.computeExternalDependencyIdentifier(
      sourceProjectName,
      targetProjectName
    )
  );
  const node = this.projectGraph.externalNodes[targetProjectName];
  let partialHash;
  if (node) {
    const partialHashes = [];
    if (node.data.hash) {
      // we already know the hash of this dependency
      partialHashes.push(node.data.hash);
    } else {
      // we take version as a hash
      partialHashes.push(node.data.version);
    }
    // we want to calculate the hash of the entire dependency tree
    if (this.projectGraph.dependencies[targetProjectName]) {
      this.projectGraph.dependencies[targetProjectName].forEach((d) => {
        if (
          !visited.has(
            this.computeExternalDependencyIdentifier(
              targetProjectName,
              d.target
            )
          )
        ) {
          partialHashes.push(
            this.hashExternalDependencyV2(targetProjectName, d.target, visited)
          );
        }
      });
    }

    partialHash = (0, impl_1.hashArray)(partialHashes);
  } else {
    // unknown dependency
    // this may occur if dependency is not an npm package
    // but rather symlinked in node_modules or it's pointing to a remote git repo
    // in this case we have no information about the versioning of the given package
    partialHash = `__${targetProjectName}__`;
  }
  this.externalDepsHashCache[targetProjectName] = partialHash;
  return partialHash;
}
```

经过测试这个修改修复了大部分因为非全树 hash 而导致 hash 不一致, 但是最后还是发现`auth-guard`在两次复现命令执行后 hash 还是不一样, 这次是`@nx/cypress`不一样, 导致这个包 hash 异常的依赖有:

```
npm:@nx/cypress npm:@nx/devkit 17501148780570736231
npm:@nx/cypress npm:@nx/js 14007381397752617907
npm:@nx/cypress npm:@nx/linter 9710830278822790515
npm:@nx/cypress npm:@nrwl/cypress 9257102550312002577
```

# 3. 总结

Nx 计算 Hash 直接影响缓存的使用, 在最新的版本中有加入的机器 ID 的检查, 所以本地缓存复用会变的更困难, Nx 的 Hash 计算和缓存复用至少在同命令同机器上好用的, 但是需要注意缓存失效及其容易发生, 缓存管理不透明也不好控制, 希望通过缓存加速的可以有限使用, 比如同 PR 同流水线同 Base 缓存命中是好用的.
