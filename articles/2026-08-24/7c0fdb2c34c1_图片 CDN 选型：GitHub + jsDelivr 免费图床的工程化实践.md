---
title: 图片 CDN 选型：GitHub + jsDelivr 免费图床的工程化实践
feedId: 34532
source: 综合讨论
publishedAt: 2026-08-24
---

## 背景

在 OpenClaw/Agent 工作流里，截图、OCR 预处理图、运行状态卡片、插件返回图片等，经常需要一个可公网访问的 URL，方便贴进 Markdown 报告、消息卡片，或作为 MCP 工具的返回值。

自建对象存储涉及备案、密钥管理、成本；公共图床 API 不稳定且有内容审核风险。GitHub + jsDelivr 组合的优势是免费、可版本化、可脚本化，适合低流量、内部工具型图床。

## 问题

直接使用 GitHub raw 链接在国内经常超时或 TLS 中断。jsDelivr 作为 CDN 能加速访问，但它不是专门的图片存储服务，对仓库策略、文件大小、缓存更新都有约束。如果无节制地当生产图床，很容易遇到 404、限流或仓库告警。

## 做法 / 步骤

1. 建立专用 public 仓库，例如 `assets` 或 `opencore-images`。不要和代码仓库混用，避免把二进制膨胀带进主仓库。
2. 目录按年月或用途划分：`/screenshots/2025/04/`、`/cards/`、`/mcp/`。
3. 上传后通过 jsDelivr 引用：

   ```text
   https://cdn.jsdelivr.net/gh/<user>/<repo>@<branch>/<path>
   ```

   例如：

   ```text
   https://cdn.jsdelivr.net/gh/octocat/assets@main/screenshots/2025/04/demo.webp
   ```

4. 自动化上传：

   - 本地用 PicGo 或自定义脚本，配置 GitHub API token 和 repo。
   - OpenClaw 侧可封装一个 `upload_image` 工具，接收本地文件或 Base64，调用 GitHub Contents API 提交，并返回 jsDelivr URL。
   - 使用 GitHub Actions 做后处理：push 图片后自动压缩、生成 WebP/AVIF，输出 `manifest.json`，避免提交原图。示例 Workflow 片段：

   ```yaml
   on:
     push:
       paths:
         - 'uploads/**'
   jobs:
     optimize:
       runs-on: ubuntu-latest
       steps:
         - uses: actions/checkout@v4
         - name: Compress images
           run: |
             for f in uploads/*.png; do
               cwebp -q 80 "$f" -o "${f%.png}.webp"
             done
         - name: Commit webp
           run: |
             git config user.name "ci"
             git config user.email "ci@example.com"
             git add -A
             git commit -m "optimize images" || true
             git push
   ```

   实际使用中，更建议在本地压缩后再上传，减少 Action 复杂度和仓库提交噪音。

5. 如果图片数量较多、需要稳定版本，可改用 npm 包发布，再通过 jsDelivr npm 源访问：

   ```text
   https://cdn.jsdelivr.net/npm/<package>@<version>/<path>
   ```

   npm 发布比直接堆 GitHub 仓库更符合 CDN 资源分发模型，也便于固定版本。

## 踩坑点

- 同名覆盖不更新缓存。jsDelivr 会缓存文件，push 同路径同名文件可能长期返回旧内容。建议文件名带内容 hash 或时间戳，如 `demo-3f2a1c.webp`。
- 单文件 50MB 限制。超过 50MB 的文件 jsDelivr 不服务，GitHub 也会拒绝大文件。图片一般不会，但注意视频、PSD。
- 仓库必须 public。私有仓库无法通过 jsDelivr 访问；如果传了敏感信息，等于公开。
- 中文路径或空格需要 URL 编码，否则可能 404。
- 国内访问不是 100% 稳定。`cdn.jsdelivr.net` 在某些网络环境下会被干扰，可准备 `fastly.jsdelivr.net`、`gcore.jsdelivr.net` 等备用域名，或在工具层做 fallback。
- 不要用 jsDelivr 承载高并发生产流量。它适合工具链产物、文档配图，不适合面向 C 端的大图床。滥用可能导致仓库或包被限制。

## 可复用建议

- 图片先压缩到 WebP，宽度控制在 1600px 内；可使用 `cwebp` 或 `sharp`。
- 命名采用内容寻址：`/2025/04/<用途>-<hash前8位>.webp`，方便排查缓存。
- 在 OpenClaw 中封装 MCP 工具时，返回三个值：原始 URL、jsDelivr URL、备用 CDN URL，避免单点故障。
- 维护一个 `manifest.json` 或 SQLite 索引，记录原图路径、压缩后路径、hash、上传时间，方便 Agent 检索和清理。
- 定期清理 90 天未引用的图片，控制仓库体积。
- 如果团队有私有需求，建议直接上对象存储 + 自有 CDN；免费方案只适合低敏、低流量场景。

## 总结

GitHub + jsDelivr 免费图床的关键不是“白嫖”，而是把图片上传、压缩、命名、URL 生成纳入自动化管线。对 OpenClaw/Agent 场景，它适合保存临时截图、运行结果、文档配图，配合版本 hash 和备用 CDN 可以降低很多麻烦。

控制好仓库体积和访问频率，这个方案可维持较长时间；一旦需要 SLA 或私有内容，就应切换到对象存储。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/73d862a13b1ac89b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/8cc26bdf6deac6d7.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/ed8c7a9e56e4df30.png)

