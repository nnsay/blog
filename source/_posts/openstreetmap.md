---
title: OpenStreetMap
date: 2025-05-30 15:35:47
tags:
  - DevOps
  - 研究
excerpt: 使用OpenStreetMap数据查询路网信息
---

# 1. [OpenStreetMap](https://wiki.openstreetmap.org/wiki/Zh-hans:%E4%B8%BB%E9%A1%B5)

## 1.1 业务目标

依赖 OpenStreetMap 来获取一个区域的道路信息.

## 1.2 简介

**OpenStreetMap** 即**开放街图**简称 OSM, 它是一个由全球志愿者共同创造和维护的开源**地理数据库**. OSM 的数据结构有三种:

- nodes: 节点
- ways: 路径
- relations: 关系

每种数据结构都是由 tag 进行描述, 而 tag 则是 key-value 的键值对, 例如: highway=footway, 真实的 OSM 数据如下:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<osm version="0.6" generator="Overpass API 0.7.62.5 1bd436f1">
<note>The data included in this document is from www.openstreetmap.org. The data is made available under ODbL.</note>
<meta osm_base="2025-05-27T05:43:34Z"/>

  <node id="246574149" lat="41.8948060" lon="12.4919399">
    <tag k="amenity" v="drinking_water"/>
    <tag k="flow" v="push-button"/>
    <tag k="fountain" v="nasone"/>
    <tag k="wikimedia_commons" v="File:Drinking fountain, Via Leonina, Roma, Italia May 03, 2023 07-31-24 PM.jpeg"/>
  </node>

 </osm>
```

## 1.3 开发工具

- [编辑器](https://wiki.openstreetmap.org/wiki/Editors)

  从官方文档看, 推荐使用 [JOSM](https://wiki.openstreetmap.org/wiki/Editors) 因为它跨平台且支持的特性最全.

- [API](https://wiki.openstreetmap.org/wiki/API)

  OSM 有编辑 API 其支持从 OSM 的数据获取和写入数据

- [Overpass API](https://wiki.openstreetmap.org/wiki/Overpass_API)

  一种只读性的 API, 查询和返回一部分 OSM 地图数据, 开发者可以通过 [overpass tubo](https://overpass-turbo.eu/index.html) 来调试查询.

- [类库](https://wiki.openstreetmap.org/wiki/Software_libraries#Web_maps)

  OSM 的类库有以下类型:

  - 绑定 OSM API

    该类型的类库中有的支持 OSM API 和 Overpass API 两种, 但也有一些仅支持其中一种

  - 处理 OSM 数据

    支持诸多语言编程类库, 但是性能最好的是 C++, 其他语言的类库也多是底层绑定 C++ 版本的类库

  - 生成地图图片

  - 展示静态图片

- [Planet OSM](https://planet.openstreetmap.org/)

  完整 OpenStreetMap 数据库的定期更新副本, 有 Planet XML 和 PBF 文件格式, 在特性的许可下可后自由处理数据, 但是文件较大, XML 格式文件 151GB, PBF 格式文件 81GB.

## 1.4 技术方案

因为本文的业务目标只是获取道路信息, 没有复杂的编辑和修改的逻辑, 所以最终选择 Overpass API 作为数据获取的方式, 该方式有以下优点:

- 简单: 只需要使用一个 interpreter API 来查询数据即可
- 效率高: Overpass API 有自己的查询语言 [Overpass QL](https://wiki.openstreetmap.org/wiki/Overpass_API/Overpass_QL), 其设计更贴合 OSM 数据库所以效率更高
- 免费: Overpass API 在全球提供多个公开的 API 实例, 不需要认证, 完全免费

> [!WARNING]
> 在免费的 Public Overpass API instances 中, 有些实例是有访问限制的, 例如: [Main Overpass API instance](https://overpass-api.de/) 限制了每天访问的次数(10000)和总大小(1G), 所以本文使用没有限制实例: VK Maps Overpass API instance (Russia), 即: **https://maps.mail.ru/osm/tools/overpass/api/interpreter**

> [!NOTE]
> Overpass API Main 和 Russia 实例经过测试国内可用, 无需代理.

# 2. 路网查询

## 2.1 代码实现

使用 TypeScript 代码实现如下, 核心 `query` 方法接受一个矩形范围, 然后返回道路和节点信息:

```typescript
import axios, { AxiosInstance } from "axios";
import axiosRetry from "axios-retry";

interface OverpassAPIResponse {
  version: number;
  generator: string;
  osm3s: {
    timestamp_osm_base: string;
    copyright: string;
  };
  elements: (
    | {
        type: "node";
        id: number;
        lat: number;
        lon: number;
      }
    | {
        type: "way";
        id: number;
        nodes: number[];
        tags: Record<string, string>;
      }
  )[];
}

/**
 * @description OpenStreetMap API service.
 * @see https://wiki.openstreetmap.org/wiki/Overpass_API
 */
export class OSMService {
  private readonly client: AxiosInstance;
  constructor(
    overpassApiInstance = "https://maps.mail.ru/osm/tools/overpass/api/interpreter"
  ) {
    // Main Overpass API instance: https://overpass-api.de/api/interpreter
    // VK Maps Overpass API instance (Russia): https://maps.mail.ru/osm/tools/overpass/api/interpreter
    this.client = axios.create({
      baseURL: overpassApiInstance,
      headers: {
        "Content-Type": "application/x-www-form-urlencoded; charset=UTF-8",
      },
    });

    axiosRetry(this.client, {
      retries: 2,
      retryDelay: axiosRetry.exponentialDelay,
    });
  }

  async query(boundingBox: {
    south: number;
    west: number;
    north: number;
    east: number;
  }) {
    const { south, west, north, east } = boundingBox;
    const query = `
[out:json];
(
  way[highway](${south},${west},${north},${east});
  node(w);
);
out;
`.trim();
    console.log(`[query] ${query}`);
    try {
      const { data } = await this.client.post<OverpassAPIResponse>("", query);
      return data;
    } catch (err) {
      if ("response" in err) {
        console.log(err.response.data);
      }
      console.log("[query] error: %s", err.message);
    }
  }
}
```

> [!NOTE]
> query 方法的四个参数其实两个经纬度坐标, south 和 west 组成的经纬度是矩形的左下角点, north 和 east 组成的经纬度是矩形的右上角点坐标, 具体含义是: _southern-most latitude_, _western-most longitude_, _northern-most latitude_, _eastern-most longitude_

## 2.2 验证

选取北京-朝阳望京公园附近来测试, 因为这里有环线道路类型比较多, 具体查询代码如下:

```typescript
const exec = async () => {
  const result = await new OSMService().query({
    south: 39.98015,
    west: 116.480808,
    north: 40.010697,
    east: 116.515645,
  });
  console.log("%j", result);
};

exec().catch((err) => console.error("catch error: ", err));
```

执行以上代码可以查询对应地区的道路和节点, 如果想在 overpass-turbo 中测试, 可以使用如下的查询语句:

```q
(
  way[highway](39.98015,116.480808,40.010697,116.515645);
  node(w);
);
out meta;
```

注意, 以上查询语句最后 out 独立语句输出的的级别设置的信息深度是 meta, 这是最细的粒度, 这里与 TypeScript 代码中不通, 这里设置为 meta 是为了导出数据在 JOSM 工具中查看, 路网结果如下:
![image-20250528165746255](https://raw.githubusercontent.com/nnsay/gist/main/imgimage-20250528165746255.png)

# 3. 总结

国内外主流地图服务商（百度、高德、谷歌等）的 API 主要聚焦于导航和本地生活服务，缺乏支持指定区域路网信息查询的接口。OpenStreetMap 作为由全球志愿者协作维护的开放平台，提供免费的公共数据访问，有效填补了商业地图服务在这一领域的空白。对于有路网数据查询需求的开发者而言，OpenStreetMap 无疑是理想的解决方案。更进一步，Overpass API 提供了无需身份验证的全球公开服务节点，让开发者能够便捷高效地获取所需数据。(🤖AI 生成)
