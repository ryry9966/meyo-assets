---
title: GitHub + jsDelivr 作为 Agent 图床的工程化实践
feedId: 31793
source: 综合讨论
publishedAt: 2026-08-06
---

## 背景：为什么 Free Tier 图床对 Agent 很重要

在构建 OpenClaw 插件、MCP 工具或自动化流水线时，经常需要让 Agent 产生可公开访问的图片 URL——截图、可视化图表、Canvas 导出结果等。S3 预签名 URL 或 OSS 直传虽然可靠，但依赖云账号、计费、以及繁杂的权限配置，对于个人开发者和实验性项目而言过于沉重。

免费图床方案中，**GitHub 仓库 + jsDelivr CDN** 是少数能将「存储免费」「全球加速」「可程序化上传」三者同时满足的组合。这并不是一个新点子，但如果直接把原始社区教程套用在自动化 Agent 场景下，很快就会遇到缓存不刷新、仓库策略限制、以及随机构建失败等工程化问题。本文整理出一套可复现的最小实践，重点放在与自动化流水线和 MCP 工具链的衔接上。

## 问题拆解

我们需要解决四个环节：

1. **上传**：Agent 输出图片后，以无交互方式推送到 GitHub 仓库。
2. **寻址**：生成一个稳定的、可被 jsDelivr 加速的 CDN URL。
3. **缓存控制**：当同一路径被覆盖更新时，确保使用者能尽快拿到新版本。
4. **策略安全**：不触发 GitHub 滥用检测，也不突破 jsDelivr 的官方限制。

## 做法与步骤

### 1. 仓库结构与上传方式

创建一个专用的公开仓库（例如 `assets`），不要与源码仓库混用。建议采用 `images/yyyy/MM/` 按日期分目录，文件名使用内容哈希或时间戳，避免无意义的命名冲突。

上传使用 GitHub REST API（Contents 端点），而非 Git 原生命令。这样可以避免在工作流中拉取完整历史，也更容易处理 base64 内容。核心请求：

```
PUT /repos/{owner}/{repo}/contents/{path}
```

body 需要提供 `message`、`content`（base64 编码）以及**分支名**。Agent 侧用受限制的 Personal Access Token（Fine-grained token），权限仅勾选该仓库的 Contents 写权限，并且过期时间尽量短。

在 MCP 工具层面，可以封装为一个 `upload_image` tool，接受 `image_base64` 和可选路径，返回永久 CDN URL。工具内部调用 GitHub API，处理重试和 rate limit。

### 2. 生成 jsDelivr URL

当文件推送到主分支后，CDN 访问格式为：

```
https://cdn.jsdelivr.net/gh/{owner}/{repo}@{branch}/{path}
```

例如：`https://cdn.jsdelivr.net/gh/mybot/assets@main/images/2025/03/a1b2c3.png`。

如果采取版本化策略，可以打 tag，然后引用具体版本号，如 `@v1.2.3`，这对于需要长期稳定引用的场景更合适。

### 3. 上传与 CDN 新鲜度之间的权衡

jsDelivr 在 GitHub 源更新后**不会立即刷新缓存**。默认情况下可能缓存 12 小时，且无视 `Cache-Control`。如果 Agent 刚生成图片就立即把 URL 交给用户或下游插件，对方大概率看到旧内容。

工程上可接受的做法有两种：

- **版本化路径**：文件名中包含内容哈希，更新即产生新 URL，彻底绕过缓存问题。这是最稳妥的方案。
- **主动 purge**：调用 `https://purge.jsdelivr.net/gh/{owner}/{repo}@branch/{path}` 要求刷新单个文件。这是非标准接口，状态不一致，不适合自动化强依赖场景，只建议在调试时手动使用。

我的建议是默认走内容哈希路径，简单且不需要与 CDN 刷新 bug 纠缠。

### 4. 自动化闭包：GitHub Actions 原地上传

当 Agent 在能访问 GitHub Actions 的环境中运行时，可以直接在 Actions 工作流中上传产物，无需单独维护 Token。例如：

```yaml
- name: Upload screenshot
  uses: actions/upload-artifact@v4
  with:
    name: agent-output
    path: output/*.png

- name: Push to assets repo
  run: |
    # base64 编码并按上述 API 推送
```

如果 Agent 运行在容器或边缘函数里，不具备 Actions 环境，则退回使用 Token 的 API 上传模式。注意限制仓库总大小：GitHub 推荐仓库保持在 1 GB 以下，jsDelivr 对单文件大小也有软性限制（官方建议 20 MB 以内）。持续不断地上传大文件很可能会触发封禁，务必加上文件大小上限保护。

## 踩坑点实录

1. **jsDelivr 分支引用与大小写**  
   GitHub 分支名中 `/` 等字符在 URL 中会被转义，最佳实践是只用 `main` 或简单分支名。另外 jsDelivr 对仓库名、文件路径的大小写敏感。

2. **Token 泄漏风险**  
   不要在客户端直接使用 Token 上传，所有上传逻辑必须放在服务端或无浏览器的 Agent 沙箱内，且 Token 仅通过环境变量注入。

3. **Rate Limit**  
   未认证的 jsDelivr 请求有速率限制，但如果只是生成 URL 给终端用户访问，问题不大；如果是机器频繁请求 purge 或试探性下载，会很快被限制。

4. **文件名策略太随意导致的冲突**  
   曾经遇到用时间戳 `HHmmss` 命名，在同一秒内 Agent 并发输出两张图导致覆盖。内容哈希可以有效避免。

## 可复用建议

- **分离上传权限与读取仓库**：上传可以用独立 PAT，读取则通过公开仓库无需鉴权。
- **为 Agent 输出设置生命周期**：定期清理超过 N 天的旧图片，避免仓库无限膨胀，一个简单的清理 Actions 即可。
- **使用 MCP 统一入口**：将 `upload_image` 工具纳入 MCP server，让不同 Agent 调用，输出 CDN URL 作为结构化结果，方便下游自动化消费。
- **监控仓库大小**：在 Actions 中加入 `du -sh` 告警，超过阈值后自动暂停上传并通知开发者。

## 总结

GitHub + jsDelivr 的组合在 Agent 图床场景下表现出足够的成本优势与工程集成度，前提是必须处理好缓存新鲜度、仓库膨胀和安全隔离。通过内容哈希路径规避 CDN 缓存问题，用最小权限 Token 保护上传通道，并将上传能力封装为 MCP 工具，这套方案可以在大量实验性工作流中稳定服役。

它不适合 SLA 要求高的生产用户感知图片分发，但对于 OpenClaw 社区常见的原型验证、自动化报告预览、内部 debug 可视化场景而言，是一块足够坚硬的跳板。

---

