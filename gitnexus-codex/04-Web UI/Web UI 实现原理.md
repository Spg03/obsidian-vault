---
type: implementation
status: codex-generated
tags:
  - gitnexus
  - web-ui
---

# Web UI 实现原理

GitNexus Web UI 是浏览器侧的可视化和交互入口。它不是另一个索引器，而是通过 `gitnexus serve` 暴露的 HTTP API 查询同一份 LadybugDB。

## 架构图

![[gitnexus-web-ui-architecture.png]]

## 分层

| 层 | 技术 | 作用 |
|---|---|---|
| React SPA | Vite + React | 展示图谱、详情、聊天、设置 |
| Graph Visualization | Sigma 等 | 浏览节点和关系 |
| AI Chat | LangChain Agent | 在浏览器里做交互式问答 |
| Express backend | `gitnexus serve` | 提供 REST、SSE、MCP HTTP endpoint |
| LadybugDB | 本地图数据库 | 所有查询来源 |

## 设计重点

Web UI 的价值在于把图谱变成可见的工程地图，适合分享、探索和人工审阅。MCP 更适合 Agent 自动调用，Web UI 更适合人理解和验证。

## 相关笔记

- [[浏览器端 Graph RAG Agent]]
- [[SSE 心跳与自动重连]]
