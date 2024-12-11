---
title: AWS/Aliyun函数计算异步调用
date: 2023-02-03 10:04:43
tags:
  - Aliyun
  - AWS
  - DevOps
  - CloudNative
excerpt: AWS/Aliyun函数计算异步调用
---

## 0. 问题背景介绍

aws 和 aliyun 的函数计算常用的都是同步调用, 同步调用当语言本身支持 CSP 方式还好(不阻塞主流程, 但是可能触发阈值限流), 但是当没有异步处理同步函数调用可能会面临超时和限流的问题, 因此对于函数计算调用较多的业务应该选择更科学的调用方式--函数计算异步调用.

## 1. 阿里云-[函数计算](https://help.aliyun.com/product/50980.html)

- [同步调用](https://help.aliyun.com/document_detail/52878.html)

  - 使用限制

    资源调用限制：您的阿里云账号（主账号）在单个地域内默认的按量实例上限数为 300.

    阿里云沟通: 文档的意思是主账号下面的 RAM，STS，主账号自己 等等各种账号 共享这 300 个实例。比如主账号同时占用了 200 个实例，ram 和 sts 只能用 100 个实例了. 1 个实例可以配置并发度从 1 到 100，所以默认 300 个实例，最大并发度是 100\*300 = 3 万个. 请求 1，2，3，4，5, ......, 300 ，这 300 个请求如果把实例用完了，第 301 个请求就报错限流了。但是如果在第 302 个请求来的时候，请求 1 已经处理完了，这个第 302 个请求就不会报错了.

    目前我们阿里所有的函数实例配置都是 1, 所以我们账号级别(app/stagging)下所有产品所有服务整合其他最大并发是 300.

  - 限制分析

    - 300 限制不大
    - 300 的限制是按服务实例的, 疗法云和科研云共享实例, m42 环境本月最高到 25

- [异步调用](https://help.aliyun.com/document_detail/181866.html)

  ```go
  package main

  import (
      "fmt"
      "os"

      "github.com/aliyun/fc-go-sdk"
  )
  func main() {
      fcClient, err := fc.NewClient(fmt.Sprintf("%s.cn-shanghai.fc.aliyuncs.com", os.Getenv("ACCOUNT_ID")),
          "2016-08-15", os.Getenv("ACCESS_KEY_ID"), os.Getenv("ACCESS_KEY_ID_SECRET"))
      if err != nil {
          panic(err)
      }

      // invoke function with delay
      invokeInput := fc.NewInvokeFunctionInput({ServiceName}, {FunctionName}).WithPayload({payload})
      invokeInput = invokeInput.WithAsyncInvocation().WithHeader("x-fc-async-delay", "200")
      _, err := FCClient.InvokeFunction(invokeInput)
      if err != nil {
          panic(err)
      }
  }
  ```

  - **_重试次数_**: 默认三次

- 代码现状

  ```java
  public void testFc() {
          InvokeFunctionRequest invokeReq = InvokeFunctionRequest.builder()
                  .serviceName("fc-debug")
                  .functionName("fc-debug-function1")
                  .body("{}".getBytes())
                  .requestConfiguration(RequestConfiguration.create()
                          .setResponseTimeout(Duration.ofMinutes(3)))
                  .build();
          String xFcInvocationType = invokeReq.getXFcInvocationType();
          log.info("invoke type: {}", xFcInvocationType);
          CompletableFuture<InvokeFunctionResponse> response = this.fcClient.invokeFunction(invokeReq);
      }
  ```

  第 10 行为空, 而源码:

  ```java
          /**
           * 调用方式:Sync或者Async，默认值：Sync
           */
          public Builder xFcInvocationType(String xFcInvocationType) {
              this.putHeaderParameter("X-Fc-Invocation-Type", xFcInvocationType);
              this.xFcInvocationType = xFcInvocationType;
              return this;
          }
  ```

  从源码我们的现在的 FC 是**_同步调用_**的, 即有 300 的并发限制, 现有调用是同步也可以从阿里云控制台得知:
  https://fcnext.console.aliyun.com/cn-hangzhou/services/m42-ng-fc/function-detail/m42-ptc-surf2surf/LATEST?tab=log
  https://fcnext.console.aliyun.com/cn-hangzhou/services/m42-ng-fc/function-detail/m42-apply-dicom/LATEST?tab=log
  (点击任意请求 ID, 在右侧弹框顶部状态栏有调用方式的显示)

- 同步改异步

  设置`xFcInvocationType`为`Async`, 如下:

  ```java
  public void testFc() {
          InvokeFunctionRequest invokeReq = InvokeFunctionRequest.builder()
              .serviceName("fc-debug")
              .functionName("fc-debug-function1")
              .body("{}".getBytes())
              .requestConfiguration(RequestConfiguration.create()
                  .setResponseTimeout(Duration.ofMinutes(3))
              )
              .xFcInvocationType("Async")
              .build();
          String xFcInvocationType = invokeReq.getXFcInvocationType();
          log.info("invoke type: {}", xFcInvocationType);
          CompletableFuture<InvokeFunctionResponse> response = this.fcClient.invokeFunction(invokeReq);
      }
  ```

- [异步任务](https://help.aliyun.com/document_detail/372531.html)

  异步调用修改配置, 开发异步任务即可实现异步调用的精细化管理, 例如取消/停止/去重等.

## 2. AWS-Lambda

- 异步调用

  - [文档](https://docs.aws.amazon.com/lambda/latest/dg/invocation-async.html)
  - [异步调用 API](https://docs.aws.amazon.com/lambda/latest/dg/API_Invoke.html)

- 源码分析

  ```java
  Invokes a Lambda function. You can invoke a function synchronously (and wait for the response), or asynchronously. To invoke a function asynchronously, set InvocationType to Event.
  For synchronous invocation , details about the function response, including errors, are included in the response body and headers. For either invocation type, you can find more information in the execution log  and trace .

  public CompletableFuture<InvokeResponse> invoke(InvokeRequest invokeRequest) {
  ```

- 同步改异步

  按照源码或者文档, 设置`InvocationType`为`Event`, 如下:

  ```java
  public void testAwsFc() {
          InvokeRequest invokeRequest = InvokeRequest.builder()
              .functionName("fc-debug-function1")
              .logType("None")
              .payload(SdkBytes.fromUtf8String("{}"))
              .invocationType(InvocationType.EVENT)
              .overrideConfiguration(a-> a.apiCallTimeout(Duration.ofMinutes(3)).build()).build();

          CompletableFuture<InvokeResponse> invoke = this.lambdaClient.invoke(invokeRequest);
      }
  ```

## 3. 总结

阿里云同步调用有并发限制 300, aws 对于同步调用的并发没有特别说明, aws 默认函数并发 1000. 鉴于同步并发在 java 进程中属于等待状态, 可能会影响性能, 不建议使用同步调用方式调用(以前 nodejs 版本也是同步调用, 但是语言特性上让产品没有感觉等待迟滞); 阿里云的同步调用限制 300 如何科研云共享服务实例, 则也有可能产品之间互有影响; 建议修改为异步调用方式.
