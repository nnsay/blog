# Product

<!-- impeccable:product-schema 1 -->

## Platform

web

## Users

两类读者并重（用户确认，2026-10-13）：

1. **作者本人 + 技术同行** —— 长期记录、沉淀与分享技术实践。这类读者关心阅读体验、检索与归档，多数从搜索引擎或 RSS 进入文章页。
2. **潜在雇主 / 合作方** —— 通过 about 页的简历与 projects 页的作品评估作者能力，需要快速看到可验证的能力证据。

到达渠道（用户确认，无统计数据，主次未定）：搜索引擎、RSS 订阅、简历/作品链接直接进入。

## Product Purpose

个人技术博客，接替作者在 OSChina 上的历史博客。承载 107 篇技术文章、个人介绍与简历、个人作品展示三块内容。

成功标准是双重的：对作者是可持续维护、能长期沉淀的写作载体；对访客是能快速建立"这个人能做什么"判断的门面。

## Positioning

中文技术写作，动手导向的实践记录。作者背景横跨 AI 应用开发、DevOps 与后端，内容跨度从内核级沙箱、云原生、跨云迁移到 3D 处理与 Agent 实现——不是资讯搬运或通用教程站，而是第一手实践与踩坑记录。

## Operating Context

- **写作**：`pnpm exec hexo new "<english-kebab-slug>"` → 填 front matter（`title` / `date` / `tags` / `excerpt`）→ 写正文
- **标签体系**：刻意收敛，复用现有标签。现有约 25 个，高频为 `DevOps` 45 / `研究` 37 / `Tips` 34 / `Code` 26
- **本地预览**：`pnpm start` → http://localhost:4000
- **发布**：push 到 `master` → GitHub Actions 构建 → GitHub Pages
- **评论**：主题 `git_commit` 配置，基于 GitHub OAuth，**仅文章页开启**，about / tags / projects 等页面均关闭
- **页面资产**：非文章页面的图片/视频放 `source/<page>/index/`，按绝对路径引用

## Capabilities and Constraints

**技术栈**：Hexo 8.1.2 静态生成 + pnpm；主题 `concise`（git submodule）+ `themes/replacement/` 覆盖层。

**约束**：

- 静态站点，无后端运行时。任何功能必须能静态生成。
- 主题定制分两路：配置改根目录 `_config.concise.yml`（Hexo 官方覆盖，优先级最高）；模板/脚本改 `themes/replacement/` 后 `pnpm run me` 同步。**不直接改 submodule**。
- 主题为 Less + CSS 自定义属性，已支持 `prefers-color-scheme: dark`；页面内联 `<style>` / `<script>` 可用（marked 透传），且能用主题的 CSS 变量（`--color-border`、`--color-muted`、`--color-primary`、`--color-hover`、`--color-bg`）。
- 图片需自行压缩后入库（工作流无图片处理步骤）；视频需预压缩，无转码服务。
- 写作规则不使用 `categories`（统一由 tags 承担）、不使用 `description`（统一用 `excerpt`）。

**未决 / 待处理**：

- **job-pilot 仓库当前为 private**（2026-10-13 核实：`gh api` 返回 `"private": true`，匿名请求 404）。projects 页的"查看源码"链接指向 `https://github.com/nnsay/job-pilot`，对外访客会 404。产品意图是公开作品，需作者决定是转为 public 还是改链接目标。
- **作品区规模会持续增加**（用户确认），两张卡当前 UI 风格不一致（作品 1 为全宽跑马灯，作品 2 为左媒体右文字），统一方案待定。

## Brand Commitments

- 站点名 **Jimmy's Blog**，副标题 `learning, thinking, sharing`，作者署名 **Jimmy Wang**
- 域名 `blog.nnsay.cn`
- 作者身份表述：AI 应用开发 / DevOps / 后端开发；求职地北京、西安
- 品牌语音：中文技术写作，平实、具体、动手导向
- **无** logo、无定制品牌色彩规范、无对外品牌视觉资产 —— 视觉身份目前完全由 concise 主题承载

## Evidence on Hand

真实可用的素材及其位置：

| 素材 | 位置 | 说明 |
|---|---|---|
| 技术文章 107 篇 | `source/_posts/` | 最早可追溯多年，持续更新 |
| 作品 1 截图 4 张 | `source/projects/index/word-fun-*.jpg` | 探索页 / 词库 / 成就 / Debug 页，均 1400px 宽 JPEG |
| 作品 2 演示视频 | `source/projects/index/job-pilot-demo.mp4` | 480×722 竖屏，54.4 秒，无音轨 |
| 作品 2 视频封面 | `source/projects/index/job-pilot-poster.jpg` | |
| 简历 | `source/about/index/resume.pdf` | 2026-07 版 |
| 资质证明 | `source/about/index/acp.jpg`、`award1.jpg` | 阿里云 ACP 认证、优秀员工 |
| 医疗器械资质 | about 页 NMPA 查询链接 ×2 | 湘械注准 20212212316、20222210103 |

**不存在、不得编造的素材**：客户案例与 logo、用户量或流量数据、性能基准测试、营收/商业指标、第三方评价或推荐语、媒体报道。

## Product Principles

1. **内容优先，界面退后** —— 文章的可读性、检索与归档优先于视觉装饰；展示页的视觉应为内容服务，不喧宾夺主。
2. **作品靠证据说话** —— 真实截图与演示视频比文字描述更有说服力；能用产品自己的画面就不用形容词。
3. **一致性靠复用** —— 标签收敛、UI 呈现统一。同类信息出现两次就应该共用一套呈现，而不是各写一套。
4. **静态、零运维** —— 不引入运行时后端依赖；发布即 push，构建与部署全自动。
5. **改动精准可追溯** —— 只动必须动的；主题定制走覆盖层而非改动 submodule，保持上游可更新。

## Accessibility & Inclusion

未确认专项无障碍需求。已知现状：主题支持 `prefers-color-scheme: dark`。未来新增交互组件时应保持键盘可达（既有跑马灯已提供按钮与 aria-label）。
