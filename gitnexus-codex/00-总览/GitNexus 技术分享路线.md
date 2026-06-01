---
type: outline
status: codex-expanded
tags:
  - gitnexus
  - 技术分享
---

# GitNexus 技术分享路线

这场分享建议按“为什么普通 Agent 会盲改”开场，再逐层回答“GitNexus 如何把代码上下文工程化”。不要从文件夹结构开始讲，那样容易变成源码导览；应该从问题开始讲，再落到源码入口。

## 45 分钟版本

| 时间 | 主题 | 核心问题 | 对应笔记 |
|---:|---|---|---|
| 5 分钟 | Agent 为什么会盲改 | Agent 缺少调用链、影响面、流程、项目规则 | [[Agent 工作流]] |
| 5 分钟 | GitNexus 不是 RAG | 它不是向量库，而是工程知识图谱 | [[GitNexus 知识库总览]] |
| 10 分钟 | analyze 如何工作 | runFullAnalysis 如何串起 Pipeline、LadybugDB、FTS、AI context | [[runFullAnalysis 编排流程]] |
| 10 分钟 | Pipeline 如何构图 | 13 个 phase 如何共享 KnowledgeGraph 并用依赖 DAG 解耦 | [[Pipeline DAG 实现]] |
| 6 分钟 | 存储与查询 | LadybugDB 为什么是查询后端，FTS/Embedding 如何辅助召回 | [[LadybugDB 图存储]] |
| 6 分钟 | MCP 与 Agent | tools/resources/prompts、LocalBackend、next-step hint 如何协同 | [[MCP Server 实现]] |
| 3 分钟 | 边界 | 静态分析、动态调用、索引新鲜度、Group 匹配风险 | [[边界与取舍]] |

## 30 分钟压缩版本

1. 一页讲问题：普通 Agent 缺的是工程关系，不是 token。
2. 一页讲架构：Repository -> Pipeline -> Graph -> LadybugDB -> MCP -> Agent。
3. 一页讲 analyze：runFullAnalysis 是总编排，Pipeline 是图谱生产线。
4. 一页讲 MCP：工具描述和 next-step hint 让 Agent 形成工作流。
5. 一页讲 Demo：query -> context -> impact -> edit -> detect_changes。
6. 一页讲边界：静态分析可覆盖大量确定性关系，但不能替代测试。

## Demo 脚本

场景：我要修改 `validateUser`。

传统方式：

```text
grep validateUser
  -> 打开几个文件
  -> 猜调用方
  -> 直接修改
  -> 靠测试兜底
```

GitNexus 方式：

```text
query("auth validation flow")
  -> context("validateUser")
  -> impact({ target: "validateUser", direction: "upstream" })
  -> 修改
  -> detect_changes()
```

这里要强调：GitNexus 并不是不读代码，而是先用图谱告诉 Agent 应该读哪些代码。

## 讲解节奏建议

每个模块都按五句话讲：

1. 它解决什么问题。
2. 源码入口在哪里。
3. 关键数据结构是什么。
4. 执行流程是什么。
5. 这个设计的边界是什么。

## 相关笔记

- [[源码入口地图]]
- [[使用演示]]
- [[边界与取舍]]
