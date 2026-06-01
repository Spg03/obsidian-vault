---
type: implementation
status: codex-generated
tags:
  - gitnexus
  - search
  - bm25
---

# FTS 与 BM25

FTS 是全文检索层，BM25 是关键词相关性排序算法。GitNexus 用它补足图遍历之外的“文本发现能力”。

## 适合解决的问题

- 我只记得一个概念，不记得符号名。
- 我要找包含某个业务词的函数、类或文件。
- query 工具需要先召回候选，再结合流程和语义排序。

## 和图遍历的关系

BM25 负责找到“可能相关”的节点，图谱负责解释这些节点之间的工程关系。两者合起来比单纯 grep 更适合 Agent，因为结果不只是文件列表，而能继续 context、impact、process 追踪。

## 相关笔记

- [[Embedding 语义搜索]]
- [[MCP 工具清单]]
