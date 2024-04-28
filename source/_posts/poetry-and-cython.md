---
title: 集成Poetry和Cython
date: 2024-04-28 11:22:08
tags:
  - 研究
  - Code
excerpt: 构建和安装编译后的Python包
---

# 1. Poetry and Cython

[Poetry](https://python-poetry.org/)是一个 Python 的包和依赖管理工具, 使用 Poetry 开发一个[PyPi](https://pypi.org/)包是非常快速的:

- poetry cli 可以快速一个 python 项目脚手架
- poetry 可以分别管理开发依赖和包依赖
- poetry 打包发布也很简单

Cython 是针对 Python 编程语言和扩展的 Cython 编程语言（基于 Pyrex）的优化静态编译器。它使为 Python 编写 C 扩展就像 Python 本身一样简单.

本文的重点不是介绍 poetry 和 cython 如何使用, 在本文中大致认为 poetry 是一种打包工具, 而 cython 是一种编译工具, 使用 cython 将敏感代码编译为.so 文件然后使用 poetry 打包这些.so 文件为一个 wheel 包可以用 pip 进行安装.

# 2. 集成 Poetry 和 Cython

## 2.1 代码文件

假设我们的代码结构如下;

```bash
.
├── README.md
├── object_storage
├── poetry.lock
├── poetry.toml
├── pyproject.toml
```

object_storage 文件夹存放的项目或者包的源码, poetry 的配置文件也在根目录, 这个结构也是 poetry 自动生成的脚手架代码, 其中 object_storage 是创建项目是提供的名称.

## 2.2 Cython 编译

cython 编译没有想象中复杂, 这里的编译代码和技巧也是从官方文档的[快速上手文档](https://cython.readthedocs.io/en/latest/src/quickstart/build.html)中了解的. 在根目录创建一个 build.py 文件, 内容如下:

```python
from Cython.Build import cythonize
from setuptools import setup

setup(
  name='object-storage',
  ext_modules=cythonize('object_storage/*.py'),
  script_args=['build_ext', '--inplace'],
)
```

> [!tip]
> 其实如果不想和 poetry 集成, 只需要将将文件名称修改为 setup.py, 然后就可以通过 pip intall git+https 的方案是安装了

## 2.3 Poetry 集成

### 增加编译依赖

修改 pyproject.toml 修改 build-system 内容如下:

```toml
[build-system]
requires = ["poetry-core", "Cython", "setuptools"]
build-backend = "poetry.core.masonry.api"
```

其中主要增加: Cython 和 setuptools 两个构建依赖

### 自定义 Poetry 构建

修改 pyproject.toml 修改 tool.poetry.build 内容如下:

```toml
[tool.poetry.build]
script = "build.py"
generate-setup-file = false
[tool.poetry.group.dev.dependencies]
cython = "^3.0.10"
setuptools = "^69.5.1"
wheel = "^0.43.0"
```

这里主要是让 poetry 构建的时候使用自定义的脚本, 因为要本地测试做在开发依赖中需要安装: cython/setuptools/wheel

### Poetry 打包

修改 pyproject.toml 修改 tool.poetry 内容如下:

```toml
[tool.poetry]
name = "object-storage"
readme = 'README.md'
include = ["object_storage/**/*.so"]
exclude = ["object_storage/**/*.py"]
```

这里主要增加: include 和 exclude, 作用是排除.py 源码文件仅保留.so, 为什么这么做和本文的主题有关, 本文的主题是构建一个源码被保护的 python 包

## 2.4 测试和使用

本文中的代码已经开源到 github, 这里直接使用 git+https 的方式进行安装和测试

### 安装

```bash
(env) ➜  pythontest pip install git+https://github.com/nnsay/object_storage.git@cython
```

### 测试

测试脚本与包的发布包的代码相关, 我的包是一个针对不同对象存储厂商服务的抽象包, 所以这里测试一下是不是可以调用生成 signedURL 即可.

```bash
(env) ➜  pythontest python testoos.py
download signed url: https://nnsay-cn.None.aliyuncs.com/hello.log?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=AKIAxxxxxxxxxxxxxxxx%2F20240428%2Fcn-northwest-1%2Fs3%2Faws4_request&X-Amz-Date=20240428T024809Z&X-Amz-Expires=3600&X-Amz-SignedHeaders=host&X-Amz-Signature=85a71f9c8c988cfda1786c1b77b4d7f428b948de30db13b31039a8fcc0cca15b
```

> [!note]
> 请问忽略测试结果输出, 理论上只要可以调用成功, 就证明.so 的 python 包安装成功了

# 3. 总结

本文从简单介绍了 cython 和 poetry 对的集成, 但是更具有核心价值的是: 利用 cython 将代码 python 代码编译, 然后结合使用 poetry 将包发布和安装. 如果结合 docker, 该方式可以交付编译后的代码制品, 更好的保护源码.
