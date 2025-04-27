---
title: 3D重建
date: 2025-03-24 11:04:23
tags:
  - DevOps
  - 研究
excerpt: 使用无人机拍摄图片进行 3D 重建
---

# 1. 3D 重建

本文主要讨论如何通过 2D 的图片进行 3D 重建, 当然这里的 2D 图片特指无人机按照航线规划飞出的图片, 一般是 JPG 格式, 这些图片的一般特征是按照一定的轨迹和角度拍摄, 图片和图片间有较多的重合.

怎么将无人机航拍的二维图片转为三维模型呢? 这是本文所探索的重点. 本文将介绍一些开源和商业工具, 然后实际对比其建模后的效果, 评估其优缺点, 最后选出一个符合自身需求的解决方案.

- 开源工具

  | 特性             | [COLMAP](https://github.com/colmap/colmap)                                          | [Meshroom](https://github.com/alicevision/Meshroom)                               | [OpenDroneMap](https://github.com/OpenDroneMap/ODM) (ODM)                       |
  | ---------------- | ----------------------------------------------------------------------------------- | --------------------------------------------------------------------------------- | ------------------------------------------------------------------------------- |
  | **精度**         | 非常高，适合复杂场景                                                                | 高，适合小范围高细节建模                                                          | 中高，适合大范围场景                                                            |
  | **纹理质量**     | 良好，但需要手动优化                                                                | 优秀，细节表现更好                                                                | 良好，但在复杂纹理上可能稍逊                                                    |
  | **地理参考支持** | 无                                                                                  | 有限                                                                              | 强，支持 GPS 和地理坐标                                                         |
  | **易用性**       | 命令行为主，需技术背景                                                              | 图形化界面，适合初学者                                                            | 命令行为主，需一定技术背景                                                      |
  | **硬件需求**     | 高（内存消耗大，GPU 加速可选）                                                      | 高（内存消耗大）                                                                  | 中高（GPU 加速可提升性能）                                                      |
  | **适用场景**     | 学术研究、复杂场景建模                                                              | 小范围高细节建模                                                                  | 大范围场景建模                                                                  |
  | **Star**         | 8.2k                                                                                | 11.6k                                                                             | 5k                                                                              |
  | **开发语言**     | C++                                                                                 | Python                                                                            | Python                                                                          |
  | **许可**         | [under the new BSD license](https://github.com/colmap/colmap?tab=License-1-ov-file) | [MPL-2.0 license](https://github.com/alicevision/Meshroom?tab=MPL-2.0-2-ov-file#) | [AGPL-3.0 license](https://github.com/OpenDroneMap/ODM?tab=AGPL-3.0-1-ov-file#) |

- 商业工具
  - [大疆智图 API(免费公测中)](https://developer.dji.com/cn/terra-api)

> [!NOTE]
>
> [大疆智图 API](https://developer.dji.com/doc/terra_api_tutorial/cn/what-is-terraapi.html) 是大疆智图桌面应用程序的 API 版本, 根据官方文档所述其: 完全继承大疆智图重建能力；在速度、精度、效果和更多功能上与 DJI 行业产品适配度更高

# 2. [OpenDroneMap](https://github.com/OpenDroneMap/ODM)

## 2.1 安装

开源工具中之所以选择 ODM 是因为未来希望可以走 SAAS 服务, 所以桌面应用程序(Meshroom)就不合适了; COLMAP 使用比较复杂, 对于三维建模技术背景不强的人使用难度大, 其配置参数较多, 对专业性要求较高, 所以也不考虑; ODM 则本身就为无人机建模创建的工具, 参数不是太多, 所以本文使用 ODM. 这里使用 ODM 的 Docker 镜像进行验证和测试:

```bash
# 测试命令
docker run -ti opendronemap/odm --version

# 查看帮助
docker run -ti opendronemap/odm --help

# 进入容器查看, 入口执行命令: ./run.sh --version
docker run -ti --entrypoint bash opendronemap/odm

# 加载数据集进入容器内部
docker run --name odm -ti --rm -v "$HOME/Downloads/tmp/odm-dataset:/datasets" --entrypoint bash opendronemap/odm
```

## 2.2 运行和验证

准备数据并运行测试:

```bash
# 查看帮助
./run.sh --help

# 查看 datasets
ls /datasets/

# 使用: 特征质量(ultra)
./run.sh --project-path /datasets --texturing-single-material --feature-quality ultra quanzhou

# 使用: 特征质量(ultra) + 特征算法(hahog)
./run.sh --project-path /datasets --texturing-single-material --feature-quality ultra --feature-type hahog quanzhou
```

## 2.3 总结 <a name=odm-summary></a>

ODM 可以较好的根据二维图像生成三维模型, 并且有质量报告产生; 但检查细节如屋顶女儿墙有扭曲和缺失, 屋顶平面的小房子的屋顶也出现扭曲呈现不规则形状.

# 3. [大疆智图 API](https://developer.dji.com/doc/terra_api_tutorial/cn/what-is-terraapi.html)

大疆智图 API 是通过**云端算力**来实现可见光重建、激光雷达重建等模型重建的一套 API 解决方案，支持企业端客户通过这套方案将大疆智图的建模能力快速集成到自有系统，形成面向测绘、电力、应急、建筑、交通、农业等垂直领域完整的应用解决方案。

| 封装大疆智图引擎                                                                  | 全自动                                     | 无缝集成                                                 | 云端算力                                         |
| --------------------------------------------------------------------------------- | ------------------------------------------ | -------------------------------------------------------- | ------------------------------------------------ |
| 完全继承大疆智图重建能力；在速度、精度、效果和更多功能上与 DJI 行业产品适配度更高 | 自定义工作流，实现全自动作业，效率全面提升 | 可无缝接入业务平台，赋能重建能力，满足更多二三维数据应用 | 多段协同，云端算力，弹性扩容，无惧高并发、大数据 |

API 引用详情参考: [大疆智图 API 云端服务](https://developer.dji.com/doc/terra_api_tutorial/cn/api/terra/)

## 3.1 API 签名和调用

本文这里已经将 大疆智图(Terra) API 封装成了 SDK, 开发者可以直接通过 SDK 请求 API 而不必关心 API 签名等复杂性操作.

- 源码: https://github.com/nnsay/dji-terra-api-sdk
- NPM: https://www.npmjs.com/package/@nnsay/dji-terra-api-sdk

如果希望了解完整 API 的使用可以参考:

- [大疆智图 API-快速入门](https://developer.dji.com/doc/terra_api_tutorial/cn/quick-start.html)
- [大疆智图 API 云端服务](https://developer.dji.com/doc/terra_api_tutorial/cn/api/terra/)

## 3.2 3D 重建

调用和验证也是通过 Typescript 脚本来做, 首先准备好无人机航拍图片, 然后按照[官方文档的流程](https://developer.dji.com/doc/terra_api_tutorial/cn/quick-start.html)进行业务开发即可, 大致流程有: 获取上传文件的 STS Token, 上传文件, 创建资源, 给资源绑定文件, 创建任务, 启动任务, 检查任务执行状态, 下载三维模型文件. 详细的实现流程如下:

```typescript
import {
  CreateJobAPIResponse,
  StartJob3DParamater,
  TerraAPI,
} from "@nnsay/dji-terra-api-sdk";
import os from "os";

const exec = async () => {
  // AK/SK come from env prameter
  const terraAPI = new TerraAPI();
  // or AK/SK hardcode
  // const terraAPI = new TerraAPI(appKey, secretKey, 'https://openapi-cn.dji.com');

  // TODO: change the dir to you drone image dir
  const imageDir = `${os.homedir()}/Downloads/tmp/test/images`;
  const apiOutputDir = `${os.homedir()}/Downloads/tmp/test/terra-api-result`;

  // 1. Get STS Token for upload files
  const stsToken = await terraAPI.obtainToken();
  // 2. Upload drone images to tmp
  const uploadedFiles = await terraAPI.uploadFile(stsToken, imageDir);
  // 3. Create resource
  const resource = await terraAPI.createResource({
    name: "test-obj",
    type: "map",
  });
  // 4. Bind the uploaded files and resource
  await terraAPI.uploadCallback(
    stsToken.callbackParam,
    uploadedFiles,
    resource.uuid
  );
  // 5. Create job
  const job = await terraAPI.createJob({ name: `job-test-obj` });
  // 6. Start job with custom parameters
  await terraAPI.startJob<StartJob3DParamater>(job.uuid, {
    parameters: {
      parameter: {
        // Texture model result parameter
        generate_ply: false,
        generate_b3dm: false,
        generate_obj: true,
      },
    },
    resourceUuid: resource.uuid,
    type: 15,
  });
  // 7. Check job status
  let checkJob: CreateJobAPIResponse;
  const sleep = (seconds = 10) =>
    new Promise((resolve) => setTimeout(() => resolve(true), seconds * 1000));
  do {
    checkJob = await terraAPI.getJob(job.uuid);
    await sleep();
  } while (checkJob.status < 6);
  if (checkJob.status == 7) {
    throw new Error("terra api job execution fail");
  }
  // 8. Downlaod 3D output
  await terraAPI.downloadFiles(checkJob.outputResourceUuid, apiOutputDir);
};

exec().catch((err) => {
  console.debug(err);
  console.error(err.response?.data);
});
```

> [!WARNING] > **createJob** 接口有并发限制, 如果同时创建 > 2 个未完成的 Job 就会报错:
>
> ```json
> { "result": { "code": 801, "msg": "too many requests: jobs over limit" } }
> ```

## 3.3 总结 <a name=terra-api-summary></a>

### 3.3.1 执行效率

- Case1: 上传 371 个无人机文件, 总计 3.3G, 生产 **obj** 和 **b3dm** 模型

  耗时: 34 分钟

- Case2: 上传 1829 个无人机文件, 总计 19G , 仅生产 **obj** 模型

  耗时: 312 分钟

### 3.3.2 三维模型质量

在 Case1 所用的无人机文件也用 ODM 生成一版 obj, 然后在用大疆智图桌面应用程序生再成一版 obj, 加上大疆智图 API 生产的这版 obj 就有三个 obj 文件, 然后使用模型渲染软件对比查看这三个 obj, 结论如下:

- 大疆智图桌面应用程序和 API 产生的 obj 效果差异不大, 纹理清晰, 扭曲较少
- ODM obj 效果较差, 特别是屋顶女儿墙扭曲变形, 屋顶小房子的屋顶轮廓变形严重

## 3.4 限制

虽然大疆智图 API 在 3.3.1 的测试中可以支持 1892 张总计 19G 的图片, 但是也不是无限制, 详细[限制](https://developer.dji.com/doc/terra_api_tutorial/cn/faq.html)如下:

- 2D/3D：要求十张以上影像，重叠率 70% 以上，3D 作业单次最多不超过 20000 张，2D 作业单次最多不超过 4000 张
- LiDAR：L1 或 L2 采集的超过 2 分钟的数据，数据文件夹里包含有 IMU、RTK、RTB、RTL、RTS、LDR、CLC、CLI 等文件，如需成果带颜色信息，还需要附带采集时图片，总大小不能超过 48 G

# 4. 技术总结

结合本文 [2.3](#odm-summary) 和 [3.3.2](#terra-api-summary) 的两个调研可知: 如果三维模型的用途较高精度的测绘, 比如根据三维结果在平面安装光伏板, 需要精确的计算那么还是选择商业产品大疆智图 API, 但是如果仅是三维展示或者做三维地图, 对于各位细节纹理呈现效果不高或者不需要那么保真, 那么 OMD 是一个不错的选择.

目前大疆智智图 API 正在公测可以申请免费试用, 虽然其并发重建任务只有 2, 但毕竟有一年的免费试用, 对于 3D 重建任务不多的场景非常适合.
