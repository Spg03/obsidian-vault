---
type: index
status: codex-generated
tags:
  - gitnexus
  - 技术分享
  - 源码级知识库
---

# GitNexus Codex 总入口

这是重新整理后的 GitNexus 源码级知识库。原始 `gitnexus` 文件夹保持备份不修改；这里按照“五层架构”组织：核心引擎层、图谱语义增强层、Agent 接入层、产品与扩展层、工程保障与边缘能力。

## 推荐阅读路线

1. [[GitNexus 五层源码架构地图]]
2. [[GitNexus 知识库总览]]
3. [[源码入口地图]]
4. [[GitNexus 技术分享路线]]
5. [[图谱 Schema 速览]]
6. [[工具层如何设计 Prompt]]
7. [[边界与取舍]]

## 00 总览

- [[GitNexus 五层源码架构地图]]
- [[源码入口地图]]
- [[GitNexus 知识库总览]]
- [[GitNexus 技术分享路线]]
- [[术语表]]
- [[边界与取舍]]
- [[使用演示]]

## 01 核心引擎层：索引与图谱构建

- [[GitNexus 索引构建实现原理]]
- [[runFullAnalysis 编排流程]]
- [[Pipeline DAG 实现]]
- [[Tree-sitter 解析层]]
- [[Semantic Model 与 Registry 解析模型]]
- [[KnowledgeGraph 内存图模型实现]]
- [[Scope Resolution 作用域解析机制]]
- [[Process 执行流生成机制]]
- [[Community Detection 与 Skill 生成机制]]
- [[MRO CrossFile 与 Wildcard Synthesis 实现]]
- [[API Route Tool ORM 提取实现]]

## 02 存储与检索

- [[LadybugDB 图存储]]
- [[LadybugDB 写入连接池与恢复机制]]
- [[图谱 Schema 速览]]
- [[Embedding Pipeline 与 Hybrid Search 实现]]
- [[FTS 与 BM25]]
- [[Embedding 语义搜索]]

## 03 MCP 与 Agent

- [[MCP Server 实现]]
- [[MCP Resources 与 Transport 协议实现]]
- [[LocalBackend 工具执行层实现]]
- [[工具层如何设计 Prompt]]
- [[Prompt Skill AGENTS 注入机制]]
- [[Context Augmentation 快速上下文增强机制]]
- [[Query 与 Context 如何实现]]
- [[Impact 影响分析实现]]
- [[Detect Changes 提交前影响验证实现]]
- [[Agent 工作流]]
- [[MCP stdout 哨兵机制]]
- [[MCP 工具清单]]

## 04 产品与扩展层：Web UI 与 HTTP 服务

- [[HTTP API 与 Serve 后端实现]]
- [[Web UI 前端组件与图可视化实现]]
- [[Web UI 实现原理]]
- [[浏览器端 Graph RAG Agent]]
- [[SSE 心跳与自动重连]]

## 05 跨仓库与 Group

- [[Group 实现原理]]
- [[Group Contract Pipeline 实现]]
- [[Bridge DB]]
- [[Contract Extraction]]
- [[Cross Repo Impact]]

## 06 Wiki 生成

- [[Wiki 生成 Pipeline 实现]]
- [[Wiki 生成实现原理]]

## 07 生成文件示例

- [[AGENTS.md 示例]]
- [[CLAUDE.md 示例]]
- [[Skills 示例]]

## 08 重点源码导读

- [[analyze.ts 源码导读]]
- [[runFullAnalysis 源码导读]]
- [[CLI 命令全集源码导读]]

## 09 性能与增量

- [[增量索引与 Parse Cache 实现]]
- [[Worker Pool 并行解析与 Quarantine 机制]]
- [[Storage Repo Manager 与 Git Staleness]]

## 10 语言与解析模型

- [[LanguageProvider 与多语言 Extractor 体系]]
- [[Import 解析系统实现]]
- [[Legacy Call Resolution DAG 实现]]

## 11 边缘与集成

- [[Eval 框架与质量评估]]
- [[Plugin 与 IDE 集成系统]]
- [[COBOL 子系统实现]]
- [[Vendored Leiden 算法实现]]

## 技术分享主线

```text
普通 Agent 为什么会盲改
  -> GitNexus 为什么不是普通 RAG
  -> analyze 如何把代码变成 KnowledgeGraph
  -> Semantic Model / Import / Call / Scope 如何提升静态解析准确性
  -> MRO / crossFile / wildcard / routes / tools / orm 如何加入工程语义
  -> LadybugDB + FTS + Embedding 如何支撑查询
  -> MCP + LocalBackend 如何把图谱能力交给 Agent
  -> AGENTS.md / Skill / Prompt 如何约束 Agent 行为
  -> Web / Wiki / Group 如何消费和扩展图谱
  -> Worker / Cache / Staleness / Eval 如何保证工程落地
```

## 四个必须讲清楚的问题

1. GitNexus 如何把源码变成图谱？看 [[GitNexus 索引构建实现原理]]、[[Pipeline DAG 实现]]、[[Semantic Model 与 Registry 解析模型]]、[[KnowledgeGraph 内存图模型实现]]。
2. 图谱里到底有哪些节点和边？看 [[图谱 Schema 速览]]、[[API Route Tool ORM 提取实现]]。
3. Agent 如何使用图谱？看 [[MCP Server 实现]]、[[MCP Resources 与 Transport 协议实现]]、[[LocalBackend 工具执行层实现]]、[[Query 与 Context 如何实现]]、[[Impact 影响分析实现]]。
4. Prompt/Skill 如何约束 Agent？看 [[工具层如何设计 Prompt]]、[[Prompt Skill AGENTS 注入机制]]、[[Community Detection 与 Skill 生成机制]]、[[Plugin 与 IDE 集成系统]]。

## 分享时的取舍

这套知识库故意做成“完整源码级”，不需要现场逐页讲完。现场可以从五层地图中挑选：30 分钟讲五层地图 + Pipeline + Semantic Model + MCP/Prompt + Demo workflow；60 分钟加 LadybugDB、Scope/Call、Search、Web/Group；内部源码分享则按目录逐篇展开。
