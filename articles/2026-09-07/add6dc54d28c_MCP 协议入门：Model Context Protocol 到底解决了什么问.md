---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 36488
source: 综合讨论
publishedAt: 2026-09-07
---

## 背景：胶水代码的 N×M 问题

模型能力这两年进步很快，但真正做过 Agent 落地的人都清楚，大头工作量不在模型，在“接线”：怎么让模型稳定地读数据库、查工单、操作文件。在 MCP 出现之前，这件事没有统一约定——Claude Desktop、Cursor、自研 Agent 各有一套 Function Calling 格式，同一个工具要按不同客户端的口味反复适配。N 个客户端对接 M 个工具，就是 N×M 份胶水代码，且互不通用。

MCP（Model Context Protocol）是 Anthropic 在 2024 年 11 月开源的协议，做的事情很朴素：把这层集成标准化。拿“USB-C”作类比有点俗，但意思是对的——工具方实现一次 Server，任何支持 MCP 的客户端都能直接用。

## 它到底定义了什么

MCP 建在 JSON-RPC 2.0 之上，核心是三种角色和三类能力：

- **Host / Client / Server**：Host 是 Claude Desktop、OpenClaw 这类宿主程序，内部为每个 Server 维护一条 Client 连接；Server 是独立进程，负责暴露能力。
- **三类原语**：Tools（模型可调用的动作，如“创建 issue”）、Resources（应用可读的数据，如文件内容）、Prompts（可复用的提示模板）。

对多数人，最先用到的是 Tools。一个最小 Server 长这样（Python / FastMCP）：

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("ops-tools")

@mcp.tool()
def query_order(order_id: str) -> str:
    """按订单号查询物流状态，返回快递公司与最新节点。"""
    return db.lookup(order_id)

mcp.run(transport="stdio")
```

在宿主的工具配置里注册启动命令，重启后模型就能看到这个工具。注意 docstring 不是注释，它会原样进模型上下文，等于工具的“说明书”。

## 踩坑点

都是实际交过学费的：

1. **stdout 污染**。stdio 传输下 stdout 只能走 JSON-RPC 帧，一行 print 调试信息就能把连接搞挂。日志一律走 stderr。
2. **工具描述写太省**。“查询数据”这种描述会让模型在相似工具间选错、编参数。把参数含义、返回格式、失败行为写清楚，是性价比最高的优化。
3. **工具数量失控**。一个 Server 挂四五十个 tool，schema 吃掉大量上下文，选择准确率也明显下降。按领域拆分，或做一层聚合入口。
4. **把本地权限当成理所当然**。stdio Server 继承的是你本机权限，接了文件系统工具就要按“模型能读到什么就能读什么”来设计边界；远程部署则必须自己补 OAuth/token，协议不管这部分。
5. **规范版本差异**。MCP 有 2024-11、2025-03、2025-06 几个修订版，传输层从 SSE 换成了 Streamable HTTP，各客户端支持不一，连不上先核对版本。

## 可复用建议

- 先用官方 **MCP Inspector** 调通再接 Agent，它可以直接列工具、模拟调用，比在 Agent 里盲猜快得多。
- 一个 Server 只做一类事，接 GitHub 的和接数据库的分开，别写大杂烩。
- 工具粒度宁粗勿细：“提交 PR”一个工具，好过 create_branch + push + open_pr 三个。
- 写操作加确认层，读操作放宽。
- 排障顺序固定下来：配置路径是否可执行 → stderr 有无报错 → 协议版本是否匹配。

## 总结

MCP 解决的不是“让模型更聪明”，而是集成层的标准化：把 N×M 的适配成本压成 N+M，工具方写一次，客户端通吃。对 OpenClaw 用户来说，它最实际的价值是把日常自动化里那些一次性的胶水脚本，沉淀成可复用的 Server——今天接的工单系统，明天换一个客户端还能接着用。协议本身一个下午能跑通第一个 Server；真正值得花时间的，是工具描述的质量和权限边界的划分。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-07/8f8386dd7296ccf4.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-07/cb1cec7be735de66.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-07/8b66379d429bd2b0.png)

