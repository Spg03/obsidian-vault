---
type: implementation
status: codex-expanded
tags:
  - gitnexus
  - mcp
  - agent
---

# MCP Server 实现

MCP Server 是 GitNexus 给 AI Agent 暴露代码图谱能力的协议层。源码入口：

```text
gitnexus/src/mcp/server.ts
gitnexus/src/mcp/tools.ts
gitnexus/src/mcp/resources.ts
gitnexus/src/mcp/local/local-backend.ts
gitnexus/src/server/mcp-http.ts
```

## 架构图

![[gitnexus-mcp-architecture-v2.png]]

## 分层讲解

```text
Agent
  -> Transport(stdio / streamable HTTP)
  -> MCP Server SDK
  -> tool/resource/prompt handlers
  -> LocalBackend
  -> LadybugDB / registry / git diff / group bridge
```

这层的关键是协议和实现分离：

- MCP Server 负责注册工具、资源、提示词，并把请求转发出去。
- LocalBackend 负责真正执行 query/context/impact 等逻辑。
- LadybugDB 负责存储和查询图谱。

## createMCPServer 做了什么

`createMCPServer(backend)` 里注册三类 handler：

### 1. Resources

处理：

- `ListResourcesRequestSchema`
- `ListResourceTemplatesRequestSchema`
- `ReadResourceRequestSchema`

资源适合读取稳定上下文，例如：

- repo context
- clusters
- processes
- schema
- group contracts/status

### 2. Tools

处理：

- `ListToolsRequestSchema`
- `CallToolRequestSchema`

工具定义来自 `GITNEXUS_TOOLS`。调用工具时，server 会执行：

```text
backend.callTool(name, args)
  -> stringify result
  -> append getNextStepHint()
  -> return text content
```

### 3. Prompts

处理：

- `ListPromptsRequestSchema`
- `GetPromptRequestSchema`

当前有典型 prompt：

- `detect_impact`
- `generate_map`

Prompt 不是工具调用结果，而是任务模板，适合引导 Agent 做完整流程。

## next-step hint 是什么

`getNextStepHint` 是一个很值得讲的设计。很多 Agent 调完一个工具就停了，所以 GitNexus 在工具结果后追加下一步提示：

| 当前工具 | 下一步提示 |
|---|---|
| list_repos | 读取 repo context |
| query | 对具体符号调用 context |
| context | 修改前调用 impact |
| impact | 先看 d=1 直接影响项 |
| detect_changes | 对高风险符号继续 context |
| rename | 运行 detect_changes |

这相当于把工作流“嵌入工具响应”。它不是强制执行，但能显著提高 Agent 顺着正确路径走的概率。

## Tool description 也是 Prompt

`tools.ts` 里每个工具的 description 都不是普通文档，而是工具级 prompt。里面有：

- WHEN TO USE
- AFTER THIS
- GROUP MODE
- SERVICE
- SCHEMA
- EXAMPLES

这让模型在选择工具时不只是看工具名，还能理解使用时机和下一步动作。

## stdio 和 HTTP 两种入口

MCP 有两种典型 transport：

- stdio：Claude Code、Cursor、Codex 等本地工具常用。
- HTTP streamable：Web 或远程客户端可以通过 `/api/mcp` 接入。

两种 transport 最终都会进入同一个 `createMCPServer`，所以工具实现不需要关心传输层。

## 分享时的重点

MCP Server 不只是“把函数暴露成工具”。GitNexus 在这里做了三件事：

1. 把本地图谱查询能力标准化成 Agent 可调用工具。
2. 把 repo context、schema、processes 暴露成稳定资源。
3. 用 tool description 和 next-step hint 影响 Agent 行为。

## 相关笔记

- [[MCP 工具清单]]
- [[MCP stdout 哨兵机制]]
- [[Prompt Skill AGENTS 注入机制]]
- [[Agent 工作流]]
