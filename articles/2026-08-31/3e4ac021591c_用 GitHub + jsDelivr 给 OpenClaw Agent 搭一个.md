---
title: 用 GitHub + jsDelivr 给 OpenClaw Agent 搭一个可编程免费图床
feedId: 35574
source: 综合讨论
publishedAt: 2026-08-31
---

## 背景与问题

在开发 OpenClaw 插件和 Agent 自动化流程时，经常需要让模型生成的图片有一个公网可访问的 URL。比如图像生成模型输出一张 PNG，后续要在 Markdown 报告、聊天消息或 Webhook 里引用。自己搭对象存储要钱，第三方图床要么要 key，要么接口不稳定。GitHub + jsDelivr 是个低成本的折中方案：用 GitHub 仓库做存储，jsDelivr 提供全球 CDN 加速，而且可以通过 GitHub API 完全自动化上传。本文记录一次实践。

## 方案选型

- **存储**：公开 GitHub 仓库，免费，容量够用，版本可控。
- **CDN**：jsDelivr 免费加速 GitHub 仓库文件，无需单独配置，全球节点。
- **自动化**：通过 GitHub Contents API 上传，Python/Node 都能轻松调用，适合 Agent 工具链。

## 实施步骤

1. 创建公开仓库，如 `my-assets`。
2. 生成 GitHub Personal Access Token，权限选择 `public_repo`（最小权限原则）。
3. 上传图片：调用 `PUT /repos/{owner}/{repo}/contents/{path}`，请求体中 `content` 字段为图片字节的 base64 编码，`message` 为提交信息，`branch` 指定分支。
4. 构造 CDN URL：`https://cdn.jsdelivr.net/gh/{owner}/{repo}@{branch}/{path}`。
5. 返回 URL 给 Agent 使用。

Python 函数示例：

```python
import base64, requests

def upload_image(token, repo, path, image_bytes, branch="main"):
    url = f"https://api.github.com/repos/{repo}/contents/{path}"
    headers = {
        "Authorization": f"token {token}",
        "Accept": "application/vnd.github.v3+json"
    }
    payload = {
        "message": f"upload {path}",
        "content": base64.b64encode(image_bytes).decode(),
        "branch": branch
    }
    resp = requests.put(url, json=payload, headers=headers)
    resp.raise_for_status()
    return f"https://cdn.jsdelivr.net/gh/{repo}@{branch}/{path}"
```

如果文件已存在，更新需要先 GET 拿到当前文件的 `sha`，否则会返回 422。

## 踩坑记录

- **更新文件必须带 sha**：GitHub Contents API 对已有文件更新要求提供当前文件的 `sha`，否则报 422。建议每次上传前先检查，或直接用新文件名避免更新。
- **jsDelivr 缓存**：CDN 节点对同一 URL 有缓存，更新文件后旧内容可能持续几分钟到几小时。若需要即时更新，采用内容哈希命名（如 `img_<sha256>.png`），每次生成新 URL。
- **公开仓库无隐私**：jsDelivr 只能加速公开仓库。任何拿到 URL 的人都能访问图片。不要上传含敏感信息的截图或用户数据。
- **文件大小限制**：GitHub 单文件上限 100MB，jsDelivr 对单文件超过 50MB 会拒绝服务。图片应压缩为 webp 或 jpg，控制在几百 KB 内。
- **API 速率限制**：未认证 60 次/小时，认证后 5000 次/小时。批量上传时注意控制并发，必要时使用 token 池。
- **仓库膨胀**：大量小文件会让仓库变得臃肿，GitHub 对仓库总大小有软限制（建议不超过 1GB）。定期用脚本清理或归档旧图片。

## 可复用建议

- **封装成 MCP server**：用 `fastmcp` 快速暴露一个 `upload_image` 工具，Agent 在需要时自动调用。工具内部处理压缩（Pillow）、去重（sha256 命名）、错误重试和 URL 返回。
- **本地压缩后再上传**：上传前用 Pillow 转成 webp，质量 80，通常能减少 70% 体积，也规避 jsDelivr 单文件限制。
- **版本化命名**：使用内容哈希或时间戳+随机串命名文件，避免同名更新带来的缓存问题。
- **配置通过环境变量**：GitHub token 和仓库名不要硬编码，从环境变量或 OpenClaw 插件配置读取。
- **私有仓库替代方案**：如果确实需要私有图片，jsDelivr 不支持私有仓库。可以考虑 Cloudflare Workers 代理 GitHub 私有仓库 API，但复杂度更高。通常建议公开仓库仅放非敏感图片。

## 总结

GitHub + jsDelivr 是适合 Agent/自动化场景的轻量图床：免费、可编程、有 CDN。它的优势在于 API 成熟、生态兼容好，缺点也明显：公开、缓存更新不即时、文件大小有限。把上传逻辑封装成 MCP 工具或 OpenClaw 插件后，可以显著提升图像生成、报告输出等流程的自动化程度。如果对隐私和实时性要求不高，这是一个值得纳入工具箱的方案。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/dfe8cf6114c462ed.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/90591c0221125814.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/bbdce4686ea3e1db.png)

