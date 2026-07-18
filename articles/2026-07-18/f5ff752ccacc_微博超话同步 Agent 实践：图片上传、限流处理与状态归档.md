---
title: 微博超话同步 Agent 实践：图片上传、限流处理与状态归档
feedId: 29537
source: 综合讨论
publishedAt: 2026-07-18
---

## 背景

内容创作者经常需要将博客、动态或精选内容同步到微博超话，以获取社区流量。手动操作时，图片上传、文案截断、添加超话标签、检查发布结果都是重复劳动。如果有多账号、多超话需要维护，很容易遗漏或重复发布。

我们希望用 OpenClaw 社区的 Agent 体系，搭建一条可靠的自动化管道：读取本地 Markdown 源文件 → 预处理 → 调用微博 API 上传图片并发帖 → 记录同步状态 → 处理限流与重试。整个流程无需打开微博客户端，也不需要依赖第三方商业中控。

## 问题分解

实现这样一个 Agent 主要面临三类工程问题：

1. **微博 API 的图片上传**  
   开放平台要求使用 multipart/form-data 方式上传图片，且不同接口（statuses/upload vs. statuses/share）对图片数量、尺寸、格式有不同限制。直接拼请求容易因编码问题失败。

2. **限流与风控**  
   发帖接口有明确的频次限制，超话连续发帖还可能触发反垃圾机制。如果不做退避和状态感知，轻则返回错误码，重则账号被限制。

3. **同步状态的归档与幂等**  
   为了避免重复发帖，必须知道哪些内容已经同步过。单纯的 MD5 比对不可靠（图片可能重新生成），需要利用微博返回的帖子 ID + 短链接做去重，并持久化记录成功、失败、限流等待等状态。

## 做法与步骤

我们基于 OpenClaw 的 MCP（Model Context Protocol）体系来构建工具，让 Agent 通过自然语言指令调用标准化工具。整体架构如下：

- **源内容**：本地 Markdown 文件，以 YAML front matter 标记标题、摘要、图片路径、目标超话。
- **MCP 服务**：自建 `weibo-mcp-server`，暴露三个工具：
  - `upload_image(file_path)` → 返回 `pic_id`
  - `post_weibo(status, pic_ids, topic)` → 返回 `post_id, short_url`
  - `check_rate_limit()` → 返回当前剩余调用次数
- **Agent 逻辑**：用 OpenClaw 的 Loop Agent 模式，遍历待同步列表，依次调用工具，根据返回结果写入状态归档。

具体步骤：

### 1. 内容源与任务队列

用文件系统作为任务队列，约定目录结构：

```
content/
  todo/
    2025-01-01-my-post.md
  done/
  failed/
```

Agent 启动后扫描 `todo` 目录，解析 front matter：

```yaml
title: "某篇技术笔记"
summary: "一篇关于…"
image: "./img/cover.png"
topic: "开源技术"
```

将文本截断至符合超话长文要求，并拼接 `#开源技术#` 标签。

### 2. 工具实现与授权

`weibo-mcp-server` 使用 `aiohttp` 封装微博开放平台 V2 接口。敏感信息（`CLIENT_ID`, `CLIENT_SECRET`, `ACCESS_TOKEN`, `REFRESH_TOKEN`）全部通过环境变量注入，OAuth2 的 token 刷新逻辑内置于服务中，对 Agent 透明。

上传图片的重点代码片段：

```python
async def upload_image(file_path: str) -> str:
    with open(file_path, "rb") as f:
        resp = await session.post(
            "https://upload.api.weibo.com/2/statuses/upload.json",
            data={"access_token": token},
            files={"pic": f},
        )
    result = await resp.json()
    return result["pic_id"]
```

发帖接口使用 `statuses/update`，将 `pic_id` 通过 `pic_id` 参数传入（支持最多 9 张）。

### 3. 限流处理策略

微博接口返回头中不直接给出限流窗口，需要根据错误码判断。常见限流错误码：

| 错误码 | 含义 |
|--------|------|
| 10022 | IP 请求频次超限 |
| 10024 | 用户请求频次超限 |

Agent 遇到这些错误码后，执行指数退避：第一次等待 60s，第二次 300s，超过 3 次则将任务移动到 `failed` 并标记 `rate_limited`，等待人工介入或次日重试。

此外，在发帖前主动调用一次 `account/rate_limit_status` 接口，若剩余次数不足 5 次则暂停并归档等待。

### 4. 状态归档与幂等

每次发帖成功后，微博返回的 `id` 和 `short_url` 被持久化到 SQLite 数据库的 `sync_log` 表：

```sql
CREATE TABLE sync_log (
    source_file TEXT PRIMARY KEY,
    post_id TEXT,
    short_url TEXT,
    status TEXT,  -- 'success','failed','rate_limited'
    created_at TIMESTAMP
);
```

Agent 在读取 `todo` 文件时，会先查询 `sync_log`，如果已有 `success` 记录则直接移动到 `done` 并跳过，避免重复。消息的全文哈希值不靠谱，因为时间戳等会变化，短链接则天然具有唯一性。

## 踩坑点

1. **图片自动压缩导致上传失败**  
   部分 Markdown 中的截图尺寸过大（超过 5MB），微博上传接口会返回 `pic size out of range`。我们加入 PIL 预处理，无脑压缩至 1080px 宽，质量 85%，几乎无损可保证通过。

2. **超话标签位置影响展示**  
   最初将 `#超话名#` 放到文末，但微博客户端有时会把超话标签前的文字截断作为摘要。改为将标签放在开头第一行后，摘要展示更理想。

3. **短链接去重的陷阱**  
   微博短链接的域名可能是 `t.cn` 也有可能被风控替换，偶尔会出现同一篇帖子返回不同短链。因此我们同时记录了 `post_id`，当短链冲突时以 `post_id` 为准去重。

4. **OAuth2 刷新令牌静默失效**  
   测试环境长时间不运行后，`refresh_token` 过期，接口返回 `invalid_grant`。Agent 没有处理，直接崩溃。后来在 MCP 服务中增加了 token 过期检测，并在即将过期前主动刷新，同时把异常通知给 Agent，让其归档并通知人工。

## 可复用建议

- **工具抽象成通用 MCP 包**  
  你可以把这个 `weibo-mcp-server` 抽象成一个可配置的 MCP 包，方便社区其他人通过 OpenClaw 直接挂载使用，只需提供 OAuth 密钥即可。
- **采用“任务文件 + 状态数据库”模式**  
  比起消息队列，文件系统更直观、方便手动干预，非常适合个人工作流。
- **引入健康检查端点**  
  在 MCP 服务中暴露 `health` 工具，Agent 可定时检测 token 是否有效、剩余次数是否充足，提前预警。

## 总结

通过 OpenClaw Agent + 自建 MCP 服务，我们搭建了一套安静可靠的微博超话同步管道。上线一个多月后，人工介入次数从每周十几次降到一两次（多为超话规则变更导致）。整个工程的核心不在于“调用 API”，而在于**限流的健壮处理**和**状态的可追溯性**——这也是任何自动化发布工具能够长期运行的基础。

如果你也有跨平台内容同步的需求，这套模式可以很容易地扩展到 Twitter、长毛象等平台，只需要替换对应 MCP 工具即可。

---

