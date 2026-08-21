---
title: 图片 CDN 选型：用 GitHub + jsDelivr 给 Agent 搭一个可编程免费图床
feedId: 33973
source: 综合讨论
publishedAt: 2026-08-21
---

## 背景

在 OpenClaw/Agent/MCP/插件这类自动化场景里，图片产出越来越多：巡检截图、插件生成的图表、Agent 任务中的配图、对话卡片等。这些图片需要一个可被外部访问的 URL，才能继续被模型、聊天窗口或下游系统引用。

如果小规模项目一上来就接对象存储，要维护 Bucket、访问密钥、生命周期、跨域和计费，对内部工具或 Side Project 来说偏重。GitHub 仓库 + jsDelivr 是一个更轻的选择：用 GitHub 仓库当存储，用 jsDelivr 当 CDN，上传过程可以通过 GitHub API 完全自动化。

## 问题与边界

直接使用 `raw.githubusercontent.com` 有两个问题：部分网络环境下访问不稳定，且没有 CDN 缓存；jsDelivr 可以对 GitHub 仓库内容做免费 CDN 加速，并在多数地区比 raw 更快、更稳定。

但选型前要明确：这不是高可用生产图床，不适合大文件、高并发、强 SLA、私密图片。它更适合内部工具、Agent 工作流、演示项目、低流量插件等场景。

## 做法 / 步骤

1. **建仓**  
   创建一个 public 仓库，例如 `assets`。建议按年月或模块建目录，例如 `2026/01/` 或 `agent-output/`。

2. **生成最小权限 Token**  
   GitHub Settings → Developer settings → Personal access tokens。建议使用 fine-grained token，只授予指定仓库的 `Contents: write` 权限。不要给整个账号的 `repo` 全量权限。

3. **用 API 上传图片**  
   通过 GitHub Contents API 提交文件。下面是一个最小可用的上传函数：

```python
import base64
import os
import hashlib
import requests

def upload_image(img_bytes, remote_dir, repo, branch="main"):
    token = os.environ["GH_TOKEN"]
    name = hashlib.sha1(img_bytes).hexdigest()[:12] + ".png"
    path = f"{remote_dir}/{name}"

    url = f"https://api.github.com/repos/{repo}/contents/{path}"
    headers = {"Authorization": f"Bearer {token}"}
    payload = {
        "message": f"upload {path}",
        "content": base64.b64encode(img_bytes).decode(),
        "branch": branch,
    }

    r = requests.put(url, json=payload, headers=headers)
    r.raise_for_status()

    cdn = f"https://cdn.jsdelivr.net/gh/{repo}@{branch}/{path}"
    raw = f"https://raw.githubusercontent.com/{repo}/{branch}/{path}"
    return {"cdn": cdn, "raw": raw, "path": path}
```

4. **拼接 jsDelivr URL**  
   固定格式为：

```text
https://cdn.jsdelivr.net/gh/{owner}/{repo}@{branch}/{path}
```

日常调试可以用 `@main`；如果需要版本稳定，建议用 release tag 或 commit hash，例如 `@v1.0.0`。

5. **接入 OpenClaw/MCP/插件**  
   把上传函数封装成 MCP tool，例如 `upload_image(local_path, remote_dir) -> url`。Agent 生成图片后调用该工具，拿到 CDN URL 再回复用户或写入消息卡片。

## 踩坑点

- **同名路径覆盖会报 422**  
  GitHub Contents API 的 PUT 请求在文件已存在时需要提供 `sha`，否则会返回 422。最简单的方法是用内容哈希作为文件名，既避免覆盖，又天然利于缓存更新。

- **jsDelivr 缓存更新不及时**  
  覆盖已有文件时，CDN 缓存可能最长保留约 12 小时。如果图片需要频繁更新，建议文件名带内容哈希，或者使用新的 tag/commit 作为版本。

- **国内访问稳定性**  
  jsDelivr 在国内部分网络环境下仍可能出现 DNS 或 HTTPS 抖动。不要把 CDN URL 当成唯一返回值，建议同时返回 raw URL 作为 fallback。

- **仓库公开意味着图片公开**  
  public 仓库里的所有文件都可以被访问。不要传敏感信息，不要把它当私有存储。

- **仓库与单文件限制**  
  GitHub 建议仓库保持轻量，单文件 API 限制为 100MB。图片建议压缩到 5MB 以下，视频不建议放。

- **Token 泄漏风险**  
  PAT 不要硬编码到插件或前端。运行时从环境变量或 secret 管理读取；最小权限只给指定仓库写权限。

## 可复用建议

- 文件名一律用内容哈希，避免覆盖和缓存混乱。
- URL 生成逻辑收敛到一个函数，不要到处手拼。
- 可以封装成一个标准 MCP tool，参数只暴露 `local_path`、`remote_dir`、`branch/tag`。
- 返回值同时提供 `cdn_url`、`raw_url`、`path`，供上层做 fallback。
- 对关键图片做简单可用性监控，例如 cron 定时请求一张测试图片。
- 如果后续量级变大，迁移到对象存储时，GitHub 仓库里的元数据和目录结构可以保留作为索引。

## 总结

GitHub + jsDelivr 是适合 Agent 工作流的轻量图片外链方案：零成本、可编程、有版本控制，配合 MCP tool 可以显著降低自动化场景里的图片交付成本。

但它也有明显边界：公开、CDN 稳定性一般、不适合大文件和高频生产调用。务实的用法是把它作为内部工具、演示或低流量插件的图床；真正面向用户的核心业务，仍然建议使用对象存储 + 自有 CDN。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/43f06d423ac0e6ab.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/47b4ae692aef1427.png)

