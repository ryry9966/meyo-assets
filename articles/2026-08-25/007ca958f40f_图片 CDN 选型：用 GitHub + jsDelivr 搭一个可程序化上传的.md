---
title: 图片 CDN 选型：用 GitHub + jsDelivr 搭一个可程序化上传的轻量图床
feedId: 34718
source: 综合讨论
publishedAt: 2026-08-25
---

## 背景

在 OpenClaw、Agent、MCP 这类自动化链路里，经常需要把截图、生成图、插件预览图丢到一个能返回直链的地方。自建 OSS 要处理鉴权、计费和生命周期；第三方免费图床要么需要登录，要么链接不稳定，要么有防盗链。GitHub 仓库 + jsDelivr 的组合，胜在免费、公开、可用 Git 版本化，并且能通过 GitHub Contents API 直接上传，适合做低频、小文件的内部图床。

## 问题

这个方案不是“无限免费 CDN”。它更合适定位为：给小工具、MCP 插件、开源项目 README、自动化报告插图提供可追溯的静态图片存储。真实限制包括：仓库体积不能无限增长；单文件过大时 GitHub API 会拒绝；jsDelivr 对 GitHub 源有缓存，更新后不一定立即生效；国内部分网络下 jsDelivr 可用性会波动。

## 做法/步骤

1. **建专用仓库**：单独创建一个 `assets` 或 `images` 仓库，不要混在代码仓库里。仓库设为 public，因为 jsDelivr 需要能读取公开文件。

2. **准备最小权限 token**：在 GitHub 生成 fine-grained token，只授予该仓库的 Contents 读写权限，不要给整个账号。把 token 放进 CI Secrets 或本地环境变量，不要写进脚本。

3. **用 Contents API 上传**：核心接口是：

   ```http
   PUT /repos/{owner}/{repo}/contents/{path}
   ```

   请求体包含 `message`、`branch`、`content`（base64）。文件名建议用 `yyyyMMdd-HHmmss-{hash}.png` 这种带时间戳和内容哈希的格式，避免覆盖和缓存混淆。

4. **拼 CDN 地址**：上传到 `main` 分支后，GitHub 原图地址是 raw.githubusercontent.com，但访问不稳定。jsDelivr 地址格式为：

   ```text
   https://cdn.jsdelivr.net/gh/{owner}/{repo}@{branch}/{path}
   ```

   需要长期稳定时，建议把 branch 换成 commit hash：

   ```text
   https://cdn.jsdelivr.net/gh/{owner}/{repo}@{commit}/{path}
   ```

5. **接入自动化**：把上传逻辑封装成一个 MCP tool 或 OpenClaw 插件函数，输入本地图片路径或 base64，输出 CDN URL 和 Markdown 片段。比如在 Agent 生图后直接调用 `upload_image`，返回 `![desc](https://cdn.jsdelivr.net/gh/...)`。

6. **压缩与控制体积**：上传前用 `pngquant`、`mozjpeg` 或 `oxipng` 压缩，单张控制在 1–2MB 以内。仓库总大小建议不要超过 1GB，否则会影响 clone 和后续维护。

## 踩坑点

- **缓存不刷新**：jsDelivr 对同一 URL 会缓存较长时间。开发期如果反复修改同一张图，最好换文件名或使用 commit hash 更新 URL。也可以用 `https://purge.jsdelivr.net/gh/...` 手动 PURGE，但不要依赖它在高并发下即时生效。
- **单文件限制**：GitHub API 对大文件不友好，超过 50MB 基本不可行；控制图片体积是前提。
- **token 泄露风险**：如果上传脚本放在前端或公开插件里，token 会直接暴露。建议把上传放在服务端或 CI 中，前端只拿结果 URL。
- **内容合规**：公开仓库意味着图片公开。不要放内部截图、用户隐私、带密钥的终端输出。即使删掉文件，Git 历史和缓存仍可能留存。
- **国内可用性**：jsDelivr 在大陆部分地区会出现间歇性不可用。如果是要发给外部用户的关键图片，建议准备 OSS 兜底，图床只作为开发期或开源项目辅助。

## 可复用建议

- 目录按 `yyyy/MM/` 或项目名组织，避免根目录几千张图。
- 维护一个 `manifest.json`，记录图片路径、CDN URL、上传时间和用途，方便检索和续用。
- 在 CI 里加一步：只对变更图片做压缩，并用 commit hash 生成 CDN URL，保证发布内容的不可变性。
- 上传接口返回至少三种格式：CDN URL、GitHub raw URL、Markdown 图片片段，减少调用方拼接错误。
- 如果后续仓库变大，可以改成 GitHub Releases 或迁移对象存储，不要硬撑。

## 总结

GitHub + jsDelivr 适合作为面向 Agent/MCP/插件的小规模图床：它程序化上传方便、零成本、可版本化，能覆盖大量内部自动化和开源文档场景。但它不适合大流量生产、敏感图片和超大文件。把它当成一个“可追溯的临时/辅助图片存储”，在体积、缓存、权限和合规上做好约束，才能真正省心。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/1a662e773bc96710.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/fca4f2e4342f53ed.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/ba9623459727b98f.png)

