---
title: 图片 CDN 选型：用 GitHub + jsDelivr 搭建面向自动化的免费图床
feedId: 32222
source: 综合讨论
publishedAt: 2026-08-09
---

## 背景

在 OpenClaw 生态里做 Agent、MCP 或插件开发时，经常需要把程序生成的图片变成可公网访问的 URL。比如：截图上报、知识卡片输出、监控看板快照、自动化日报配图等。传统的免费图床要么限制 API 调用，要么去重策略难受，自建 OSS 又增加了运维成本和计费担忧。

一次「刚好够用」的选型是：把 GitHub 仓库当作图床存储，用 jsDelivr 作为免费 CDN 加速层。这个组合天然可编程、可版本化，且对个人及小团队项目几乎是零成本。

## 问题分析

直接使用 GitHub Raw 链接（`raw.githubusercontent.com`）有几个痛点：
- 国内访问时断时续，境外也谈不上快速；
- 没有 CDN 缓存策略，大量请求对仓库不友好；
- 不支持 URL 版本切换，发布后更新图片可能造成缓存混乱。

jsDelivr 作为全球 CDN，能代理 GitHub 仓库文件并提供稳定加速，同时保留版本化能力。对自动化场景来说，只需学会一套上传 + 拼接规则，就能让 Agent 稳定产出可访问的图片链接。

## 实践做法

### 1. 创建图床仓库
新建公开 GitHub 仓库，例如 `assets-cdn`。目录可以按项目或日期组织，例如 `images/2025/`。建议不要放代码混合，保持纯图床用途。

### 2. 上传图片（可编程方式）
在 Agent 或自动化脚本中，使用 GitHub REST API 上传文件。核心请求：

```python
import requests, base64, os

GITHUB_TOKEN = os.getenv("GITHUB_TOKEN")
REPO = "owner/assets-cdn"
BRANCH = "main"
file_path = "images/demo.png"
image_bytes = open("demo.png", "rb").read()

content_b64 = base64.b64encode(image_bytes).decode()
url = f"https://api.github.com/repos/{REPO}/contents/{file_path}"
headers = {"Authorization": f"Bearer {GITHUB_TOKEN}"}
payload = {
    "message": "upload image via api",
    "content": content_b64,
    "branch": BRANCH
}
resp = requests.put(url, json=payload, headers=headers)
resp.raise_for_status()
commit_sha = resp.json()["content"]["sha"]  # 后续更新若需要可保留
```

注意事项：
- Token 需要 `repo` 权限，建议通过环境变量注入；
- 同一路径再次写入必须提供上次文件的 `sha`，否则会返回 422，这点在更新图片时要特别处理。

### 3. 构造 jsDelivr CDN 链接
上传成功后，按以下格式拼接永久 URL：

```
https://cdn.jsdelivr.net/gh/{owner}/{repo}@{branch}/{file_path}
```

例如：`https://cdn.jsdelivr.net/gh/owner/assets-cdn@main/images/demo.png`

若希望链接永远不变且不需要手动刷新缓存，可以用 commit hash 代替分支名，这样内容可永久缓存，但更新时需要替换链接。

### 4. 自动化刷新缓存
如果使用分支名且更新了图片，jsDelivr 会缓存在 12 小时左右过期。如需立即生效，可以调用 purge API：

```
GET https://purge.jsdelivr.net/gh/{owner}/{repo}@{branch}/{file_path}
```

注意 purge 存在频率限制，不要在循环中乱刷。更适合的方式是采用内容哈希命名文件（如 `abc123.png`），每次更新生成新文件名，从根本上避免缓存问题。

### 5. 与 OpenClaw 工作流集成
可封装成一个上传工具，挂接到 Agent 的知识卡输出节点。例如：Agent 用 Pillow 生成卡片图片后，直接调用该工具上传并返回 `cdn_url`，再嵌入到最终推送的 Markdown 或消息中。这种方法将图片生成与分发解耦，让自动化更干净。

## 踩坑点

- **文件大小限制**：GitHub 单文件最大 100 MB，但 jsDelivr 对超大文件优化有限，建议普通图片压缩在 2 MB 以内，文件总数控制好，仓库总容量建议别超过 1 GB。
- **缓存策略混乱**：直接用分支名且频繁覆盖图片，会因 CDN 缓存不一致产生「更新了但看到的还是旧图」的问题。最佳实践是文件名带内容哈希，永远不覆盖同一路径。
- **API 速率限制**：GitHub API 匿名每小时 60 次，认证后 5000 次。对于中低频自动化足够，但如果高频上传，需要做本地缓冲合并。
- **国内可达性**：jsDelivr 在中国大陆有节点，但个别地区可能被干扰。作为内部工具图床够用，若面向 C 端大规模服务需谨慎评估。

## 可复用建议

1. **统一图床模块**：把上传、命名、CDN 拼接、purge 封装成单一 Python/JS 模块，所有 Agent 共用，降低重复工作。
2. **自动压缩**：上传前用 Pillow 或 sharp 做尺寸裁剪和质量压缩，减小体积也节省流量。
3. **版本路径规划**：`images/{hash[:2]}/{hash}.png` 这种结构可以分散目录，避免单目录文件过多。
4. **CI 同步**：如果图片是本地生成后批量上传，可在 GitHub Actions 里用 `git` 推送，而不是每次通过 API，适用于定时任务。
5. **配合 Git LFS**：若有较大二进制文件（如动画 GIF），启用 Git LFS 避免仓库急速膨胀，但需确认 jsDelivr 对 LFS 文件同样生效。

## 总结

GitHub + jsDelivr 的组合，为 OpenClaw 社区里的 Agent、MCP 以及各类自动化插件提供了一套低成本、可编程的图片托管手段。它不需要管理服务器，不需要付费，配合良好的文件命名与缓存策略，可以支撑中小规模自动化场景的图片分发。对于追求工程效率且希望聚焦在自动化逻辑的开发老手而言，这个方案简单却扎实。

---

