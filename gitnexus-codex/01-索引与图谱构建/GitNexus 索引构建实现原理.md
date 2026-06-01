---
type: implementation
status: codex-expanded
tags:
  - gitnexus
  - indexing
---

# GitNexus 索引构建实现原理

索引构建是 GitNexus 的核心：它把仓库里分散的源码、文档、路由、工具、ORM 调用、继承关系和执行流统一变成一张 KnowledgeGraph，然后落到 LadybugDB，供 MCP、HTTP、CLI 查询。

## 它解决什么问题

普通 Agent 进入一个陌生仓库时，通常只能靠 grep、文件名、局部代码片段来猜上下文。这个方式的问题是：

- grep 能找到文本，但不知道调用关系。
- RAG 能找到相似片段，但不知道谁依赖谁。
- LLM 临时读文件，会受 token、顺序和注意力限制。
- 修改前最关键的影响面，往往需要跨文件、跨入口、跨流程推断。

GitNexus 的选择是把这些关系提前算好。分析阶段越重，查询阶段越轻；Agent 调工具时拿到的是结构化结果，而不是临时拼凑的文本。

## 总体链路

```text
npx gitnexus analyze
  -> cli/analyze.ts
  -> core/run-analyze.ts::runFullAnalysis
  -> core/ingestion/pipeline.ts::runPipelineFromRepo
  -> KnowledgeGraph
  -> core/lbug/lbug-adapter.ts::loadGraphToLbug
  -> .gitnexus/lbug + meta.json
  -> FTS / Embedding / registry / AGENTS.md / CLAUDE.md / Skills
```

## 三层实现思想

### 1. 代码结构化

入口在 `gitnexus/src/core/ingestion/pipeline-phases/parse.ts`。

这一层把文件文本变成 AST，再从 AST 提取统一语义：

- 文件和文件夹：File、Folder。
- 代码符号：Function、Class、Method、Interface、Enum、Struct 等。
- 依赖关系：IMPORTS、DEFINES。
- 调用候选：CALLS 的基础信息。
- 入口边界：Route、Tool。

Tree-sitter 只提供语法树，GitNexus 的语言 provider 才负责把不同语言翻译成统一图模型。

### 2. 关系预计算

入口在 `gitnexus/src/core/ingestion/pipeline.ts` 和各 phase 文件。

这一层不满足于“抽符号”，还继续推导工程关系：

- crossFile：跨文件类型传播。
- scopeResolution：基于全局作用域索引解析引用。
- mro：继承链、方法重写、接口实现。
- communities：用图结构做功能模块聚类。
- processes：从路由/工具等入口追踪执行流。

这就是 GitNexus 和普通代码搜索最大的区别：它关心的是可遍历关系。

### 3. Agent 编排

入口在 `gitnexus/src/mcp/server.ts`、`gitnexus/src/mcp/tools.ts`、`gitnexus/src/cli/ai-context.ts`。

索引完成后，GitNexus 不只是把数据库放在那里，还生成和暴露 Agent 能用的约束：

- MCP tools：query、context、impact、detect_changes 等。
- MCP resources：repo context、schema、processes。
- tool description：WHEN TO USE / AFTER THIS。
- AGENTS.md / CLAUDE.md：项目级 MUST / NEVER 规则。
- Skills：任务级工作流。

## 索引结果里有什么

| 类别 | 例子 | 用途 |
|---|---|---|
| 结构节点 | File、Folder | 回答代码在哪里 |
| 符号节点 | Function、Class、Method | 回答有哪些代码实体 |
| 关系边 | CALLS、IMPORTS、DEFINES | 回答谁依赖谁 |
| 入口节点 | Route、Tool | 回答外部请求从哪里进入 |
| 流程节点 | Process | 回答完整执行路径 |
| 社区节点 | Community | 回答功能模块边界 |
| 搜索索引 | FTS、Embedding | 回答概念或关键词检索 |
| Agent 文件 | AGENTS.md、CLAUDE.md、Skill | 约束 Agent 行为 |

## 分享时的关键句

GitNexus 不是把代码拆 chunk 后塞进向量库，而是把代码库编译成一张可查询的工程知识图谱。Embedding 是召回能力，图谱关系才是核心能力。

## 容易讲错的点

- 不要说“完全理解业务逻辑”。它主要是静态分析和预计算。
- 不要说“100% 精准”。动态调用、反射、字符串拼路由都可能漏。
- 不要把 Pipeline 阶段说成固定 12 个。当前代码已包含 `scopeResolution`，完整口径是 13 个阶段。
- 不要把 Agent 文件当成索引结果本身。它们是把图谱能力注入 Agent 行为的提示层。

## 相关笔记

- [[runFullAnalysis 编排流程]]
- [[Pipeline DAG 实现]]
- [[LadybugDB 图存储]]
- [[Prompt Skill AGENTS 注入机制]]
