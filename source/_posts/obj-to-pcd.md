---
title: OBJ 转为 PCD
date: 2025-05-14 16:31:58
tags:
  - DevOps
  - 研究
excerpt: 从 3D 模型到 PCD 文件的转换与优化
---

> [!NOTE]
> 本文包含 AI 生成的内容。

# 1. 点云文件简介

本文主要讨论如何通过 OBJ 文件生成点云（PCD）文件。关于 OBJ 文件的生成，我已经在之前的[3D 重建](https://nnsay.cn/2025/03/24/3d-reconstruction/)博客中详细介绍过，因此本文将专注于 PCD 文件的生成与处理。

PCD（Point Cloud Data）是一种专门用于存储三维点云数据的文件格式，在多个领域有着广泛应用：

- **三维建模与重建**：用于记录和重构真实物体的形状
- **机器人视觉与导航**：帮助机器人感知周围环境并进行路径规划
- **计算机视觉与图像处理**：提供空间数据支持，用于物体识别和场景理解

作为空间计算领域的重要文件格式，理解和处理 PCD 文件是 3D 开发的基础技能。本文将涵盖以下内容：

- OBJ 到 PCD 的转换方法
- 点云简化与优化处理
- 结果可视化与验证

# 2. 技术实现

经过调研，我选择使用[Open3D](https://www.open3d.org/)这个 Python 库来实现点云生成和简化。Open3D 是一个开源的库，专为 3D 数据处理而设计，提供了丰富的功能和优秀的性能。以下是具体实现代码：

```python
import sys
import time

import numpy as np
import open3d as o3d


class Obj2Pcd:
    def obj_to_pcd(self, obj_path: str, pcd_path: str, min_distance=0.2):
        """
        将OBJ文件转换为PCD并进行点云简化

        参数:
            obj_path: OBJ文件路径
            pcd_path: 输出PCD文件路径
            min_distance: 点之间的最小距离（米）
                        如果设置为0或负数，则只进行简单转换，不控制点距离
        """

        start_time = time.perf_counter()
        print(f'[obj_to_pcd] 正在加载OBJ文件: {obj_path}')
        mesh = o3d.io.read_triangle_mesh(obj_path)
        print(f'[obj_to_pcd] 网格信息: {len(mesh.vertices)} 顶点, {len(mesh.triangles)} 三角形')

        # 计算网格的总表面积
        triangles = np.asarray(mesh.triangles)
        vertices = np.asarray(mesh.vertices)
        v0 = vertices[triangles[:, 0]]
        v1 = vertices[triangles[:, 1]]
        v2 = vertices[triangles[:, 2]]
        triangle_areas = 0.5 * np.linalg.norm(np.cross(v1 - v0, v2 - v0), axis=1)
        total_surface_area = np.sum(triangle_areas)
        print(f'[obj_to_pcd] 模型总表面积: {total_surface_area:.4f} 平方米')

        # 根据表面积和最小距离要求计算采样点数
        approx_point_count = int(total_surface_area / (min_distance**2))
        print(f'[obj_to_pcd] 估计采样点数: {approx_point_count}')

        # 均匀采样生成点云
        # https://www.open3d.org/docs/release/python_api/open3d.geometry.TriangleMesh.html#open3d.geometry.TriangleMesh.sample_points_uniformly
        pcd = mesh.sample_points_uniformly(number_of_points=approx_point_count)

        # 保存为PCD文件
        o3d.io.write_point_cloud(pcd_path, pcd)
        elapsed = (time.perf_counter() - start_time) / 60
        print(f'[obj_to_pcd] 转换完成，耗时: {elapsed:.4f} 分钟')
        return pcd

# 使用示例
if __name__ == '__main__':
    # 获取命令行参数中 "--" 后的部分
    args = sys.argv
    if '--' in args:
        idx = args.index('--')
        script_args = args[idx + 1:]
    else:
        script_args = []

    if len(script_args) != 2:
        print('❌ 用法错误：请提供 obj_path 和 pcd_path 参数')
        print('例如: python script.py -- /path/to/input.obj /path/to/output.pcd')
        sys.exit(1)

    obj_path, pcd_path = script_args
    task = Obj2Pcd()
    pcd = task.obj_to_pcd(
        obj_path,
        pcd_path,
    )

    # 如需可视化结果，取消以下注释
    # o3d.visualization.draw_geometries([pcd])
```

# 3. 核心技术解析

代码中涉及到的 Open3D 关键方法有：

1. **数据转换**：使用`open3d.utility.Vector3dVector`将 NumPy 数组转换为 Open3D 可处理的向量格式（代码中未直接使用，但在`mesh.sample_points_uniformly`内部会调用）

2. **点云采样**：
   - `mesh.sample_points_uniformly()`：均匀采样，在整个网格表面生成均匀分布的点
   - 另一种选择是`mesh.sample_points_poisson_disk()`：基于泊松盘采样，可以生成更加均匀且避免密集聚集的点云分布

这两种采样方法的选择取决于你的具体需求：

- **均匀采样**适合大多数一般应用场景，计算速度较快
- **泊松盘采样**在保持点之间最小距离方面表现更好，但计算开销更大，适合对质量要求较高的场景

# 4. 点云文件可视化

生成的 PCD 文件可以通过多种工具进行查看和进一步处理。我推荐以下几个工具：

- **[CloudCompare](https://www.danielgm.net/cc/)**：免费开源的跨平台点云编辑软件，功能强大且易于使用
- **[MeshLab](https://www.meshlab.net/)**：同样支持点云查看和处理的开源工具
- **Open3D 自带的可视化功能**：通过`o3d.visualization.draw_geometries([pcd])`可直接在 Python 环境中查看结果

# 5. 总结

[Open3D](https://www.open3d.org/)提供了一套完善的解决方案，能够高效地实现 OBJ 到 PCD 的转换及点云优化。本文介绍的方法适用于多种 3D 数据处理场景，特别是在机器人视觉、自动驾驶和增强现实等领域有着广泛应用。

对于有 3D 数据处理需求的开发者，Open3D 是一个值得深入学习的工具库，它不仅提供了本文展示的基础功能，还包含了点云配准、重建、分割等高级功能，能够满足大多数点云处理的需求。
