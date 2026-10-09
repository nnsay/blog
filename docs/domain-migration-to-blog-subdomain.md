# 博客域名迁移：nnsay.cn → blog.nnsay.cn（Cloudflare 301 跳转）

> 记录时间：2026-10-09

## 背景

博客原托管在 GitHub Pages，自定义域名绑定为裸域 `nnsay.cn`。后来域名 DNS 迁移到了 Cloudflare，原计划继续用 `nnsay.cn` 访问博客，但没有跑通（GitHub Pages 只认一个自定义域名，裸域配置未成功）。最终改为三级域名方案：

1. GitHub Pages 重新绑定自定义域名为 `blog.nnsay.cn`
2. Cloudflare DNS 增加 CNAME 记录：`blog.nnsay.cn` → `nnsay.github.io`

三级域名工作正常，但查看访问数据发现 0 访问——原因是大量老用户和搜索引擎收录的入口仍是 `nnsay.cn`，而此时 Cloudflare 上 `nnsay.cn` 和 `www.nnsay.cn` **没有任何 DNS 记录**（`www` 甚至 NXDOMAIN），用户访问直接失败。

本文记录如何让 `nnsay.cn` 的所有访问 301 跳转到 `blog.nnsay.cn`。

## 方案选型

### 为什么不能用 CNAME/A 记录直接指回 GitHub Pages？

GitHub Pages 一个站点只认一个自定义域名（当前为 `blog.nnsay.cn`）。即使把 `nnsay.cn` 解析到 GitHub Pages 的 IP（185.199.108.153 等），请求到达 GitHub 时带的 `Host: nnsay.cn` 不在其认领列表里，返回 404。

### 为什么用 Cloudflare Redirect Rule？

域名 DNS 已托管在 Cloudflare，可以在 Cloudflare 边缘节点直接做 301 重定向，请求不需要落到任何源站：

```
用户 → nnsay.cn (Cloudflare 边缘) --301--> blog.nnsay.cn (Cloudflare 边缘) → GitHub Pages
```

优点：

- 不需要任何服务器
- 301 永久重定向，搜索引擎会把权重逐步转移到新域名
- 路径、query string 完整保留，旧文章链接不失效

### 跳转类型选择：Static vs Dynamic

- **Static**：所有请求统一跳到固定 URL（如首页），路径会丢失
- **Dynamic**：用表达式拼接，保留原请求路径 —— 本文采用

## 执行步骤

### 1. Cloudflare DNS 添加占位记录

Cloudflare 面板 → DNS → Records，添加：

| 类型 | 名称 | 内容 | 代理状态 |
|---|---|---|---|
| A | `@` | `192.0.2.1` | 已代理（Proxied，橙色云） |

说明：

- `192.0.2.1` 是 RFC 5737 保留的文档专用地址，**不是真实服务器**。因为记录开启了 Cloudflare 代理，流量在边缘就被 Redirect Rule 拦截跳走，永远不会真正回源到这个 IP
- 裸域 `nnsay.cn` 无法用 CNAME（DNS 规范限制，裸域与 CNAME 冲突），所以用 A 记录占位
- `www.nnsay.cn` 未配置——历史上从来没有过 `www` 的访问习惯，不需要

### 2. 创建 Redirect Rule

Cloudflare 面板 → **Rules → Overview → Create rule → Redirect Rule**：

- **Rule name**: `blog`
- **If incoming requests match**: 选 `Custom filter expression`
  - Field: `Hostname`
  - Operator: `equals`
  - Value: `nnsay.cn`
  - 生成的表达式预览：`(http.host eq "nnsay.cn")`
- **Then**:
  - Type: **Dynamic**
  - URL 表达式:

    ```
    concat("https://blog.nnsay.cn", http.request.uri.path)
    ```

  - Status code: **301 - Permanent Redirect**
  - **Preserve query string**: 打开

关键点：`http.request.uri.path` 是原请求的路径（如 `/2024/03/05/some-post/`），拼接后实现了"路径原样保留"的跳转；Preserve query string 保证 `?from=xxx` 之类参数不丢。

### 3. 确认 GitHub Pages 侧配置

GitHub 博客仓库 Settings → Pages → Custom domain 只保留 `blog.nnsay.cn`，避免裸域残留配置造成干扰。

## 验证

### curl 验证跳转

```bash
$ curl -sI https://nnsay.cn/2024/some-post/
HTTP/2 301
date: Fri, 09 Oct 2026 03:15:40 GMT
content-type: text/html; charset=UTF-8
location: https://blog.nnsay.cn/2024/some-post/
server: cloudflare
```

三个关键点：

1. `HTTP/2 301` —— 永久重定向生效
2. `location: https://blog.nnsay.cn/2024/some-post/` —— 路径完整保留
3. `server: cloudflare` —— 跳转发生在 Cloudflare 边缘

### 首页与参数验证

```bash
# 首页跳转
curl -sI https://nnsay.cn/ | grep -i location
# 期望: location: https://blog.nnsay.cn/

# 带 query string 的跳转
curl -sI "https://nnsay.cn/?from=wechat" | grep -i location
# 期望: location: https://blog.nnsay.cn/?from=wechat
```

### 浏览器验证

无痕窗口访问 `https://nnsay.cn`，确认落到博客首页，并观察统计工具（GA 等）开始记录来自 `blog.nnsay.cn` 的流量。

## 收尾事项

- **搜索引擎收录**：301 会引导搜索引擎转移权重。可在 Google Search Console 提交 `blog.nnsay.cn` 的 sitemap 加速新域名收录
- **Hexo 配置**：`_config.yml` 的 `url` 已是 `https://blog.nnsay.cn`，无需改动（影响生成的绝对链接与 sitemap）
- **排查技巧**：本机若开着代理（fake-ip 模式），`dig` 查询会返回 `198.18.x.x` 的假 IP，不可信。改用 DoH 查真实记录：

  ```bash
  curl -s "https://cloudflare-dns.com/dns-query?name=nnsay.cn&type=A" \
    -H "accept: application/dns-json"
  ```
