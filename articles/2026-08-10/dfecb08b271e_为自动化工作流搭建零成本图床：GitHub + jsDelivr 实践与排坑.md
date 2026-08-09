---
title: 为自动化工作流搭建零成本图床：GitHub + jsDelivr 实践与排坑
feedId: 32299
source: 综合讨论
publishedAt: 2026-08-10
---

## 背景：为什么自动化流程需要“自建”图床

在搭建 Agent、MCP 工具或 OpenClaw 这类自动化流水线时，经常需要产出图片并返回可访问的 URL。典型场景包括：

- Agent 调用绘图模型生成示意图，需要临时存储并嵌入返回消息；
- MCP 插件截图浏览器页面，供下游工具引用；
- 自动化脚本周期性生成监控图表，推送到通知渠道。

直接用 API 返回 base64 数据既浪费 token 又难以在多个系统间传递。一个稳定、免费、可通过程序上传的图床就成了基础设施的一部分。云服务商的对象存储收费虽低，但需备案、配置凭证，轻量场景下显得繁琐。本文探讨一种零成本方案：**GitHub 仓库 + jsDelivr CDN**，并记录一次工程实践中的细节与踩坑。

## 方案原理

核心思路很简单：

1. 创建一个 **公开** 的 GitHub 仓库（因为 jsDelivr 只能给公开仓库的文件加速）；
2. 通过 GitHub API 将图片文件上传到仓库中的指定路径；
3. 利用 jsDelivr 提供的 CDN 链接访问图片：`https://cdn.jsdelivr.net/gh/用户名/仓库名@版本/文件路径`。

CDN 会将请求分发到全球边缘节点，解决 GitHub raw 服务器直连速度慢、稳定性差的问题。同时，GitHub 仓库本身提供版本控制，方便回溯。

## 工程化步骤

### 1. 准备仓库与凭证

创建一个空仓库，建议在根目录下规划 `images/yyyy/MM/` 这样按时间归档的目录结构，避免单目录文件数过多。

生成一个 **Fine-grained personal access token**，权限仅需 `Contents` 读写（用于上传和获取文件信息）。将 token 存入环境变量，不要在代码中硬编码。

### 2. 上传图片的自动化脚本

下面是一个可复用的 Python 函数，通过 GitHub Contents API 上传任意二进制内容（图片），并返回可用的 jsDelivr CDN 链接：

```python
import requests
import base64
import os
from datetime import datetime

GITHUB_TOKEN = os.getenv("GITHUB_TOKEN")
REPO_OWNER = "your-username"
REPO_NAME = "your-repo"
BRANCH = "main"

def upload_image(image_bytes: bytes, filename: str = None) -> str:
    # 自动以日期组织路径
    date_path = datetime.now().strftime("%Y/%m")
    if not filename:
        filename = f"{datetime.now().strftime('%H%M%S')}.png"
    repo_path = f"images/{date_path}/{filename}"

    url = f"https://api.github.com/repos/{REPO_OWNER}/{REPO_NAME}/contents/{repo_path}"
    headers = {
        "Authorization": f"Bearer {GITHUB_TOKEN}",
        "Accept": "application/vnd.github+json"
    }
    content_b64 = base64.b64encode(image_bytes).decode("utf-8")
    payload = {
        "message": f"upload {filename}",
        "content": content_b64,
        "branch": BRANCH
    }

    resp = requests.put(url, json=payload, headers=headers)
    if resp.status_code not in (201, 200):
        raise Exception(f"上传失败: {resp.status_code} {resp.text}")

    commit_sha = resp.json()["content"]["sha"]
    # 使用 commit hash 构建 CDN 链接，保证立刻可访问且绕过缓存问题
    cdn_url = f"https://cdn.jsdelivr.net/gh/{REPO_OWNER}/{REPO_NAME}@{commit_sha}/{repo_path}"
    return cdn_url
```

调用方只需传入图片字节数据，即可获得 CDN 链接。返回的 URL 使用了 **commit SHA** 而非分支名，这是关键设计，后面会解释。

### 3. 集成到自动化工具

- **MCP 工具**：可以封装成 `upload_image` 工具，供 Agent 调用，参数为 `base64_encoded_image`，返回 `url`。
- **Agent 流水线**：在生成图片后直接调用该函数，将 url 写入消息的 `image_url` 字段。
- **定时任务**：通过 cron 驱动脚本，生成监控图后推送到钉钉/飞书等，图片链接使用 CDN 地址。

## 踩坑记录与工程化要点

### 坑 1：jsDelivr 缓存与刷新时机

如果 CDN 链接使用分支名（如 `@main`），jsDelivr 会周期性拉取 GitHub 仓库更新，周期最长可能达 12 小时。新上传的图片在这个窗口内无法通过 CDN 访问，这对实时性要求较高的自动化流程是不可接受的。

**解决方案**：上传完成后，利用 API 响应中的 `content.sha`（文件 blob 的 sha，非 commit sha）**重新请求一次该文件的 Git blob URL** 并不直接有用。实际上，上传 API 返回的 `commit.sha` 是本次提交的 commit SHA。jsDelivr 支持精确到 commit 的版本号：`@commitSHA`。使用 `@commitSHA` 可以强制 CDN 立刻拉取该提交中的文件，几乎没有延迟。上面的示例代码已经这样实现。

> 注意：commit SHA 必须在 URL 中做相应替换。以后每次上传都会产生新 commit，链接随之改变，这对一次写入、长期引用的场景是理想的。

### 坑 2：文件大小与 API 限制

GitHub Contents API 允许的最大单文件大小为 **100 MB**，但 jsDelivr 限制单文件 **50 MB**。对图片而言绰绰有余，但如果你的自动化会产出视频或高分辨率拼接图，需提前压缩或改用其他存储。另外，GitHub API 对认证用户有 5000 次/小时的速率限制，上传频率不会成为瓶颈。

### 坑 3：仓库必须是公开的

jsDelivr 只能加速公开仓库。这意味着你上传的所有图片均可被任何人通过 CDN 链接访问。如果图片包含敏感信息，应该在上传前做脱敏处理、加密，或者放弃此方案。对于一般性的自动化输出图表，风险可控。

### 坑 4：滥用政策与流量考量

jsDelivr 的 acceptable use policy 禁止将其作为通用图床用于网站热图等大流量场景。个人开发者用于少量自动化图片的分发，通常不会被判定为滥用。但如果你的开放服务每天产生数十万次图片请求，建议套一层自己的 CDN 或直接用对象存储。**在 Agent 场景下这种可能性很小，但值得注意。**

## 可复用的扩展建议

1. **添加清理策略**：通过 GitHub Actions 每周删除超过 30 天的图片，避免仓库无限膨胀。
2. **备用 CDN**：在代码中生成多个 CDN 链接作为 fallback，如 `https://cdn.staticaly.com/gh/...`，但需要测试可用性。
3. **自定义域名**：可以在 jsDelivr 前方挂自己的域名（如 `img.example.com`），通过 CNAME 或反向代理，提升品牌感。
4. **封装为 MCP 服务器**：将上传、列表、删除操作封装成 MCP 工具，供 Claude Desktop 或 OpenClaw 这类 Agent 框架调用，极大简化图片交互。
5. **限制上传格式与大小**：在脚本里增加校验，只允许上传 PNG/JPEG/WEBP 且小于 5 MB，防止误操作。

## 总结

GitHub + jsDelivr 的组合是个人自动化工作流中非常实用、零成本的图片托管方案。通过合理使用 commit 版本号，可以绕过 CDN 缓存延迟问题；通过清晰的路径规划和自动化清理，可以长期维护一个轻量图床。相较市面上的免费图床，它完全可控、无隐私泄露风险（除了公开性本身），更适合工程化集成。

如果你的 Agent 需要一个随叫随到的图片出口，不妨用半小时搭建这一套体系。

---

