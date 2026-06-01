---
type: implementation
status: codex-generated
tags:
  - gitnexus
  - embedding
---

# Embedding 语义搜索

Embedding 用于语义召回，帮助 GitNexus 找到“词面不一样但语义相关”的代码元素。它不是 GitNexus 的全部，只是搜索层的一部分。

## 典型流程

```text
节点文本
  -> embedding model
  -> 384D vector
  -> vector search
  -> 与 BM25 结果融合
```

## RRF 融合

GitNexus 可以把 BM25 和向量搜索结果用 Reciprocal Rank Fusion 合并。这样既保留关键词精确匹配，也能兼顾语义相似。

## 分享时要避免的误解

不要把 GitNexus 讲成“代码向量库”。Embedding 只能告诉我们“可能相关”，不能证明“谁调用谁”。调用链、影响面和执行流主要来自静态分析图谱。

## 相关笔记

- [[FTS 与 BM25]]
- [[边界与取舍]]
