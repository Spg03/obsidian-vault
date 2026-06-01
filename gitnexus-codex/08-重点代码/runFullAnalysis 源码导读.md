---
type: source-guide
status: codex-generated
tags:
  - gitnexus
  - source-reading
---

# runFullAnalysis 源码导读

原始备份里的 `runFullAnalysis().md` 是空文件，这里补成导读页。

## 建议重点看

1. 早期返回逻辑：commit 没变时如何避免重复索引。
2. 缓存加载：解析缓存和嵌入缓存如何保护已有结果。
3. Pipeline 调用：`runPipelineFromRepo` 如何把仓库变成 KnowledgeGraph。
4. 数据落库：图谱如何进入 LadybugDB。
5. 后处理：FTS、Embedding、meta、registry、AI context。

## 一句话

`runFullAnalysis` 是 analyze 的总调度层：它把“解析仓库”扩展成“生成一个可被人和 Agent 使用的本地代码情报系统”。

## 相关笔记

- [[analyze.ts 源码导读]]
- [[Pipeline DAG 实现]]
- [[LadybugDB 图存储]]
