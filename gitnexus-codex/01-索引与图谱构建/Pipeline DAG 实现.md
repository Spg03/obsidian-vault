---
type: implementation
status: codex-expanded
tags:
  - gitnexus
  - pipeline
---

# Pipeline DAG 实现

Pipeline 是 GitNexus 把源码变成 KnowledgeGraph 的生产线。源码入口：

```text
gitnexus/src/core/ingestion/pipeline.ts
gitnexus/src/core/ingestion/pipeline-phases/runner.ts
gitnexus/src/core/ingestion/pipeline-phases/types.ts
```

## 核心思想

Pipeline 不是一个巨大的顺序函数，而是一组带依赖声明的 Phase。每个 Phase 只声明自己依赖哪些上游阶段，Runner 用拓扑排序决定执行顺序。

这样做的好处是：

- 阶段边界清楚。
- 新增阶段更容易，例如 `scopeResolution`。
- 可以禁止隐式依赖，减少“偷偷读别的阶段输出”的耦合。
- 错误信息能定位到具体 phase。
- 测试可以围绕 phase 和 runner 写。

## 当前 13 个阶段

`buildPhaseList` 当前包含：

| 顺序 | Phase | 作用 |
|---:|---|---|
| 1 | scan | 扫描文件系统，得到文件清单 |
| 2 | structure | 建 File、Folder、CONTAINS |
| 3 | markdown | 提取 Markdown section 和文档链接 |
| 4 | cobol | 处理 COBOL 特殊结构 |
| 5 | parse | Tree-sitter 解析代码，抽符号、导入、调用候选 |
| 6 | routes | 识别 Next.js、Express、PHP 等路由 |
| 7 | tools | 识别 MCP/RPC 工具定义 |
| 8 | orm | 识别 Prisma、Supabase 等 ORM 访问 |
| 9 | crossFile | 跨文件类型传播 |
| 10 | scopeResolution | 新一代作用域解析 |
| 11 | mro | 继承链、重写、实现关系 |
| 12 | communities | Leiden 社区发现，得到功能模块 |
| 13 | processes | 从入口追踪执行流 |

注意：旧材料里常说 12 阶段，现在应讲 13 阶段，因为 `scopeResolution` 已进入主 Pipeline。

## Phase 类型

每个阶段都遵循 `PipelinePhase` 合约，概念上长这样：

```text
{
  name: string
  deps: string[]
  execute(ctx, deps) -> output
}
```

`ctx` 里最关键的是共享的 `graph`：

```text
PipelineContext
  -> repoPath
  -> graph
  -> onProgress
  -> options
  -> pipelineStart
```

所以 Pipeline 的设计是“单图累加器”：每个阶段都往同一张 KnowledgeGraph 上补节点或边。

## Runner 如何保证依赖顺序

Runner 的核心在 `topologicalSort`：

1. 建 phaseMap，检查重复 phase name。
2. 校验每个 dep 是否存在。
3. 计算 inDegree 和 reverseDeps。
4. 用 Kahn 算法从入度 0 的阶段开始出队。
5. 如果排序结果数量不等于 phase 数量，说明有环。
6. 用 DFS 找出一个具体 cycle path，生成可读错误。

## 隐式依赖如何被限制

Runner 执行某个 phase 时，不把所有历史结果都传进去，而是只传它声明过的 dependencies：

```text
declaredDeps = results filtered by phase.deps
phase.execute(ctx, declaredDeps)
```

这点非常重要。它强迫阶段作者在 deps 里显式说明依赖，避免后续维护时出现“这个 phase 实际上依赖另一个 phase，但配置里看不出来”的问题。

## 为什么不是并行执行

Runner 注释里说明当前是顺序执行。理论上 DAG 可以并行跑互不依赖的阶段，但 GitNexus 的多数阶段存在明显线性关系，而且共享 KnowledgeGraph 写入会引入并发复杂度。当前设计更偏向可预测、可调试。

## 和传统流水线的区别

传统脚本式流水线：

```text
scan()
structure()
parse()
routes()
...
```

问题是阶段耦合隐式存在，添加新阶段要人工找位置。

GitNexus DAG：

```text
phase declares deps
runner validates graph
runner sorts and executes
```

这更接近构建系统的思想。

## 分享时可以画的图

```text
scan -> structure -> markdown/cobol -> parse -> routes/tools/orm
  -> crossFile -> scopeResolution -> mro -> communities -> processes
```

讲解时不要把所有细节塞进一张图。第一张讲阶段顺序，第二张讲 Runner 内部拓扑排序。

## 相关笔记

- [[Tree-sitter 解析层]]
- [[runFullAnalysis 编排流程]]
- [[图谱 Schema 速览]]
