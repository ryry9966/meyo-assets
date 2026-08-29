---
title: 图片 CDN 选型：GitHub + jsDelivr 免费图床的工程化实践
feedId: 35309
source: 综合讨论
publishedAt: 2026-08-30
---

## 背景
在 OpenClaw、Agent、MCP 插件开发中，经常需要托管图片：文档截图、bot 生成的图表、插件预览图、MCP 工具返回的示意图。这些图片需要一个稳定、可自动化、低成本的托管方案。市面上的图床要么收费，要么有防盗链、不稳定、限制 API 等问题。GitHub 仓库配合 jsDelivr CDN 是一个零成本、可编程的轻量图床方案，适合个人项目、开源插件和自动化流程。

## 问题
直接用 GitHub 的 raw 链接（`raw.githubusercontent.com`）访问图片，速度慢、不稳定，且不适合频繁请求。jsDelivr 是一个全球 CDN，可以加速 GitHub 仓库中的文件，提供免费、无带宽限制的服务。但使用它需要解决几个问题：仓库公开、文件大小限制、缓存更新、自动化上传等。

## 做法 / 步骤
1. **创建公开仓库**  
   在 GitHub 新建一个专用仓库，如 `assets` 或 `cdn-images`，设为 Public。jsDelivr 只能加速公开仓库。

2. **上传图片**  
   将图片文件放入仓库目录，例如 `images/2025/01/example.png`。可以通过网页上传、git 提交或 API 自动上传。

3. **构造 jsDelivr URL**  
   引用格式为：  
   `https://cdn.jsdelivr.net/gh/用户名/仓库名@版本号/文件路径`  
   版本号可以是分支名（如 `main`）或 commit hash。推荐使用 commit hash 固定版本，避免分支更新后引用自动变化。

4. **自动化上传**  
   在脚本或 Agent 中，使用 GitHub API 上传图片。例如 Python 脚本：
   ```python
   import requests, base64, os
   token = os.getenv("GITHUB_TOKEN")
   repo = "user/repo"
   path = "images/test.png"
   with open("test.png", "rb") as f:
       content = base64.b64encode(f.read()).decode()
   url = f"https://api.github.com/repos/{repo}/contents/{path}"
   headers = {"Authorization": f"token {token}"}
   data = {"message": "upload image", "content": content}
   r = requests.put(url, json=data, headers=headers)
   print(r.json()["content"]["html_url"])
   ```
   这样可以在 OpenClaw 的自动化流程中，截图后直接上传并返回 jsDelivr 链接。

5. **缓存与刷新**  
   jsDelivr 对文件有约 12 小时的缓存。新上传的文件可能不会立即通过 CDN 访问到，但通常几分钟内可用。如果更新了已有文件，需要等待缓存过期，或者使用新的版本号（如 commit hash）来强制刷新。jsDelivr 也提供 purge 缓存 API，但有速率限制，不建议频繁使用。

## 踩坑点
- **仓库大小限制**：jsDelivr 对 GitHub 仓库有 50MB 的总大小限制，单个文件不超过 20MB。超出后服务可能不可用。务必控制图片体积，使用压缩工具。
- **私有仓库无效**：只有公开仓库能被 jsDelivr 加速。如果图片涉及隐私，不要使用此方案。
- **文件名大小写敏感**：路径必须完全匹配，否则 404。
- **缓存延迟**：更新图片后，CDN 可能仍返回旧图。解决办法：每次更新后使用新的 commit hash 作为版本号，或给 URL 加 query string（但 jsDelivr 对 query string 支持有限）。
- **服务稳定性**：jsDelivr 依赖第三方 CDN 提供商，偶尔会有波动或地区访问问题。不建议用于生产关键业务。
- **API 限流**：自动化上传时，注意 GitHub API 每小时 5000 次限制（认证后），批量上传要控制频率。
- **合规风险**：不要托管侵权或违规内容，否则仓库可能被 GitHub 或 jsDelivr 封禁。

## 可复用建议
- **目录规范**：按日期或项目划分目录，如 `assets/项目名/年月/`，便于管理和清理。
- **版本化引用**：在文档或代码中引用图片时，固定 commit hash 或 tag，避免分支变动导致图片错乱。
- **封装上传函数**：写一个通用的上传函数或 CLI 工具，集成到 OpenClaw 的 action 或 MCP server 中，实现“截图 → 上传 → 返回 URL”的一键流程。
- **定期清理**：监控仓库大小，删除不再使用的图片，保持 50MB 以内。可以用 GitHub Action 定时检查。
- **兜底方案**：对于重要图片，同时备份到对象存储或本地，避免单点故障。

## 总结
GitHub + jsDelivr 是一个低成本、可编程的图片托管方案，特别适合 OpenClaw/Agent/MCP 插件中的轻量图片需求。它免费、无需服务器、支持自动化上传，但必须注意仓库公开、大小限制和缓存问题。工程化使用时，建议封装上传逻辑、使用版本化引用，并把它定位为“够用但不完美”的图床，重要场景搭配其他 CDN 或对象存储。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/7de70b645114a7ef.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/30db9379bc87e5df.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/f0febf5742bc71bb.png)

