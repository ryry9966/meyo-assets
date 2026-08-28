---
title: 图片 CDN 选型：GitHub + jsDelivr 免费图床的工程化实践
feedId: 35051
source: 综合讨论
publishedAt: 2026-08-28
---

## 背景

在 OpenClaw / Agent 自动化流程中，经常需要把生成的截图、架构图、报表图片发布到文档、Issue 或聊天窗口。稳定的图片外链是刚需。对象存储（如 S3 / R2）虽然可靠，但需要配置凭证和计费；第三方图床质量参差不齐。GitHub 公共仓库配合 jsDelivr 免费 CDN 是一个轻量、可版本管理、可 API 自动化的替代方案。

## 问题

核心需求是：图片能长期访问、上传自动化、成本尽量低。GitHub 负责存储和版本管理，jsDelivr 负责全球 CDN 加速。但这不是官方推荐的图床用法，存在文件大小、缓存刷新、仓库体积等限制。下面记录一次实际接入过程。

## 做法 / 步骤

### 1. 创建公共仓库

在 GitHub 新建一个 public 仓库，例如 `assets`。jsDelivr 只能加速公共仓库，私有仓库会返回 404。初始化时添加一个 README 即可。

### 2. 规划目录结构

建议按日期和用途分目录，例如 `images/2025/01/screenshot.png`。这样便于后续清理和定位。文件名使用内容 hash 或时间戳，避免同名覆盖。

### 3. 上传图片

手动测试可以直接在网页拖拽上传，但自动化场景需要脚本。使用 GitHub API 上传需要 Personal Access Token（PAT），权限勾选 `repo` 即可。示例 Python 脚本：

```python
import base64
import requests

def upload_image(token, repo, path, local_file):
    with open(local_file, 'rb') as f:
        content = base64.b64encode(f.read()).decode()
    url = f'https://api.github.com/repos/{repo}/contents/{path}'
    headers = {'Authorization': f'token {token}'}
    data = {'message': f'upload {path}', 'content': content}
    r = requests.put(url, headers=headers, json=data)
    return r.json()
```

也可以用 `gh` CLI 或 `git push`。推荐把 token 放在环境变量中，不要硬编码。

### 4. 生成 jsDelivr URL

上传成功后，图片的 CDN 地址格式为：

```
https://cdn.jsdelivr.net/gh/{user}/{repo}@{branch}/{path}
```

例如 `https://cdn.jsdelivr.net/gh/myuser/assets@main/images/2025/01/screenshot.png`。可以省略分支，默认 `main`，但明确写出更稳妥。如果希望固定版本，可以用 commit hash 替代分支名。

### 5. 集成到 OpenClaw / Agent

把上传函数封装成 MCP 工具或插件。当 Agent 生成图片后，调用 `upload_image` 返回 jsDelivr URL，直接写入回复或文档。这样整个流程无需人工干预。

## 踩坑点

- **缓存刷新困难**：jsDelivr 对同一 URL 的缓存时间较长，覆盖同名文件后，CDN 可能继续返回旧图。官方不提供手动 purge。解决办法是文件名带内容 hash 或时间戳，上传新文件而不是覆盖。实际项目中我采用“新文件名”策略，简单可靠。
- **文件大小限制**：GitHub 单文件建议不超过 100MB，但 jsDelivr 对超大文件支持不佳，超过 20MB 的图片可能出现加载缓慢或超时。上传前用 `sharp` 或 `imagemin` 压缩到合理尺寸。
- **仓库体积增长**：长期使用会让仓库膨胀，GitHub 建议仓库保持在 1GB 以下。需要定期清理无用图片，或者按项目拆分仓库。
- **路径大小写敏感**：URL 路径必须与仓库实际路径完全一致，包括大小写。Windows 开发环境尤其要注意。
- **Token 安全**：PAT 只给最小权限，避免泄露。如果用在 CI 中，使用 GitHub Actions 的 `secrets`。
- **访问频率限制**：jsDelivr 适合低频访问，高并发场景会被限流或降级。生产环境慎用。

## 可复用建议

- 所有图片上传前统一压缩，推荐 `sharp` 转 WebP 或压缩 PNG / JPG。
- 文件名使用内容哈希（如 MD5 前 8 位），避免同名覆盖，也方便去重。
- 封装一个 `upload_image` 函数，输入本地路径，输出 CDN URL，作为 OpenClaw 的通用工具。
- 如果团队使用，可以配置 GitHub Actions：监听 push 自动压缩图片、生成索引文件。
- 监控仓库大小，可以写一个 weekly job 检查并提醒。

## 总结

GitHub + jsDelivr 免费图床方案适合轻量、可版本管理、自动化要求高的图片分发场景，尤其适合 OpenClaw / Agent 生成配图后直接发布。但它不是生产级 CDN，不适合大流量或高可用需求。合理使用能显著降低成本和运维复杂度，同时保持工程可控。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/64b36de92d3b0f5b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/2ffe98ed5e19a6f5.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/ddd12c64c3e01ceb.png)

