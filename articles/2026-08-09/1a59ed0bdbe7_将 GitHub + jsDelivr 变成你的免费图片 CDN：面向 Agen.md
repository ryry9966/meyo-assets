---
title: 将 GitHub + jsDelivr 变成你的免费图片 CDN：面向 Agent 自动化的工程实践
feedId: 32265
source: 综合讨论
publishedAt: 2026-08-09
---

## 背景

在 OpenClaw 的 Agent 流水线里，经常会产出需要外网访问的视觉产物：截图、图表、二维码、Markdown 中引用的插图……一个稳定、快速、低成本的图床就成了基础设施需求。传统图床要么收费，要么自定义度低；自建 OSS 又涉及备案、HTTPS、费用等一系列运维负担。

GitHub 仓库本身可以托管静态文件，但通过 `raw.githubusercontent.com` 直接访问速度慢、容易被墙，且无法享受 CDN 加速。jsDelivr 作为免费、全球多节点的开源 CDN，原生支持从 GitHub 仓库拉取文件并提供加速，这使我们能免费获得一个「仓库即图床 + CDN 分发」的方案。

本文面向需要将图片上传集成到 MCP 工具、插件或自动化流程中的开发者，分享一套经过验证的实践方案。

## 问题拆解

搭建可用图床需要解决三个核心问题：

1. **上传**：如何将图片可靠、可编程地提交到 GitHub 仓库。
2. **访问**：如何生成稳定、可缓存的 CDN 链接，避免 URL 频繁变动导致客户端缓存失败。
3. **集成**：在 Agent/工作流中如何以最小侵入性调用上传与链接生成。

直接使用 `git push` 的方式不适合程序化调用，也不利于并发和速率控制。我们需要利用 GitHub REST API 实现轻量级的「文件上传-链接返回」闭环。

## 实践步骤

### 1. 创建专用图床仓库

建议单独建立一个 public 仓库（例如 `assets`），结构简单：

```
assets/
└── images/
    └── 2025/
        ├── screenshot_20250301.png
        └── chart_20250302.jpg
```

按年月分目录方便后期管理，也可直接用 UUID 或 timestamp 命名避免冲突。

### 2. 配置上传 Token

在 GitHub Settings → Developer settings → Personal access tokens 中生成一个细粒度 token，权限仅勾选 `Contents: Read and write`，限定到该仓库。将 token 作为环境变量保存，切忌硬编码到代码中。

### 3. 编写上传脚本

使用 Python 的 `requests` 库调用 GitHub Contents API 即可完成文件上传。核心逻辑：

- 读取本地图片二进制内容。
- 计算 Base64 编码。
- 构造 PUT 请求到 `https://api.github.com/repos/{owner}/{repo}/contents/{path}`。
- 携带 JSON body：`message`, `content`, `branch`。如果是新建文件，不需要 `sha`；若需覆盖同路径文件，则需先在 HEAD 中获取该文件的 `sha` 再更新（避免丢失历史版本）。
- 请求头包含 `Authorization: token <your-token>`。

**示例片段（思路）**：

```python
def upload_image(image_bytes, repo, path, branch="main"):
    content_b64 = base64.b64encode(image_bytes).decode("utf-8")
    url = f"https://api.github.com/repos/{repo}/contents/{path}"
    payload = {
        "message": f"Upload {path}",
        "content": content_b64,
        "branch": branch
    }
    resp = requests.put(url, headers={"Authorization": f"token {TOKEN}"}, json=payload)
    return resp.json()
```

成功响应中会包含 `content.download_url`，但我们实际不使用它，因为我们要走 jsDelivr。

### 4. 生成 jsDelivr CDN 链接

假设仓库为 `myuser/assets`，分支 `main`，文件路径为 `images/2025/photo.png`，则 jsDelivr 链接为：

```
https://cdn.jsdelivr.net/gh/myuser/assets@main/images/2025/photo.png
```

注意：jsDelivr 对 GitHub 文件大小限制为 **50 MB**，适合绝大多数图片。首次访问时，jsDelivr 会自动拉取文件到全球节点，之后的访问会命中缓存并加速分发。

### 5. 封装为 MCP 工具

对于 OpenClaw 生态，可以把上传+链接生成封装为一个 MCP 工具 `upload_image_to_cdn`，输入图片 base64 或文件路径，输出 CDN 链接。这样在 Agent 工作流中只需调用该工具即可获取可访问的图片 URL，无需关心底层实现。

类似地，也可以做成 Action 插件或命令行工具，随用随调。

## 踩坑点与避坑指南

- **jsDelivr 缓存刷新**：更新同路径文件后，CDN 边缘节点可能仍缓存旧版本（最长 24 小时）。jsDelivr 提供了手动 purge 接口：`https://purge.jsdelivr.net/gh/user/repo@branch/file`。可以在上传脚本中加入自动 purge 逻辑，或在 URL 中附加版本 query 参数强制刷新，但最简单的方案是使用 **内容哈希文件名**（如 `img_<sha256>.png`），从根本上避免缓存冲突。
  
- **API 速率限制**：对 `api.github.com` 的未认证请求仅有 60 次/小时，使用 token 后提升至 5000 次/小时，对于自动化上传足够。但如果批量上传大量文件，需控制并发，避免 `403 rate limit` 错误。
  
- **文件名特殊字符**：路径中的空格或 Unicode 字符容易导致 422 错误。建议仅使用小写字母、数字、连字符和下划线，并对路径做 URL 编码后生成 CDN 链接。
  
- **仓库容量**：GitHub 官方建议仓库小于 1GB，单个文件不超过 100MB。日常图片使用基本不会触及，但仍建议加入简单的容量监控脚本。
  
- **隐私与安全性**：仓库为 public，所有图片均公开可见。若需要私密图片，这条路不适用，应考虑 OSS + 签名 URL。

## 可复用建议

1. **标准化命名**：在文件名中加入时间戳和内容哈希，如 `20250301_a3f2_chart.png`，既保证唯一性，又便于后期检索。
2. **自动化压缩**：通过 GitHub Actions 在推送到仓库时自动压缩图片（如使用 `imagemin`），减少存储和流量。
3. **集成健康检查**：在上传函数中，校验文件大小不超过 50MB，并自动将 CDN 链接写入日志，方便追踪。
4. **避免重复上传**：可计算本地文件 SHA256，先查询仓库中是否已存在同名文件（利用 API 获取目录列表），存在则直接返回 CDN 链接，减少 API 调用。
5. **作为内部工具分发**：可将工具脚本打包为 Docker 镜像或 Python 包，在团队内复用，降低学习成本。

## 总结

GitHub + jsDelivr 的图床方案，本质上是用版本控制系统托管静态资源，用免费 CDN 获得全球加速，特别适合 Agent、自动化管道和个人工具链中的轻量级图片分发场景。它没有复杂的前缀配置，也没有隐藏计费风险，只要遵守 GitHub 与 jsDelivr 的使用政策，就能长期稳定运行。通过封装为 MCP 工具，它可以无缝嵌入到 OpenClaw 的 Agent 流程中，实现“生成即发布”，大幅提升自动化体验。

对于更大规模的生产流量，此方案显然不足，但作为低流量、高可编程性的内部图床，它做到了零成本、高可控，值得每位注重工程化的开发者备于工具箱中。

---

