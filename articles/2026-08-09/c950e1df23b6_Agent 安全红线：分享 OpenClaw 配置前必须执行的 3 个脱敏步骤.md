---
title: Agent 安全红线：分享 OpenClaw 配置前必须执行的 3 个脱敏步骤
feedId: 32175
source: 综合讨论
publishedAt: 2026-08-09
---

## 背景
随着 Agent、MCP、插件化工作流的普及，OpenClaw 社区出现了越来越多分享 YAML 配置、工具链和自动化脚本的帖子。这些分享极大地降低了上手成本，但也无意中打开了内部信息泄露的窗口。一个被截图的对话历史、一段看似无害的工具配置，都可能包含未经察觉的敏感信息。

过去我们习惯在代码仓库中检查硬编码密钥，但 Agent 配置更像是一张交织了凭证、内部端点、业务逻辑和交互指令的地图。一旦泄露，攻击者不仅能拿到钥匙，还能看懂整栋楼的安保布局。

## 问题：为什么 Agent 配置分享更危险？
- **凭证与逻辑紧密耦合**：MCP 服务器的 URL 或 API key 往往是配置的一部分，例如 `http://internal.corp:8080/query?token=sk-xxx`。
- **隐式信任上下文**：社区成员可能在演示中直接导出生产环境的 `agent.yaml`，忽略了其中嵌入了内部知识库的 endpoint 或者包含内部业务术语的 system prompt。
- **二次传播**：论坛、GitHub issue、Discord 消息中的 attachment 可能会被搜索引擎收录，让错误分享难以撤销。

一个真实（已脱敏）的案例：某用户分享了一段用于生成周报的 Agent 配置，其中 tools 部分引用了公司内部的 Jira 实例地址和 Basic Auth 凭证。该配置被 fork 了上百次，直到有同事在搜索引擎中发现。

## 做法：三层脱敏流水线
### 第一步：结构拆分（Secrets 外置）
永远不要在配置文件中硬编码凭证。使用环境变量或外部 Secret 引用：

```yaml
tools:
  - name: jira
    type: mcp
    server_url: ${JIRA_SERVER_URL}
    headers:
      Authorization: "Bearer ${JIRA_API_TOKEN}"
```

分享时，只需提供 `.env.example` 模板，表明需要哪些变量，而不带真实值。

### 第二步：配置匿名化（Redact）
编写一个简单的检查脚本，在提交前扫描敏感关键字。可使用 Python 快速实现：

```python
import re, sys

SENSITIVE_PATTERNS = [
    r'sk-[a-zA-Z0-9]{32,}',     # OpenAI key
    r'Bearer [A-Za-z0-9\-_\.]+',# Bearer token
    r'internal\.corp',          # 内网域名
    r'192\.168\.\d+\.\d+',      # 内网 IP
    r'password["\s:=]+[\S]+',   # 明文密码
]

with open(sys.argv[1]) as f:
    content = f.read()
for pat in SENSITIVE_PATTERNS:
    if re.search(pat, content):
        print(f"发现疑似敏感信息: {pat}")
```

在执行 `git commit` 前跑一遍，并集成到 pre-commit hook。

### 第三步：人工回放审查（Review）
机器无法理解业务语义。分享前人工进行以下检查：
- 截图或日志：使用马赛克或替换文本遮盖任何 UI 中的业务数据（客户名、项目代号）。最简单的方法是重新录制一个 **仅包含 mock 数据** 的演示。
- system prompt：若 prompt 中定义了公司内部角色（如“你是一家位于上海的私募基金分析师”），删除或替换为通用描述。
- MCP 工具描述：如果 `tools` 的描述字段泄露了内部 API 的请求格式或返回字段，考虑移除详细 schema，仅保留功能说明。

## 踩坑点
1. **.env 文件被误打包**：检查 `.gitignore`，但分享到论坛时往往直接粘贴文件夹压缩包，会泄露 .env。打包前务必删除或使用 `zip -r project.zip project/ -x "*.env"`。
2. **Docker 配置残留**：若使用 docker-compose，其中的 `environment` 字段可能包含明文。养成使用 `env_file` 的习惯，并只分享去除 env_file 引用的 compose 文件。
3. **会话历史泄露**：分享的聊天记录（markdown 或截图）可能包含之前调试时注入的敏感上下文。应使用全新对话或脚本重新执行，确保仅包含 demo 数据。
4. **URL 中的参数**：内网地址可能带有调试 token，即便替换了值，路径格式（如 `/api/v2/project/`）也可能暗示内部系统结构。考虑替换为 `https://your-instance.com/api/`。
5. **二进制文件与日志**：不要分享 `*.db`, `*.log`, `node_modules` 等，它们可能包含编译进去的凭证。

## 可复用建议
- **建立分享检查清单**：在团队或个人笔记中固化一份 checklist，每次公开分享前逐项确认。
- **使用配置模板引擎**：如 `envsubst` 或更复杂的 Helm/Kustomize 思想，把环境差异彻底剥离。
- **社区规范**：OpenClaw 等社区的版主可以在“展示区”置顶安全分享指南，鼓励使用最小化可运行示例（MRE）并附带脱敏说明。
- **自动化 CI 扫描**：如果分享是通过 GitHub 仓库进行的，配置 GitHub Actions 调用上述扫描脚本，在 PR 或 push 时提醒。

## 总结
Agent 时代的配置分享是推动生态发展的重要方式，但我们必须用工程化的手段守住安全底线。记住三个原则：凭证外置、自动化扫描、业务语义脱敏。不要因为一次热情的分享，让整个内部网络的拓扑暴露在公网。

**安全不是你做了多少，而是你没漏掉多少。**

---

