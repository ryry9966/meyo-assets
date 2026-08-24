---
title: GitHub + jsDelivr 搭建免费图床：给 Agent 工作流一个可引用的图片层
feedId: 34600
source: 综合讨论
publishedAt: 2026-08-25
---

## 背景

在 OpenClaw/Agent 自动化场景里，经常会出现截图、架构图、二维码、插件预览图等产物。图片如果只存在本地，就无法通过聊天、API 或 Markdown 报告直接引用；如果临时传到某些公共图床，链接存活、鉴权、限流和内容审查都不可控。

GitHub 公开仓库 + jsDelivr CDN 是一套低依赖方案：免费、可直链、可版本管理、可以通过 API 完全自动化。它不适合当生产对象存储，但很适合个人项目、内部工具、Agent 输出预览等小规模场景。

## 问题

直接使用 GitHub 的 `raw.githubusercontent.com` 链接，部分网络环境访问不稳定，而且它本质不是 CDN。jsDelivr 可以代理 GitHub 公开仓库，提供更稳定的 CDN 地址。

但这个方案也有边界：

- 仓库必须公开，否则无法通过 jsDelivr 稳定访问；
- 单文件大小有限制，大图不适合；
- CDN 缓存可能导致同名更新后仍返回旧内容；
- 公开意味着任何人可见，隐私图片必须脱敏。

因此更合理的做法不是把它当成“无限相册”，而是封装成一个上传和引用工具，让 Agent 在需要时自动生成可访问图片链接。

## 做法/步骤

### 1. 建一个公开图片仓库

创建一个 public 仓库，比如 `assets` 或 `images`。目录建议按项目或时间拆分：

```text
/2025/12/
/plugin-preview/
/qrcode/
```

根目录放一个简短 README，说明上传规则、命名规范和可接受的文件类型。

### 2. 使用最小权限 Token

在 GitHub 新建 Fine-grained token，只授予该图片仓库的 Contents 读写权限。不要把 token 给到全局仓库权限，也不要长期放在 Agent 对话里。

### 3. 通过 GitHub Contents API 上传

不建议每次上传都 clone 整个仓库，尤其是图片仓库越来越大时。使用 GitHub Contents API 的 `PUT` 操作即可：

```http
PUT /repos/{owner}/{repo}/contents/{path}
```

请求体示例：

```json
{
  "message": "upload image",
  "content": "<base64 content>",
  "branch": "main"
}
```

文件名建议加短内容哈希，例如：

```text
plugin-arch-3f2a9c1b.png
```

这样可以避免同名文件互相覆盖，也能天然解决部分缓存问题。

### 4. 生成 jsDelivr CDN 链接

上传成功后，按照以下规则生成链接：

```text
https://cdn.jsdelivr.net/gh/{owner}/{repo}@main/{path}
```

例如：

```text
https://cdn.jsdelivr.net/gh/user/assets@main/2025/12/plugin-arch-3f2a9c1b.png
```

路径中不要使用空格、中文或特殊字符。建议统一为小写字母、数字、连字符和下划线。

### 5. 封装成脚本或 MCP 工具

可以写一个简单上传函数，输入文件路径，返回 Markdown 片段：

```python
def upload_image(path, owner, repo, token):
    with open(path, "rb") as f:
        b64 = base64.b64encode(f.read()).decode()
    dest = build_dest_with_hash(path)
    # 调用 GitHub Contents API 上传
    # 返回 jsDelivr URL
    return f"![image](https://cdn.jsdelivr.net/gh/{owner}/{repo}@main/{dest})"
```

如果使用 MCP，可以暴露一个 `upload_image` 工具。OpenClaw Agent 在生成图片后可以直接调用该工具，把本地图片转成 CDN URL，并将 Markdown 链接插入最终回复中。

### 6. 处理缓存更新

jsDelivr 会在首次访问后缓存文件。如果你覆盖了同名文件，旧内容可能还会存在一段时间。可以通过以下地址发起 purge：

```text
https://purge.jsdelivr.net/gh/{owner}/{repo}@main/{path}
```

但在自动化流程里，更推荐文件名携带内容哈希，即“不可变上传”。这样每次内容变化都会得到新 URL，不依赖缓存刷新。

## 踩坑点

- **文件大小**：GitHub 单文件限制 100MB，但 jsDelivr 代理大文件体验很差。图床图片建议压缩到 1–2MB 以内，PNG 可转 WebP。
- **公开性**：公开仓库里的图片任何人都能访问。截图、二维码、架构图必须提前脱敏，不要包含内部 IP、凭证、用户隐私或未公开项目信息。
- **缓存延迟**：同名覆盖会带来旧内容问题，优先使用内容哈希命名。
- **GitHub API 限流**：认证请求 5000 次/小时，未认证 60 次/小时。批量上传要控制频率，必要时加重试与排队。
- **不要当网盘用**：视频、大文件、高频热更新不适合走这套方案，容易被限流，也偏离图床用途。
- **不要用 raw 链接替代 CDN 链接**：`raw.githubusercontent.com` 在部分网络环境不可控，且不是 CDN。

## 可复用建议

- 做一个统一上传工具，任何 MCP、插件、CI 都调用同一个入口，输出统一格式的 Markdown。
- 目录规划使用 `{project}/{date}-{slug}-{hash8}.{ext}`，便于按项目清理。
- 在仓库中维护一个 `manifest.json`，记录图片 URL、来源、生成时间、生成 prompt，方便后续回溯。
- 对重要图片保留本地原图或额外备份，不要把 GitHub 仓库当作唯一存储。
- 在 OpenClaw 场景中，把上传函数作为工具注入，Agent 生成图片后自动转成 CDN URL，避免用户手动下载再上传。

## 总结

GitHub + jsDelivr 搭建免费图床，优势是零成本、可编程、可版本管理，适合 OpenClaw/Agent 工作流中的小文件图片引用。公开、缓存和大小限制是必须接受的硬条件。把它封装成统一上传工具、使用内容哈希文件名、做好权限最小化与脱敏，就能在较少复杂度下稳定运行。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/913beb2bb1b91090.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/fade26205f10f770.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/9db5f3ad8d18edf6.png)

