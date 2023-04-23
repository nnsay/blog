---
title: Sentry最佳实践(不定期更新)
date: 2023-04-23 12:56:17
tags:
  - Tips
  - DevOps
excerpt: 了解Sentry真实项目中的最佳使用实践
---

## 更新日志

- 2023-04-23 13:00: 增加性能和问题分配的实践
- 2023-04-23 16:41: 增加问题过滤实践

## 1. 实践一: 性能优化

最近的实践发现服务加上 Sentry 会性能降低很多, 做了一些对比测试即有 Sentry 和无 Sentry 的相同 API 访问测试, 发现有 Sentry 很影响性能, 推断影响性能的原因:

- Sentry 默认会开启性能监控, 性能监控会进行采样, 采样会影响性能
- 服务本身除了集成 Sentry 还开启了 Source Map, 该特性也影响性能

省略测试过程, 测试结果如下:

- Source Map 和 Sentry 都会影响性能
- 影响性能因素:
  - Sentry Trace
  - Sentry Senssion
  - Source Map
- Sentry Trace 影响效果 > Source Map
- 减少 Sentry Trance Sample Rate 可以提高性能

根据以上测试和实验, 最佳实践是: **保留 Source Map, 禁用性能监控**. 具体做法如下:

```typescript
Sentry.AWSLambda.init({
  dsn: process.env.SENTRY_DSN,
  environment,
  enableTracing: false,
  autoSessionTracking: false,
  enabled: environment !== "local",
});
```

## 2. 实践二: 问题自动分配

Sentry 的 Issue 默认不会分配到某个人, 但是通过配置归属规则可以指定哪些 Issue 分配到哪个人负责解决问题, 这个分配的行为和 Github Issue Assign 的行为一样. 具体设置分配规则可以参考文档: [ownership-rules](https://docs.sentry.io/product/issues/ownership-rules/), Sentry 的规则写法是: type:pattern owners

- type: 匹配类型, 可以选择: path, module, url, tag
- pattern: 是默认匹配, 匹配规则: https://en.wikipedia.org/wiki/Glob_(programming)
- owners: 是分配的人或者团队, 分配到人可以写邮箱, 分配到团队可以写团队名称(带#)

假设我们是用使用 Lambda, user-service 有问题分配给 a(a@mail.com), report-service 有问题分配给 b(b@mail.com)则规则如下:

```
tags.transaction:user-service* a@mail.com
tags.transaction:report-service* b@mail.com
```

其中`tags.transaction`是 Sentry SDK 集成后默认会有的 Tag, 根据分析, 这个值是和 Lambda 的名字有关, 而`*`是为了默认匹配, 因为这个服务被部署到多个环境, 所以后缀是不一样的

## 3. 实践三: 异常过滤

为了避免非必要的异常淹没重要的异常, 也为了开发者可以更关注, 过滤异常可以有以下两个方法:

### 3.1 捕获时过滤

因为捕获是通过 SDK 的`captureException`方法实现的, 所以可以在执行`captureException`之前进行条件判断, 代码如下:

```typescript
// 自定义错误
class ValidationError extends Error {}

// 捕获前过滤
if (exception.constructor.name !== "ValidationError") {
  Sentry.captureException(exception);
}
```

### 3.2 通知时过滤

创建 Alert 时可以增加过滤, 这个方式与上面的方式不同之处是: 通知时过滤 Event 是已经被捕获和发送了, 只是在通知的时候不关注. 继续使用`3.1`中的案例, 通知时过滤的过滤条件可以如下设置:

The event's `exception.type` value `does not equal` `ValidationError`

当然通知过滤支持更丰富的过滤条件, 具体可以从[这里](https://docs.sentry.io/product/alerts/create-alerts/issue-alert-config/#if-conditions-filters)查看.
