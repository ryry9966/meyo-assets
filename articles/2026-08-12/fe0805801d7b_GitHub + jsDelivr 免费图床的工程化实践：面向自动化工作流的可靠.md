---
title: GitHub + jsDelivr 免费图床的工程化实践：面向自动化工作流的可靠选型
feedId: 32677
source: 综合讨论
publishedAt: 2026-08-12
---

## 背景：为什么还在折腾免费图床

OpenClaw、Agent、MCP 插件这类工具链中，图片生成或截图存储是一个高频需求。当我们用 Playwright 截图、用 matplotlib 画图、或用 DALL·E 生成结果时，总需要把二进制对象持久化成一个可公网访问的 URL，才能传递给下游 Agent 或消息通道。

公有云的对象存储（S3/R2/OSS）是最直接的答案，但个人项目或实验阶段，为几十张图养一个 bucket 略显笨重。于是不少自动化工作流开始转向 **GitHub + jsDelivr** 这条组合链路：用 Git 仓库当存储，用 jsDelivr 当 CDN，零费用、免运维，而且完全可以通过 GitHub API 或 Actions 编程控制。经历三次迭代后，我为自己的 Agent 截图管线形成了以下稳定实践。

## 问题：不是所有免费链路都适合自动化

尝试初期踩了几个坑：

1. **PicGo 等桌面工具不贴合 Agent 环境**：它们依赖 GUI 和手动热键，自动化脚本没法直接调用。  
2. **直接 push 到 master 分支后再用 jsDelivr，缓存周期不可控**：jsDelivr 对 `https://cdn.jsdelivr.net/gh/user/repo@master/file.png` 有 12 小时强制缓存，更新图片需要等半天才能生效。  
3. **仓库体积膨胀**：无脑上传原图很快把仓库推到 1GB 限制边缘，触发 GitHub 的警告邮件，甚至被 jsDelivr 限制访问。  
4. **Git LFS 幻想破灭**：jsDelivr 无法代理 LFS 对象，最终得到的只是一个指针文件，CDN 返回的是文本而不是图片。

目标是：**轻量、可编程、缓存可控、不怕把仓库撑爆**。

## 做法与步骤

### 1. 仓库设计：分支策略 + 体积控制

新建一个公开仓库 `static-bed`，并采用 **release 分支 + tag 版本化** 的方式组织文件。日常自动化上传固定走 `main` 分支用于快速测试；生产引用时一律使用 commit hash 或 tag 对应的 jsDelivr 链接，格式如下：

```
https://cdn.jsdelivr.net/gh/user/static-bed@<commit-hash>/path/file.png
```

优势：commit 不可变，CDN 认为是新资源立即回源，无缓存等待。同时不同版本间互不干扰。

控制体积的关键是**在 Agent 侧做预处理**：截图后转换成 WebP（质量 80 通常视觉无损），限制最长边 1200px，得到的图片多在 30–100KB 之间。这样即使每天生成几百张，仓库也能撑很久。

### 2. 上传：一个可嵌入 Agent 的 Python 脚本

不依赖任何 GUI，直接调用 GitHub Contents API。核心函数：

```python
import requests
import base64

def upload_to_github(token, owner, repo, path, content_bytes, branch="main"):
    url = f"https://api.github.com/repos/{owner}/{repo}/contents/{path}"
    headers = {"Authorization": f"token {token}"}
    data = {
        "message": f"upload {path}",
        "content": base64.b64encode(content_bytes).decode("ascii"),
        "branch": branch,
    }
    resp = requests.put(url, json=data, headers=headers)
    resp.raise_for_status()
    commit_sha = resp.json()["content"]["sha"]
    return commit_sha
```

调用后拿到此次提交的 commit hash，拼接 jsDelivr URL：

```python
def get_jsdelivr_url(owner, repo, commit_sha, file_path):
    return f"https://cdn.jsdelivr.net/gh/{owner}/{repo}@{commit_sha}/{file_path}"
```

在 OpenClaw 或自定义 Agent 中，只要提供 `GITHUB_TOKEN` 环境变量，截图生成后就能立刻得到一个稳定的 CDN 地址。

### 3. 用 GitHub Actions 做兜底清理（可选）

为防止某天不小心上传了大文件，可以创建一个每周运行的 Action，扫描仓库中大文件（`git rev-list --objects --all | git cat-file ...`），超过 500KB 的文件发 issue 提醒或自动压缩替换。这不是必选项，但对长期使用很有益处。

## 踩坑点复盘

**坑一：jsDelivr 的 aggressive caching**  
用 `@master` 或 `@main` 引用时，CDN 会无视较新的 push 长达 12 小时。解决方法是上传后立即获取 commit SHA，用 `@<sha>` 代替分支名。jsDelivr 对 commit 资源的缓存策略更宽松，几乎是即时生效。

**坑二：GitHub API 的速率限制**  
认证后每小时 5000 次请求，对自动化来说绰绰有余。但未认证请求仅有 60 次/小时，在一次调试脚本中容易耗尽。务必始终携带 token。

**坑三：文件名冲突**  
如果 Agent 多个实例同时向同一个路径 `screenshot.png` 上传，会出现竞态条件，API 会要求提供 `sha` 参数。推荐生成带时间戳或 UUID 的文件名，如 `2025-04-17/ab12cd.png`，既能规避冲突，也便于后续清理。

**坑四：仓库被 GitHub 标记**  
偶尔收到 “仓库体积过大” 的提醒，只要及时处理（交互式 rebase 从历史中删除大文件，或直接创建新仓库迁移），不影响 CDN 可用性。但这是信号，提醒要持续做图片转码和尺寸限制。

## 可复用建议

- **为 Agent 封装一个轻量 Uploader 工具**：输入 `bytes` 加 `extension`，返回 CDN URL。可以直接做成一个 MCP 工具，挂载到 OpenClaw 的工具列表中。  
- **生产环境用 tag 而非 commit**：当需要长期引用且稳定不变时，上传到某个 release tag（如 `v1-snapshots`），可以用 `@v1-snapshots` 的 jsDelivr 链接，便于语义化管理。  
- **设置仓库的 `description` 明确用途**，并在 README 添加 “仅用于 CDN 加速，不存储高价值资源” 的声明，减少被误判为滥用的风险。  
- **监控使用量**：虽然 jsDelivr 无用量统计面板，但可以通过 Cloudflare Analytics 私有域名前置做一层代理，但会增加复杂度。个人使用不必强求。

## 总结

GitHub + jsDelivr 这条链路远非完美，它的天花板是仓库体积和 GitHub 的风控策略。但如果甲方是“个人自动化管线里的百 KB 级截图存储”，它几乎是无敌的：零成本、API 优先、全球加速、版本天然可追溯。我现在的 OpenClaw 截图 Task 结束时会自动吐出 `@commit_sha` 的 jsDelivr 链接，下游 Discord 通知、日报汇总直接引用，从未掉过链子。

当你的 Agent 需要一个不用养 bucket、不用配 CORS、不用管 SSL 的图片 URL 时，这个组合值得放进工具箱里。仓库里躺着的全是 50KB 的 WebP，安静、可控、刚刚好。

---

