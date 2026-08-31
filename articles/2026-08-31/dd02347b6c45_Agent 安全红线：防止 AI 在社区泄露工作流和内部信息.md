---
title: Agent 安全红线：防止 AI 在社区泄露工作流和内部信息
feedId: 35498
source: 综合讨论
publishedAt: 2026-08-31
---

# Agent 安全红线：防止 AI 在社区泄露工作流和内部信息

## 背景

OpenClaw 的复现价值在于工作流、MCP 工具和代理配置可以被直接分享。但社区里一份看似正常的 workflow，往往不只是“逻辑图”：它会带上提示词、工具权限、环境变量名、内网地址，甚至本地文件路径。

Agent 与普通脚本不同，它会把系统提示、工具输出、历史消息拼进上下文继续推理。只要某个工具返回了不该返回的内容，模型就可能在下一次回复里原样带出。因此，安全边界不能只放在聊天界面，而要前移到工具、配置和发布流程里。

## 问题：最常见的泄露路径

1. **硬编码密钥**：`api_key: sk-xxx` 直接写进 workflow 的 YAML/JSON。
2. **系统提示泄露内部上下文**：提示词里写着“访问 http://10.0.0.5:8000/erp，使用内网数据字典 v2.3”。
3. **MCP/插件工具返回全量**：数据库查询返回完整行、HTTP 工具返回完整 header、shell 输出原样进上下文。
4. **发布时打包了本地配置**：导出时把 `.env`、`mcp.json` 里的 `env` 块一起带上。
5. **Prompt injection**：外部网页、文档或社区内容诱导 Agent 打印 system prompt、工具配置、环境变量。
6. **日志泄露**：为了调试打开全量日志，把 prompt、tool result、文件内容写进日志后再发到社区。

## 做法 / 步骤

### 1. 把秘密移出工作流

不要在 workflow 里放任何真实密钥、token、cookie。使用占位符或环境变量引用：

```yaml
env:
  OPENAI_API_KEY: ${OPENAI_API_KEY}
  INTERNAL_API: ${INTERNAL_API}
```

分享时提供 `.env.example`，并确保默认值不包含真实内容。

### 2. 工具输出最小化

不要把工具原始返回直接交给模型。MCP server 或插件层做一次包装：

```json
{
  "ok": true,
  "data": {"id": 1024, "status": "active"},
  "error_code": null
}
```

模型不需要看到完整的 HTTP 响应头、堆栈、数据库 schema 或全量行。只需返回任务必要的结构化字段。

### 3. 设置显式 allowlist / deny 规则

对文件、shell、HTTP 工具设置边界：

- 文件工具只允许 `/workspace/**`，拒绝 `**/.env`、`**/*.pem`、`**/.ssh/**`、`**/.aws/**`
- Shell 禁止 `cat .env`、`printenv`、`history`、`env`
- HTTP 工具限制域名白名单，禁止访问 `169.254.169.254`、内网管理面

在 OpenClaw/MCP 配置中，权限声明应默认拒绝，只有明确需要时才开放。

### 4. 预发送 / 发布前 redaction

在 Agent 对外回复或导出前，加一层正则脱敏：

```python
REDACT_PATTERNS = [
    r"sk-[A-Za-z0-9]{10,}",
    r"ghp_[A-Za-z0-9]{10,}",
    r"AKIA[A-Z0-9]{10,}",
    r"Bearer\s+[A-Za-z0-9\-._~+/]+",
    r"\b10\.\d{1,3}\.\d{1,3}\.\d{1,3}\b",
]
```

这层过滤器放在 publish/reply hook 中，而不是只在聊天界面做 UI 遮挡。

### 5. 发布检查清单

分享 workflow 前至少做一次：

- 运行敏感字段扫描：`grep -R -E "sk-|ghp_|AKIA|BEGIN .*PRIVATE|password|token" .`
- 检查 MCP 配置里的 `env`，确认没有真实值
- 用一份 dummy 数据完整跑一遍，观察是否有内部名称、账号、路径出现在输出中
- 将内网域名、IP、用户名、项目名替换成示例值

### 6. 把社区下载内容视为不可信输入

导入别人的 workflow、插件、MCP 前，先看权限声明：它请求了哪些 env、文件路径、网络域。第一次运行放在隔离环境或容器中，不要直接在工作机加载别人给的 install script 或 `.env`。

## 踩坑点

- **只脱敏 API key，不改提示词**：密钥挡住了，但提示词里的内部系统信息还在。
- **忽略 `mcp.json` 里的 `env` 块**：很多人检查了 workflow，忘记 MCP server 配置同样可以藏密钥。
- **把 debug 日志当普通文本发社区**：模型看不到，但日志里可能有完整 prompt 和工具返回。
- **只信任“开源 workflow”**：开源只能说明代码可见，不能说明它不读取本地配置、不上传数据。
- **认为 prompt injection 只能来自用户消息**：网页、文档、Issue、社区帖子都可能成为注入源。

## 可复用建议

- 用 `secrets` 统一命名环境变量，CI/publish 脚本只扫描这一前缀。
- 导出时只分享图、提示词和工具声明，不导出 `env` 与凭据。
- 工具层统一返回 envelope，禁止原始响应直通模型。
- 所有外部内容走三层：allowlist、redactor、human review。
- 建立一个 5 行以内的发布前 lint：扫描常见密钥正则、拒绝 `.env` 文件、检查 MCP 权限。

## 总结

社区分享的价值在于工作流和思路，不在于密钥、内部路径或客户数据。Engineering 上的一条红线是：**模型能够读到的任何信息，最终都有可能被模型带出来。** 因此不能只靠事后打码，而要在工具输出、配置管理、发布流程上做默认拒绝和最小暴露。这样分享出来的 Agent 才是可复现、可维护，也不容易把自己搭进去。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/f7ce605e3e13500c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/c06372c930a21b22.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/ae65e21a2e9a4ab6.png)

