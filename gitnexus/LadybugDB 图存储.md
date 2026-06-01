
```markdown
# GitNexus 中 LadybugDB 的嵌入式图数据库实现

LadybugDB 在 GitNexus 里不是“直接存一坨 JSON 图”，而是作为**嵌入式图数据库**承接 `KnowledgeGraph` 的持久化。实现思路可以概括成：

```text
KnowledgeGraph 内存图
  -> 流式生成 CSV
  -> 创建 LadybugDB schema
  -> COPY 批量导入节点
  -> 按 from/to 类型拆分关系 CSV
  -> COPY 批量导入 CodeRelation
  -> 建 FTS / Vector 索引
  -> MCP / CLI / HTTP 用 Cypher 查询
```

## 1. Schema 设计：多节点表 + 单关系表

GitNexus 采用 hybrid schema：

- 每种节点一个表：`File`、`Folder`、`Function`、`Class`、`Method`、`Route`、`Tool`、`Community`、`Process` 等。
- 所有边统一进一张关系表：`CodeRelation`。
- 关系类型不拆表，而是放在 `CodeRelation.type` 字段里。

调用关系、导入关系、继承关系、执行流步骤都存在同一张关系表中：

```cypher
MATCH (a)-[r:CodeRelation {type: "CALLS"}]->(b)
RETURN a.name, b.name, r.confidence
```

`CodeRelation` 主要字段：

```text
type        CALLS / IMPORTS / EXTENDS / MEMBER_OF / STEP_IN_PROCESS ...
confidence 关系置信度
reason     解析原因，例如 import-resolved、same-file、global
step       流程步骤序号，主要给 Process 用
```

对应源码在 [schema.ts](E:/test/GitNexus/gitnexus/src/core/lbug/schema.ts:1)。

## 2. 落库位置

每个仓库的索引结果存在自己的 `.gitnexus/` 目录下：

```text
<repo>/.gitnexus/
├── lbug        LadybugDB 主数据库
├── lbug.wal    写前日志
├── lbug.lock   写锁
└── meta.json   lastCommit、indexedAt、stats、fileHashes
```

同时全局注册到：

```text
~/.gitnexus/registry.json
```

这样 MCP server 启动后就能发现所有已索引仓库。

相关实现是 [repo-manager.ts](E:/test/GitNexus/gitnexus/src/storage/repo-manager.ts:1)。

## 3. 写入流程：不是一条条 INSERT，而是 CSV + COPY

真正落库发生在 `runFullAnalysis()` 里：

```text
runPipelineFromRepo()
  -> 得到 KnowledgeGraph
  -> initLbug(lbugPath)
  -> loadGraphToLbug(graph, repoPath, storagePath)
  -> createSearchFTSIndexes()
  -> 保存 meta.json
  -> registerRepo()
```

关键函数是 [loadGraphToLbug](E:/test/GitNexus/gitnexus/src/core/lbug/lbug-adapter.ts:814)。

它做的事情是：

```text
KnowledgeGraph
  -> streamAllCSVsToDisk()
  -> COPY node CSVs
  -> split relationship CSV by from/to pair
  -> COPY relationship CSVs
```

为什么用 CSV？因为代码仓库可能很大，如果逐条 `INSERT`，性能会很差；批量 `COPY` 更适合大图导入。

## 4. CSV 生成：单次遍历图，懒加载源码内容

`csv-generator.ts` 会遍历 `KnowledgeGraph`，把不同类型的节点写入不同 CSV：

```text
file.csv
function.csv
class.csv
method.csv
community.csv
process.csv
...
relations.csv
```

它还有几个细节：

- 文件内容按需读取，不一次性把整个仓库读进内存。
- 有一个 LRU 文件内容缓存，避免同一个文件被多个 symbol 反复读取。
- 代码片段会截断，避免 CSV 过大。
- CSV 严格做 RFC 4180 转义，处理引号、换行、逗号。
- 每 500 行 buffer flush 一次，减少频繁 IO。

对应源码在 [csv-generator.ts](E:/test/GitNexus/gitnexus/src/core/lbug/csv-generator.ts:1)。

## 5. 关系为什么要按 from/to 拆分？

这是一个很关键的实现细节。

LadybugDB 的关系表虽然叫统一的 `CodeRelation`，但关系 schema 里声明了很多合法的起点/终点组合：

```text
FROM Function TO Method
FROM Class TO Method
FROM File TO Function
FROM Route TO Process
...
```

导入关系时，LadybugDB 的 `COPY` 需要知道当前这批关系的 `from` 和 `to` label。

所以 GitNexus 先把总的关系 CSV 拆成：

```text
rel_Function_Method.csv
rel_File_Function.csv
rel_Class_Method.csv
...
```

然后分别执行类似：

```cypher
COPY CodeRelation
FROM "rel_Function_Method.csv"
(from="Function", to="Method", HEADER=true, ...)
```

这部分在 [lbug-adapter.ts](E:/test/GitNexus/gitnexus/src/core/lbug/lbug-adapter.ts:866)。

## 6. 查询层如何使用

导入完成后，MCP / CLI / HTTP 都通过后端执行 Cypher 查询：

```text
MCP tool / CLI command / HTTP API
  -> LocalBackend
  -> executeQuery / streamQuery
  -> LadybugDB
```

例如：

```cypher
MATCH (caller)-[r:CodeRelation {type: "CALLS"}]->(target:Function {name: "validateUser"})
RETURN caller.name, caller.filePath, r.confidence
```

所以 `impact`、`context`、`query` 本质上都是在这张图上做不同形态的图查询和结果包装。

## 7. 搜索和 embedding 怎么存

LadybugDB 不只存结构图，还承接搜索能力：

- FTS：对 `File`、`Function`、`Class`、`Method` 等表建全文索引。
- Embedding：单独有 `CodeEmbedding` 表。
- 向量维度默认 384，对应 Snowflake arctic-embed-xs。
- 支持 HNSW vector index，平台支持时会创建向量索引。

Embedding schema 在 [schema.ts](E:/test/GitNexus/gitnexus/src/core/lbug/schema.ts:303)，FTS 创建入口在 [fts-indexes.ts](E:/test/GitNexus/gitnexus/src/core/search/fts-indexes.ts:1)。

## 一句话总结

> LadybugDB 在 GitNexus 里承担的是“代码知识图谱的本地持久化层”：  
> Pipeline 先在内存里构建 `KnowledgeGraph`，再通过 CSV + COPY 批量写入 LadybugDB；节点按类型分表，所有关系统一进 `CodeRelation`；查询时 MCP、CLI、HTTP 都用 Cypher 查同一张图，从而支持 `context`、`impact`、`query`、`detect_changes` 这些 Agent 工具。
