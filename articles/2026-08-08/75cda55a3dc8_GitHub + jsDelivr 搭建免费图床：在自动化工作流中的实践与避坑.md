---
title: GitHub + jsDelivr 搭建免费图床：在自动化工作流中的实践与避坑
feedId: 32163
source: 综合讨论
publishedAt: 2026-08-08
---

## 背景：自动化流程中图片外链的刚需

在 OpenClaw 或 Agent 编排里，经常需要将生成的图表、截图、日志可视化结果持久化为可访问的链接，以便嵌入报告、发送到聊天工具、喂给下一个 MCP 工具。直接使用云服务的对象存储固然可靠，但对个人开发者、小规模自动化来说，成本、配置复杂度往往过高。免费、可控、能被 CDN 加速的方案于是进入视野。通过 GitHub 仓库托管静态资源并用 jsDelivr CDN 分发，是目前实践最多、门槛最低的组合之一。

本文将围绕“自动化生成图片 → 推送到 GitHub 仓库 → 通过 jsDelivr 产出永久链接”这一链路，给出可操作的搭建步骤、真实踩坑记录以及围绕 OpenClaw/Agent 场景的复用建议。

## 问题：为什么不用 raw 链接

GitHub 仓库中的文件本身可以通过 `raw.githubusercontent.com` 直接访问，但它存在几个工程上不可忽视的问题：

- **速率限制与稳定性**：raw 端点不适合高频、短时间内大量请求，容易触发限制或访问过慢，且不提供 CDN 级别的边缘缓存。
- **CORS 与 MIME 类型**：有时自动化流程中拉取图片会因 CORS 策略被拦截，而 jsDelivr 对静态资源返回合适的头信息。
- **缓存与版本控制**：raw 链接几乎没有可配置的缓存行为；jsDelivr 则允许通过版本号或分支名固化引用，缓解缓存更新不及时的问题。

因此，将 GitHub 作为存储后端、jsDelivr 作为分发层，是成本最低的“图床 CDN 化”方案。

## 做法：从零搭建可自动化调用的图床

### 1. 创建专门的 GitHub 仓库

建议新建一个 public 仓库（若存放隐私图片则必须 private，但 jsDelivr 仅能代理 public 仓库，下文会说明私有化替代）。仓库结构可按项目、日期或功能建目录，例如：

```
images/
  2025/
    report/
    screenshots/
```

将仓库克隆到本地或直接通过 API 操控。

### 2. 获取 jsDelivr 永久链接规则

对 public 仓库 `https://github.com/<user>/<repo>`，文件 `<path>` 在默认分支（如 `master`/`main`）下的 jsDelivr 链接为：

```
https://cdn.jsdelivr.net/gh/<user>/<repo>@<branch>/<path>
```

用版本 Tag 代替分支可固化引用：

```
https://cdn.jsdelivr.net/gh/<user>/<repo>@v1.0.0/logo.png
```

### 3. 自动化上传与链接生成

在 Agent 或脚本中完成“截图/生成 → 保存临时文件 → 通过 GitHub API 上传”这一流程。核心代码逻辑（伪代码）：

```python
import base64
import requests
from pathlib import Path

def upload_to_github(token, repo, file_path, commit_msg="auto upload"):
    url = f"https://api.github.com/repos/{repo}/contents/{file_path}"
    with open(file_path, "rb") as f:
        content_b64 = base64.b64encode(f.read()).decode()
    data = {
        "message": commit_msg,
        "content": content_b64,
        "branch": "main"  # 可根据实际调整
    }
    headers = {"Authorization": f"token {token}"}
    r = requests.put(url, json=data, headers=headers)
    if r.status_code == 201:
        return f"https://cdn.jsdelivr.net/gh/{repo}@main/{file_path}"
    else:
        raise Exception(f"Upload failed: {r.json()}")
```

将此函数封装为 MCP Server 工具，Agent 即可在流程中调用，返回可直接引用的 CDN 链接。同时可搭配 GitHub Actions，在 push 后自动进行图片优化。

### 4. GitHub Actions 辅助：压缩与自动发布

避免原图占用过多仓库空间，可以添加一个 Action，在每次 push 到指定目录时运行 `pngquant` 或 `imagemin`：

```yaml
name: Compress images
on:
  push:
    paths:
      - 'images/**'
jobs:
  compress:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Compress PNG
        uses: namelivia/pngquant-action@v2
        with:
          path: images
          ext: .png
          quality: 0.8
      - name: Commit changes
        run: |
          git config user.name github-actions
          git config user.email github-actions@github.com
          git add .
          git diff --quiet && git diff --staged --quiet || git commit -m "auto compress"
          git push
```

这使仓库保持干净，也减少 CDN 加载体积。

## 踩坑点与避坑指南

### 缓存更新滞后

jsDelivr 对 GitHub 文件的缓存时间较长，官方说明约一周甚至更久。如果覆盖同名文件，新内容不会立即生效。**解锁方式**：

- 每次上传使用含有时间戳或 hash 的文件名，如 `screenshot_20250409_0832.png`，规避缓存。
- 通过清除缓存 API 刷新：`https://purge.jsdelivr.net/gh/<user>/<repo>@<branch>/<path>`（仅限已缓存的版本，新版本可能需要等待）。
- 版本化：将图床仓库打 Tag，每次重大更新使用新 Tag，旧链接保留不变。自动脚本可使用 `@master`，但 Master 分支缓存刷新常延迟，适合非关键场合。

### 仓库容量与滥用限制

单个文件最大 50 MB，仓库建议小于 1 GB（GitHub 有软限制）。jsDelivr 虽无明确硬限，但大量超大文件或极高频访问可能被临时封禁。实测中，一个每天产生数百张小截图的自动化流程，通过压缩后完全可控。务必在 Agent 中加入文件大小检查与压缩步骤。

### 隐私与安全问题

**Public 仓库中所有图片均可被外部访问**。切勿上传包含 token、个人数据的截图。如有隐私需求，可使用 Private 仓库 + 自建反向代理，但这失去免费 CDN 特性；更务实的做法是在截图前自动脱敏（通过 OCR 检测敏感字段并像素化）。

### 频繁提交可能导致的仓库污染

自动化若以极高频（例如每秒一次）提交，会污染 commit 历史并浪费存储。建议引入缓冲队列，聚合一批图片后一次提交。

## 可复用建议：与 OpenClaw/Agent/MCP 结合

- **封装为 MCP 工具**：暴露 `upload_image_to_cdn(local_path)` 和 `purge_cdn_cache(url)` 两个函数，让 Agent 拥有图床读写能力。
- **结合 Playwright 截图链**：在 Agent 执行 Web 自动化时，截图后直接上传，返回可分享链接并附在任务报告中。
- **图片处理管道**：在 Actions 中加入水印、格式转换（webp）流程，可定制业务相关加工。
- **生成永久二维码/分享卡**：一些自动化需要产出带图标的卡片，可直接合成后 push 到仓库，返回 CDN 链接供对外使用。

## 总结

GitHub + jsDelivr 构建的“穷人的图床”绝非高可用方案，但在个人或小团队自动化场景下，其零成本、可 API 化、可定制流水线的特点，使它成为值得储备的工程技巧。建立时需特别注意缓存策略和隐私边界，略微多写几行代码即可化坑为路。将这套能力注入到 OpenClaw 的 Agent 工具集中，相当于为你的自动化流程免费配备了一个可编程、可加速的静态资源发布通道。

---

