---
title: GitHub + jsDelivr：给 OpenClaw 搭一套可脚本集成的免费图床
feedId: 31292
source: 综合讨论
publishedAt: 2026-08-02
---

## 背景

在 OpenClaw 这类 Agent 自动化的日常里，图片产出远比你想象的多：截图证据、数据可视化、AI 绘图的中间结果、markdown 渲染的预览……这些图片需要一个“落脚点”。上传到临时文件服务容易过期，挂在自己服务器上又增加维护成本。真正的需求是：**一个免费、稳定、而且能被脚本和 MCP 工具链轻松集成的图床**。

GitHub 仓库 + jsDelivr CDN 的组合恰好可以满足这套约束。它不需要额外域名，没有流量账单，上传、分发全部可以通过 GitHub API 自动化，非常适合嵌入到 OpenClaw 的图片管线里。本文记录我在自己的 OpenClaw 工作流中实践这套方案的过程、遇到的实际坑以及可复用的封装思路。

## 问题分析与选型取舍

选择 GitHub + jsDelivr 而非传统图床，主要基于三点：

1. **完全脚本化**：传统免费图床大多依赖 Web 表单上传，自动化需要通过逆向接口，随时可能变动或封禁。GitHub API 则稳定且有官方保证，可直接用 token 调用。
2. **版本追溯**：图片文件随 commit 记录在仓库中，可以方便地回滚或审计。
3. **CDN 加速**：jsDelivr 在全球有数百节点，能自动对 GitHub 仓库内容进行分发，免去自行配置 CDN 的烦恼。

但它也有先天限制：仓库容量（建议 <1GB）、单文件不要超过 20MB 以避免 CDN 拒绝服务、API 速率限制（认证后 5000 次/小时）、以及国内部分运营商对 `cdn.jsdelivr.net` 的劣化路由。工程上必须把这些边界条件提前处理干净。

## 实践步骤

### 1. 创建专用图床仓库

新建一个 **public** 仓库（例如 `assets`），图片统一存放在 `/images/` 路径下。保持仓库干净，便于后续清理。

### 2. 生成 Personal Access Token

在 GitHub Settings → Developer settings → Personal access tokens (classic) 中生成一个 token，勾选 `repo` 权限。部分自动化场景如果只需要上传，可选用 fine-grained token 限定到单个仓库的 Contents 写权限，缩减风险面。将 token 存入 OpenClaw 的环境变量 `GITHUB_TOKEN`。

### 3. 上传脚本（Python 示例）

核心逻辑是通过 GitHub Contents API 将文件以 base64 编码后 PUT 到仓库。下面是量产环境里经过验证的片段：

```python
import base64
import os
import requests

OWNER = "your-username"
REPO = "assets"
BRANCH = "main"
TOKEN = os.environ["GITHUB_TOKEN"]

def upload_image(local_path: str, remote_name: str):
    with open(local_path, "rb") as f:
        content = base64.b64encode(f.read()).decode()
    url = f"https://api.github.com/repos/{OWNER}/{REPO}/contents/images/{remote_name}"
    headers = {"Authorization": f"token {TOKEN}"}
    payload = {
        "message": f"upload {remote_name}",
        "content": content,
        "branch": BRANCH
    }
    r = requests.put(url, json=payload, headers=headers)
    r.raise_for_status()
    # 返回 jsDelivr CDN 链接
    return f"https://cdn.jsdelivr.net/gh/{OWNER}/{REPO}@{BRANCH}/images/{remote_name}"
```

文件名推荐使用内容 SHA-256 的前 12 位 + 后缀，避免冲突并天然去重。

### 4. 封装为 MCP 工具或 OpenClaw 插件

将上述函数暴露为一个简单的 MCP 工具 `upload_image`，接收本地路径参数，返回 CDN URL。这样在 Agent 的推理循环里即可通过工具调用完成上传，而无需繁琐的 shell 命令拼接。示例 MCP 定义：

```json
{
  "tools": [{
    "name": "upload_image_to_cdn",
    "description": "Upload an image to GitHub and return its jsDelivr CDN URL.",
    "parameters": {
      "type": "object",
      "properties": {
        "file_path": {"type": "string"}
      },
      "required": ["file_path"]
    }
  }]
}
```

Agent 随后输出 `![image](https://cdn.jsdelivr.net/gh/.../xxx.png)` 即可在各种客户端渲染。

### 5. 缓存与可靠性优化

jsDelivr 对全新文件会回源 GitHub，首字节延迟可能较高。我的做法是：在上传后统一构造一个带版本号的路径（如 `/images/v3/`），这样新目录树强制 CDN 获取最新资源，也便于按批次清理旧文件。

在国内网络环境下，`cdn.jsdelivr.net` 经常绕路甚至被 DNS 污染。我内置了一个备用域名列表：`fastly.jsdelivr.net`、`gcore.jsdelivr.net`，以及社区镜像 `jsdelivr.pai233.top`。工具返回 URL 时默认提供 `cdn`，但 Agent 侧可以实现一个简单的 fallback 逻辑：当某些客户端报告加载失败时，切换域名重试。

## 踩坑记录

- **速率限制**：上传文件算一次写入请求，批量处理 100 张图片时会快速消耗配额。建议每次上传间隔 200ms，并捕获 403/429 后等待重试。
- **jsDelivr 单文件 20MB 硬限制**：超过 20MB 的文件虽然仍能推送到 GitHub，但 CDN 会拒绝缓存并返回 404。如果有大图需求，可在上传前用 sharp 等库将 PNG 转 JPG 并降低质量。
- **缓存刷新不可靠**：官方 purge API (`purge.jsdelivr.net`) 有频率限制且经常延迟。生产经验是：**永远通过换文件名/路径来强制更新**，不要依赖 purge。
- **仓库 size 膨胀**：自动化运行几个月后，我的图床仓库轻松突破了 300MB。设置 GitHub Actions 定时清理30天前的图片是不错的防治办法（配合 `git-filter-repo` 或直接删除文件后 force push，但 force push 会丢失历史，建议使用新 commit 删除旧文件，再跑 `git gc`）。

## 可复用建议

1. **开始就设计文件名规范**：`/year-month/uniqueid.ext` 或内容哈希，不要用原始标题命名。
2. **将上传能力内建到 Agent 的工具箱**，而不是期望用户手动运行脚本。
3. **监控 API 使用量**，在接近限制时降级到 local 存储并提醒运维。
4. **对于国内用户群体，额外提供一层 CDN 中继**（如用 Cloudflare Workers 反代 jsDelivr），以解决运营商劫持问题。

## 总结

GitHub + jsDelivr 的方案并不完美，但它是目前个人自动化场景下**成本最低、集成最平滑**的图床解决路径。在 OpenClaw 的上下文中，它把“图片持久化”变成了一个简单的函数调用，并且和版本控制天然绑定。只要提前吃透容量、缓存和网络三方面的限制，它会是一个能长期跑稳的组件。

> 工程上没有银弹，只有对边界条件的清醒认知和兜底策略。

---

