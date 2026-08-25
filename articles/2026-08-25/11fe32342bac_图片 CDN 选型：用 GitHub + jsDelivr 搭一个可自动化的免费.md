---
title: 图片 CDN 选型：用 GitHub + jsDelivr 搭一个可自动化的免费图床
feedId: 34723
source: 综合讨论
publishedAt: 2026-08-25
---

## 背景

在 OpenClaw、Agent、MCP、插件这类自动化实践里，图床不只是“给博客配图”。很多场景需要机器生成图片或截图后，立刻拿到一个可公开访问的 URL：插件返回预览、MCP 工具输出结果、文档自动插图、CI 生成产物归档。这个 URL 要稳定、可外部访问，最好还能自己控制。

第三方免费图床经常出现 token 回收、限流、图片过期、接口变更；自建对象存储加 CDN 又偏重。GitHub 仓库 + jsDelivr 是一个相对均衡的免费方案：用 GitHub 做存储和版本管理，用 jsDelivr 做 CDN 分发，再通过脚本或 MCP 工具把“上传图片 → 返回 URL”自动化。

## 问题

GitHub 本身可以托管图片，但直接引用 raw.githubusercontent.com 体验一般，部分网络下不够稳定。jsDelivr 提供全球 CDN，可以按 GitHub 仓库路径访问文件，但它不是图片处理服务，不会做裁剪和格式转换，并且对缓存、仓库大小、访问频率都有限制。

所以最终目标不是“免费不限量”，而是：**用 GitHub 当存储，用 jsDelivr 当分发，再把上传过程纳入自动化链路。**

## 做法/步骤

### 1. 建一个公开仓库

新建 public 仓库，建议独立使用，例如 `assets` 或 `image-hosting`。公开仓库才能被 jsDelivr 稳定访问；私有仓库会增加 token 复杂度，不适合作为默认图床。

### 2. 生成最小权限 token

GitHub Settings → Developer settings → Personal access tokens。推荐使用 fine-grained token：

- 只授予目标仓库；
- Repository permissions 中 Contents 设为 Read and write；
- 过期时间尽量短。

token 只放在环境变量里，例如 `GITHUB_IMAGE_TOKEN`，不要写进代码或配置文件。

### 3. 上传脚本

下面是一个 Python 示例，通过 GitHub API 上传图片，并返回 jsDelivr URL。需要安装 `requests`。

```python
import base64
import os
import requests

TOKEN = os.environ["GITHUB_IMAGE_TOKEN"]
REPO = "yourname/assets"
BRANCH = "main"

def upload_image(path: str, content: bytes) -> str:
    safe_path = path.strip("/")
    url = f"https://api.github.com/repos/{REPO}/contents/{safe_path}"
    headers = {
        "Authorization": f"Bearer {TOKEN}",
        "Accept": "application/vnd.github+json",
    }
    payload = {
        "message": f"upload {safe_path}",
        "content": base64.b64encode(content).decode(),
        "branch": BRANCH,
    }

    r = requests.put(url, json=payload, headers=headers, timeout=30)

    if r.status_code == 422:
        # 同名文件已存在，需要先取 sha 再覆盖
        existing = requests.get(url, headers=headers, timeout=30).json()
        payload["sha"] = existing["sha"]
        r = requests.put(url, json=payload, headers=headers, timeout=30)

    r.raise_for_status()
    sha = r.json()["commit"]["sha"][:8]
    return f"https://cdn.jsdelivr.net/gh/{REPO}@{sha}/{safe_path}"
```

关键点是返回的 URL 带 commit hash。这样即使以后覆盖同名文件，历史链接仍然指向旧版本，避免缓存混乱。

### 4. 接入 MCP/Agent

把上面的函数封装成 MCP stdio tool 即可。工具定义可以很简单：

- 名称：`store_image`
- 输入：`{ "path": "agent/2025-04-01/preview.png", "data_base64": "..." }`
- 输出：`{ "url": "https://cdn.jsdelivr.net/gh/..." }`

在 Agent 或插件流程中，生成图片后直接调用这个 MCP 工具拿 URL。上传前先用 Pillow 或 sharp 做本地压缩，比如宽 1200px、WebP 或 PNG 格式，避免把原始大图直接推到仓库。

### 5. 引用方式

建议使用 commit hash 或 release tag：

```text
https://cdn.jsdelivr.net/gh/owner/repo@<sha>/path
```

这样不会因为 jsDelivr 缓存旧版本而引用到错误内容。

## 踩坑点

- **缓存刷新慢**：jsDelivr 对同一 URL 有较长缓存，覆盖同名文件不会立刻生效。解决方式是文件名带内容哈希，或 URL 带 commit。临时 purge 可用 `https://purge.jsdelivr.net/gh/owner/repo@sha/path`，但不要依赖它。
- **仓库限制**：GitHub 单文件超过 100MB 会被拒绝，仓库建议不要超过 1GB。适合低频小图，不适合大量原图。
- **隐私与合规**：public 仓库里的图片可被枚举和下载。截图、用户数据、内部信息、有版权素材不要传。
- **特殊字符路径**：中文、空格等路径需要 URL encode，否则 API 可能返回 404 或 422。
- **Token 泄露**：如果 MCP 配置可能出现在公开仓库，务必用环境变量注入；fine-grained token 能减小泄露后的影响范围。
- **可用性**：jsDelivr 不是 SLA 服务，偶有波动。对外关键链接建议做降级，或迁移到对象存储。

## 可复用建议

- 图床仓库保持单一职责，不要和业务代码混在一起。
- 文件名采用 `{date}/{content_hash}.webp`，避免重名和缓存问题。
- 上传前统一压缩：宽度、格式、质量在脚本里做，比事后处理可靠。
- MCP 工具保持幂等：同名同内容直接返回已有 URL；同名不同内容生成新路径，而不是覆盖。
- 定期检查仓库体积，删除无用图片。超过几百 MB 时认真考虑迁移到 OSS。
- 不要用 jsDelivr 承载高频调用图片，比如每秒几百次请求；那只适合内部分发、文档配图和临时预览。

## 总结

GitHub + jsDelivr 适合“够用、公开、可自动化”的小型图床：个人项目文档、Agent 生成物临时预览、插件调试、CI 产物归档。它免费、可版本化、容易写脚本接入。

但它不解决带宽、隐私、大文件和高可用问题。工程上的关键不是选它，而是把上传做成一条可信的工具链，并明确边界：公开、小文件、低频、非敏感。

这个方案最值得投入的，其实是上传前的图片压缩和文件名策略，而不是折腾 CDN 参数。免费图床的稳定性，最终取决于你的使用边界。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/6546cda30dd2e25d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/2c6df187ec80329f.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/095bd6bdd4cc51a0.png)

