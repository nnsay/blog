---
title: Dynamodb在Nest框架中如何写单元测试
date: 2023-04-18 19:24:24
tags:
  - Code
  - Tips
excerpt: 了解如何实现DynamoDB在Nest框架中如何进行依赖注入/mock数据
---

# 1. 问题背景

因为 Nestjs 是高度抽象的框架, 而单元测试通常是比较聚焦到某个文件(service/controller)的测试, 这个时候能 mock 这些文件的依赖对于单元测试来说至关重要. 本文以测试 service 文件为例, 通过 mock service 依赖注入的 DynamoDB 为例来做单元测试. 本文使用的开发框架和工具有:

- Nestjs: Nodejs 后端开发框架, 支持多种模式
- nestjs-dynamoose: 一个针对`dynamoose` ORM 的 Nest 封装, 适应 Nest 的 model 组织方式
- jest: 单元测试框架, 本文使用它 mock DB 操作的返回值

# 2. 代码实现

## 2.1 编写 servie

假设使用`app.service.ts`文件, 具体代码如下:

```typescript
import { Injectable } from "@nestjs/common";
import { InjectModel, Model } from "nestjs-dynamoose";
import { UserRecord, UserRecordKey } from "./app.schema";

@Injectable()
export class AppService {
  constructor(
    @InjectModel("User")
    private readonly userModule: Model<UserRecord, UserRecordKey>
  ) {}

  addData() {
    return this.userModule.create({
      userId: `uid-${Date.now()}`,
      info: {
        name: "demoUser",
        age: 100,
      },
    });
  }
  listData() {
    return this.userModule.scan().exec();
  }
}
```

上面的代码是一个常见的 Nest service, 需要注意的地方如下:

- 在构造函数中依赖注入`User` 的 DynamoDB Table
- 实际上`User`对象是`dynamoose`的`model`, 这里是我们 mock 的重点

## 2.2 单元测试

使用`app.service.spec.ts`文件, 具体代码如下:

```typescript
import { Test } from "@nestjs/testing";
import { AppService } from "./app.service";
import { DemoSchema, UserRecord } from "./app.schema";
import * as dynamoose from "dynamoose";

describe("AppService", () => {
  let service: AppService;
  const createdUser: UserRecord = {
    userId: "1",
    info: {
      name: "John Doe",
      age: 30,
    },
  };
  const userModel = dynamoose.model("User", DemoSchema);
  const now = new Date();

  beforeAll(async () => {
    jest.useFakeTimers({ now });
    jest
      .spyOn(userModel, "create")
      .mockImplementation(jest.fn().mockResolvedValue(createdUser));
    jest.spyOn(userModel, "scan").mockImplementation(
      jest.fn().mockReturnValue({
        exec: jest
          .fn()
          .mockResolvedValue([createdUser, { ...createdUser, id: "2" }]),
      })
    );

    const app = await Test.createTestingModule({
      providers: [
        AppService,
        {
          provide: "UserModel",
          useValue: userModel,
        },
      ],
    }).compile();

    service = app.get<AppService>(AppService);
  });
  afterAll(() => {
    jest.restoreAllMocks();
  });

  it("add", async () => {
    const user = await service.addData();
    // 测试mock的返回值, 这里真实的走DB
    expect(user.userId).toEqual(createdUser.userId);
    // 测试真实逻辑
    expect(userModel.create).toBeCalledWith({
      userId: `uid-${Date.now()}`,
      info: {
        name: "demoUser",
        age: 100,
      },
    });
  });

  it("list", async () => {
    const users = await service.listData();
    expect(users.length).toEqual(2);
    expect(users[0].userId).toEqual(createdUser.userId);
  });
});
```

代码解释:

- 15 行, 使用`dynamoose.model`创建一个 mock 对象, 这个对象在单元测试中会被注入到 service 中
- 19~25 行
  - 使用 useFakeTimers 冻结时间, 因为业务代码有使用时间戳的逻辑, 冻结时间后比较好写断言
  - 使用 jest 分别 mock `userModel`的`create`和`scan`方法, 每个方法都 mock 返回值, 返回值是 18 行定义的测试数据
- 30~33 行, 因为 nest 使用依赖注入, 所以这里设置一个 provider 提前将依赖准备好, 这里需要注意一下问题
  - `provide`这个属性的值属于 Nest 自身逻辑, Nest 所有的 model 都是以`Model`结尾的, 因为我们代码使用`@InjectModel('User')`, 所以这里是`User`+`Model`, 最终为: `UserModel`
  - `useValue` 这里需要定义注入的依赖, 因为之前的步骤已经定义和 mock 了`userModel`这里直接使用这个对象
- 40 行, 清除 mock
- 44, 46 行, 这里在执行的时候使用的是我们 mock 注入的对象, 所以不走真实数据库
- 48 行, 这里断言我们定义的 server 中的方法被执行了, 是测试代码逻辑的核心, 这里需要注意的`userId`和`createdUser`中的值是不一样的,因为前者走的真实业务代码逻辑, 而后者只是我们 mock 了这个方法的返回值

## 2.3 总结

要解决 Nest 单元测试的问题, 主要思路是解决依赖注入, 按照上面的例子可以使用自定`provider`的方式提供依赖, 但是需要注意依赖的名字(provide 的值); 而注入的依赖对象可以结合 jest, mock 其各个具体的方法和返回值

# 3. 参考文档

- [计时器模拟](https://jestjs.io/zh-Hans/docs/timer-mocks)
- [mock 对象函数](https://jestjs.io/zh-Hans/docs/jest-object#jestspyonobject-methodname)
- [nest provider](https://docs.nestjs.com/providers#providers)
- [nest modules:](https://docs.nestjs.com/modules): 该文档中还有一个官方例子, 代码位置在[这里](https://github.com/nestjs/nest/blob/master/sample/25-dynamic-modules/src/config/config.module.ts), 例子中使用了动态`module`这是依赖注入方案的启示性的例子
