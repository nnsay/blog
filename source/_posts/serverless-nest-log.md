---
title: AWS Lambda 日志优化
date: 2023-06-07 18:11:54
tags:
  - Tips
  - DevOps
excerpt: 自定义 Nest 日志让 AWS Lambda 日志自动携带 awsRequestId
---

# 1. 业务场景

AWS Lambda 的日志中会打印`awsRequestId`, 过该 id 可以获取一次调用的相关执行日志进而帮助开发者查找问题.

虽然 Nodejs 中`console`对象写日志 Lambda 运行时会自动加上`awsRequestId`, 但是如果其他基于标准输出写的日志的工具运行时是无法加上`awsRequestId`的, 比如 Nestjs 中的`Logger`.

本文介绍一个代码片段提供一个自定义的 Logger, 可以记录`awsRequestId`.

# 2. 代码片段

```typescript
/* eslint-disable @typescript-eslint/no-explicit-any */
import { ConsoleLogger, ConsoleLoggerOptions } from "@nestjs/common";
import { AsyncLocalStorage } from "async_hooks";
import { Handler } from "aws-lambda";

export class AWSLogger extends ConsoleLogger {
  private storage: AsyncLocalStorage<string>;

  constructor(
    storage: AsyncLocalStorage<string>,
    context?: string,
    options?: ConsoleLoggerOptions
  ) {
    super(context, options);
    this.storage = storage;
  }

  private getRequestId() {
    return this.storage.getStore();
  }

  private getLogParams(message: any, ...optionalParams: any[]) {
    const requestId = this.getRequestId();
    return requestId
      ? {
          content: `RequestId: ${requestId} | ${message}`,
          params: optionalParams,
        }
      : { content: message, params: optionalParams };
  }

  /**
   * Write a 'log' level log.
   */
  log(message: any, ...optionalParams: any[]) {
    const { content, params } = this.getLogParams(message, ...optionalParams);
    super.log(content, ...params);
  }

  /**
   * Write an 'error' level log.
   */
  error(message: any, ...optionalParams: any[]) {
    const { content, params } = this.getLogParams(message, ...optionalParams);
    super.error(content, ...params);
  }

  /**
   * Write a 'warn' level log.
   */
  warn(message: any, ...optionalParams: any[]) {
    const { content, params } = this.getLogParams(message, ...optionalParams);
    super.warn(content, ...params);
  }

  /**
   * Write a 'debug' level log.
   */
  debug(message: any, ...optionalParams: any[]) {
    const { content, params } = this.getLogParams(message, ...optionalParams);
    super.debug(content, ...params);
  }

  /**
   * Write a 'verbose' level log.
   */
  verbose(message: any, ...optionalParams: any[]) {
    const { content, params } = this.getLogParams(message, ...optionalParams);
    super.verbose(content, ...params);
  }
}

class AwsRequestIdHelper {
  private storage: AsyncLocalStorage<string>;
  private logger: AWSLogger;
  constructor() {
    this.storage = new AsyncLocalStorage<string>();
    this.logger = new AWSLogger(this.storage);
  }

  getRequestId(): string {
    return this.storage.getStore();
  }

  wrapHandler(handler: Handler, logEvent = false): Handler {
    return async (event, context, callback?) => {
      return await this.storage.run(context.awsRequestId, () => {
        if (logEvent) this.logger.log(`Event: ${JSON.stringify(event)}`);
        return handler(event, context, callback);
      });
    };
  }

  getLogger(context?: string, options?: ConsoleLoggerOptions) {
    return new AWSLogger(this.storage, context, options);
  }
}

export const awsRequestIdHelper = new AwsRequestIdHelper();
```

使用方法:

1. 通过导出对象`awsRequestIdHelper`的`wrapHandler`封装已经定义好的 Lambda Handler 函数
2. 通过导出对象`awsRequestIdHelper`的`getLogger`获得一个 Nest Logger 实例写日志,

# 3. 使用技巧

- 替换 Nest 默认`Logger`

  无侵入的替换掉默认 Logger,让已经使用默认 Logger 写的日志都带上`awsRequestId`

  ```typescript
  const app = await NestFactory.create(AppModule, {
    logger: awsRequestIdHelper.getLogger(),
  });
  ```

  如此替换后, 以下写法的日志均可以带`awsRequestId`

  ```typescript
  import { Logger } from "@nestjs/common";
  import { awsRequestIdHelper } from "xxxx";

  const log = awsRequestIdHelper.getLogger("DebugLog");
  log.log("log comes from custom name logger");
  Logger.log("log comes from the nest default logger");
  ```

- Nest 拦截器(中间件)中使用自定义的`AWSLogger`记录请求响应日志

  ```typescript
  import {
    Injectable,
    NestInterceptor,
    ExecutionContext,
    CallHandler,
    Logger,
    HttpException,
  } from "@nestjs/common";
  import { Observable, throwError } from "rxjs";
  import { catchError, tap } from "rxjs/operators";
  import { awsRequestIdHelper } from "./lib";

  @Injectable()
  export class LoggingInterceptor implements NestInterceptor {
    private readonly logger = new Logger(LoggingInterceptor.name);

    intercept = (
      context: ExecutionContext,
      next: CallHandler
    ): Observable<any> => {
      const request = context.switchToHttp().getRequest();
      const response = context.switchToHttp().getResponse();
      const { method, url, params, query } = request;
      const requestId = awsRequestIdHelper.getRequestId();

      this.logger.log(
        `IN ${requestId} ${method} ${url} ${context.getClass().name} ${
          context.getHandler().name
        } ${JSON.stringify(params)} ${JSON.stringify(query)}`
      );
      const now = Date.now();
      return next.handle().pipe(
        tap(() =>
          this.logger.log(
            `OUT ${requestId} ${Date.now() - now}ms ${response.statusCode}`
          )
        ),
        catchError((err) => {
          const code = err.getStatus ? err.getStatus() : err.status || "10000";
          this.logger.error(
            `OUT ${requestId} ${Date.now() - now}ms ${
              response.statusCode
            } ${code} '${err.message}'`
          );
          return throwError(() => err);
        })
      );
    };
  }
  ```

  > 注意:
  >
  > 15 行, 虽然这里用的 Nest 默认 Logger 但是通过在`NestFactory.create`修改默认`logger`属性为自定义 Logger 让中间件可以打印`awsReqeustId`这就是无侵入的好处.

​ 日志效果如下:

```txt
[Nest] 8  - 06/07/2023, 4:12:08 AM     LOG [LoggingInterceptor] 6686f59f-ff0a-4b0c-a49b-a978db74cdd5 IN 6686f59f-ff0a-4b0c-a49b-a978db74cdd5 GET /nestlog AppController nestlog
{}

{}

[Nest] 8  - 06/07/2023, 4:12:08 AM     LOG [LoggingInterceptor] 6686f59f-ff0a-4b0c-a49b-a978db74cdd5 OUT 6686f59f-ff0a-4b0c-a49b-a978db74cdd5 3ms 200
```
