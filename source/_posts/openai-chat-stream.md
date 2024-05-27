---
title: openai chat stream
date: 2024-05-27 17:57:39
tags:
  - 研究
  - Code
excerpt: 使用openai sdk产生对话流数据并在前端处理显示
---

# 1. 介绍

本教程使用 openai sdk 调用大模型并将大模型流式返回给前端, 以及前端如何承接这种数据流和处理.

注意: 本文虽然使用了 openai sdk 但是大模型并不是 openai 的 chatgpt 系列模型, 而是 [moonshot](https://platform.moonshot.cn/docs/intro#%E4%B8%BB%E8%A6%81%E6%A6%82%E5%BF%B5), 该模型出自月之暗面 openai 的 api 已经成为其他大模型服务标准, 经过简单的调研: 阿里云(qwen)和月之暗面(moonshot)都是支持使用 openai sdk 记性调用的, 比如: 文件操作(创建/列表/删除/基本信息), 会话

使用 openai sdk 调用其他模型也比较简单, 只需要按照各个厂商的文档, 基本只需要修改以下两项:

```ts
import { OpenAI } from "openai";

const client = new OpenAI({
  apiKey: process.env.MOONSHOT_API_KEY,
  baseURL: "https://api.moonshot.cn/v1",
});
```

# 2. 后端设计和实现

通常不会在前端直接调用大模型 API, 因为 Key 有泄露的风险, 所以一般会在自己的后端服务中封装一下, 这里使用 nodejs 的后端服务框架 nest 实现, 因为逻辑简单这里不写 Controller 路由的, 仅实现 Service.

要实现和大模型交互并返回结果核心的步骤有以下几个:

- 初始化 openai sdk
- 调用大模型
- 返回大模型流式结果

```ts
import { Injectable, Logger } from "@nestjs/common";
import { ConfigService } from "@nestjs/config";
import { OpenAI } from "openai";
import type { Response } from "express";

@Injectable()
export class LLMService {
  private readonly logger = new Logger(LLMService.name);
  private readonly client: OpenAI;

  constructor(private readonly configService: ConfigService) {
    this.client = new OpenAI({
      apiKey: this.configService.get("MOONSHOT_API_KEY"),
      baseURL: "https://api.moonshot.cn/v1",
    });
  }

  /**
   * 对话
   * @param data 对话上下文参数
   * @returns stream
   */
  async chat(data: { question: string }, res: Response) {
    this.logger.log("chat");
    const chunkStream = await this.client.chat.completions.create({
      model: "moonshot-v1-8k",
      messages: [
        {
          role: "system",
          content: "你是数学老师的智能助理, 可以帮助老师处理各类教学问题",
        },
        {
          role: "system",
          content: "请准确精简回答, 请勿过度发散",
        },
        { role: "user", content: data.question },
      ],
      stream: true,
    });
    res.setHeader("Content-Type", "text/event-stream");
    res.setHeader("Cache-Control", "no-cache");
    res.setHeader("Connection", "keep-alive");
    for await (const chunk of chunkStream) {
      // NOTE: Print the token usage when the chunk stop
      if (chunk.choices[0] && chunk.choices[0]["usage"]) {
        this.logger.log(`usage: ${JSON.stringify(chunk.choices[0]["usage"])}`);
      }
      res.write(chunk.choices[0]?.delta?.content || "");
    }
    res.end();
  }
}
```

在上面的代码中, 调用大模型传递参数: stream: true, 这样模型会实时吐出文本信息, 我们将每个 chunk 的文本信息直接通过 res.write 返回, 在最后使用 res.end 结束 HTTP 请求.

在返回 chunk 流之前我们设置了 http 返回数据类型是: text/event-stream, 这是一种后端推送数据技术, 和 websocket 不一样的只支持从服务器端推送到客户端.

# 3. 前端设计和实现

前端采用 vue 来演示如何请求后端 API 并处理 text/event-stream 数据.

```vue
<template>
  <h1>获取EventSource流</h1>
  <div class="container">
    <input
      type="text"
      name="question"
      placeholder="请详细介绍一下安妮海瑟薇"
      style="width: 100%;"
    />
    <button type="button" @click="sendQuestion" style="width: 10%;">
      提问
    </button>
  </div>
  <div class="container">
    <textarea type="text" name="answer"></textarea>
  </div>
</template>

<script setup lang="ts">
import axios from "axios";

const sendQuestion = async () => {
  const answer = document.querySelector<HTMLTextAreaElement>(
    'textarea[name="answer"]'
  )!;
  const questionInput = document.querySelector<HTMLInputElement>(
    'input[name="question"]'
  )!;
  const question = questionInput.value;
  console.log({ question });

  const url = "http://localhost:4020/chatbot/llm/chatstream";
  const res = await fetch(url, {
    method: "POST",
    headers: {
      "Content-Type": "application/json", // 该header明确指定传递的参数是JSON格式, 去除后会导致后端参数验证失败
      Accept: "text/event-stream", // 可忽略
    },
    body: JSON.stringify({ question }),
  });
  const reader = res.body!.getReader();
  const decoder = new TextDecoder();

  while (true) {
    const { done, value } = await reader.read();
    if (done) break;
    const text = decoder.decode(value, { stream: true });
    answer.textContent += text;
  }
};
</script>

<style scoped>
.container {
  display: flex;
  flex-direction: row;
  align-items: stretch;
  /* 默认值，可选 */

  /* 可以根据需要调整高度 */
}

textarea {
  height: 300px;
  flex: 1;
  resize: none;
  /* 可选，防止用户调整文本框大小 */
  margin-bottom: 0px;
  /* 可选，设置按钮和文本框之间的间距 */
}
</style>
```

在上面的代码中使用 fetch 请求后端接口, 当然也可以用别的, 前提是支持 ReadableStream, 经过代码实践 axios 是不可用的.

# 4. 总结

本文的代码虽然不复杂, 但是提供了两个开发技巧:

- openai 成为了后续大模型的 api 交互标准, 所以可以使用 openai 的 sdk 操作其他大模型
- 可以使用 event-stream 流向客户端推送实时处理结果
