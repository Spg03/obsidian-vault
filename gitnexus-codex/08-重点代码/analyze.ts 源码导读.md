---
type: source-guide
status: codex-generated
tags:
  - gitnexus
  - source-reading
---

# analyze.ts 源码导读

这篇不再复制整份 `analyze.ts`，而是给技术分享准备阅读路线。

## 读代码顺序

1. CLI 参数定义：理解 analyze 支持哪些模式，例如 embeddings、dropEmbeddings、index-only、skills。
2. 环境准备：堆内存、embedding 环境变量、错误处理。
3. `runFullAnalysis` 调用：看 CLI 如何把用户参数转为索引编排参数。
4. 结果处理：状态输出、错误提示、模型下载失败提示。
5. AI context：看 analyze 如何触发 AGENTS.md、CLAUDE.md、Skill 生成。

## 分享时重点

`analyze.ts` 是产品入口，它把复杂能力包装成一个用户能执行的命令。真正的图谱构建在 Pipeline，真正的持久化在 LadybugDB，真正的 Agent 接入在 MCP。

## 相关笔记

- [[runFullAnalysis 编排流程]]
- [[GitNexus 索引构建实现原理]]
