---
title: 图片 CDN 选型：GitHub + jsDelivr 搭建免费图床的实践
feedId: 34557
source: 综合讨论
publishedAt: 2026-08-24
---

# 图片 CDN 选型：GitHub + jsDelivr 搭建免费图床的实践

## 背景：自动化流程里需要一个可编程图床

在 OpenClaw、Agent、MCP 以及插件自动化场景里，图片经常作为中间产物或最终交付物出现：截图留证、告警卡片、渲染预览、巡检报告、插件素材。Agent 生成图片后，往往需要返回一个公网可访问的 URL，供通知、日志、前端或下游系统使用。

商业图床 API 会引入额外依赖、配额和隐私约束；自建对象存储或服务器对个人项目来说偏重。相比之下，GitHub 公开仓库 + jsDelivr CDN 是一个成本低、可 Git 审计、便于脚本集成的折中方案。它不解决所有问题，但很适合非敏感图片的分发。

## 问题：GitHub raw 不够用

直接使用 `raw.githubusercontent.com` 有几个明显问题：

- 它不是 CDN，跨区域读取体验不稳定；
- 国内访问可能抖动；
- 响应头、缓存策略并不适合频繁读取；
- 对自动化流程来说，缺少稳定的静态资源分发能力。

jsDelivr 可以把 GitHub 公开仓库文件分发到 CDN 节点，改善访问速度和可用性。但它也带来缓存、版本、文件大小、滥用限制等新问题。选型前要先确认：图片非敏感、体积中等、可以接受公开访问。

## 做法步骤

### 1. 建仓

创建一个 public 仓库，例如 `assets`。目录建议按 `{project}/{yyyyMM}/` 组织，避免单目录堆积过多文件。

### 2. 上传图片

手动使用 git push 可以，但自动化流程建议走 GitHub Contents API。这样 Agent 或插件可以在运行时直接上传二进制内容，无需预先落盘到 git 工作区。

一个最小可用的上传函数示意：

```python
import base64
import requests
import os

def upload_image(repo, path, content_bytes, token):
    url = f"https://api.github.com/repos/{repo}/contents/{path}"
    body = {
        "message": f"upload {path}",
        "content": base64.b64encode(content_bytes).decode(),
    }
    headers = {
        "Authorization": f"Bearer {token}",
        "Accept": "application/vnd.github+json",
    }
    r = requests.put(url, json=body, headers=headers, timeout=30)
    r.raise_for_status()
    return r.json()["content"]["sha"]
```

建议使用 fine-grained token，并只授予单个仓库的 Contents 读写权限。

### 3. 拼接 jsDelivr URL

URL 规则如下：

```text
https://cdn.jsdelivr.net/gh/{owner}/{repo}@{commit_or_tag}/{path}
```

长期使用的 URL 应尽量固定 commit SHA 或 tag，不要依赖 `@latest`。因为 `@latest` 在覆盖更新时存在缓存语义不一致的问题，不利于自动化系统做稳定性判断。

### 4. 接入 Agent/MCP

把上传逻辑封装成 MCP tool 或 OpenClaw 插件命令。输入可以是图片路径或二进制内容，内部完成压缩、哈希命名、上传、URL 拼接，最后返回可直接用于 Markdown 或通知的链接。Agent 只需要调用工具，不需要关心 git 和 CDN 细节。

## 踩坑点

### 同名覆盖与缓存

如果对同一路径重复上传，GitHub 会生成新 commit，但 jsDelivr 上的旧 URL 缓存可能继续返回旧内容。虽然可以使用 `purge.jsdelivr.net` 刷新缓存，但频繁调用并不友好。更稳妥的做法是文件名带内容哈希，从设计上避免覆盖更新。

### 公开性与删除不彻底

仓库必须 public 才能被 jsDelivr 读取。公开仓库加上 CDN 缓存意味着删除或修改文件后，旧内容可能仍可被访问一段时间。因此敏感图片不要走这个方案。

### 文件大小

GitHub 仓库单文件有 100MB 上限，但大文件会拖慢仓库、增加 API 负担。jsDelivr 对超大文件的支持也不理想。上传前应做压缩，建议单张控制在小几 MB 以内。

### Token 与限流

不要把 token 写进代码、日志或 prompt 可见上下文。批量上传要做限速和重试，避免触发 GitHub 的 secondary rate limit。自动化流程里建议使用环境变量或密钥管理系统注入 token。

### 国内可用性波动

jsDelivr 在国内并非始终稳定。可以维护一个 fallback base URL 列表，失败时回退到 raw 或其他可达镜像。但不要依赖非官方代理处理敏感数据。

## 可复用建议

- **命名规范**：`{project}/{yyyyMM}/{sha1前8位}.png`，语义清晰且避免冲突。
- **版本固定**：URL 中带 tag 或 commit SHA，不做无版本依赖。
- **统一封装**：实现一个 `upload_image` 工具，内部处理压缩、哈希、重试、URL 返回。OpenClaw/Agent 只看到工具能力。
- **批量任务**：使用队列加低并发，做失败重试和去重，避免一次性大量请求。
- **容量治理**：定期清理无用旧图，按项目拆库，避免单仓库历史膨胀。
- **监控与 fallback**：记录上传失败和 URL 访问失败，遇到 CDN 不可用可切换 base URL。

## 总结

GitHub + jsDelivr 免费图床适合自动化流程中的非敏感图片分发：预览图、截图留证、告警卡片、插件素材等。它的优点是零服务器成本、可版本化、API 易于集成；缺点是公开性、缓存策略、文件大小限制和国内可达性存在边界。

工程化的关键不是“能传上去”，而是通过 hash 命名、固定版本、API 封装和 fallback 策略，把不确定性关进工具层。这样 Agent 和 OpenClaw 插件侧只需要面对一个稳定的图片上传接口。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/a5687d53e2f37f03.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/7190b6ff5e2ff78c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/4ac194a438f41b79.png)

