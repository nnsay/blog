---
title: 3D重建--OBJ处理
date: 2025-04-27 10:19:13
tags:
  - DevOps
  - 研究
excerpt: 学习使用 Blender 脚本自动处理 OBJ 模型文件，包括简化、烘焙和导出等操作
---

# 1. 使用 blender 处理 obj 文件

前一篇文章讲述了如何使用无人机拍摄的图片做 [3D 重建](https://nnsay.cn/2025/03/24/3d-reconstruction/), 重建的结果是 OBJ+MTL+JPG 文件, 本文则重要聚焦如何自动化的处理 OBJ 文件及其材质.

[blender](https://www.blender.org/)是一个开源的 3D 处理软件, 其还支持[python 脚本](https://docs.blender.org/api/current/index.html), 利用这个工具可以针对 [obj](https://en.wikipedia.org/wiki/Wavefront_.obj_file) 做一些业务处理, 以下是本文的重点:

- 如何简化 obj, 缩小文件大小, 方便传输和加载
- 如何烘焙贴图, 适应简化 obj 后的模型
- 如何压缩贴图质量, 缩小贴图文件大小, 方便传输和加载

因为 blender 有自己 python 包 bpy, 如果业务需求可以通过 bpy 处理则可以实现自动化. 一般来说即使使用 bpy 也需要安装 blender 软件, 但是本文后面将介绍 [dokcer-blender](https://github.com/linuxserver/docker-blender) 这个工具, 从而实现在容器中使用 bpy.

## 1.1 安装 blender

从官方[下载](https://www.blender.org/download/)页面选择合适的版本安装即可. 本文使用 **macOS** **Apple Silicon** 版本. 安装完毕后执行以下命令验证安装:

```bash
# 设置别名
alias blender="/Applications/Blender.app/Contents/MacOS/Blender"

# 查看版本
blender --version
```

## 1.2 简化与烘焙

obj 简化的目的是缩小大小, 缩小一般的算法是减少面和点的个数, 这样的话 UV 就会发生改变, 为了让贴图更保真, 所以需要做 UV 烘焙; 考虑到烘焙的贴图也可能很大, 所以贴图也需要拆分为多张. 代码(run-bpy.py)具体实现如下, 涵盖以下功能:

- 支持 1 个或者多个 obj 文件简化
- 单文件简化支持配置面数阈值, 小于此阈值则跳过
- 多文件简化会根据总面数按比例对于每个 obj 文件进行简化
- 烘焙贴图
- 支持自定义数量的烘焙贴图

```python
from typing import Dict
import bpy
import bmesh
import mathutils
import numpy as np
import os
import shutil
import sys
import time


class SimplifyObj:
    def __init__(self, input_dir: str, output_dir: str):
        self.input_dir = input_dir
        self.output_dir = output_dir

    def clear_scene(self):
        bpy.ops.wm.read_factory_settings(use_empty=True)

    def ensure_dir(self, path: str):
        if not os.path.exists(path):
            os.makedirs(path)

    def simplify_and_bake(
        self,
        obj_file: str,
        decimate_ratio: float,
        tile_num=5,
        is_single_obj=True,
        rotation_matrix=None,
    ):
        if rotation_matrix is not None and (
            not isinstance(rotation_matrix, list)
            or len(rotation_matrix) != 4
            or not all(
                isinstance(row, list) and len(row) == 4 for row in rotation_matrix
            )
        ):
            raise ValueError("旋转矩阵必须是4x4的嵌套列表")
        start_time = time.perf_counter()
        # 准备路径
        folder_path = os.path.dirname(obj_file)
        rel_path = os.path.relpath(folder_path, self.input_dir)
        base_name = os.path.splitext(os.path.basename(obj_file))[0]
        output_subdir = os.path.join(self.output_dir, rel_path)

        # -----------------------------------------
        # Step 1: Clean everything from the scene
        # -----------------------------------------
        self.ensure_dir(output_subdir)
        self.clear_scene()

        # -----------------------------------------
        # Step 2: Load the OBJ file and rename it to 'HighPoly'
        # ---------------------------------------------
        bpy.ops.wm.obj_import(filepath=obj_file)
        high_poly = bpy.context.selected_objects[0]
        high_poly.name = "HighPoly"

        # === 如果提供了旋转矩阵，则应用旋转 ===
        if rotation_matrix is not None:
            matrix = mathutils.Matrix(rotation_matrix)
            high_poly.matrix_world = matrix @ high_poly.matrix_world
        # === 如果无需简化则直接拷贝 ===
        if is_single_obj and decimate_ratio >= 1.0:
            for item in os.listdir(folder_path):
                s = os.path.join(folder_path, item)
                d = os.path.join(output_subdir, item)
                if os.path.isfile(s):
                    shutil.copy2(s, d)
            print(f"文件复制完成: {folder_path} -> {output_subdir}")
            return

        print(
            f"简化前({high_poly.name}): 顶点数: {len(high_poly.data.vertices)}, 面数: {len(high_poly.data.polygons)}"
        )

        ## -----------------------------------------
        # Step 3: Duplicate HighPoly and rename the duplicate to 'LowPoly'
        # -----------------------------------------
        low_poly = high_poly.copy()
        low_poly.data = high_poly.data.copy()
        low_poly.name = "LowPoly"
        bpy.context.collection.objects.link(low_poly)

        # -----------------------------------------
        # Step 4: Remove texture, material, and UV map from LowPoly
        # -----------------------------------------
        # Remove materials
        low_poly.data.materials.clear()
        # Remove all UV maps
        if low_poly.data.uv_layers:
            for uv_layer in list(low_poly.data.uv_layers):
                low_poly.data.uv_layers.remove(uv_layer)

        # -----------------------------------------
        # Step 5: Decimate LowPoly and apply the modifier
        # -----------------------------------------
        bpy.context.view_layer.objects.active = low_poly
        decimate_mod = low_poly.modifiers.new(name="Decimate", type="DECIMATE")
        decimate_mod.ratio = decimate_ratio
        bpy.ops.object.modifier_apply(modifier=decimate_mod.name)
        print(
            f"简化后({low_poly.name}): 顶点数: {len(low_poly.data.vertices)}, 面数: {len(low_poly.data.polygons)}"
        )

        # -----------------------------------------
        # Step 6: Split LowPoly into tiles using robust face clustering
        # -----------------------------------------
        # Compute clustering on faces of LowPoly

        # Build a bmesh from LowPoly's mesh to compute centroids for all faces.
        bm = bmesh.new()
        bm.from_mesh(low_poly.data)
        face_centroids = [face.calc_center_median()[:] for face in bm.faces]
        bm.free()

        # Convert the list of centroids to a NumPy array
        centroids_np = np.array(face_centroids)

        # Initialize K-means clustering
        np.random.seed(0)  # For reproducibility
        indices = np.random.choice(len(centroids_np), tile_num, replace=False)
        centers = centroids_np[indices]

        # Run K-means clustering iteratively until convergence
        while True:
            # Compute Euclidean distances from each face centroid to each cluster center
            distances = np.linalg.norm(
                centroids_np[:, None, :] - centers[None, :, :], axis=2
            )
            labels = np.argmin(distances, axis=1)
            # Recompute centers as the mean of all centroids assigned to each cluster
            new_centers = np.array(
                [
                    centroids_np[labels == i].mean(axis=0)
                    if np.any(labels == i)
                    else centers[i]
                    for i in range(tile_num)
                ]
            )
            if np.allclose(centers, new_centers, atol=1e-6):
                break
            centers = new_centers

        tiles = []
        # For each cluster (tile), duplicate LowPoly and delete faces that do not belong to that cluster.
        for i in range(tile_num):
            bpy.ops.object.select_all(action="DESELECT")
            low_poly.select_set(True)
            bpy.context.view_layer.objects.active = low_poly
            bpy.ops.object.duplicate()
            tile_obj = bpy.context.active_object
            tile_obj.name = f"{base_name}_{i + 1}"

            bpy.ops.object.mode_set(mode="EDIT")
            bm_tile = bmesh.from_edit_mesh(tile_obj.data)

            # For each face, delete it if its cluster label is not equal to the current tile index.
            for idx, face in enumerate(bm_tile.faces):
                if labels[idx] != i:
                    face.select = True
                else:
                    face.select = False

            bmesh.update_edit_mesh(tile_obj.data)
            bpy.ops.mesh.delete(type="FACE")
            bpy.ops.object.mode_set(mode="OBJECT")
            tiles.append(tile_obj)

        # -----------------------------------------
        # Step 7: For each tile, unwrap using Smart UV Project
        # -----------------------------------------
        for tile in tiles:
            bpy.ops.object.select_all(action="DESELECT")
            bpy.context.view_layer.objects.active = tile
            tile.select_set(True)
            bpy.ops.object.mode_set(mode="EDIT")
            bpy.ops.mesh.select_all(action="SELECT")
            bpy.ops.uv.smart_project()
            bpy.ops.uv.select_all(action="SELECT")
            bpy.ops.uv.pack_islands()
            bpy.ops.uv.select_all(action="DESELECT")
            bpy.ops.object.mode_set(mode="OBJECT")
            tile.select_set(False)

        # -----------------------------------------
        # Step 8: For each tile, create a 4096 x 4096 empty image texture and assign a new material
        # -----------------------------------------
        for tile in tiles:
            # Create new image
            image = bpy.data.images.new(name=f"{tile.name}", width=4096, height=4096)

            # Create new material with nodes
            mat = bpy.data.materials.new(name=f"{tile.name}")
            mat.use_nodes = True
            nodes = mat.node_tree.nodes
            links = mat.node_tree.links

            # Remove default nodes except output
            for node in list(nodes):
                if node.type != "OUTPUT_MATERIAL":
                    nodes.remove(node)

            # Create new nodes: Principled BSDF and Image Texture
            bsdf = nodes.new(type="ShaderNodeBsdfPrincipled")
            tex_image = nodes.new(type="ShaderNodeTexImage")
            tex_image.image = image
            output = nodes.get("Material Output")

            # Link nodes
            links.new(tex_image.outputs["Color"], bsdf.inputs["Base Color"])
            links.new(bsdf.outputs["BSDF"], output.inputs["Surface"])

            # Assign the material to the tile
            if len(tile.data.materials) > 0:
                tile.data.materials[0] = mat
            else:
                tile.data.materials.append(mat)

        # -----------------------------------------
        # Step 9: Bake the diffuse color from HighPoly onto each tile's image texture
        # -----------------------------------------
        bpy.context.scene.render.engine = "CYCLES"
        bpy.context.scene.cycles.device = "CPU"
        bpy.context.scene.cycles.use_denoising = False
        bpy.context.scene.cycles.bake_type = "DIFFUSE"
        bpy.context.scene.render.bake.use_pass_color = True
        bpy.context.scene.render.bake.use_pass_direct = False
        bpy.context.scene.render.bake.use_pass_indirect = False
        bpy.context.scene.render.bake.use_selected_to_active = True
        bpy.context.scene.render.bake.cage_extrusion = 0.1
        bpy.context.scene.render.bake.use_clear = True

        for tile in tiles:
            # Hide all objects except the current tile and HighPoly
            for obj in bpy.data.objects:
                if obj != tile and obj != high_poly:
                    obj.hide_viewport = True
                    obj.hide_render = True

            bpy.ops.object.select_all(action="DESELECT")
            tile.select_set(True)
            high_poly.select_set(True)
            bpy.context.view_layer.objects.active = tile

            # Ensure the image texture node is active in the tile material
            mat = tile.data.materials[0]
            nodes = mat.node_tree.nodes
            tex_nodes = [n for n in nodes if n.type == "TEX_IMAGE"]
            if tex_nodes:
                mat.node_tree.nodes.active = tex_nodes[0]

            # Perform the bake
            bpy.ops.object.bake(type="DIFFUSE")

            # Save the baked image
            baked_image = tex_nodes[0].image
            baked_image.filepath_raw = f"{output_dir}/{baked_image.name}.jpg"
            baked_image.file_format = "JPEG"
            baked_image.save(quality=30)

            # Unhide all objects after baking
            for obj in bpy.data.objects:
                obj.hide_viewport = False
                obj.hide_render = False

        print("3D reconstruction baking complete.")

        # -----------------------------------------
        # Step 10: Combine all tile objects into one mesh and export as one OBJ and MTL
        # -----------------------------------------

        # Deselect all objects
        bpy.ops.object.select_all(action="DESELECT")

        # Select all tile objects
        for tile in tiles:
            tile.select_set(True)

        # Set the active object to the first tile
        bpy.context.view_layer.objects.active = tiles[0]

        # Join all selected tile objects into one
        bpy.ops.object.join()

        combined_obj = bpy.context.active_object
        combined_obj.name = base_name

        # Define the export file path (this will generate both .obj and .mtl files)
        export_path = os.path.join(output_subdir, f"{base_name}.obj")
        # Export the combined object as OBJ with materials
        bpy.ops.wm.obj_export(
            filepath=export_path,
            export_selected_objects=True,
            export_materials=True,
            path_mode="COPY",
            export_uv=True,
            export_normals=False,
        )
        # 计算耗时
        elapsed = (time.perf_counter() - start_time) / 60
        print(f"简化和烘焙完成: {export_path}, 耗时: {elapsed:.4f}分钟")

    def batch_process(self):
        # === 遍历所有子目录，查找含 .obj 的模型包 ===
        obj_files: Dict[str, int] = {}
        for root, _, files in os.walk(self.input_dir):
            for file in files:
                if file.endswith(".obj"):
                    obj_file = os.path.join(root, file)
                    obj_dir = os.path.dirname(obj_file)
                    rel_path = os.path.relpath(obj_dir, self.input_dir)
                    bpy.ops.wm.read_factory_settings(use_empty=True)
                    bpy.ops.wm.obj_import(filepath=obj_file)
                    obj = bpy.context.selected_objects[0]
                    obj_files[obj_file] = len(obj.data.polygons)
                    bpy.ops.wm.read_factory_settings(use_empty=True)
                    break  # 防止重复处理同一文件夹

        total_files = len(obj_files)
        total_face_count = sum(obj_files.values())

        # 确定每个OBJ文件的简化目标面数
        individual_max_face_count = int(
            os.getenv("MULTI_OBJ_INDIVIDUAL_MAX_FACE_COUNT", 500000)
        )  # 单个OBJ文件最大面数
        all_obj_max_face_count = int(
            os.getenv("MULTI_OBJ_ALL_OBJ_MAX_FACE_COUNT", 2500000)
        )  # 所有OBJ文件总面数上限
        single_obj_skip_face_num = int(
            os.getenv("SINGLE_OBJ_SKIP_SIMPLIFY_FACE_NUM", 1000000)
        )
        obj_face_simplify_target: Dict[str, int] = {}
        # 单文件和多文件场景统一处理
        if total_files == 1:
            # 单文件情况简单处理
            single_obj_skip_face_num = int(
                os.getenv("SINGLE_OBJ_SKIP_SIMPLIFY_FACE_NUM", 1000000)
            )
            print(
                f"总面数 {total_face_count}，按照阈值 {single_obj_skip_face_num} 裁剪"
            )
            obj_face_simplify_target[obj_file] = min(
                total_face_count,
                single_obj_skip_face_num,
            )
        elif total_face_count > all_obj_max_face_count:
            # 总面数超过限制，按比例分配
            print(
                f"总面数 {total_face_count} 超过限制 {all_obj_max_face_count}，按比例裁剪"
            )
            for obj_file, face_count in obj_files.items():
                if face_count > 0:  # 跳过面数为0的文件
                    obj_face_simplify_target[obj_file] = min(
                        int(face_count / total_face_count * all_obj_max_face_count),
                        individual_max_face_count,
                    )
        else:
            # 总面数在限制内，仅限制单个文件面数
            print(
                f"总面数 {total_face_count} 在限制 {all_obj_max_face_count} 内，仅限制单个文件面数"
            )
            for obj_file, face_count in obj_files.items():
                obj_face_simplify_target[obj_file] = min(
                    face_count, individual_max_face_count
                )

        # 简化
        for obj_file, face_count in obj_files.items():
            rotation_matrix = [
                [0, 0, 1, 0],  # 将X轴映射到Z轴
                [0, 1, 0, 0],  # Y轴保持不变
                [-1, 0, 0, 0],  # 将Z轴映射到-X轴
                [0, 0, 0, 1],  # 齐次坐标
            ]
            rotation_matrix = None
            self.simplify_and_bake(
                obj_file=obj_file,
                decimate_ratio=obj_face_simplify_target[obj_file]/total_face_count,
                is_single_obj=total_files == 1,
                rotation_matrix=rotation_matrix,
            )

        print("✅ 所有模型处理完毕。")


if __name__ == "__main__":
    # 获取命令行参数中 "--" 后的部分
    args = sys.argv
    if "--" in args:
        idx = args.index("--")
        script_args = args[idx + 1 :]
    else:
        script_args = []

    if len(script_args) != 2:
        print("❌ 用法错误：请提供 input_dir 和 output_dir 参数")
        print(
            "例如: blender --background --python script.py -- /path/to/input /path/to/output"
        )
        sys.exit(1)

    input_dir, output_dir = script_args
    task = SimplifyObj(input_dir, output_dir)
    task.batch_process()

```

执行脚本:

```bash
blender --background --python bpy.py -- /config/guoxuan-YXBX /config/simplified
```

> [!NOTE]
>
> **设置 Cycles 渲染器和烘焙参数**是根据容器处理结果调整而来的, 如果脚本不在容器中跑, 可以使用 GPU 和降噪.

# 2. 容器化

容器话的目的是将处理逻辑封装为弹性的服务或者任务, 方便业务上按需执行. 根据调研 [linuxserver/blender](https://github.com/linuxserver/docker-blender/) 是一个较新的基础镜像, 以此来验证 Blender 容器化脚本运行的可行性.

## 2.1 创建容器

```bash
docker run --gpus all --name blender -it -d linuxserver/blender bash
docker logs -f blender
```

## 2.2 复制测试脚本

```bash
docker cp ~/Downloads/tmp/odm-dataset/orginobjs/guoxuan-YXBX blender:/config/
# 复制之前修改 INPUT_DIR 和 OUTPUT_DIR
docker cp bpy.py blender:/config/
```

## 2.3 执行

首先进入容器:

```bash
docker exec -it blender bash
```

检查命令:

```bash
blender --version
```

安装脚本依赖:

```bash
apt-get update && \
    apt-get install -y --no-install-recommends  && \
    add-apt-repository -y ppa:deadsnakes/ppa && \
    apt-get update && \
    apt-get install -y python3.11 python3.11-venv python3-pip && \
    rm -rf /var/lib/apt/lists/*
python3.12 -m pip install -r requirements.txt --target="/usr/lib/python3.12/dist-packages"
```

执行脚本:

```bash
blender --background --python run-bpy.py -- /app/guoxuan-YXBX /app/simplified
```

查看结果:

```bash
# 拷贝容器的结果到宿主机方便对比查看
docker cp blender:/config/simplified ~/Downloads/tmp/odm-dataset/simplified2
```

调试技巧:

因为业务脚本中通常不是只包含 bpy 这一个包, 如果包含其他包就要考虑怎么安装, 以下是一些调试 blender python 的有用脚本

```bash
blender --background --python-expr "import site; print(site.getsitepackages())"
blender --background --python-expr "import sys; print(f'Blender Python 版本: {sys.version_info.major}.{sys.version_info.minor}.{sys.version_info.micro}')"
blender --background --python-expr "import sys; print(sys.executable)"
blender --background --python-console
```

# 3. 结论

使用同样的脚本和 OBJ 源文件(50.3MB), 分别在容器和宿主机运行, 对比和查看结果:

- 执行时间

  宿主机上运行更快(6 分钟), 容器较慢(12 分钟)

- 结果质量

  容器结果在个别细节上略有瑕疵, 但两者渲染质量都很高, 逼近为简化的模型和材质.

所以如果想实现 obj 文件的自动化处理, 那么 [docker-blender](https://github.com/linuxserver/docker-blender) + [bpy](https://docs.blender.org/api/current/index.html) 是一个可行方案.
