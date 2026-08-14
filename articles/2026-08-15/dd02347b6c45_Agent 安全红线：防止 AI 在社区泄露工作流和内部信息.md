---
title: Agent 安全红线：防止 AI 在社区泄露工作流和内部信息
feedId: 33202
source: 综合讨论
publishedAt: 2026-08-15
---

## 背景

在 OpenClaw 这类 Agent 编排环境里，MCP/插件让模型可以读文件、查数据库、调接口、发消息。很多用户会把 Agent 接入社区群、Issue 自动回复、周报生成、公开博客。风险已经不是模型说错话，而是它把内部工作流、目录结构、密钥、客户名、未公开代码片段当成“上下文”带出去。

## 问题在哪

泄露通常不是模型主动背叛，而是来自三类情况：

1. **上下文过载**：为了效果把内部文档、全量配置、env 全塞进 system prompt，模型在公开输出时复述了这些内容。
2. **工具描述过细**：MCP server 的 tool description 写得像内部 SOP，模型调用失败时把参数、路径、账号名打出来。
3. **自动化发布缺少脱敏**：Agent 直接写文件、发 issue、推送到公开仓库，没有人工确认。

常见泄露形式包括：错误堆栈带 `/home/user/project/.env`、API key、内网 IP；工作流 YAML 被摘要后发到群；插件读取内部 CRM 客户名后生成公开案例。

## 做法/步骤

### 1. 做上下文分层

把 system prompt 拆成三层：

- 公开人设/任务说明
- 内部工作流知识
- 运行时数据

内部知识用 MCP 工具按需取用，不要全部常驻上下文。例如，不要在 prompt 里写“我们内部用 A 系统，路径是 `/data/ops`”，改为“当需要部署信息时调用 `internal_ops_query`”。

### 2. 给工具加最小权限和描述脱敏

MCP server 只暴露必要读写接口。写操作单独授权，默认只读。工具描述不要包含真实路径、账号、内部系统命名。

不推荐：

```yaml
name: get_deploy_status
description: 查询 /opt/deploy/xxx 的 k8s 状态，需 admin token
```

推荐：

```yaml
name: get_deploy_status
description: Get deployment status by service id. Do not expose internal hostnames.
```

### 3. 设置输出过滤

在 Agent 输出后、发布前加一层 filter。低成本做法：

- 正则脱敏：手机号、邮箱、API key、token、内网 IP、路径。
- 关键词阻断：若输出含 `internal/`、`secret`、`Bearer `、`.env` 等则阻止发送。
- 结构化校验：JSON schema 只允许公开字段，不允许携带 raw config。

可以在发布前挂一个 `pre_publish` hook：

```python
import re

DANGER_PATTERNS = [
    r"sk-[A-Za-z0-9]{16,}",
    r"10\.\d+\.\d+\.\d+",
    r"/home/",
    r"/data/",
    r"internal_",
    r"Bearer [A-Za-z0-9\-._~+/]+=*",
]

def pre_publish(text):
    for pattern in DANGER_PATTERNS:
        if re.search(pattern, text):
            raise BlockPublish("sensitive content detected")
    return text
```

### 4. 审计日志与回放

记录 Agent 的 tool call 和最终输出。每周抽检一次：哪些内部数据被读到了？哪些出现在公开输出？重点看错误路径和“总结”类输出。

### 5. 发布流程加人工闸门

公开仓库、社区帖、群公告默认进入草稿目录，不直接发布。只有人工确认后才推送。对自动回复类 bot，限制只能回复预设短句，不生成自由文本。

## 踩坑点

- **只靠 prompt 说“不要泄露内部信息”基本无效**。模型在长上下文总结时仍可能带出原词。
- **`.env` 全量塞进环境变量**，然后 MCP 工具能枚举 env，等于把密钥暴露给模型。建议使用变量白名单。
- **错误信息比正文更危险**。OpenClaw 的 debug log 可能包含完整 tool input，发 issue 时不要直接贴 logs。
- **截图自动发送**：UI 截图含内部导航、项目名、同事头像，应该关闭自动截图或做模糊。
- **“只发一次没关系”的临时脚本**，后来被改成定时任务，泄露面扩大。

## 可复用建议

- 建一个 internal-info 清单：哪些字段、路径、系统名、客户名不允许出现。把它做成正则和 keyword 配置。
- 使用两个 Agent：一个内部 Agent 获取原始数据；另一个公开 Agent 只接收脱敏后的摘要，负责对外表达。
- 对 MCP server 做 capability review：每个 tool 是否真的需要写权限？是否返回过多字段？
- 在 CI 里做配置扫描：禁止在公开仓库提交 `.env`、`internal/`、token 等。
- 分享工作流时只分享“结构”，不分享“数据”。把真实路径、客户名、token 替换成占位符。

## 总结

Agent 安全红线不是不让自动化，而是把“读取、生成、发布”三段切开。读取最小化，生成过滤，发布人工确认。对 OpenClaw/MCP 用户来说，最划算的三件事是：**上下文分层、输出过滤、发布闸门**。先做这三件，再谈更复杂的权限系统。

---

