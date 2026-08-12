---
title: GitHub + jsDelivr 搭建生产可用图床的工程化实践
feedId: 32724
source: 综合讨论
publishedAt: 2026-08-12
---

## 为什么还要折腾图床

在构建 OpenClaw 智能体或 MCP 工具链时，图片往往是推理结果的重要载体——可能是 `matplotlib` 生成的趋势图、`playwright` 截取的页面快照，或是某个插件渲染的架构草图。这些图片需要一个稳定的外网可访问地址，才能继续传递给下游 agent 或嵌入最终报告。

自建对象存储 + CDN 的方案最可控，但对于个人项目、内部工具或快速原型来说，维护一套 OSS/Bucket 会让工程复杂度陡升。第三方图床服务又常常有政策变动、限流、插入广告等问题。

一个折中方案是将 GitHub 仓库当作图片源，再通过 jsDelivr 全球 CDN 分发。不需要额外注册服务，完全基于已有 GitHub 账号即可运作，且访问速度对海外和国内部分区域都算可用。更重要的是，这套流程可以完全自动化，很适合嵌入 MCP server 或 agent 的图片上传动作。

## 方案核心

整个链路非常简单：

```
local image → GitHub repo (raw file) → jsDelivr CDN URL
```

GitHub 仓库里的任何文件都可以通过 `raw.githubusercontent.com` 直接访问，但那个域名在某些地区体验很差。jsDelivr 对 GitHub 提供了加速支持，规则是：

```
https://cdn.jsdelivr.net/gh/<user>/<repo>@<branch>/<path>
```

例如文件在 `myuser/myimages` 仓库的 `main` 分支 `screenshots/demo.png`，CDN 地址就是：

```
https://cdn.jsdelivr.net/gh/myuser/myimages@main/screenshots/demo.png
```

这个地址可以直接嵌入 Markdown、HTML 或 API 响应中，没有跳转，没有广告。

## 工程化落地步骤

### 1. 仓库准备

创建一个专用图床仓库（建议设为 Public，因为 jsDelivr 加速面向公开仓库；Private 仓库不适用）。仓库内可以按业务模块或日期规划目录，例如：

```
/images
  /2025
    /03
      chart-q1.png
  /agents
    arch-overview.png
```

不建议直接把几百张图塞在根目录，否则后期维护和自动清理都会麻烦。

### 2. 手动验证

先把一张图片放到仓库对应路径，提交后等待几秒，访问 CDN 地址确认生效。jsDelivr 会拉取文件并缓存到边缘节点，首次访问可能稍慢，后续会明显提速。可以通过在浏览器中直接打开链接或使用 `curl -I` 查看响应头，确认 `x-cache: HIT` 状态。

### 3. 通过 GitHub API 实现自动上传

绕过本地 git 操作，直接用 Personal Access Token 调用 GitHub Contents API 上传文件。核心请求：

```
PUT /repos/:owner/:repo/contents/:path
Authorization: token <PAT>
Content-Type: application/json
{
  "message": "upload screenshot",
  "content": "<base64 encoded image>",
  "branch": "main"
}
```

一次调用就完成了提交。这里的 base64 内容不需要包含类似 `data:image/png;base64,` 前缀，只需要纯编码后的字符串。

### 4. 封装为 MCP 工具或 Agent 动作

为了方便在 OpenClaw 等 agent 环境中调用，可以把上传逻辑封装成一个简单的 MCP tool，接受图片路径和仓库目标路径，返回 CDN 链接。大致步骤：

1. 读取本地图片并转为 base64
2. 调用 GitHub API 上传
3. 拼接 jsDelivr URL 返回

如果需要处理重复文件名覆盖问题，可以让 tool 在目标路径中附加时间戳或短 UUID，例如 `screenshots/20250322-153045-a1b2.png`，避免意外覆盖重要图片。

也可以更进一步，通过 GitHub Actions 监听 Push 事件，在图片上传后自动生成缩略图或优化体积，但这对图床场景来说有些过度设计。

## 踩坑实录

### 缓存刷新不及时

jsDelivr 对同一 URL 的缓存时间很长（CDN 层通常 7 天以上）。如果上传新版本覆盖原文件，旧 URL 并不会立即失效。解决办法是**每次都上传到新文件名**（如带时间戳或内容哈希），或者通过 jsDelivr 的 purge API（`purge.jsdelivr.net`）手动刷新。工程中更推荐改名策略，因为它消除了缓存一致性问题。

### 仓库大小限制

GitHub 建议仓库大小在 1GB 以内，且单个文件不能超过 100MB。对于图片场景，100MB 的单张图几乎不会遇到，但仓库总大小需要留意。如果长期高频上传而且不清理，仓库会膨胀。可以制定清理策略，比如保留最近 30 天的图片，其余通过脚本归档或删除。注意，删除后 CDN 链接会失效，仅适用于临时图片场景。

### 国内部分运营商访问波动

jsDelivr 在国内不同地区和时段的表现有差异。移动、联通大部分时间能用，电信偶尔丢包。如果面向大量国内最终用户，可以考虑配合一个简单的 Nginx 反代缓存，但这已经是另一个话题。对于面向 agent 内部消费的场景，偶尔超时重试即可接受。

### 敏感信息泄漏

如果把包含敏感数据的截图、调试信息上传到 Public 仓库，就等于公开了。必须在 agent 内部做好过滤，只上传脱敏后的图片，或者为敏感场景使用另一个私有方案（但这套 GitHub + jsDelivr 就不适用了）。

## 可复用建议

- **目录规范**：按 `project/year-month/` 组织，方便批量清理。
- **命名策略**：使用 `<type>-<timestamp>-<shortid>.<ext>`，保证唯一且可追溯。
- **上传时检查 Content-Type**：确保文件扩展名与 MIME 对应，否则浏览器访问时可能被当成下载而不是预览。
- **搭配 CDN 容错**：在代码里对 CDN 返回做超时和重试处理，并可回退到 `raw.githubusercontent.com` 作为兜底（虽然慢但一般可用）。
- **监控用量**：GitHub 对 API 有速率限制（认证用户 5000 次/小时），一般图片上传绰绰有余，但批量操作时要注意。

## 总结

用 GitHub 仓库 + jsDelivr 搭建图床是一个低成本、高透明度的方案，尤其适合嵌入自动化工作流。不用引入额外账号体系或账单，所有资源可审计、可复现、可脚本化。它虽然不是生产级海量图片的最终答案，但足以支撑 agent 输出、工具链集成与原型验证阶段的图片分发需求。

下一步可以考虑把上传逻辑固化为一个 Shared MCP tool，让 OpenClaw 中的不同 agent 直接通过自然语言描述就能存放和引用图片，进一步降低操作摩擦。

---

