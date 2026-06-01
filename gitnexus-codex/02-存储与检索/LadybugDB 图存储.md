---
type: implementation
status: codex-expanded
tags:
  - gitnexus
  - ladybugdb
  - storage
---

# LadybugDB 图存储

LadybugDB 是 GitNexus 的本地图数据库层。它把 Pipeline 生成的 KnowledgeGraph 持久化为 `.gitnexus/lbug`，并提供图查询、全文检索、向量检索等能力。

源码入口：

```text
gitnexus/src/core/lbug/schema.ts
gitnexus/src/core/lbug/lbug-adapter.ts
gitnexus/src/core/lbug/csv-generator.ts
gitnexus/src/core/search/fts-indexes.ts
```

## 它解决什么问题

Pipeline 生成的是内存图。内存图适合构建阶段，但不适合长期查询：

- Agent 每次问问题时不能重新解析全仓库。
- Web UI 需要快速加载图节点和边。
- CLI query/context/impact 需要跨进程访问同一份索引。
- Group 跨仓库需要打开多个仓库的索引。

LadybugDB 的作用就是把“离线分析结果”变成“持续可查询的本地数据库”。

## 文件形态

```text
<repo>/.gitnexus/
  lbug       # 主数据库文件
  lbug.wal   # WAL，崩溃恢复
  lbug.lock  # 写锁，避免并发写入冲突
  meta.json  # commit、统计、fileHashes、schemaVersion
```

`meta.json` 不只是展示信息，也参与 early return、增量判断、embedding 保存策略。

## Schema 设计

Schema 入口在 `schema.ts`。核心思想是：

- 节点按 label 分表，例如 File、Function、Class、Method。
- 关系统一用 CodeRelation，并用 `type` 字段区分 CALLS、IMPORTS、DEFINES 等。
- Embedding 单独表，和节点 ID 关联。

这种设计兼顾了两个需求：

1. 节点分表方便按类型查询。
2. 关系统一表方便跨类型图遍历。

## 写入流程

`loadGraphToLbug` 不是逐条 insert。它走的是批量导入思路：

```text
KnowledgeGraph
  -> streamAllCSVsToDisk()
  -> nodes.csv / rels.csv
  -> splitRelCsvByLabelPair()
  -> COPY FROM
  -> create indexes
```

为什么要 CSV？

- 大仓库节点和边很多，逐条写入慢。
- CSV 可以流式生成，避免一次性拼超大 SQL。
- 按 label pair 拆分关系后，批量导入更稳定。

## 关系 CSV 拆分

`splitRelCsvByLabelPair` 会把总关系 CSV 拆成类似：

```text
rel_File_Function.csv
rel_Function_Function.csv
rel_Class_Method.csv
```

它还处理了几个工程细节：

- line-by-line 流式读取。
- 校验 from/to label 是否有效。
- 写流 backpressure。
- 输入流和输出流异常时清理文件句柄。

这些细节看起来琐碎，但决定了它在 Windows 和大仓库上的稳定性。

## 锁、WAL 和恢复

`lbug-adapter.ts` 里有不少看似“边角”的逻辑：

- init lock：防止多个进程同时初始化数据库。
- BUSY retry：写锁被占用时做有限重试。
- WAL corruption / sidecar recovery：处理崩溃或 shadow sidecar 异常。
- Windows handle release：避免文件句柄尚未释放就删除目录。

这些点说明 LadybugDB 层不是简单的存取封装，而是在处理本地数据库真实运行时问题。

## FTS 和 Embedding 的位置

FTS 和 Embedding 不是图谱本体，但和 LadybugDB 一起成为查询后端：

- FTS：关键词召回和 BM25 排序。
- Embedding：语义召回。
- RRF：混合排序。

注意：Embedding 只帮助找相关节点，不证明调用关系。调用和影响分析仍依赖图边。

## 分享时的讲法

可以把 LadybugDB 讲成 GitNexus 的“本地代码情报仓库”。Pipeline 是生产线，LadybugDB 是仓库，MCP/HTTP/CLI 是前台。

## 相关笔记

- [[图谱 Schema 速览]]
- [[FTS 与 BM25]]
- [[Embedding 语义搜索]]
- [[runFullAnalysis 编排流程]]
