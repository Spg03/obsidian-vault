---
type: implementation
status: codex-generated
tags:
  - gitnexus
  - mcp
  - stdout
---

# MCP stdout 哨兵机制

MCP stdio 模式要求 stdout 只输出 JSON-RPC 协议消息。如果业务代码随手 `console.log`，就会污染协议流，导致 Agent 无法解析。

GitNexus 用 stdout 哨兵机制拦截非 MCP 输出：真正的 MCP 写入会被标记允许通过，未标记输出会被重定向到 stderr，并做截断和限流。

## 流程图

![[gitnexus-mcp-stdout-sentinel-flow.png]]

## 核心思想

不是 Proxy 包装整个 stdout 对象，而是替换或包裹 stdout 的写入路径。通过 AsyncLocalStorage 标记当前写入是否属于 MCP 协议上下文。

## 为什么这个细节重要

它说明 GitNexus 不只做“代码分析”，还解决了 Agent 工具集成里的真实工程问题：协议通道必须干净，否则再好的工具返回都会被破坏。

## 相关笔记

- [[MCP Server 实现]]
- [[边界与取舍]]
