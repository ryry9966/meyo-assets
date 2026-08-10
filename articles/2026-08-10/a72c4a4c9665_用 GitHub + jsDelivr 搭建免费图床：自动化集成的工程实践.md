---
title: 用 GitHub + jsDelivr 搭建免费图床：自动化集成的工程实践
feedId: 32377
source: 综合讨论
publishedAt: 2026-08-10
---

## 背景

在为 OpenClaw、各类 Agent 或自动化流水线输出结果时，常常需要把截图、生成的图表或富文本中引用的图片持久化到一个可公网访问的地址。如果用对象存储（S3/R2/OSS），会产生固定成本；用 Imgur、SM.MS 等服务，又面临稳定性、链接持久性和 API 限制的担忧。对于个人项目、CI 产物展示或插件插图等非高频、静态图片场景，**GitHub + jsDelivr** 是很多人选择的免费方案：把图片推送到一个公共仓库，通过 jsDelivr CDN 获得一个稳定、全球可用的 URL。

这篇文章不是“用 Git 传图”的新手教程，而是从自动化集成视角，梳理选型考量、实际搭建步骤，以及容易掉进去的坑和可复用的经验。

## 问题拆解

我们要解决的问题是：

1. 在自动化脚本、MCP 工具或插件中生成图片后，如何 **无感上传** 并获得 https 直链。
2. 图床需要：
   - 免费且无需额外账号体系（依赖已有的 GitHub 账号）
   - 直链稳定，不会被墙或突然失效
   - 一定的访问速度
   - 能嵌入 Markdown 或作为网页资源

GitHub 本身允许在仓库中存储文件，raw 链接可以访问，但国内速度不稳定，而且 GitHub 官方不建议直接用 raw 做热链。jsDelivr 是一个开源 CDN，支持从 GitHub、npm 等源自动拉取文件，并提供了全球节点和 1 年期缓存（对静态图片基本足够）。所以组合之后，能把 GitHub 仓库变成一个近似图床。

## 搭建步骤

### 1. 准备图床仓库

创建一个 **公开** 仓库，建议命名类似 `assets` 或 `images`。目录结构推荐按日期或功能分类：

```
assets/
├── 2025/
│   ├── 04/
│   │   ├── chart-01.png
│   │   └── screenshot-agent.png
│   └── ...
└── icons/
    └── logo.svg
```

关键：确保文件大小控制在 **20 MB 以内**（jsDelivr 对单个文件大小的软限制，实际还可以更小，后文细说）。

### 2. 上传图片

手动上传阶段，可以通过 GitHub 网页端直接拖拽，或使用 Git 客户端。但为了自动化，需要脚本化。推荐以下方式：

**使用 `gh` 命令行（GitHub CLI）**

```bash
gh auth login
gh repo clone your-org/assets /tmp/assets
cp output.png /tmp/assets/2025/04/
cd /tmp/assets
git add .
git commit -m "add image $(date +%Y%m%d-%H%M%S)"
git push origin main
```

**通过 GitHub API 直接上传**

如果不想克隆整个仓库（仓库较大时），可以用 API 上传文件：

```bash
CONTENT=$(base64 -w 0 output.png)
PAYLOAD=$(jq -n \
  --arg message "upload image" \
  --arg content "$CONTENT" \
  --arg branch "main" \
  '{message: $message, content: $content, branch: $branch}')
curl -X PUT \
  -H "Authorization: token $GITHUB_TOKEN" \
  -H "Accept: application/vnd.github.v3+json" \
  -d "$PAYLOAD" \
  "https://api.github.com/repos/your-org/assets/contents/2025/04/output.png"
```

在 OpenClaw 或 Agent 的步骤中，只要能把文件编码为 base64 并调用 API 即可，不需要完整的 Git 环境。

### 3. 获取 jsDelivr CDN 链接

推送文件到 GitHub 后，jsDelivr 会自动同步主要分支（通常 `main` 或 `master`）。CDN 链接格式如下：

```
https://cdn.jsdelivr.net/gh/{user}/{repo}@{branch}/{path}
```

例如：

```
https://cdn.jsdelivr.net/gh/your-org/assets@main/2025/04/output.png
```

你也可以省略 `@branch` 使用默认分支，但生产环境建议明确指定分支，避免意外。

### 4. 验证缓存与刷新

jsDelivr 的 Edge 节点会缓存文件。新文件首次访问后会被缓存，**旧文件覆盖**时 CDN 不一定自动刷新。可以通过在 URL 后加查询参数强制刷新：

```
https://cdn.jsdelivr.net/gh/your-org/assets@main/2025/04/output.png?ts=123
```

更好的方法是使用 **版本化文件名**（如 `chart-v2.png`）或者用 `purge` API（有配额限制）。jsDelivr 也支持 `npm` 包方式，但图片场景没必要。

## 踩坑与注意事项

### 1. 文件大小限制
jsDelivr 对 GitHub 源文件的大小限制为 **50 MB**（新规则），但实际图片超过 **10 MB** 后加载很慢，且可能触发 CDN 的防滥用策略。日常截图、图表控制在 **2 MB** 以内比较安全。如果确实需要大文件，考虑压缩或缩放后再上传。

### 2. 仓库容量和速率限制
GitHub 建议仓库保持在 1 GB 以下，单个文件推送频繁会被限制 API 速率。自动化上传时注意错误处理，必要时加入退避重试。

### 3. 私密性问题
公开仓库的图片所有人都可访问，不适合存储敏感截图。如果必须私有，需要私有仓库 + 其他 CDN 方案（或者使用带 token 的 URL，但 jsDelivr 不支持认证拉取）。OpenClaw 场景中通常输出是公开的文档或分析图，问题不大。

### 4. 国内访问
jsDelivr 在中国大陆有节点，但并非总能直接访问，某些区域可能被干扰。可以考虑配合自定义域名，将其 CNAME 到 CDN 前缀 `cdn.jsdelivr.net`，再套一层你自己的代理或 Cloudflare Workers，但会增加复杂度。纯图床不建议过度设计，稳定优先。

### 5. 分支选择
推送图片到一个独立分支（如 `gh-pages` 或 `cdn`）会更好管理，也避免和代码仓库混淆。你需要确保 jsDelivr 拉取的目标分支配置正确。

### 6. 日志与监控
自动化流程中上传失败是常见问题（网络、token 过期、API 限流）。建议在脚本中捕获 `curl` 或 `git` 的输出，记录上传状态，并设置告警（例如通过 Telegram Bot 发通知），避免生成的文档悄悄丢图。

## 可复用建议

- **封装成 MCP Tool**：把上传和获取 CDN 链接的功能做成一个 MCP Server 工具，OpenClaw 等 Agent 可以直接调用。输入本地文件路径，返回 CDN URL。
- **结合 PicGo/Custom Uploader**：如果习惯 GUI 操作，PicGo 支持自定义图床上传接口，可以封装一个上传到 GitHub 的插件，自动转 CDN 链接。
- **与截图工具集成**：Playwright/Puppeteer 自动化截图后，顺手走上面流程推送到图床，在报告中使用 CDN URL，实现端到端自动化。
- **清理策略**：图片越积越多，仓库会膨胀。建议在自动化脚本中加入定时清理逻辑，删除过时的旧图（例如超过 30 天的临时产物）。

## 总结

GitHub + jsDelivr 搭建图床，**本质上是一个零成本、有限制的静态文件托管方案**。它非常适合 OpenClaw 这类场景下“生成图片 → 获得可嵌入链接”的轻量需求。只要控制文件大小、处理好缓存刷新问题、并合理封装成自动化环节，就能得到一个相当可靠的免费图床。但它不是高性能图片服务，也不适合存储私密或超大文件。理解它的边界，才能让工程实践更干净。

> 在动手前，默认先问自己：这些图片真的需要永久公网可访问吗？如果是，那么 GitHub + jsDelivr 是一个务实、可控的选项。

---

