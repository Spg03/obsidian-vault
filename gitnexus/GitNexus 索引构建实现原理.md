一、总览：从 analyze 命令到一张可查询的知识图谱

  gitnexus analyze ──→ runFullAnalysis() ──→ 10 步编排流程
                           │
                           ├── 1. 早期返回检查
                           ├── 2. 嵌入缓存
                           ├── 3. runPipelineFromRepo() ← 12 阶段 DAG
                           ├── 4. LadybugDB 入库
                           ├── 5. FTS 全文索引
                           ├── 6. 嵌入恢复
                           ├── 7. 嵌入生成
                           ├── 8. 元数据持久化
                           ├── 9. 注册 + .gitignore
                           └── 10. AI 上下文文件

 核心文件：

| 文件                                       | 职责                              |
| ---------------------------------------- | ------------------------------- |
| cli/analyze.ts                           | CLI 入口：解析参数、管理堆内存（16GB re-exec） |
| core/run-analyze.ts                      | 10 步总编排器                        |
| core/ingestion/pipeline.ts               | 管道入口：创建图，注册阶段，调用 runner         |
| core/ingestion/pipeline-phases/runner.ts | 拓扑排序 + 顺序执行                     |
| core/ingestion/pipeline-phases/*.ts      | 12 个具体阶段实现                      |
| core/lbug/lbug-adapter.ts                | LadybugDB 图数据库操作                |
| core/lbug/csv-generator.ts               | 图 → CSV 流式写入                    |
| core/search/fts-indexes.ts               | FTS 全文索引                        |
| core/embeddings/embedding-pipeline.ts    | 嵌入生成管道                          |

  ---
  二、CLI 入口：[[analyze.ts]]

  用户敲下: gitnexus analyze --embeddings --skills --verbose
                │
                ▼
       cli/analyze.ts: analyzeCommand()
                │
                ├── ensureHeap(16GB)    ← 重新以 16GB 堆 exec 自身
                ├── 解析 CLI 参数
                └── runFullAnalysis(repoPath, options, callbacks)

  CLI 层只做三件事：
  1. 堆内存管理 — 如果当前堆 < 16GB，用 --max-old-space-size=16384 重新 exec 自己
  2. 参数解析 — Commander.js 把 flag 转成 AnalyzeOptions
  3. 进度回调 — 用 ora spinner 把 onProgress 事件渲染成进度条

  ---
  三、10 步编排器：[[runFullAnalysis()]]

  这是 run-analyze.ts 中的核心函数（~800 行），是整条流水线的"总指挥"：

  runFullAnalysis(repoPath, options, callbacks)

    Step 1 ── 清理旧 KuzuDB 文件（迁移残留）
    Step 2 ── 早期返回：lastCommit == HEAD 且工作区干净 → 直接返回
    Step 3 ── 加载嵌入缓存（下次运行保留已有嵌入）
    Step 4 ── 加载增量解析缓存（内容寻址，跨 --force 重用）
    Step 5 ── runPipelineFromRepo() → KnowledgeGraph [0-60%]
    Step 6 ── 计算文件哈希 → 决定增量/全量模式
    Step 7 ── LadybugDB 入库 [60-85%]
    Step 8 ── FTS 全文索引 [85-90%]
    Step 9 ── 嵌入恢复 + 生成 [88-98%]
    Step 10 ── 元数据 + 注册 + .gitignore + AI 上下文 [98-100%]

  增量 vs 全量判断（Step 6）

  const isIncremental =
    !options.force &&
    !!existingMeta &&
    existingMeta.schemaVersion === INCREMENTAL_SCHEMA_VERSION &&
    !!existingMeta.fileHashes &&
    Object.keys(existingMeta.fileHashes).length > 0 &&
    repoHasGit &&
    allFilePaths.length > 0;

  全部条件满足 → 走增量写回路径（只替换变化文件的行）；任一条件不满足 → 删除 DB 文件，全量重建。

  增量写回的核心逻辑：

  4. diffFileHashes() — 对比新旧文件哈希，产出 {changed, added, deleted, toWrite}
  5. 传染性扩展 — 对 changed/deleted 文件做 BFS（深度 ≤ 4），沿 IMPORTS 链把"可能受影响"的导入者也拉进来
  6. Shadow-seed — 新增文件可能"抢夺"已有文件的模块解析结果，将这些候选者也种子进 BFS 前沿
  7. computeEffectiveWriteSet() — 遍历新图的边，把跨越"已变/未变"边界的节点也拉进来
  8. deleteNodesForFile() — 先删除写集内文件的旧行
  9. extractChangedSubgraph() — 从完整图中提取变化子图
  10. loadGraphToLbug() — 只写子图到 DB

  ---
  四、管道引擎：12 阶段 DAG

  4.1 拓扑排序执行器：runner.ts

  // runner.ts 核心流程
  export async function runPipeline(phases, ctx) {
    // 1. Kahn 拓扑排序
    const sorted = topologicalSort(phases);

    // 2. 顺序执行
    for (const phase of sorted) {
      // 只暴露声明式依赖的输出（防止隐藏耦合）
      const declaredDeps = new Map();
      for (const depName of phase.deps) {
        declaredDeps.set(depName, results.get(depName));
      }
      // 执行 + 计时
      const output = await phase.execute(ctx, declaredDeps);
      results.set(phase.name, { phaseName, output, durationMs });
    }
  }

  验证逻辑：
  - 重复阶段名 → 抛错
  - 依赖了不存在的阶段 → 抛错
  - 循环依赖 → DFS 追踪具体环路径（如 A -> B -> C -> A），并报告被阻塞的传递依赖数量

  关键设计决策：
  - 声明式依赖过滤：declaredDeps 只包含 phase.deps 中声明的阶段，即使 results 里有更多已完成阶段也不传入
  - 错误封装：阶段崩溃时包装为 Phase 'X' failed: reason，同时发射 error 进度事件
  - Dev 模式计时：每个阶段打印耗时

  4.2 完整阶段列表

  scan → structure → [markdown, cobol] → parse → [routes, tools, orm]
    → crossFile → scopeResolution → mro → communities → processes

| #   | 阶段              | 文件                                 | 依赖                                    | 做什么                                                |     |
| --- | --------------- | ---------------------------------- | ------------------------------------- | -------------------------------------------------- | --- |
| 1   | scan            | scan.ts                            | (无)                                   | 遍历文件系统，返回 {path, size}[]，不读文件内容                    |     |
| 2   | structure       | structure.ts                       | scan                                  | 创建 File/Folder 节点 + CONTAINS 边，构建 allPathSet       |     |
| 3   | markdown        | markdown.ts                        | structure                             | 解析 .md/.mdx，提取章节节点和交叉链接                            |     |
| 4   | cobol           | cobol.ts                           | structure                             | COBOL 程序/段落提取（正则，不用 tree-sitter）                   |     |
| 5   | parse           | parse.ts + parse-impl.ts           | structure, markdown, cobol            | 核心：tree-sitter 解析 → 符号节点 + IMPORTS/CALLS/EXTENDS 边 |     |
| 6   | routes          | routes.ts                          | parse                                 | 识别 HTTP 路由（Next.js, Express, PHP, 装饰器）             |     |
| 7   | tools           | tools.ts                           | parse                                 | 识别 MCP/RPC 工具定义                                    |     |
| 8   | orm             | orm.ts                             | parse                                 | 识别 ORM 查询（Prisma, Supabase）                        |     |
| 9   | crossFile       | cross-file.ts                      | parse, routes, tools, orm             | 跨文件类型传播（拓扑导入顺序）                                    |     |
| 10  | scopeResolution | scope-resolution/pipeline/phase.ts | parse, structure                      | RFC #909 新一代作用域解析（迁移语言：Python, C#, JS）             |     |
| 11  | mro             | mro.ts                             | crossFile, structure                  | METHOD_OVERRIDES + METHOD_IMPLEMENTS 边             |     |
| 12  | communities     | communities.ts                     | mro, structure                        | Leiden 算法社区检测 → 功能模块聚类                             |     |
| 13  | processes       | processes.ts                       | communities, routes, tools, structure | 从路由/工具出发追踪执行流 → 流程节点                               |     |
  4.3 各阶段详解

  Phase 1: scan — 文件扫描

  // scan.ts
  export const scanPhase: PipelinePhase<ScanOutput> = {
    name: 'scan',
    deps: [],
    async execute(ctx) {
      const scannedFiles = await walkRepositoryPaths(ctx.repoPath, (current, total, filePath) => {
        // 进度回调
      });
      return { scannedFiles, allPaths, totalFiles };
    },
  };

  调用 walkRepositoryPaths()，内部：
  1. 读取 .gitignore / .gitnexusignore 规则
  2. 递归遍历目录
  3. 过滤：跳过 node_modules、.git 等
  4. 大小过滤：跳过 >512KB 的文件（可通过 --max-file-size 或 GITNEXUS_MAX_FILE_SIZE 调整，上限 32768KB）
  5. 返回 {path, size}[]

  不读取文件内容 — 只拿路径和大小。

  Phase 2: structure — 结构构建

  // structure.ts
  export const structurePhase: PipelinePhase<StructureOutput> = {
    name: 'structure',
    deps: ['scan'],
    async execute(ctx, deps) {
      const { scannedFiles, allPaths, totalFiles } = getPhaseOutput(deps, 'scan');
      processStructure(ctx.graph, allPaths);
      const allPathSet: ReadonlySet<string> = new Set(allPaths);
      return { scannedFiles, allPaths, allPathSet, totalFiles };
    },
  };

  processStructure() 做的事：
  6. 为每个文件创建 File 节点（属性：filePath, extension, size）
  7. 为每个目录创建 Folder 节点（属性：folderPath）
  8. 创建 CONTAINS 边：Folder → File / Folder → Folder
  9. 记录语言信息到 File 节点的 language 属性

  allPathSet 在这里构建一次，后续阶段（cobol, markdown, crossFile）直接复用，避免每个阶段都 new Set(allPaths)。

  Phase 3-4: markdown + cobol — 特殊文件处理

  markdown： 解析 .md/.mdx 文件，提取：
  - Section 节点 — 文档章节（#, ##, ###）
  - Cross-link 边 — Markdown 内部链接 ([text](./other.md))

  cobol： COBOL 程序解析（正则匹配，不走 tree-sitter），提取：
  - Program / Paragraph / Section 节点
  - 调用关系

  这两个阶段与 parse 并行（都只依赖 structure，互不依赖）。

  Phase 5: parse — 核心解析（最复杂的阶段）

  // parse.ts
  export const parsePhase: PipelinePhase<ParseOutput> = {
    name: 'parse',
    deps: ['structure', 'markdown', 'cobol'],
    async execute(ctx, deps) {
      const { scannedFiles, allPaths, allPathSet, totalFiles } =
        getPhaseOutput<StructureOutput>(deps, 'structure');
      const result = await runChunkedParseAndResolve(
        ctx.graph, scannedFiles, allPaths, totalFiles, ...
      );
      return { ...result, allPaths, allPathSet, totalFiles };
    },
  };

  runChunkedParseAndResolve() 在 parse-impl.ts 中，核心流程：

                            ┌─ 有缓存 → 直接回放 ParseWorkerResult[]
   chunkedFiles (2MB/块) ──┤
                            └─ 无缓存 → 分发到 Worker 池
                                            │
                                            ▼
                                ┌─── detectLanguage(filePath)
                                ├─── loadGrammar(lang)
                                ├─── parser.parse(source)  → Tree
                                ├─── runQueries(Tree)       → 捕获节点
                                ├─── extractImports()       → import 信息
                                ├─── extractCalls()         → 调用信息
                                ├─── extractHeritage()      → 继承信息
                                └─── extractParsedFile()     → ParsedFile (RFC #909)
                                            │
                                            ▼
                                主线程收集所有 Worker 结果：
                                ├─── 写入 Symbol 节点到 ctx.graph
                                ├─── 写入 IMPORTS/CALLS/EXTENDS/IMPLEMENTS 边
                                ├─── 合成通配符绑定（wildcard imports）
                                ├─── 构建 exportedTypeMap（跨文件类型传播用）
                                └─── 收集 fetchCalls, routes, tools, ORM queries

  分块机制： 文件按字节预算（默认 2MB）分组为 chunk，每个 chunk 分发到 Worker 池。目的：
  - 控制内存峰值（不一次性加载所有文件）
  - 粒度适中的缓存键（内容寻址，SHA1）

  Worker 池： workers/worker-pool.ts — 核心数量 = cpuCores - 1，上限 16。每个 slot 维护一个长期存活的 Worker
  线程，支持：
  - 零拷贝数据传输（SharedArrayBuffer）
  - 崩溃恢复（per-slot 重生限制）
  - 断路器（连续失败达阈值 → 关闭该 slot）

  内容寻址缓存： parse-cache.ts — 以 chunk 文件的 SHA1 为 key：
  chunkHash = SHA1(file1Content + file2Content + ...)
    → 命中：跳过 worker，直接复用上次的 ParseWorkerResult[]
    → 未命中：跑 worker，结果写入缓存

  --force 不清理缓存，因为内容是幂等的。

  Phase 6-8: routes, tools, orm — 语义提取

  这三个阶段并行（都只依赖 parse），从 parse 产出的符号图中提取特定语义：

  routes：
  - Next.js 文件路由（pages/, app/）
  - Express/PHP 路由定义
  - 装饰器路由（@Get, @Post 等）
  - 产出：Route 节点 + HANDLES_ROUTE 边

  tools：
  - MCP 工具定义
  - RPC 工具定义
  - 产出：Tool 节点 + HANDLES_TOOL 边

  orm：
  - Prisma 查询
  - Supabase 查询
  - 产出：FETCHES 边（controller → database）

  Phase 9: crossFile — 跨文件类型传播

  拓扑导入顺序处理每个文件：
    for each file (按 import 依赖的拓扑序):
      1. 从 importedTypeMap 获取上游文件的类型导出
      2. 对本文件中使用了上游类型的调用进行类型细化
      3. 更新 CALLS 边的置信度
      4. 本地修改变更传播到 exportedTypeMap

  这是确保"类型信息跨文件流动"的关键阶段。例如：
  // a.ts: export class User { getName() {} }
  // b.ts: import { User } from './a';
  //        const u = new User();
  //        u.getName();  ← crossFile 阶段才确定这是 User.getName()

  Phase 10: scopeResolution — RFC #909 新一代解析

  针对已迁移语言（Python, C#, JavaScript），用作用域注册表替代旧 DAG：

  ParsedFile[] → ScopeResolutionIndexes → ReferenceIndex → KnowledgeGraph 边

  与旧 DAG 并存，通过 isRegistryPrimary(lang) 开关控制。CI 上两条管线都跑，结果必须一致。

  Phase 11: mro — 方法解析顺序

  遍历所有类的 EXTENDS/IMPLEMENTS 边：
    ├── Python: C3 线性化
    ├── Java/C#/C++/TS/Ruby/Go: first-wins
    └── 其他: none

  产出：
    ├── METHOD_OVERRIDES 边 (子类覆盖父类方法)
    └── METHOD_IMPLEMENTS 边 (类实现接口方法)

  Phase 12: communities — 社区检测

  使用 Leiden 算法 在图结构上检测功能模块聚类：
  - 输入：File/Folder 结构 + 符号调用关系 + 继承关系
  - 输出：Community 节点 + MEMBER_OF 边
  - 每个 Community 有一个 heuristicLabel（如 "Authentication", "Database"）

  Phase 13: processes — 执行流检测

  从 HTTP 路由 / MCP 工具入口出发，沿 CALLS 边做 DFS/BFS 追踪：
  - 输入：Route 节点 + Tool 节点 + 调用图
  - 输出：Process 节点 + STEP_IN_PROCESS 边
  - 每个 Process 代表一个完整的"请求 → 处理 → 响应"执行路径

  ---
  五、LadybugDB 存储层

  5.1 图 → CSV → 批量导入

  KnowledgeGraph (内存)
        │
        ▼
  streamAllCSVsToDisk()  ← csv-generator.ts
        │
        ├── 为每种节点类型生成一个 CSV 文件
        │   (File.csv, Function.csv, Class.csv, ...)
        │
        ├── 为每条关系生成一行到 rel.csv
        │   (fromNodeId, toNodeId, type, reason, confidence)
        │
        ▼
  splitRelCsvByLabelPair()  ← lbug-adapter.ts
        │
        ├── 将 rel.csv 按 (fromLabel, toLabel) 拆分为多个小 CSV
        │   (rel_File_Folder.csv, rel_Function_Calls_Function.csv, ...)
        │
        ▼
  COPY FROM 批量导入  ← LadybugDB 原生语法
        │
        └── 每个 CSV 一个 COPY FROM 语句，比逐行 INSERT 快 10-100 倍

  为什么拆分关系 CSV？
  - LadybugDB 的节点表按类型分开存储（File 表、Function 表等）
  - COPY FROM 对每种（fromLabel, toLabel）组合更高效
  - 同一对组合内的关系批量插入，减少索引更新开销

  5.2 图 Schema

  节点表（44 种类型，NODE_TABLES）：

  // schema.ts
  export const NODE_TABLES: Record<NodeTableName, { label: string; properties: PropertyDef[] }> = {
    File:       { label: 'File',       properties: [{ name: 'filePath', type: 'STRING' }, ...] },
    Folder:     { label: 'Folder',     properties: [{ name: 'folderPath', type: 'STRING' }, ...] },
    Function:   { label: 'Function',   properties: [{ name: 'name', type: 'STRING' }, ...] },
    Class:      { label: 'Class',      properties: [{ name: 'name', type: 'STRING' }, ...] },
    Method:     { label: 'Method',     properties: [...] },
    // ... 共 44 种
    Embedding:  { label: 'Embedding',  properties: [{ name: 'embedding', type: 'FLOAT[384]' }, ...] },
  };

  关系表（单表，21 种关系类型，REL_TABLE_NAME）：

  CREATE REL TABLE CodeRelation (
    FROM <node_table> TO <node_table>,
    type STRING,        -- CONTAINS, CALLS, IMPORTS, EXTENDS, ...
    reason STRING,      -- 'import-resolved', 'global', 'local-call', ...
    confidence DOUBLE,  -- 0.0 ~ 1.0
    ...
  )

  关系类型：CONTAINS, DEFINES, CALLS, IMPORTS, EXTENDS, IMPLEMENTS, HAS_METHOD, HAS_PROPERTY, ACCESSES,
  METHOD_OVERRIDES, METHOD_IMPLEMENTS, MEMBER_OF, STEP_IN_PROCESS, HANDLES_ROUTE, FETCHES, HANDLES_TOOL,
  ENTRY_POINT_OF。

  ---
  六、FTS 全文索引

  // fts-indexes.ts
  export async function createSearchFTSIndexes(options) {
    for (const { table, indexName, properties } of FTS_INDEXES) {
      await createFTSIndex(table, indexName, [...properties]);
    }
  }

对以下节点表建 FTS 索引（全文搜索）：
  
| 表        | 索引字段                     |
|-----------|------------------------------|
| File      | filePath, name               |
| Function  | name, qualifiedName          |
| Class     | name, qualifiedName          |
| Method    | name, qualifiedName          |
| Interface | name, qualifiedName          |
| ...       | ...                          |

底层是 LadybugDB 的 FTS 扩展，支持 BM25 算法。

  ---
  七、嵌入（Embeddings）

  // embedding-pipeline.ts
  export async function runEmbeddingPipeline(query, batchInsert, onProgress, opts, ...) {
    // 1. 加载模型：Snowflake arctic-embed-xs (384维)
    // 2. 对可嵌入节点生成文本描述
    // 3. 模型推理 → 384维浮点向量
    // 4. 批量写入 Embedding 表
  }

  可嵌入类型： File, Function, Class, Method, Interface

  文本描述生成： 对每个节点，组合其名称、签名、所在文件、上下文等信息，生成一段自然语言描述，再交给嵌入模型编码。

  双重用途：
  - 有 VECTOR 索引 → 向量语义搜索（ANN）
  - 无 VECTOR 索引 → 精确扫描回退（exact-scan，限制 10000 chunk）

  ---
  八、元数据与注册

  // .gitnexus/meta.json
  {
    "repoPath": "/path/to/repo",
    "lastCommit": "abc123...",
    "indexedAt": "2026-05-24T...",
    "remoteUrl": "https://github.com/user/repo",
    "stats": {
      "files": 149,
      "nodes": 5230,
      "edges": 12450,
      "communities": 28,
      "processes": 15,
      "embeddings": 342
    },
    "schemaVersion": 2,
    "fileHashes": { "src/main.ts": "sha1...", ... },
    "incrementalInProgress": null
  }

  // ~/.gitnexus/registry.json
  {
    "repos": {
      "sky-take-out-project": {
        "path": "/path/to/repo",
        "name": "sky-take-out-project",
        "lastCommit": "abc123...",
        "indexedAt": "...",
        "stats": { ... }
      }
    }
  }

  ---
  九、增量索引完整流程

  第二次运行 gitnexus analyze：
    │
    ├── meta.json 存在，lastCommit !== HEAD
    ├── 加载 parse-cache（内容寻址，跨 --force 复用）
    ├── 加载 embedding cache（保留已有嵌入）
    │
    ├── 跑完整 12 阶段 DAG（解析所有文件）
    │   └── parse-cache 命中 → 跳过 worker 解析
    │
    ├── 计算文件哈希差异
    │   changed: [A, B]  added: [C]  deleted: [D]
    │
    ├── 传染性扩展 (BFS ≤ 4):
    │   queryImporters(A) → [E, F]  (E import 了 A)
    │   queryImporters(B) → [G]
    │   writableFiles = {A, B, C, D, E, F, G}
    │
    ├── deleteNodesForFile(A, B, C, D, E, F, G)
    │
    ├── extractChangedSubgraph(graph, writableFiles)
    │
    ├── loadGraphToLbug(subgraph)  ← 只写变化的部分
    │
    ├── createSearchFTSIndexes()
    │
    ├── batchInsertEmbeddings(cached)  ← 恢复旧嵌入
    │
    ├── runEmbeddingPipeline(newOnly)  ← 只生成新节点的嵌入
    │
    ├── saveMeta()  ← 更新 meta.json + 清除 dirty flag
    └── registerRepo()

  关键点：管线总是全量跑（因为跨文件解析需要完整上下文），但 DB
  只写变化的部分。这就是为什么增量模式下解析阶段耗时不减（所有文件都要读），但 DB 写入快很多（只替换有变化的行）。

  ---
  十、整体时序图

                                      ┌──────────┐
    CLI analyze                       │  用户敲命令 │
        │                              └─────┬────┘
        ▼                                     │
    ensureHeap(16GB)                          │
        │                                     │
        ▼                                     │
    runFullAnalysis() ◄───────────────────────┘
        │
        ├─ [0%]  早期返回检查（commit 没变 → 直接返回）
        ├─ [0%]  加载嵌入缓存（上次的嵌入别弄丢）
        ├─ [0%]  加载解析缓存（内容没变的文件别重解析）
        │
        ├─ [0-60%]  runPipelineFromRepo()
        │    │
        │    ├── scan         遍历文件系统
        │    ├── structure    建文件/夹节点
        │    ├── markdown     提取 .md 章节
        │    ├── cobol        COBOL 提取
        │    ├── parse        ★ tree-sitter 解析 (Worker池)
        │    ├── routes       识别路由
        │    ├── tools        识别工具
        │    ├── orm          识别ORM查询
        │    ├── crossFile    跨文件类型传播
        │    ├── scopeRes     RFC #909 作用域解析
        │    ├── mro          方法重写/实现
        │    ├── communities  Leiden 社区检测
        │    └── processes    执行流追踪
        │
        ├─ [60-85%]  LadybugDB 入库
        │    ├── streamAllCSVsToDisk()    图 → CSV
        │    ├── splitRelCsvByLabelPair()  按 (from,to) 类型拆分
        │    └── COPY FROM                批量导入
        │
        ├─ [85-90%]  创建 FTS 索引
        │
        ├─ [88%]     恢复缓存嵌入
        ├─ [90-98%]  生成新嵌入（可选）
        │
        └─ [98-100%] 保存 meta.json → registerRepo → AI 上下文文件


