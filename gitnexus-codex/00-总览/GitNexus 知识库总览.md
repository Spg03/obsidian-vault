---
type: moc
status: codex-expanded
tags:
  - gitnexus
  - 技术分享
---

# GitNexus 知识库总览

这套笔记的定位不是“GitNexus 使用说明”，而是面向程序员的实现原理分享。主线只有一条：

> GitNexus 把代码库从“文本集合”预计算成“工程关系图谱”，再把这张图通过 MCP、HTTP、CLI 和 Agent 规则注入到开发工作流里。

如果只说“它能帮 AI 理解代码”，听众很难信服。应该把它拆成四个工程问题：

1. 代码如何结构化：源码如何从文件、AST、符号、调用候选变成图节点和边。
2. 关系如何预计算：调用、导入、继承、路由、工具、流程、社区如何提前算好。
3. 查询如何工程化：LadybugDB、FTS、Embedding、MCP、HTTP 如何让这些关系可查询。
4. Agent 如何被约束：tool description、next-step hint、AGENTS.md、CLAUDE.md、Skill 如何让 Agent 形成固定工作流。

## 推荐阅读顺序

第一层先建立全局心智模型：

1. [[GitNexus 技术分享路线]]
2. [[源码入口地图]]
3. [[术语表]]
4. [[边界与取舍]]

第二层讲核心链路：

1. [[GitNexus 索引构建实现原理]]
2. [[runFullAnalysis 编排流程]]
3. [[Pipeline DAG 实现]]
4. [[Tree-sitter 解析层]]
5. [[LadybugDB 图存储]]

第三层讲 Agent 接入：

1. [[MCP Server 实现]]
2. [[MCP 工具清单]]
3. [[MCP stdout 哨兵机制]]
4. [[Prompt Skill AGENTS 注入机制]]
5. [[Agent 工作流]]

第四层讲扩展能力：

1. [[Web UI 实现原理]]
2. [[Group 实现原理]]
3. [[Wiki 生成实现原理]]

## 一句话架构

```text
Repository
  -> analyze / Pipeline
  -> KnowledgeGraph
  -> LadybugDB + FTS + Embedding
  -> MCP / HTTP / CLI
  -> Agent 工作流约束
```

## 讲给程序员听的重点

不要把 GitNexus 讲成“更聪明的 grep”，也不要讲成“把代码塞进向量库”。更准确的说法是：

- grep 只能告诉你文本在哪。
- RAG 主要告诉你哪些文本语义相似。
- GitNexus 试图告诉 Agent：谁定义了谁、谁调用了谁、从哪个入口进来、属于哪个执行流、改了会影响哪些调用方。

这就是它更适合 Agent 编程的原因：Agent 改代码前最缺的不是“再多读几段文本”，而是“知道应该读哪些关系、哪些路径、哪些风险”。

## 关键图片

![[gitnexus-mcp-architecture-v2.png]]

![[gitnexus-web-ui-architecture.png]]

![[gitnexus-group-data-flow-overview.png]]

## 相关笔记

- [[源码入口地图]]
- [[GitNexus 技术分享路线]]
- [[Agent 工作流]]
