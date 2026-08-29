---
title: 图片 CDN 选型：GitHub + jsDelivr 搭建免费图床的实践
feedId: 35239
source: 综合讨论
publishedAt: 2026-08-29
---

## 背景

在 OpenClaw/Agent/MCP 这类自动化流程里，图片 URL 经常是中间产物：Agent 生成的截图、流程图、预览卡片、状态图需要插入 Markdown 报告或消息卡片。公共图床有审核和稳定性问题，对象存储要账号、密钥、可能付费；对于“只要能公开访问、可脚本化、可追溯”的场景，GitHub 仓库 + jsDelivr CDN 是一个轻量选择。

## 问题

直接用 GitHub 的 raw 链接存在访问不稳定、速度一般的问题；jsDelivr 可以加速 GitHub 公开仓库文件，免费且免备案。但常见做法里，用分支名作为 URL 版本容易遇到 CDN 缓存，自动化上传后拿不到新地址。这里需要把上传、命名、URL 拼接放到工程化流程里。

## 做法

### 1. 仓库与 token

创建一个独立公开仓库，例如 `image-host`，结构按年月存放：

```
image-host/
  2025/
    04/
      3f2a...c1.png
```

创建 fine-grained Personal Access Token，权限只给该仓库的 Contents 读写。不要将 token 放进仓库。

### 2. 上传脚本

用 GitHub Contents API 写入文件，返回 commit SHA，再拼接 jsDelivr URL。文件名使用内容哈希避免覆盖：

```python
import base64
import hashlib
import os
import time
from pathlib import Path

import requests

GITHUB_TOKEN = os.environ["GITHUB_TOKEN"]
REPO = "yourname/image-host"
BRANCH = "main"

def upload_image(path: str) -> str:
    data = Path(path).read_bytes()
    digest = hashlib.sha1(data).hexdigest()[:16]
    ext = Path(path).suffix.lower() or ".png"
    remote_path = f"{time.strftime('%Y/%m')}/{digest}{ext}"
    content = base64.b64encode(data).decode()

    resp = requests.put(
        f"https://api.github.com/repos/{REPO}/contents/{remote_path}",
        headers={
            "Authorization": f"Bearer {GITHUB_TOKEN}",
            "Accept": "application/vnd.github+json",
        },
        json={
            "message": f"upload {remote_path}",
            "content": content,
            "branch": BRANCH,
        },
    )
    resp.raise_for_status()
    commit_sha = resp.json()["commit"]["sha"]
    return f"https://cdn.jsdelivr.net/gh/{REPO}@{commit_sha}/{remote_path}"
```

这里的关键是用 `commit_sha` 作为 jsDelivr 的版本号，而不是 `@main`。这样同一个文件更新后也不会被旧缓存干扰，且每个 URL 都能追溯到具体提交。

### 3. 压缩与 Agent 封装

在上传前做一次压缩，减少仓库体积和请求失败率。可以用 Pillow 将长边缩到 1200px，转 JPEG：

```python
from PIL import Image

def compress(path: str, max_width: int = 1200, quality: int = 85) -> str:
    im = Image.open(path).convert("RGB")
    if im.width > max_width:
        h = int(im.height * max_width / im.width)
        im = im.resize((max_width, h))
    out = str(Path(path).with_suffix(".jpg"))
    im.save(out, quality=quality)
    return out
```

封装成 OpenClaw 插件或 MCP 工具时，建议入参为本地路径，返回结构化 JSON：`url`、`markdown`、`remote_path`、`sha`。这样 Agent 可以直接把结果插进报告。

## 踩坑点

- **缓存问题最大**：用 `@main` 或 `@latest` 时，更新同名文件可能长时间命中旧缓存；新文件首次访问也可能需要 CDN 回源。用 commit SHA 版本化 URL，基本能避开。
- **文件大小**：GitHub API 允许 100MB，但 jsDelivr 对 GitHub 大文件支持并不好，单张建议控制在 1–2MB，超过 20MB 有失败风险。
- **权限**：fine-grained token 只给单仓库的 Contents 读写，够用即可；不要把 token 暴露到前端日志或仓库文件。
- **仓库膨胀**：图床仓库会持续变大，按年月分目录并压缩，不要存放原图；长期可考虑定期归档旧图。
- **国内访问波动**：jsDelivr 偶尔不稳定，重要场景可以加 GitHub raw 或其他备用域名做 fallback。

## 可复用建议

1. 文件名使用内容 SHA1 前 16 位，天然去重，避免缓存和覆盖。
2. URL 使用 commit SHA 版本化，永久缓存、可审计。
3. 上传逻辑做成独立函数或 MCP server，Agent/OpenClaw 插件都调用同一入口。
4. 批量上传时串行并加 429/5xx 重试，认证 API 每小时 5000 次，注意频率。
5. 只放非敏感、可公开的图片；私有仓库无法直接用 jsDelivr。

## 总结

GitHub + jsDelivr 不是高可用生产图床，但对于 Agent/OpenClaw 的临时预览、报告插图、自动化产物来说，成本低、可脚本化、可追溯，足够可靠。相比公共图床，它更透明；相比对象存储，它更省事。如果后续图片量大了，再迁到 R2/S3/OSS 也不复杂，因为上传入口已经封装好了。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/36c3f1f97237c7db.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/3b5b9a8659a94ac4.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/923895d516dce2e2.png)

