---
title: 用 GitHub + jsDelivr 给 Agent 工作流搭一个够用的免费图床
feedId: 32990
source: 综合讨论
publishedAt: 2026-08-13
---

## 背景

在 OpenClaw、Agent、MCP 插件这类自动化场景里，经常会遇到一个很小的需求：Agent 运行后产出一张截图、预览图、二维码或结果卡片，需要转成一个公开 URL，再传给下游插件、消息推送或 Markdown 报告。

这个需求很轻，但选项并不算多。自建 OSS/COS 要开通服务、配置密钥、考虑计费；一些免费图床接口不稳定、有防盗链或内容限制。后来我把图片统一放到 GitHub 公开仓库，再通过 jsDelivr CDN 输出，配合 GitHub API 做自动化上传，整体成本很低，也比较好复现。

## 为什么选 GitHub + jsDelivr

这个组合的优点比较明确：

- 无需服务器，仓库本身就是存储。
- jsDelivr 提供免费 CDN 加速，支持 `https://cdn.jsdelivr.net/gh/user/repo@branch/path` 形式的 URL。
- 图片进入 GitHub 后天然版本化，删除、覆盖都有记录。
- GitHub API 成熟，方便在 Agent、MCP 工具或插件里直接上传，不需要额外 SDK。

但它不是“生产级图床”。适合个人工具、开源项目、内部自动化里公开且非关键的小图，不适合隐私内容、大流量或高频写入。

## 做法与步骤

先创建一个公开仓库，建议单独命名，例如 `assets` 或 `opencLaw-images`。不要和业务代码混在一起，方便管理。

手动上传可以直接用网页或者 git push。自动化上传推荐走 GitHub Contents API。核心逻辑是把图片 base64 编码后 PUT 到仓库路径。

用 Python 写一个最小上传函数：

```python
import base64
import os
import requests

def upload_image(path, repo, token, remote_path):
    with open(path, "rb") as f:
        content = base64.b64encode(f.read()).decode()

    url = f"https://api.github.com/repos/{repo}/contents/{remote_path}"
    resp = requests.put(
        url,
        headers={"Authorization": f"token {token}"},
        json={
            "message": f"upload {os.path.basename(path)}",
            "content": content,
        },
    )
    resp.raise_for_status()
    return resp.json()["content"]["download_url"]
```

也可以直接用 `curl`：

```bash
curl -X PUT \
  -H "Authorization: token $GITHUB_TOKEN" \
  https://api.github.com/repos/OWNER/REPO/contents/images/test.png \
  -d '{"message":"upload image","content":"'"$(base64 -w0 test.png)"'"}'
```

上传成功后，jsDelivr 地址为：

```text
https://cdn.jsdelivr.net/gh/OWNER/REPO@main/images/test.png
```

这个 URL 可以直接塞进 Markdown、卡片、插件输出或通知里。

## 踩坑点

**1. jsDelivr 缓存不会立刻更新**

同名覆盖文件后，CDN 可能继续返回旧内容。默认缓存时间不固定，手动 purge 也不总是即时生效。最稳的办法是文件名带内容哈希，例如 `preview-3f2a1c.png`。这样每次内容变化都是新 URL，避免缓存问题。可以写进上传函数：

```python
import hashlib

digest = hashlib.sha1(content.encode()).hexdigest()[:8]
remote_path = f"images/{digest}-{os.path.basename(path)}"
```

**2. 仓库和文件大小有限制**

GitHub 单文件不能超过 100MB，仓库建议控制在 1GB 以内。图床图片应该提前压缩，最好转成 WebP 或压缩过的 PNG/JPEG。原图、大图不要直接传。

**3. 只能放公开内容**

公开仓库里的所有图片都可以被访问，不能放敏感信息。Agent 输出截图时要注意自动打码或过滤隐私字段。

**4. Token 权限最小化**

自动化上传时，GitHub Token 只需要 `repo` 下的内容写入权限。不要把 token 写进仓库、日志或错误堆栈。可以使用 GitHub Actions secrets 或环境变量注入。

**5. API 速率限制**

未认证请求每小时 60 次，认证后 5000 次。对低频图床足够；如果批量上传很大，要控制频率，不要并发打满。

**6. 网络可用性问题**

jsDelivr 在某些网络环境下可能不稳定。建议在插件里保留 `raw.githubusercontent.com` 作为备用 URL，失败时回退。也可以用一个小函数自动检测。

## 可复用建议

这个方案更适合作为自动化插件的“默认轻量图床”，而不是对外服务的正式 CDN。

几个工程化建议：

- 上传前统一压缩和转换格式，控制单张图在 500KB 以内。
- 文件名使用内容哈希，避免 CDN 缓存造成旧图。
- 单独建一个只读或最小权限的 token，定期轮换。
- 如果你的 MCP 工具会输出图片，可以封装一个 `upload_to_image_cdn()` 工具，把 GitHub + jsDelivr 作为默认 provider。
- 在配置里预留 `IMAGE_CDN_PROVIDER` 或 `FALLBACK_URL`，以后切换 R2、OSS、Cloudflare Images 会更容易。
- 如果某天流量上去了，不要继续依赖公共 CDN。迁移到对象存储或 Cloudflare R2 是更合理的选择。

## 总结

GitHub + jsDelivr 是一个成本很低、易于自动化、可版本化的图片分发方案。它适合 Agent、MCP 插件、OpenClaw 工作流里“需要公开 URL 但不关键”的小图输出场景。只要控制文件大小、使用哈希命名、做好权限隔离和备用 URL，它可以稳定地跑很久。

但它不是银弹，也不该被当成无限免费对象存储。用对场景，它会是一个非常顺手的组件。

---

