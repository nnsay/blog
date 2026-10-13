# CLAUDE.md

本仓库的写作与改动规则。项目背景、命令、架构、部署见 `README.md`。

## 新建文章

```bash
pnpm exec hexo new "english-kebab-slug"
```

**文件名即 URL**（`new_post_name: :title.md` + permalink `:year/:month/:day/:title/`）。必须传英文 kebab-case slug，中文标题写进 front matter 的 `title`。改文件名会改变 URL 并破坏既有外链。

## Front Matter

只写这四个字段：

```yaml
---
title: 文章标题
date: 2026-10-13 10:00:00
tags:
  - DevOps
excerpt: 一句话说明这篇文章讲什么
---
```

- 不写 `categories`，分类统一由 tags 承担
- 不写 `description`，摘要一律用 `excerpt`
- 不写 `slug` / `permalink` 覆盖字段

## tags：精准且收敛

优先复用现有标签，不要为语义细致而新造近义词。

**现有标签表**（全量统计，新增前先在这里找同类）：

```text
DevOps 45   研究 37    Tips 34    Code 26     AI 14      工具 13
AWS 12      Aliyun 9   CloudNative 7          3D 4       工作 4
Linux 4     Troubleshooting 4   Reading 3      效率 3
Shell 2     Security 1  Kubernetes 1  JavaScript 1  TypeScript 1
Java 1      Git 1     Ubuntu 1   数据库 1     OKR 1      日常 1
Book 1      Tools 1
```

- 每篇 **1–3 个**，宁少勿多
- 高频标签语义分工：`研究` = 调研/选型/原理探究，`Tips` = 技巧与小抄，`Code` = 动手实现，`工具` = 工具介绍与推荐，`DevOps` = 运维/部署/CI
- **能复用就不新建**。确实要新建时选语义更宽的上位词 —— 当前已有 12 个仅出现一次的标签，不要再增加
- `Tools` 与 `工具` 语义重复，统一并入 `工具`
- 技术名词用英文（`DevOps`、`AWS`、`Kubernetes`），抽象概念用中文（`研究`、`工作`、`效率`、`数据库`）
- 大小写照表中写法：`DevOps` 而非 `devops`

## excerpt

- **必填**，正文可以不写但这段不能省
- **一行纯文本，不写 Markdown** —— 列表页是原样输出该字段的，不经过 Markdown 渲染，反引号和链接会直接显示成符号
- 一句话概括全文，约 20–50 字
  - `容器保交付、内核沙箱防暴破、密钥不进沙箱——Agent 安全的三层防御架构与 nono 集成实战`
  - `用 Rclone 一站式统一对象存储与服务器文件流转`

## 正文

- **不要写 `<!-- more -->`**。Hexo 的 excerpt filter 检测到 front matter 已有 `excerpt` 时直接走 `data.more = content` 分支，截断逻辑整个跳过：注释不会变成 `<span id="more">` 锚点，还会原样留在正文里（见 `source/_posts/yueqian.md:34`）
- **图片**：`post_asset_folder: true`，`hexo new` 会创建同名文件夹，图片放进该文件夹后按相对路径引用
- **提示块**：用 `marked-alert` 语法
  ```markdown
  > [!NOTE]
  > 说明内容
  ```
- **图表**：`mermaid` 代码块可用，主题已设为 dark

## 改动主题

`themes/concise` 是 submodule，**不要直接改**。定制分两类：

- **主题配置** → 改根目录 `_config.concise.yml`（Hexo 官方覆盖机制，自动生效，无需同步）
- **模板/脚本** → 改 `themes/replacement/` 下的 `footer.ejs`、`include.js`，然后 `pnpm run me` 同步

## 提交

`public/`、`node_modules/`、`db.json` 已在 `.gitignore` 中，不要提交。发版只需 push 到 `master`，GitHub Actions 自动构建发布。
