---
title: GitHub + jsDelivr 搭建免费图床：面向自动化工具的图片托管实践
feedId: 35291
source: 综合讨论
publishedAt: 2026-08-30
---

## 背景

在开发 OpenClaw 插件、MCP 工具或 Agent 自动化流程时，经常需要托管图片：生成的图表、截图、二维码、OCR 结果、状态卡片等。把这些图片直接 base64 塞进日志或消息里，不仅体积膨胀，也不利于版本追踪。第三方图床要么限流、要么链接不稳定，而且很多不允许程序化上传。

GitHub 仓库天然适合做这类资源的版本管理，配合 jsDelivr 的全球 CDN，可以低门槛获得一个“免费且相对稳定”的图片外链方案。这篇文章整理一次完整的实践过程，包括自动化上传、踩坑点和可复用建议，供有类似需求的开发者参考。

## 问题

直接用 GitHub 的 `raw.githubusercontent.com` 访问图片，速度慢，部分网络环境下还会超时。jsDelivr 通过 CDN 边缘节点加速，国内可达性明显更好。但直接用 jsDelivr 也存在几个工程问题：

- 缓存更新不及时：push 后 CDN 可能仍返回旧内容。
- 路径规则和大小限制需要摸清。
- 大量图片托管可能触碰 jsDelivr 的使用边界。
- 需要一套自动化流程，让 Agent/MCP 工具能“上传即得链接”。

下面按步骤说明如何搭建，并穿插实际踩坑点。

## 做法 / 步骤

### 1. 创建公开仓库并规划目录

新建一个公开仓库，例如 `assets` 或 `imgbed`。目录结构建议固定：

```
assets/
  images/
    2025/
      03/
        chart-01.png
        qr-01.png
```

统一小写命名，避免大小写敏感问题（后面会提到）。

### 2. 上传图片并生成 jsDelivr 链接

假设仓库为 `username/assets`，图片路径为 `images/2025/03/chart-01.png`，则 jsDelivr 链接格式为：

```
https://cdn.jsdelivr.net/gh/username/assets@latest/images/2025/03/chart-01.png
```

推荐生产环境使用 commit hash 固定版本：

```
https://cdn.jsdelivr.net/gh/username/assets@a1b2c3d/images/2025/03/chart-01.png
```

原因：`@latest` 会随着推送更新，但 CDN 缓存可能存在延迟；固定 commit hash 则保证链接永远指向该版本内容，适合关键场景。个人项目或原型阶段用 `@latest` 足够。

### 3. 自动化上传与链接生成

手动 git add/commit/push 太麻烦，尤其当 MCP 工具或 Agent 需要频繁上传时。可以这样做：

**方案 A：GitHub Actions 自动压缩并生成清单**

在仓库中加一个 workflow，监听 push 到 `images/` 目录，自动运行图片压缩脚本（如 `imagemin`、`sharp`），并把生成的 jsDelivr 链接追加到 `LINKS.md`。

简化版 workflow（YAML 示意）：

```yaml
name: process-images
on:
  push:
    paths:
      - 'images/**'
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
      - run: npm install sharp
      - run: node scripts/compress-and-link.js
      - run: |
          git config user.name "bot"
          git config user.email "bot@example.com"
          git add -A
          git commit -m "auto: process images" || true
          git push
```

`compress-and-link.js` 扫描 `images/` 下新增图片，压缩后覆盖原文件，并生成 Markdown 链接写入 `LINKS.md`。

**方案 B：写一个可被 MCP 调用的命令行工具**

如果已有 MCP 服务或 Agent 框架，可以写一个 `upload_image` 工具：接收本地图片路径，自动执行以下操作：

1. 复制图片到仓库 `images/` 对应日期目录；
2. 执行 `git add` 和 `git commit`；
3. push 到 GitHub；
4. 返回 jsDelivr URL（默认 `@latest`，可选 hash）。

伪代码：

```python
def upload_image(local_path, repo_path, remote_url):
    # 1. 压缩图片
    compress(local_path)
    # 2. 移动到仓库目录
    target = f"{repo_path}/images/{date.today().strftime('%Y/%m')}/"
    shutil.copy(local_path, target)
    # 3. git 提交
    subprocess.run(["git", "add", target], cwd=repo_path)
    subprocess.run(["git", "commit", "-m", f"add image {filename}"], cwd=repo_path)
    subprocess.run(["git", "push"], cwd=repo_path)
    # 4. 返回链接
    return f"https://cdn.jsdelivr.net/gh/user/repo@latest/images/{date_path}/{filename}"
```

这种方式让 Agent 能像调用本地函数一样上传图片，适合 OpenClaw 插件或 MCP 工具集成。

## 踩坑点

### 1. 大小写敏感问题

GitHub 文件路径是大小写敏感的，但 jsDelivr 的 CDN 在某些节点上可能不区分大小写，导致同一路径的不同大小写文件互相覆盖或 404。解决方案：统一使用小写文件名和目录名。

### 2. 单文件和仓库大小限制

GitHub 仓库建议不超过 1GB，单文件硬限制 100MB，但 jsDelivr 对单文件超过 20MB 的内容可能无法正常缓存，且会拖慢加载。图片务必压缩到 500KB 以下，常用格式用 WebP/AVIF。

### 3. CDN 缓存刷新

`@latest` 更新后，jsDelivr 边缘节点可能几分钟到几小时才刷新。如果对时效性要求高，使用 commit hash 作为版本号，每次更新生成新链接，绕过缓存。也可以手动调用 jsDelivr 的 purge API（需登录），但不适合自动化。

### 4. 私有仓库不可用

jsDelivr 只能服务公开仓库。如果图片涉及敏感信息，这个方案不适合。

### 5. 使用政策风险

jsDelivr 官方定位是服务开源项目的前端资源，并非通用图床。大量、高频的图片托管可能触发限流，甚至被禁止。个人项目、内部工具低频使用问题不大，但不要把核心业务图片全放在上面。曾有用户因上传大量非代码文件被限制。

## 可复用建议

- **固定目录和命名规范**：所有图片按 `images/YYYY/MM/` 存放，小写、短横线分隔。
- **版本策略**：需要更新的图片保留旧版本，新文件用新路径；不需要更新的直接用 commit hash 固定链接。
- **同时生成 HTML 和 Markdown**：自动化脚本输出两种格式，方便复制到不同场景。
- **监控可用性**：用 uptime 工具（如 Uptime Kuma）监控几个关键 jsDelivr 链接的状态，出现 404 或超时及时告警。
- **备选方案**：如果规模变大，迁移到 Cloudflare R2、Backblaze B2 + Cloudflare CDN，成本仍然低，但合规性和稳定性更好。GitHub + jsDelivr 适合原型、个人工具、低频图片。

## 总结

GitHub + jsDelivr 是一套低成本、版本化的图片托管方案，尤其适合 OpenClaw/Agent/MCP 这类自动化工具中的非关键图片资源。通过简单的工作流即可实现“上传即得链接”，减少手动操作。但要注意大小限制、缓存刷新和合规边界，避免过度依赖。工程上把它当作一个“轻量图床”，而非生产级 CDN，是合理的定位。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/ccefde35f25d63fd.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/a19b550daa136be5.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/7e677f9ae1a82ce0.png)

