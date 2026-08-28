---
title: GitHub + jsDelivr 图床实践：给 OpenClaw 自动化一个轻量图片出口
feedId: 35107
source: 综合讨论
publishedAt: 2026-08-28
---

## 背景

在 OpenClaw、Agent、MCP 或插件自动化链路里，经常需要把本地生成的截图、预览图、图表转成公网 URL，供下游模型读取或嵌入报告。直接挂对象存储要开通、计费、配权限；临时图床服务又不可控。更轻量的做法是：用 GitHub 公开仓库做存储，用 jsDelivr 做 CDN 分发，再通过 GitHub API 把上传封装成自动化工序。

这个方案适合小规模、公开、低频的图片预览场景，不适合私有图片或强生产依赖。

## 问题

直接引用 GitHub raw 地址有两个麻烦：部分地区访问不稳，且 URL 路径较长、不适合作为插件输出。jsDelivr 提供：

```text
https://cdn.jsdelivr.net/gh/{owner}/{repo}@{branch}/{path}
```

它会把公开仓库文件缓存到 CDN 节点，图片、截图等小文件加载速度通常比 raw 更稳定。对自动化来说，关键是解决两个问题：如何安全上传，以及如何避免缓存过期导致旧图不更新。

## 做法/步骤

### 1. 建公开仓库

必须先建 public 仓库，jsDelivr 只能读取公开 GitHub 文件。建议独立建一个 `assets` 或 `image-hosting` 仓库，不要和主项目代码混用。分支建议固定为 `main` 或 `release`。

### 2. 创建受限 Token

GitHub PAT 只勾选仓库权限中的 `contents: write`，并限制到该 assets 仓库。Token 放服务端环境变量，不要写进前端、客户端或公开脚本。

### 3. 用 GitHub Contents API 上传

核心逻辑是 PUT 到：

```text
/repos/{owner}/{repo}/contents/{path}
```

二进制图片要 Base64 编码；如果文件已存在，必须同时传 `sha`，否则 GitHub 会返回 422。精简版 Python 实现如下：

```python
import base64
import requests

def upload_and_cdn(token, owner, repo, path, data, branch="main"):
    headers = {
        "Authorization": f"Bearer {token}",
        "Accept": "application/vnd.github+json",
    }

    r = requests.get(
        f"https://api.github.com/repos/{owner}/{repo}/contents/{path}",
        headers=headers,
        params={"ref": branch},
    )
    sha = r.json().get("sha") if r.status_code == 200 else None

    payload = {
        "message": f"upload {path}",
        "content": base64.b64encode(data).decode(),
        "branch": branch,
    }
    if sha:
        payload["sha"] = sha

    resp = requests.put(
        f"https://api.github.com/repos/{owner}/{repo}/contents/{path}",
        headers=headers,
        json=payload,
    )
    resp.raise_for_status()

    return f"https://cdn.jsdelivr.net/gh/{owner}/{repo}@{branch}/{path}"
```

### 4. 封装成 OpenClaw 工具或 MCP

在 OpenClaw 侧把上述代码封装成一个工具：接收本地图片路径或 Base64 内容，返回公网 URL。工具描述里要注明输出是公开可读的，避免误传敏感图片。

## 踩坑点

### 缓存刷新并不总是即时

同名覆盖后，CDN 可能继续返回旧图。最稳的做法是从源头避免覆盖：文件名里引入内容哈希或时间戳。紧急情况下可以调用 purge：

```text
https://purge.jsdelivr.net/gh/{owner}/{repo}@{branch}/{path}
```

但 purge 也不保证全节点瞬间生效，生产环境不要依赖它做实时更新。

### 覆盖必须带 sha

GitHub Contents API 更新已有文件时缺 sha 会 422。上面代码先 GET 再 PUT，可以规避。自动化代码里不要假设文件一定不存在。

### Token 权限与泄漏

公开仓库不代表 Token 可以公开。只给 `contents: write`，限制到单个仓库，并定期轮换。服务端调用时通过环境变量读取。

### GitHub API 频率限制

认证后约 5000 req/h，对普通自动化上传足够。但如果批量生成大量图片，要加退避重试或队列，避免触发 403/429。

### 不适合大文件与私密内容

图片建议控制在几 MB 到十几 MB。大视频、大压缩包不适合。私有图片、证书、用户敏感数据不要放公开仓库，即便 URL 看似难猜也不安全。

### 分支 URL 不是不可变版本

`@main` 是可变的。若需要稳定版本，建议在自动化里改用 commit hash 或 tag 拼 URL，否则分支推进后旧 URL 内容可能变化。

## 可复用建议

- 文件名固定为：`{业务前缀}-{sha256前8位}-{timestamp}.png`。从源头避免覆盖和缓存混乱。
- MCP 工具返回两个字段：`url` 和 `purge_url`，排障时可以直接用。
- 高频生成场景先压缩或攒批，再上传，降低 GitHub API 和 CDN 压力。
- 公开 URL 只能用于预览、报告等非敏感场景，不要进入核心业务数据库。
- 定期删除过期图片。但删除后 CDN 不一定会即时失效，需要配合 purge 或容忍一段缓存时间。

## 总结

GitHub + jsDelivr 不是生产级对象存储，也不适合私有图片。但在 OpenClaw/Agent 自动化里作为公开图片出口，它成本低、接口简单、容易封装进 MCP 或插件。

真正决定稳不稳的，不是 CDN 本身，而是文件命名、sha 更新、缓存清理和 Token 权限这些工程细节。把这些处理干净，这个小方案就能长期复用。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/16d578859aa04d2a.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/e5f5bb36297ae5d4.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/57dafcb5c7732668.png)

