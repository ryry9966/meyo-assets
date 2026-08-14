---
title: 图片 CDN 选型：用 GitHub + jsDelivr 搭一个够用的免费图床
feedId: 33182
source: 综合讨论
publishedAt: 2026-08-15
---

## 背景

在 OpenClaw、Agent、MCP 和插件自动化场景里，经常会出现一类需求：脚本生成了截图、预览图、OCR 结果或执行记录，需要得到一个稳定、可公开访问的图片 URL，方便接入通知、聊天机器人、Issue 报告或 Webhook。

本地文件路径对 Agent 内部可用，但发给外部系统就不行；临时图床经常过期、限流或夹带广告；对象存储虽然正规，但要备案、配密钥、付费，对少量公开图片来说偏重。

一个务实的折中是：**GitHub 公开仓库 + jsDelivr CDN**。它免费、可脚本化、有版本记录，适合小体量、非敏感的自动化图片托管。

## 选型边界

这套方案不是万能图床，先明确边界：

- 只能使用 **公开仓库**，jsDelivr 不能加速 GitHub 私有仓库。
- jsDelivr 对单文件有 **20MB** 上限，GitHub 仓库建议控制在 **1GB** 以内。
- 图片一旦上传，基本等于公开，任何人拿到 URL 都能访问。
- CDN 缓存更新不实时，覆盖旧文件可能长时间不生效。

如果你的场景满足“小图、公开、非敏感”，继续往下看。

## 做法 / 步骤

### 1. 创建公开图片仓库

新建一个 GitHub 仓库，例如 `assets` 或 `opencław-images`，设为 Public。目录建议按模块或月份组织：

```text
images/
  2025-01/
  screenshots/
  previews/
```

### 2. 创建受限 Token

在 GitHub 后台创建 **Fine-grained personal access token**，只授权目标仓库的 `Contents` 读写权限。不要把 Token 写进仓库，通过环境变量传入脚本。

### 3. 上传图片

可以用 GitHub REST API 上传，返回 URL 可直接交给 Agent 或 MCP 工具。

核心脚本示例：

```python
import base64
import os
import time
import requests

def upload_image(path: str, repo: str, branch: str = "main") -> str:
    token = os.environ["GH_TOKEN"]
    owner = os.environ["GH_OWNER"]
    fname = os.path.basename(path)
    target = f"images/{int(time.time())}-{fname}"

    with open(path, "rb") as f:
        content = base64.b64encode(f.read()).decode()

    resp = requests.put(
        f"https://api.github.com/repos/{owner}/{repo}/contents/{target}",
        headers={
            "Authorization": f"Bearer {token}",
            "Accept": "application/vnd.github+json",
        },
        json={
            "message": f"upload {fname}",
            "content": content,
            "branch": branch,
        },
    )
    resp.raise_for_status()

    return f"https://cdn.jsdelivr.net/gh/{owner}/{repo}@{branch}/{target}"
```

GitHub CLI 也可以：

```bash
gh api repos/<owner>/<repo>/contents/images/demo.png \
  -X PUT \
  -f message="upload demo" \
  -f content="$(base64 -w0 demo.png)" \
  -f branch=main
```

### 4. URL 规则

jsDelivr 的 GitHub 路径格式是：

```text
https://cdn.jsdelivr.net/gh/<owner>/<repo>@<branch>/<path>
```

例如：

```text
https://cdn.jsdelivr.net/gh/niceking/assets@main/images/2025-01/demo.png
```

如果不写分支，默认 master/main 行为可能因仓库设置不同而变化，建议显式指定分支。

### 5. 接入 OpenClaw / MCP

可以把上传能力封装成一个 MCP 工具，供 Agent 在生成图片后调用。

MCP 工具可以简单包装上面的 `upload_image`，输入本地图片路径或 base64，返回 `cdn.jsdelivr.net` URL。这样 OpenClaw 中的插件只需关心“我生成了一张图”，不需要关心 GitHub API、Token 和 CDN 细节。

## 踩坑点

### 1. CDN 缓存刷新很慢

jsDelivr 对 GitHub 文件有缓存。覆盖同一路径后，旧内容可能持续数小时甚至更久。工程上不要用固定文件名覆盖图片，而是让文件名带内容哈希或时间戳，如：

```text
preview-20250112-a3f9c2.png
```

这样每次上传都是新 URL，从根本上绕开缓存问题。

### 2. Git 历史会膨胀

图片进入 Git 后，即使删除，历史里仍保留二进制内容。频繁上传会让仓库体积增大，clone 变慢。建议：

- 定期用一个新仓库或 orphan 分支做轮换；
- 非必要不把临时图长期保留；
- 对于高频场景，可以每天一个目录，过段时间归档或清理。

### 3. 单文件 20MB 上限

jsDelivr 对超过 20MB 的文件支持不友好，可能返回错误或加载失败。上传前应压缩、裁剪或转换格式。截图先转 WebP/PNG 小尺寸，预览图控制在 1–2MB 内比较稳。

### 4. URL 大小写和转义

GitHub 路径大小写敏感，`Demo.png` 和 `demo.png` 是不同文件。路径中有空格、中文或特殊字符时，上传 API 需要 URL 编码，最好在脚本里统一生成纯小写、无空格的路径。

### 5. 公开即暴露

不要传内部截图、用户数据、密钥信息、带个人隐私的内容。即使 URL 看似随机，只要知道完整地址就能访问。Agent 自动截图尤其要注意，建议截图前做一次敏感信息过滤，或只上传脱敏后的图片。

### 6. 分支删除导致链接失效

如果图床仓库使用临时分支，删除分支后 CDN 链接会失效。用于长期引用的图片，推送到 `main` 或其他长期分支，不要挂在不稳定的 feature 分支上。

## 可复用建议

- **文件名不可变**：上传后不要覆盖，采用 `module-YYYYmmdd-hash.png` 规则。
- **Token 最小化**：只给单个仓库的 Contents 写权限，不碰其他仓库。
- **脚本统一入口**：维护一个 `upload_image.py` 或 MCP 工具，避免各插件各写一套。
- **上传前处理**：统一进行压缩、重命名、大小限制检查，失败时返回本地路径作为兜底。
- **不要当生产主图床**：这个组合适合工具链、内部自动化和公开示例。高可用、隐私或大流量场景还是需要对象存储。

## 总结

GitHub + jsDelivr 是一个适合开源自动化项目的轻量图床方案。它把图片管理并入 GitHub 工作流，脚本化成本低，配合 MCP 工具后，Agent 可以自然地获得“生成图片并返回公网 URL”的能力。

但它不是生产级方案。缓存刷新、单文件限制、公开访问和 Git 仓库膨胀都是现实约束。只要把这些边界设计进流程，它就是一个够用、可审计、低维护的免费图床选择。

---

