# Jimmy's Blog

基于 [Hexo](https://hexo.io/) 8.1.2 的静态博客，包管理器为 [pnpm](https://pnpm.io/)。

- **线上地址**：https://blog.nnsay.cn
- **托管**：GitHub Pages（自定义域名通过 `source/CNAME` 声明）
- **主题**：`concise`（git submodule）+ `themes/replacement/` 覆盖层

> Cloudflare 只托管该域名的 **DNS**，必须是 **DNS only（灰色云）**。开橙色云代理会导致 GitHub Pages 的域名校验不通过、Let's Encrypt 证书签不出来。

## 命令

| 命令 | 作用 |
|---|---|
| `pnpm start` | 本地预览：构建 + 启动 `hexo server`（默认 http://localhost:4000） |
| `pnpm build` | `pnpm run me` → 删除 `public/` → `hexo generate` |
| `pnpm clean` | 清理 Hexo 缓存与 `public/` |
| `pnpm me` | 同步主题覆盖：执行 `lnconfig`，再把 `themes/images/*` 复制进主题 |
| `pnpm lnconfig` | 把 `themes/replacement/` 的 `footer.ejs`、`include.js` 复制进主题 |

新建文章用项目本地的 Hexo，避免全局 CLI 与项目版本不一致：

```bash
pnpm exec hexo new "english-kebab-slug"
```

`pnpm deploy` 是遗留脚本（`hexo-deployer-git` 推 `gh-pages` 分支），**当前线上不走这条路**，仓库也不存在 `gh-pages` 分支。

## 部署

push 到 `master` 即触发 `.github/workflows/deploy.yaml`：

1. `pnpm install --frozen-lockfile`
2. `pnpm run build`
3. 上传 `public/` 为 Pages artifact
4. `actions/deploy-pages` 发布到 GitHub Pages

发版就是提交并 push 到 `master`，无需手工构建。

## 目录结构

```text
blog/
├── _config.yml            # Hexo 主配置
├── source/
│   ├── _posts/            # 文章（.md 与同名资源文件夹）
│   ├── about/             # 关于页
│   ├── tags/              # 标签页
│   └── CNAME              # blog.nnsay.cn
├── _config.concise.yml    # 主题配置（Hexo 官方覆盖机制，优先级最高）
├── themes/
│   ├── concise/           # 主题本体（submodule，勿直接改）
│   ├── replacement/       # 模板覆盖层：footer.ejs / include.js
│   └── images/            # 需复制进主题的图片
├── scaffolds/post.md      # hexo new 的模板
└── .github/workflows/deploy.yaml
```

## 主题定制

主题采用**覆盖式定制**，`themes/concise` 是 submodule，不要直接改它。两类定制走不同机制：

**1. 主题配置 → `_config.concise.yml`（Hexo 官方机制）**

配置文件放在仓库根目录 `_config.concise.yml`，Hexo 会自动读取并 `deepMerge` 进主题配置，**优先级高于主题目录内的 `_config.yml`**。无需任何同步脚本，`themes/concise/_config.yml` 已不再使用。

主题菜单、`excerpt_link`、搜索、sitemap、百度统计等配置都在这里。

**2. 模板与静态资源 → `themes/replacement/` + `pnpm run me`**

`footer.ejs`、`include.js` 是主题模板源码，Hexo 没有覆盖机制，只能复制：

- 改 `themes/replacement/` 下的 `footer.ejs` / `include.js`，然后 `pnpm run me` 同步到主题
- `themes/images/*` 由 `pnpm me` 复制到 `themes/concise/source/images/`（该路径被根 `.gitignore` 忽略）
- `pnpm build` / `pnpm start` 内部都会调用 `me`，正常构建流程无需手动执行

## 功能

- **图表**：`mermaid` 代码块（`_config.yml` 中已开启，主题为 dark）
- **提示块**：`marked-alert` 语法，`> [!NOTE]`
- **搜索**：`hexo-generator-search` 生成 `search.xml`
- **RSS**：`hexo-generator-feed` 生成 `atom.xml`
- **站点地图**：`hexo-generator-sitemap` 生成 `sitemap.xml`
- **评论**：主题 `git_commit` 配置，基于 GitHub OAuth

## 目录约定

- `public/`、`node_modules/`、`db.json` 已在 `.gitignore` 中，不要提交
- `source/` 下的 `about/`、`tags/` 是页面而非文章，注意与 `_posts/` 区分
- 文章文件名一旦修改，文章 URL 随之改变，会破坏既有外链

## 附：主题 submodule 接入记录

> 以下为最初接入主题时的手工步骤，其中脚本名与路径已被后续重构取代（`npm run cp` → `pnpm run me`/`lnconfig`，`themes/_config.yml` → `themes/replacement/_config.yml`），仅作历史记录保留。

```
# 添加适合的主题
git submodule add https://github.com/sanonz/hexo-theme-concise.git ./themes/concise

# 配置主题的分支(版本)
git config -f .gitmodules submodule.themes/concise.branch master

# 配置git pull自动拉取子模块
git config submodule.recurse true

# 忽略子模块变更
git config -f .gitmodules submodule."themes/concise".ignore dirty 
```

除此之外, 需要把配置安装到主题中, 主题配置放在themes目录根目录, 然后copy到主题中, 所以修改package script如下:
```
"scripts": {
    "build": "npm run cp && hexo generate",
    "clean": "hexo clean",
    "deploy": "npm run cp && hexo deploy",
    "server": "npm run cp && hexo server",
    "cp": "cp themes/_config.yml themes/concise/ && cp images/* themes/concise/source/images"
  },
```
