---
title: 把 GitHub 仓库当图床：jsDelivr 免费 CDN 的自动化实践与坑
feedId: 35123
source: 综合讨论
publishedAt: 2026-08-29
---

## 背景

在 OpenClaw / Agent / MCP 这类自动化工作流里，经常会出现一种“轻图片需求”：Agent 生成一张异常截图、一段运行监控图、一张流程示意图，然后要把它插入 Markdown 报告或通知里。这时需要的不是高级图床，而是一个能通过脚本稳定上传、返回确定 URL、并且能直接嵌入 Markdown 的存储方案。

对象存储要开通、配密钥、算费用；自建 Nginx 要维护服务；公共图床不适合自动化调用。GitHub 公开仓库 + jsDelivr CDN 是成本很低、协议成熟的过渡方案，尤其适合内部脚本、个人自动化报告、非敏感图片。

## 问题

这个方案的核心不在“免费”，而在“是否适合自动化”。GitHub API 成熟，jsDelivr 提供固定格式的 CDN URL，能直接拼出可访问的图片地址。缺点也明确：仓库必须公开、缓存不可控、国内访问不完全稳定、单文件不能太大。只要把这些限制纳入工程判断，它就能很好地服务轻量场景。

## 做法与步骤

### 1. 建公开仓库

创建一个 public 仓库，例如 `assets-bucket`。注意：jsDelivr 只能访问公开仓库，私有仓库无法通过 jsDelivr 暴露。

### 2. 上传图片

两种常见方式：

- 本地用 `git add` + `git push` 提交图片。
- 自动化脚本调用 GitHub Contents API：

```http
PUT /repos/{owner}/{repo}/contents/{path}
```

请求体里写 base64 文件内容、commit message。脚本可以用 Python、Node 或现成 MCP 工具封装。

### 3. 拼 CDN URL

上传后可以直接拼出 jsDelivr 地址：

```text
https://cdn.jsdelivr.net/gh/{owner}/{repo}@{branch}/{path}
```

例如：

```text
https://cdn.jsdelivr.net/gh/user/assets-bucket@main/2025/01/a1b2c3.webp
```

### 4. 封装成 MCP 工具

对 Agent 来说，最好封装成一个 `upload_image` 工具：

- 输入：本地图片路径或 base64
- 处理：压缩、转格式、按内容 hash 命名
- 输出：jsDelivr URL、Markdown 片段、可能的 purge URL

这样 Agent 生成图片后，一次工具调用即可得到可嵌入报告的链接。

### 5. 在 OpenClaw 流程中使用

Agent 处理截图或图表后，调用 `upload_image`，再写入报告：

```markdown
![异常截图](https://cdn.jsdelivr.net/gh/user/assets-bucket@main/2025/01/a1b2c3.webp)
```

如果希望 URL 与内容强一致，可以用 `@commit_hash` 代替 `@main`，避免后续提交覆盖造成混乱。

## 踩坑点

### 1. 仓库必须公开

私有仓库无法走 jsDelivr，这是最常见的误解。不要把任何私密图片放进这个流程，哪怕仓库先公开再转私有也不行。

### 2. 缓存不实时

jsDelivr 对同一 URL 有较长的缓存。覆盖同名文件后，旧图可能还会继续显示一段时间。解决方式有两种：

- 用内容 hash 做文件名，避免覆盖；
- 调 purge 接口强制刷新：

```text
https://purge.jsdelivr.net/gh/{owner}/{repo}@{ref}/{path}
```

工程上更推荐 hash 命名，刷新接口只作为应急手段。

### 3. 文件名和编码

路径里避免中文、空格、`#`、`?` 等字符。统一小写、短横线或下划线。否则在 URL 编码、Markdown 链接、shell 参数传递中都容易出问题。

### 4. 国内可达性不保证

`cdn.jsdelivr.net` 在国内部分地区和运营商下会有劣化甚至不可达。可以备选：

```text
https://fastly.jsdelivr.net/gh/...
https://gcore.jsdelivr.net/gh/...
```

但也不建议作为生产关键资源的唯一 CDN。轻量、内部、可重试场景更适合。

### 5. 文件大小

上传前必须压缩。原图几 MB 以上会拖慢访问，也可能触发 jsDelivr 对单文件的限制。图床场景建议压缩到 200KB～1MB，优先 webp/avif。

### 6. GitHub API 限流

未认证的 GitHub API 请求只有每小时 60 次，认证后是 5000。批量自动化上传时要加队列、重试、缓存已上传结果。对 OpenClaw 工具来说，可以用 fine-grained PAT 限制到单个仓库的 `contents:write` 权限，不要把整个账号 token 放进去。

## 可复用建议

- **文件名用内容 hash**：`sha256(file)[:16].webp`，天然去重、避免覆盖、避免缓存坑。
- **目录按时间分片**：`yyyy/mm/hash.webp`，避免根目录堆积大量文件。
- **MCP 返回结构化数据**：不要只返回字符串 URL，返回 `{ url, markdown, purge_url, file_hash }`，Agent 可以直接用。
- **压缩放工具内做**：用 sharp（Node）或 Pillow（Python），上传前自动限制宽高和体积。
- **提交版本用 commit hash**：push 后取 `git rev-parse HEAD` 拼 URL，保证链接指向具体内容。
- **失败不要硬重试**：CDN 不可达时让 Agent 降低频率，回退到 `raw.githubusercontent.com` 或本地路径，避免限流叠加。

## 总结

GitHub + jsDelivr 作为免费图床，价值不在“白嫖”，而在于它的上传协议简单、URL 规则明确、自动化成本低。对 OpenClaw / Agent / MCP 这类需要自动生成并嵌入图片的工作流来说，它足够务实。

但它并不适合所有场景：公开仓库、缓存不实时、国内可达性波动、单文件大小限制，注定了它更适合内部报告、轻量配图、个人自动化。关键资源还是应该走对象存储或自有 CDN。

工程化的做法不是“换成免费 CDN”，而是把它封装成一个小工具：一条命令或一次 MCP 调用，完成压缩、上传、返回可嵌入链接。这样 OpenClaw 里的 Agent 才能真正把图床用起来，而不是每次手动传图。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/0d9e243a0e4a41c0.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/de344e6cd699fead.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/117cb167875300d9.png)

