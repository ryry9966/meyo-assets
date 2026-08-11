---
title: 免费图床 × 自动化工作流：用 GitHub + jsDelivr 搭一条图片上传到 CDN 的快捷通路
feedId: 32596
source: 综合讨论
publishedAt: 2026-08-11
---

## 背景

用 Markdown 做技术笔记、写文档或者让 Agent 生成技术博客时，图片引用是个绕不开的问题。本地相对路径无法跨平台分发，第三方图床要么收费，要么担心数据归属和长期稳定性。GitHub 仓库免费存储文件，jsDelivr 提供全球 CDN 加速，这套组合在静态博客圈子里已经是非常成熟的“零成本图床”方案。但对于习惯在 OpenClaw、MCP 这类自动化环境里串联工具的用户来说，只有把图床上传接进自己的工作流，它才算真正好用。

## 问题

每次想插入图片，都得手动打开 GitHub 网页上传、复制 raw 链接，再转成 jsDelivr 的 CDN 地址。效率低下，而且稍不注意就会引入空格、大小写之类的小问题。更关键的是，Agent 生成图表、截图之后，如果还要人去手动“传图”，自动化的链条就断了。

我们需要的是一条流水线：本地/Agent 产出图片 → 一键上传到 GitHub → 立刻返回一个可访问的 CDN URL。

## 做法：利用 GitHub API + jsDelivr 搭建图床

### 1. 创建公共仓库

jsDelivr 只能代理公开仓库的文件，所以需要建一个 **public** 仓库。仓库名随意，比如 `static-images`，专门存放图片资源。私有仓库走不了 jsDelivr，如果一定要保密，可以考虑 Cloudflare R2 之类的方案，这里不展开。

### 2. 获取 Personal Access Token

在 GitHub 的 `Settings > Developer settings > Personal access tokens > Fine-grained tokens` 裡创建一个新 token，只授予目标仓库的 **Read and Write** 权限（Contents 读写就够）。Token 仅上传时需要，用来调用 GitHub API。

### 3. URL 生成规则

把原始 GitHub raw URL（`https://raw.githubusercontent.com/用户名/仓库名/分支/文件路径`）换成 jsDelivr 的格式：

```
https://cdn.jsdelivr.net/gh/用户名/仓库名@分支/文件路径
```

比如 `@main` 或 `@master`。如果想用版本号控制缓存，可以打 tag 后改用 `@1.0.0`，但日常图片更新我们一般直接固定在主分支，通过改变文件名来解决缓存刷新（后面细说）。

### 4. 编写上传脚本

核心是调用 GitHub Contents API：

```
PUT /repos/{owner}/{repo}/contents/{path}
```

请求体包含：

- `message`：commit message，写清楚是哪张图片
- `content`：文件的 base64 编码
- `branch`：目标分支

我用 Python 写了一个最小实现，接收本地文件路径，自动生成 UUID 做文件名防止冲突，编码后上传并返回 jsDelivr 链接。脚本可以做成命令行工具，也可以直接封装成一个 MCP Server 的 tool，例如 `upload_image_to_cdn(local_path: str) -> str`，这样 OpenClaw 的 Agent 就能在生成图片后自动调用。

```python
import uuid
import base64
import requests

def upload_image(file_path: str, token: str, repo: str) -> str:
    with open(file_path, "rb") as f:
        data = f.read()
    content_b64 = base64.b64encode(data).decode()
    ext = file_path.split(".")[-1] if "." in file_path else "png"
    remote_path = f"images/{uuid.uuid4()}.{ext}"
    url = f"https://api.github.com/repos/{repo}/contents/{remote_path}"
    headers = {"Authorization": f"token {token}"}
    payload = {
        "message": f"Upload {remote_path}",
        "content": content_b64,
        "branch": "main"
    }
    r = requests.put(url, json=payload, headers=headers)
    if r.status_code in [201, 200]:
        # 拼接 jsdelivr 链接
        owner, repo_name = repo.split("/")
        return f"https://cdn.jsdelivr.net/gh/{owner}/{repo_name}@main/{remote_path}"
    else:
        raise Exception(f"Upload failed: {r.text}")
```

### 5. 接入自动化工作流

- **手动使用**：可以把脚本绑到 VSCode 的 Task 或 Shell 别名，截图后一键上传。
- **Agent 集成**：在 MCP Server 配置中注册该 tool，这样 OpenClaw 的 agent 在需要发布图片时直接调用，获得 CDN 地址后插入 Markdown。
- **批量/监控**：配合 inotify 或文件夹监控，自动上传新增图片；再配合 GitHub Actions 进行缩略图生成或压缩，但日常使用脚本已经足够。

## 踩坑点

**缓存刷新是最大的坑。**  
jsDelivr 对同一 URL 会缓存很长时间，即使你替换了文件内容，旧版本可能 24 小时甚至更久都不更新。官方建议使用版本号来强制刷新，但每张图片打 tag 太重。

我的做法是 **用文件内容哈希做文件名**（例如 `md5(content).png`），这样一旦图片变化，就会生成全新的 URL，天然避免缓存失效问题。缺点是文件没法原地更新了，不过图片图床通常不需要覆盖旧文件，这完全可接受。

如果一定要原地更新，可以用 Purge API 请求 `https://purge.jsdelivr.net/gh/...`，但不一定实时生效，而且限额较严格。

**仓库公开但不想暴露其他文件。**  
可以把图片集中放在一个目录，仓库 README 里写清楚这是图片仓库，不要放敏感数据。别人即使访问整个仓库也没关系。

**GitHub 仓库有 1GB 推荐上限。**  
个人图床很难超过，定时清理一下无用的图片即可。脚本里加个前端日期前缀也方便日后扫除。

**文件名里的特殊字符。**  
jsDelivr 对空格、中文支持不稳定，应当只用字母、数字、连字符和下划线。UUID 解决了这个问题。

## 可复用建议

1. **Token 用环境变量，不要硬编码**。在 OpenClaw 的环境配置里设为 `GITHUB_TOKEN`，脚本直接读取。
2. **把上传逻辑做成最小化的 MCP tool**，一个 `upload_image` 函数足够，遵循 MCP JSON-RPC 协议，其他 Agent 也能复用。
3. **保留原始图片备份**，虽然 GitHub 很难丢数据，但重要截图还是建议本地留底。
4. **在 Markdown 中使用相对简洁的 CDN 链接**，避免把 `@main` 写死，如果以后想切到 Release 版本，也方便全局替换。
5. **控制仓库垃圾**：可以用一个 GitHub Actions workflow，每晚删除 30 天前未引用的图片。

## 总结

GitHub 仓库加 jsDelivr 的图床方案成本为零，搭配简单的 API 上传脚本，就能完全融入自动化工作流。对于个人技术笔记、Agent 自动产出的图片，这一套已经足够鲁棒和方便。它不适合高并发生产环境，但在小规模、自用的场景里，是一个工程上非常清爽的解法。

---

