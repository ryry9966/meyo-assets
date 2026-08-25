---
title: GitHub + jsDelivr 免费图床实践：给 Agent 工作流一个可引用的图片 CDN
feedId: 34701
source: 综合讨论
publishedAt: 2026-08-25
---

# GitHub + jsDelivr 免费图床实践：给 Agent 工作流一个可引用的图片 CDN

## 背景

在 OpenClaw、Agent、MCP 插件和自动化脚本里，图片资源的稳定引用经常被忽略。典型场景包括：

- 插件生成架构图、流程图或数据可视化后，需要在 Markdown 中返回一个可访问的 URL；
- MCP 工具截图后，要把截图交给下游 Agent 或写入日志；
- 自动化任务生成的封面图、预览图、二维码，需要在多个容器或服务间复用。

本地路径、临时上传服务通常无法跨环境访问；直接使用 GitHub raw 又存在访问速度不稳定、内容缓存不可控的问题。GitHub 仓库 + jsDelivr CDN 是一种低成本、可脚本化、带版本控制的图床方案，适合小规模自动化素材的托管。

## 问题

这个方案并不是“建个仓库传图”这么简单。实践中需要处理几个问题：

1. 仓库必须公开，否则 jsDelivr 无法拉取资源；
2. 自动化上传需要处理 GitHub API 的覆盖冲突；
3. jsDelivr 对 GitHub 资源有缓存，更新同一路径时 URL 可能不刷新；
4. 文件名、路径和单文件大小需要约束，否则容易出现访问失败或缓存混乱。

下面按实际工程化步骤展开。

## 做法 / 步骤

### 1. 准备公开仓库与目录

创建一个 public 仓库，目录按用途拆分，避免所有图片堆在根目录：

```text
repo/
  assets/
    agent-cards/
    mcp-demo/
    charts/
    screenshots/
```

文件名建议使用 ASCII 小写字母、数字、连字符，不要包含空格、中文或特殊字符。这样可以减少 URL 编码问题。

### 2. 上传图片：手动或通过 GitHub API

手动上传适合少量图片；自动化场景建议直接调 GitHub REST API。首次上传某个路径时，直接 PUT 即可：

```http
PUT /repos/{owner}/{repo}/contents/assets/charts/foo.png
Authorization: token ghp_xxx
Content-Type: application/json

{
  "message": "upload chart",
  "content": "<base64-encoded-image>",
  "branch": "main"
}
```

如果路径已经存在，GitHub 会返回 409 Conflict，要求提供当前文件的 `sha`。所以自动化封装时需要注意：先尝试获取该路径的 `sha`，再决定是创建还是覆盖。

简化 Python 逻辑如下：

```python
import base64
import requests
from pathlib import Path

def upload_image(local_path, repo, token, remote_dir="assets"):
    headers = {
        "Authorization": f"token {token}",
        "Accept": "application/vnd.github+json",
    }
    data = base64.b64encode(Path(local_path).read_bytes()).decode()
    remote_path = f"{remote_dir}/{Path(local_path).name}"
    api_url = f"https://api.github.com/repos/{repo}/contents/{remote_path}"

    # 先尝试取 sha，避免覆盖冲突
    existing = requests.get(api_url, headers=headers)
    payload = {
        "message": f"upload {Path(local_path).name}",
        "content": data,
        "branch": "main",
    }
    if existing.status_code == 200:
        payload["sha"] = existing.json()["sha"]

    resp = requests.put(api_url, headers=headers, json=payload)
    resp.raise_for_status()
    return remote_path
```

### 3. 拼接 jsDelivr URL

上传完成后，图片的 CDN 地址格式为：

```text
https://cdn.jsdelivr.net/gh/{owner}/{repo}@{branch}/{path}
```

如果要固定版本，可以替换为 commit hash：

```text
https://cdn.jsdelivr.net/gh/{owner}/{repo}@{commit}/{path}
```

建议在自动化流程中使用 commit hash。原因是分支 URL 在内容更新后可能被 CDN 缓存，而 commit hash 天然对应唯一内容，适合归档和复现。

### 4. 封装成 MCP 工具或内部函数

在 OpenClaw 插件或 Agent 工具层，可以封装一个 `upload_image_to_jsdelivr`：

- 输入：本地图片路径；
- 处理：压缩尺寸、生成文件名、上传到 GitHub；
- 输出：`![alt](https://cdn.jsdelivr.net/gh/...)` Markdown 片段。

这样 Agent 生成图片后，可以直接把结果插入回复或文档，不需要手动处理 CDN。

## 踩坑点

### 1. 仓库隐私

jsDelivr 不支持从私有 GitHub 仓库拉取文件。仓库一旦设为 public，所有内容都可被访问。不要上传敏感截图、内部系统信息或带有凭证的图片。

### 2. 缓存不刷新

更新某个路径的图片后，jsDelivr 旧缓存可能继续生效。分支 URL 的缓存可能持续较长时间。解决办法：

- 使用 commit hash URL；
- 或者调用 jsDelivr 的 purge API：

```text
https://purge.jsdelivr.net/gh/{owner}/{repo}@{branch}/{path}
```

### 3. GitHub API 覆盖冲突

覆盖已有文件时必须提供 `sha`，否则会收到 409。自动化封装务必先 GET 现有文件信息。

### 4. 文件大小限制

GitHub 单文件硬限制为 100MB，但 jsDelivr 对 GitHub 源通常建议单文件不超过 20MB。为了加载速度和稳定性，图片建议压缩到 1MB 以下。可以使用 Pillow、Sharp 等工具预处理。

### 5. 网络访问波动

jsDelivr 在某些网络环境下可能出现域名解析异常或访问超时。自动化流程中建议加重试机制，并为关键场景准备 fallback URL。

## 可复用建议

1. **固定版本**：生产环境尽量使用 commit hash URL，避免缓存和内容漂移。
2. **内容哈希命名**：文件名加入图片内容哈希，例如 `arch-{sha256前8位}.png`，避免同名覆盖和缓存混淆。
3. **封装成 MCP 工具**：将上传、压缩、URL 生成做成一个独立工具，供多个 Agent 调用。
4. **上传前压缩**：限制宽度和文件大小，减少仓库体积和加载时间。
5. **使用 token 提高 API 限额**：GitHub API 未认证请求约 60 次/小时，认证后可达 5000 次/小时。
6. **README 说明用途**：在仓库 README 中写明这是图床资源仓库，避免被误删或误解。

## 总结

GitHub + jsDelivr 适合小规模、需要版本控制、希望脚本化管理的图片资源。对 Agent 和 MCP 工具链来说，它把图片引用从“临时链接”变成“可复现的 Git 资源”，配合 commit hash 还能保证内容一致性。

但它不适合大流量生产环境，也不能替代对象存储 + 专业 CDN。如果你的插件、Agent 需要稳定、隐私、高可用的图片服务，仍应选择更专业的方案。作为一个轻量、免费、易自动化的图床实践，它值得进入工具库。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/d1ff9072c2fd4e01.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/e0428e5021b4c14e.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/95c27438afda885c.png)

