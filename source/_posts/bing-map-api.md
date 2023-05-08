---
title: 必应地图API调研
date: 2023-05-08 14:33:35
tags:
  - Code
  - 研究
  - Tips
excerpt: 使用必应地图API替换谷歌地图API
---

# 1. 背景介绍

Google Map 的 API 在国内不可用, 社区里面有一些通过反向代理的方式使用 Google Map API 的方式也不稳定, 经常遇到的问题就是`ECONNREFUSED`. 为了提供稳定的服务本文试图调研 Bing Map API 来替换 Google Map API 的可行性, 结合业务背景, 主要调研替换两个接口:

- [海拔数据](https://developers.google.com/maps/documentation/elevation/overview?hl=zh-cn): Elevation API, 可返回地球上某个位置的海拔数据，或沿路径的抽样海拔数据
- [时区](https://maps.googleapis.com/maps/api/timezone/json): Time Zone API 服务可接受纬度/经度坐标的 HTTP 请求以及所需的日期和时间。它返回该位置的时区数据，包括世界协调时间 (UTC) 的偏移量和夏令时

# 2. Bing Map 开发者

与其他服务一样, 使用 Bing Map 需要注册开发者账号并创建一个 Key, 使用 API 时需要提供 Key. 具体获取 Key 的步骤如下:

- 在[开发者中心](https://www.bingmapsportal.com/)注册账号
- 在`My Account`中选择`My keys`: 创建账号

完整的帮助文档请查看:[Getting a Bing Maps Key](https://learn.microsoft.com/en-us/bingmaps/getting-started/bing-maps-dev-center-help/getting-a-bing-maps-key)

# 3. 使用和对比

## 3.1 代码

使用 nodejs 的 axios 客户端发送请求, 详细代码如下:

```typescript
import axios from "axios";

const BingMapsKey = "使用个人开发者中心生成的Key";

// https://learn.microsoft.com/en-us/bingmaps/rest-services/timezone/find-time-zone
const getTimezone = async (point: string, datetime_utc: string) => {
  const url = `https://dev.virtualearth.net/REST/v1/TimeZone/${point}?datetime=${datetime_utc}&key=${BingMapsKey}`;
  const res = await axios.get(url, {
    headers: {
      "Content-Type": "application/json",
    },
  });
  const { resourceSets, statusCode, errorDetails } = res.data;
  if (errorDetails) {
    console.error("%j", errorDetails);
  }
  if (
    statusCode === 200 &&
    resourceSets[0]?.resources[0]?.timeZone?.utcOffset
  ) {
    console.log(statusCode);
    console.log("%j", resourceSets);
    const utcOffsets =
      resourceSets[0].resources[0].timeZone.utcOffset.split(":");
    const hours =
      +utcOffsets[0] > 0
        ? +utcOffsets[0] + utcOffsets[1] / 60
        : +utcOffsets[0] + (-1 * utcOffsets[1]) / 60;
    console.log(hours);
  } else {
    // 没有报错, 但是无数据, 提供坐标和UTC时间不匹配时会出现
    console.warn("%j", resourceSets);
  }
};

// https://learn.microsoft.com/en-us/bingmaps/rest-services/elevations/get-elevations
const getElevation = async (point: string) => {
  const url = `http://dev.virtualearth.net/REST/v1/Elevation/List?points=${point}&key=${BingMapsKey}`;
  const res = await axios.get(url, {
    headers: {
      "Content-Type": "application/json",
    },
  });
  const { resourceSets, statusCode, errorDetails } = res.data;
  if (errorDetails) {
    console.error("%j", errorDetails);
  }
  if (statusCode === 200 && resourceSets[0]?.resources[0]?.elevations?.length) {
    console.log(statusCode);
    console.log("%j", resourceSets);
    console.log(resourceSets[0].resources[0].elevations[0]);
  } else {
    // 没有报错, 但是无数据
    console.warn("%j", resourceSets);
  }
};

(async () => {
  // 北京故宫: 8
  // await getTimezone('39.918953,116.397357', new Date().toUTCString())
  // 新疆乌鲁木齐: 8
  // await getTimezone('43.844556,87.69757', new Date().toUTCString())
  // Google: -28800(秒), Bing: -8(小时)
  await getTimezone(
    "39.6034810,-119.6822510",
    new Date(1331766000000).toUTCString()
  );

  // 北京故宫: 52
  // await getElevation('39.918953,116.397357')
  // 新疆乌鲁木齐: 882
  // await getElevation('43.844556,87.69757')
  // Google: 1608.637939453125, Bing: 1609
  await getElevation("39.7391536,-104.9847034");
})()
  .then(() => {
    console.log("all done");
  })
  .catch((err) => {
    // console.error(err)
    console.log(err.response.data);
  });
```

## 3.2 测试说明

- 海拔数据 Google Map API 取其[返回值](https://developers.google.com/maps/documentation/elevation/start?hl=zh-cn#sample-request)中的`elevation`字段, 而 Bing Map API 取`elevations`数组中的值, 单位都是米
- 时区数据 Google Map API 取其[返回值](https://developers.google.com/maps/documentation/timezone/requests-timezone?hl=zh-cn#TimeZoneResponse)中的`rawOffset`字端,单位是秒, 而 Bing Map API 取`utcOffset`, 是`(+/-)hh:mm`格式, 上面的代码中将其换算成小时进行比较
- 代码中使用 Google Map API 中文档中的例子中的数据请求 Bing Map API 并将结果和 Google Map 文档中的案例值进行对比
  - [Google Map API 时区案例](https://developers.google.com/maps/documentation/timezone/requests-timezone?hl=zh-cn#TimeZoneResponse)
  - [Google Map API 海拔案例](https://developers.google.com/maps/documentation/elevation/start?hl=zh-cn#sample-request)

## 3.3 测试结论

​ **数据有效性上如果 Google Map 的结果四舍五入和 Bing Map 结果相差不大, 可以完全替换.**
