---
type: source-map
status: codex-generated
tags:
  - gitnexus
  - leiden
  - community
---

# Vendored Leiden 算法实现

Leiden 算法是 GitNexus 社区发现的数学基础之一。当前仓库中 vendored 版本位于 Web 侧。

| 路径 | 作用 |
|---|---|
| `gitnexus-web/src/vendor/leiden/index.js` | Leiden 实现 |
| `gitnexus-web/src/vendor/leiden/utils.js` | 辅助工具 |
| `gitnexus-web/src/vendor/leiden/index.d.ts` | 类型声明 |

## 它在 GitNexus 中的意义

Community Detection 的目标是把符号聚成“功能模块”，后续用于 Web 图谱上色和聚类展示、`query` 结果的 cluster context、`context` 输出符号所属社区、generated community skills、技术分享中解释“repo-specific skill 不是纯人工文档，而是从图谱聚类来”。

## 为什么要讲但不深讲

Leiden 的数学细节不是 GitNexus 分享主线。更重要的是讲清楚：CALLS / IMPORTS / DEFINES 等关系形成 weighted graph，然后 community detection 生成 Community nodes 和 MEMBER_OF edges，最后被 skill-gen 和 query context 消费。

## 边界

社区发现是启发式聚类，不是架构真理。一个 community 通常代表代码关系紧密的一组符号，但不一定完全对应团队定义的模块边界。

## 讲解抓手

> Leiden 给 GitNexus 提供自动功能聚类能力。它让图谱不只是节点和边，还能形成面向 Agent 和人类的模块视角。
