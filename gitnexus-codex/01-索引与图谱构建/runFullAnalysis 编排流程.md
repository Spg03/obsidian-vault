---
type: implementation
status: codex-expanded
tags:
  - gitnexus
  - analyze
---

# runFullAnalysis 编排流程

`runFullAnalysis` 是 analyze 的总控层，源码在：

```text
gitnexus/src/core/run-analyze.ts
```

它的职责不是“解析代码”这么简单，而是把一次索引任务完整工程化：什么时候跳过、什么时候增量、什么时候全量、如何保护旧 embedding、如何写 LadybugDB、如何生成 Agent 上下文。

## 它在架构里的位置

```text
cli/analyze.ts
  -> runFullAnalysis(repoPath, options, callbacks)
      -> runPipelineFromRepo()
      -> loadGraphToLbug()
      -> createSearchFTSIndexes()
      -> generate embeddings
      -> saveMeta()
      -> registerRepo()
      -> generateAIContextFiles()
```

可以把它理解成 analyze 的“事务编排器”。Pipeline 只负责产出图，`runFullAnalysis` 负责让这张图成为一个可复用的本地索引。

## 输入和输出

### 输入

`AnalyzeOptions` 控制索引行为：

| 选项 | 作用 |
|---|---|
| `force` | 强制全量重建 |
| `repairFts` | 只修复 FTS，不重跑 Pipeline |
| `embeddings` | 是否生成或补齐 embedding |
| `dropEmbeddings` | 是否显式丢弃旧 embedding |
| `skipAgentsMd` | 是否跳过 AGENTS.md / CLAUDE.md |
| `skipSkills` | 是否跳过标准 Skill 安装 |
| `registryName` | 注册到全局 registry 的名字 |
| `workerPoolSize` | parse worker pool 大小 |

### 输出

`AnalyzeResult` 返回：

- repoName / repoPath。
- files、nodes、edges、communities、processes、embeddings 统计。
- alreadyUpToDate。
- ftsRepairedOnly。
- pipelineResult。

## 分阶段流程

### 1. 准备阶段

做三件事：

1. 计算 `.gitnexus` 存储路径。
2. 清理旧 KuzuDB 文件，处理 LadybugDB 迁移。
3. 读取 git commit 和已有 meta。

这里已经体现一个关键设计：索引不是纯内存任务，它必须和磁盘状态、git 状态、历史 meta 协同。

### 2. FTS repair 快速路径

如果用户只想修复 FTS，`repairFts` 会绕过 Pipeline：

```text
load meta
  -> initLbug
  -> createSearchFTSIndexes
  -> verifySearchFTSIndexes
  -> return ftsRepairedOnly
```

这个路径说明 FTS 是索引后处理的一层，不需要每次都重新解析源码。

### 3. early return

当已有 meta、没有 force、commit 没变，并且工作区干净时，GitNexus 可以直接返回 already up to date。

注意它会排除 GitNexus 自己写的路径：

- `.gitnexus/`
- `.claude/`
- `.cursor/`
- `AGENTS.md`
- `CLAUDE.md`

否则 analyze 刚生成的上下文文件会让下一次 analyze 永远认为工作区 dirty。

### 4. 缓存保护

`runFullAnalysis` 在重建前会处理两类缓存：

- embedding cache：避免 routine analyze 把旧向量静默清空。
- parse cache：内容寻址，文件内容没变时复用 worker 输出。

这个设计的意义是：全量图谱重建很贵，Embedding 更贵；缓存让 GitNexus 能成为日常工具，而不是偶尔跑一次的离线脚本。

### 5. Pipeline

核心调用是：

```text
runPipelineFromRepo(repoPath, progress, { parseCache, workerPoolSize })
```

Pipeline 负责产出 `KnowledgeGraph`。进度会映射到总进度的 0-60%。

### 6. LadybugDB 写入

Pipeline 完成后，`runFullAnalysis` 会计算所有 File 节点的 hash，并决定走增量写入还是全量重建。

全量路径：

```text
closeLbug
  -> remove lbug / wal / lock
  -> initLbug
  -> loadGraphToLbug
```

增量路径：

```text
diffFileHashes
  -> compute changed / added / deleted
  -> expand writable set with importers
  -> delete affected rows
  -> write changed subgraph
```

这里有一个很工程化的细节：变更文件的 importer 也可能受影响。例如 barrel export 变化后，两个未改文件之间的 CALLS 边可能被重新解析，所以增量写入需要扩展写集合。

### 7. FTS、Embedding、meta、registry、AI context

最后做后处理：

1. 建 FTS 索引。
2. 恢复或生成 embedding。
3. 保存 meta.json。
4. 注册 repo 到全局 registry。
5. 确保 `.gitnexus` 被 gitignore。
6. 生成 AGENTS.md、CLAUDE.md、Skills。

## 分享时怎么讲

`runFullAnalysis` 的价值在于把“分析代码”变成“可持续维护的本地代码情报系统”。没有它，Pipeline 只是一次性图构建；有了它，GitNexus 才能支持增量、缓存、查询、Agent 注入和多入口复用。

## 相关笔记

- [[Pipeline DAG 实现]]
- [[增量索引机制]]
- [[LadybugDB 图存储]]
- [[Prompt Skill AGENTS 注入机制]]
