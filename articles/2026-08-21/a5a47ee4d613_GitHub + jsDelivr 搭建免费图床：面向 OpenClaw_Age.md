---
title: GitHub + jsDelivr 搭建免费图床：面向 OpenClaw/Agent 工作流的图片分发实践
feedId: 34011
source: 综合讨论
publishedAt: 2026-08-21
---

## 背景

在 OpenClaw/Agent/MCP/插件这类自动化场景里，经常需要把截图、OCR 结果卡片、插件产物、可视化图发布成可公网访问的 URL。比如 Agent 跑完一个网页巡检后要返回截图，MCP 工具要把生成的知识卡发给用户，或者自动化脚本需要把本地 PNG 推到文档里。

如果只是本地调试，文件路径够用；一旦要跨设备、跨会话、嵌入 Markdown 或喂给下游 Agent，就需要一个相对稳定的公网图床。免费、可脚本化、可版本化是主要诉求。GitHub + jsDelivr 是一套成本低、工程上可控的组合，但也要接受它不是商业 CDN，存在缓存、容量、权限等边界。

## 问题

直接用 GitHub 的 `raw.githubusercontent.com` 可以访问文件，但它在缓存策略、访问速度、跨域引用上不如专门 CDN 友好。jsDelivr 提供 GitHub 仓库的 CDN 加速，支持 `commit`、`branch`、`tag` 引用，适合自动化生成 URL。

需要解决的问题有：如何安全地上传图片、如何生成稳定 URL、如何避免缓存旧图、如何控制仓库体积、如何把上传能力封装给 Agent/MCP 使用。

## 做法/步骤

### 1. 建立公开图床仓库

创建一个 public 仓库，例如 `cdn-assets`。建议按用途分目录：

```text
screenshots/
cards/
plugin-output/
tmp/
```

jsDelivr 只能加速 public 仓库，私有仓库不可用。仓库名和路径尽量小写，避免大小写问题。

### 2. 生成最小权限 Token

在 GitHub 设置里生成 Personal Access Token，只勾选 `public_repo` 权限即可。不要用全仓库读写权限的大 Token。也可以直接用 GitHub CLI 的已登录状态，或者用 GitHub Actions 里的 `GITHUB_TOKEN`，但要注意默认 Token 只能在当前仓库使用。

### 3. 上传图片

手动上传可以直接用网页拖拽，但自动化场景更推荐脚本。下面是一个基于 GitHub Contents API 的最小上传函数：

```python
import base64
from pathlib import Path
import requests

def upload_image(token, owner, repo, path, local_file, branch="main"):
    data = base64.b64encode(Path(local_file).read_bytes()).decode()
    url = f"https://api.github.com/repos/{owner}/{repo}/contents/{path}"
    headers = {"Authorization": f"Bearer {token}"}
    payload = {
        "message": f"upload {path}",
        "content": data,
        "branch": branch,
    }
    r = requests.put(url, headers=headers, json=payload)
    r.raise_for_status()
    return r.json()["commit"]["sha"]
```

上传返回 `commit sha`，这个值后面要用来固化 CDN URL。

### 4. 拼接 jsDelivr URL

推荐格式：

```text
https://cdn.jsdelivr.net/gh/<owner>/<repo>@<commit>/<path>
```

例如：

```text
https://cdn.jsdelivr.net/gh/yourname/cdn-assets@a1b2c3d/screenshots/2025-01-01-home.png
```

调试阶段可以先用 `@main` 分支引用，但生产或跨会话引用一定要用 commit sha。分支会移动，commit 是固定的。

### 5. 接入 OpenClaw/Agent/MCP

可以把上传逻辑封装成一个 MCP 工具，比如 `upload_image`，输入本地文件路径或 base64，输出结构化结果：

```json
{
  "cdn_url": "https://cdn.jsdelivr.net/gh/...@.../...png",
  "commit": "a1b2c3d",
  "size": 245760,
  "sha256": "..."
}
```

这样 Agent 在任务中可以直接调用工具，把返回的 `cdn_url` 插入 Markdown、消息卡片或下游文档。也可以在 OpenClaw 插件里挂一个本地命令：

```bash
./upload-image.sh ./result.png --dir screenshots --commit
```

脚本负责压缩、上传、输出 URL。

## 踩坑点

### 1. 缓存不即时

如果你用 `@main` 引用同名文件，更新后 jsDelivr 可能不会立刻返回新图。可以访问 `https://purge.jsdelivr.net/gh/<owner>/<repo>@main/<path>` 触发清除，但缓存刷新有延迟，也不建议频繁依赖。最稳的是上传后使用新 commit 生成 URL，旧 URL 继续指向旧版本，天然解决缓存问题。

### 2. 文件大小限制

GitHub 对单个文件超过 50MB 会警告，超过 100MB 直接拒绝。图片一般到不了这么大，但截图原图容易几十 MB。建议上传前统一压缩到 5–10MB 以下，转成 WebP 或 JPEG。jsDelivr 对超大静态文件的分发性能也不友好。

### 3. 仓库膨胀

个人公开仓库有容量建议上限，通常建议控制在 1GB 以下。自动化跑多了，`tmp/` 和旧截图会堆积。可以加清理策略：按日期目录管理，超过 30 天的临时图自动删除；或者每月跑一次 GitHub Action 清理未引用图片。

### 4. 安全边界

仓库必须 public 才能被 jsDelivr 访问。任何上传上去的图片都等于公开。不要传包含内网 IP、账号信息、密钥、个人隐私的截图。如果涉及敏感数据，应该换对象存储 + 私有签名方案。

### 5. 路径大小写

GitHub 路径大小写敏感。上传时文件名为 `Home.PNG`，引用时写 `home.png` 会 404。建议统一小写命名，脚本里自动 `str.lower()`。

### 6. 滥用风险

免费 CDN 不是 SLA 承诺。如果资源被大量异常访问，或者仓库被判定为滥用，可能被限流。不要把核心业务流量完全压在 jsDelivr 上，它更适合开源项目、内部工具、Agent 输出预览、文档配图。

## 可复用建议

- 用 commit sha 固化 URL，不要在生产里依赖 `@main`。
- 上传函数统一返回 `cdn_url + commit + sha256`，方便追踪和回滚。
- 图片先压缩再上传，控制在 5MB 以内，优先 WebP。
- 将上传封装成 MCP 工具或 OpenClaw 插件，不让 Agent 直接拼 HTTP 请求。
- 仓库 README 写清目录规则、命名规则、清理周期。
- 对需要长期保存的资源，打 tag 批量标记，比如 `v2025-01`。

## 总结

GitHub + jsDelivr 的图床方案，优点是免费、可版本化、容易接入自动化工作流，尤其适合 OpenClaw/Agent/MCP 这类需要可编程图片分发的场景。它的缺点也很明确：缓存不即时、仓库必须公开、容量有限、不适合敏感图片或商业 SLA。

工程上的关键在于两点：用 commit 固化引用，把上传能力封装成结构化工具。做到这两点后，它可以在自动化链路里稳定承担“图片暂存 + 公网分发”的角色。对于更严肃的生产需求，还是建议迁移到对象存储 + 正规 CDN。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/5717674b6dc1bb70.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/c07d8fc193f0088e.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/2ae38e5e822a343d.png)

