---
title: GitHub + jsDelivr 搭免费图床：OpenClaw 自动化场景下的图片 CDN 实践
feedId: 34055
source: 综合讨论
publishedAt: 2026-08-21
---

# GitHub + jsDelivr 搭免费图床：OpenClaw 自动化场景下的图片 CDN 实践

## 背景

在 OpenClaw/Agent 工作流里，很多插件会产出图片：截图、渲染封面、生成二维码、OCR 对比图、MCP 工具返回的图表预览。这些图片要被聊天通道、Webhook 或文档引用，必须有一个稳定可公网访问的 URL。

自建 OSS/CDN 有成本、鉴权和区域问题；第三方图床不稳定，还会引入额外 API 与隐私风险。GitHub 仓库 + jsDelivr 是一个相对可控的折中：GitHub 负责存储与版本管理，jsDelivr 提供全球 CDN 加速，免费、无需备案，且能与现有 Git/CI 流程复用。

## 问题

直接用 GitHub raw 链接问题很多：国内可达性一般、Content-Type 有时不对、缓存不可控。jsDelivr 虽然能加速 GitHub 仓库文件，但也不是没有限制：单文件大小、缓存更新、路径编码、滥用风险等。如果只是手动传图，这些问题不明显；一旦接入 Agent 自动上传，踩坑概率会迅速上升。

## 做法/步骤

1. 建独立 public 仓库，如 `assets`，不要和主项目混用。目录按 `YYYYMM/` 或 `agent-name/` 分。

2. 命名用内容哈希：`{sha1[:12]}.webp`，避免覆盖和缓存碰撞。文件名只保留 `[a-zA-Z0-9._/-]`。

3. 上传推荐 GitHub Contents API：

```bash
PUT /repos/{owner}/{repo}/contents/{path}
{
  "message": "add image",
  "content": "<base64>",
  "branch": "main"
}
```

Token 用 fine-grained PAT，只授予该仓库 Contents 读写权限。

4. URL 转换。GitHub 路径：

```text
https://github.com/owner/assets/blob/main/202503/ab12cd34.webp
```

对应 jsDelivr：

```text
https://cdn.jsdelivr.net/gh/owner/assets@main/202503/ab12cd34.webp
```

5. 封装成 MCP 工具或 CLI：输入图片路径或 base64，压缩、计算 hash、检查去重、上传、返回 `![alt](jsDelivr-url)`。

6. 需要更新同名文件时，优先用新文件名或新 tag；紧急情况可调用 `https://purge.jsdelivr.net/gh/owner/repo@version/file` 刷新缓存。

## 踩坑点

- **图片大小**：单文件最好控制在 1MB 以下，超过 20MB 成功率明显下降。上传前用 Pillow/sharp 转 WebP 或 JPEG。
- **Git LFS 不支持**：jsDelivr 只认仓库里的真实 blob，LFS 指针文件会直接 404。大二进制别放这个方案。
- **路径编码**：中文名、空格、`#`、`?` 会让 URL 出错，即使浏览器能打开，CDN 缓存 key 也可能不一致。
- **缓存不更新**：覆盖同路径后 CDN 可能长时间返回旧图。自动化流程里不要用 `latest.png` 这种固定文件，用内容 hash 或 tag。
- **API 限流**：未认证 GitHub API 每小时 60 次，认证后 5000 次。批量上传建议本地 git 一次性提交。
- **首次访问 404**：jsDelivr 第一次拉取可能有延迟，等 1–5 分钟再试；工具里可以加重试。
- **国内可达性**：`cdn.jsdelivr.net` 在大陆部分地区偶尔不稳定。MCP 返回时可以同时给 `fastly.jsdelivr.net` 或 `gcore.jsdelivr.net` 备选，但不要承诺 SLA。

## 可复用建议

- 实现一个 `imgpush` MCP server，配置项只有 `GITHUB_TOKEN`、`REPO`、`BRANCH`、`BASE_PATH`。
- 去重逻辑：上传前先按 hash 查询仓库是否已有同路径，若有直接返回 CDN URL，减少 commit 数。
- 返回结构化结果：

```json
{
  "url": "https://cdn.jsdelivr.net/gh/owner/assets@main/202503/ab12cd34.webp",
  "markdown": "![image](https://cdn.jsdelivr.net/gh/owner/assets@main/202503/ab12cd34.webp)",
  "fallback_urls": [
    "https://fastly.jsdelivr.net/gh/owner/assets@main/202503/ab12cd34.webp"
  ]
}
```

- 稳定资源打 tag，例如 `@2025-03`，URL 使用 tag 而不是 `@main`，避免主分支滚动更新导致缓存不一致。
- 控制使用边界：这个方案适合低频、小体积、内部工具/文档配图。生产高流量或大文件，还是上对象存储 + 专业 CDN。

## 总结

GitHub + jsDelivr 在 Agent 自动化里是一个低成本、易集成、可版本化的图片分发方案。它的价值不在于性能有多强，而在于能用现有 Git/CI 体系把“图片上传”变成可审计、可去重、可回滚的工程步骤。关键是做好命名版本化、压缩、最小权限 token 和降级策略，不要把免费 CDN 当生产高可用。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/29878a0e5fd56d50.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/8370f7a9c20cfd4b.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/7962b7d55ded2124.png)

