---
title: MCP 协议入门：为什么你的 Agent 需要统一的上下文接口
feedId: 32313
source: 综合讨论
publishedAt: 2026-08-10
---

## 背景：碎片化的工具调用

过去一年，几乎每个大模型应用都在做同一件事：让模型调用外部工具、读取外部数据。OpenAI 有 Function Calling，LangChain 有 Tool 抽象，各种 Agent 框架又各自定义了自己的插件接口。结果就是，同一个天气查询功能，为了接入 Claude、GPT 和开源模型，你可能要写三套适配代码。

这很像 LSP（Language Server Protocol）出现之前的 IDE 生态——每个编辑器都要为每种语言单独实现补全、跳转、诊断。Anthropic 在 2024 年底提出的 Model Context Protocol (MCP)，试图在 AI 应用和外部工具/数据源之间扮演类似 LSP 的角色：一个标准协议，让任何模型客户端都能通过统一方式和任意 MCP 服务端通信。

## 它到底解决了什么问题？

表面看，MCP 就是另一种工具调用协议。但它真正解决的，是**上下文供给的标准化**。

LLM 的上下文窗口虽然在增长，但依然昂贵且有限。实际工程中，我们不会把整个数据库塞进 prompt，而是按需查询。这就需要一种机制，让模型在推理过程中主动“拉开抽屉”，找到需要的信息。Function Calling 提供了调用能力，但没规定资源的发现、描述和安全边界。MCP 把上下文需求抽象成三种原语：

- **Resources**：结构化只读数据，比如文件内容、数据库记录、API 响应。模型可以像打开文件一样读取，并附带元数据（MIME 类型、描述）。
- **Tools**：可执行的函数。模型可以请求调用，并接收结果。与 Function Calling 类似，但增加了标准化的输入/输出 schema 和错误格式。
- **Prompts**：预置的提示模板，帮助模型更好地使用服务器能力。

通过这三种原语，一个 MCP 服务端可以同时服务 Claude Desktop、OpenClaw 或其他客户端，而无需为每个平台重写集成。开发者只需要维护一个“上下文提供者”，而不是 N 个插件的胶水代码。

## 动手：5 分钟搭一个 MCP 服务端

MCP 官方提供了 Python 和 TypeScript 的 SDK。我们以 Python 为例，实现一个极简的“文件字数统计”工具。

```bash
pip install mcp
```

创建 `server.py`：

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("Word Counter")

@mcp.tool()
def count_words(text: str) -> int:
    """统计文本中的单词数量"""
    return len(text.split())

if __name__ == "__main__":
    mcp.run(transport="stdio")
```

服务端通过标准输入/输出（stdio）与客户端通信。现在，在 OpenClaw 中配置这个 MCP 服务器：在 `clawd.yaml`（或对应的插件配置文件）中添加：

```yaml
mcp_servers:
  word_counter:
    command: python
    args: ["/path/to/server.py"]
```

重启 OpenClaw 后，模型就可以通过 `count_words` 工具统计任何文本的单词数了。工具描述、参数类型、返回类型完全由服务端定义，客户端无需再写额外的类型适配。

对于需要持久化连接或远程访问的场景，MCP 支持 Server-Sent Events (SSE) 传输。只需将 `transport` 改为 `'sse'`，并在 `run()` 中配置端口即可。

## 踩坑记录

**1. stdio 传输的日志干扰**  
服务端如果在 stdout 打印日志，会污染 MCP 的 JSON-RPC 消息通道，导致客户端解析失败。务必把所有日志输出到 stderr，或使用标准的 logging 模块并指定 StreamHandler 指向 stderr。

**2. 大资源的分块问题**  
当 Resource 返回海量数据（如几万行日志），客户端可能因 token 限制截断。MCP 协议定义了分页和订阅机制，但 SDK 的实现还在完善。实践中建议先在服务端做聚合或摘要，再暴露给模型。

**3. 安全认证的缺失**  
当前 MCP 协议没有内置认证层。通过 SSE 暴露服务端时，需要反向代理添加鉴权（如 API Key 或 OAuth2），否则任何能访问端口的客户端都能调用工具。官方已表示后续会引入认证能力，但目前还是“裸奔”状态。

**4. 工具描述的重要性**  
模型完全依赖工具的名称和 docstring 来决定何时调用。如果你的工具描述含糊，模型会“用脚投票”。写好详细的说明、参数约束和返回示例，是工具被正确调用的前提，这比在客户端写 Prompt 工程更可靠。

## 可复用建议

- **渐进式接入**：不必一次性把所有外部服务都变成 MCP。可以先从高频、数据量小的工具（如日期计算、文本处理）开始，验证链路。
- **统一管理**：维护一个 MCP 服务端集合，用配置管理工具（如 Ansible/Docker Compose）启动。OpenClaw 等客户端只需一套 YAML 描述即可接入全部能力。
- **优先使用资源暴露**：如果数据是只读的，用 Resource 而非 Tool。这能避免模型无意中修改生产数据，且客户端可以做本地缓存。
- **监控与降级**：为每个工具调用添加超时、重试和 Fallback 逻辑。MCP 规范支持返回标准化错误码，可在客户端统一处理“服务不可用”的情况，提示模型换一种方式获取信息。

## 总结

MCP 并不是一个全新的发明，而是把已经存在于各家 Agent 框架中的“工具 / 资源 / 提示”抽成了一个网络协议。它的价值在于打破 AI 客户端与数据源之间的紧耦合，让上下文供给从“平台锁定”走向“即插即用”。

在大模型能力还在快速演进的阶段，把工程精力投入这种协议层的标准化，是值得的。毕竟，你肯定不会希望每次换个模型底座，都要重写一遍插件系统。

---

